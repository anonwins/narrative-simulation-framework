# Sim Economy Service — Implementation Architecture

- Spec: [systems/sim-economy.md](../systems/sim-economy.md)
- Roadmap: [Phase 10](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why economy is pressure, not power progression

NSF financial loops create **survival tension** (rent, debt, food) rather than gear upgrades. `IEconomyService` tracks liquid currency and purchase attempts; narrative consequences of unpaid obligations flow through story state and dialogue, not instant game-over.

### Why currency is multi-ID capable

MVP implements single `currency_cash` integer; interface uses `currencyId` string so content can add scrip, favors, or faction credits without API churn. All amounts are integers — no floating money.

### Why TryPurchase is vendor-scoped

Prices and stock live on vendor definitions in content store. Service validates funds, deducts currency, and delegates item grant to `IInventoryService` integration point — economy does not duplicate item logic.

### Why economy stays in Simulation

Financial state participates in save/load and gate rules ("need 50 cash"). No UI pricing display in core — Presentation reads balances through view models in Phase 13+.

---

## Implementation summary

| MVP (Phase 10) | Full |
|---|---|
| `GetCurrency` / `AddCurrency` / `TrySpend` | Debt ledger + interest `[FULL]` |
| `TryPurchase(vendorId, itemId)` | Rent obligations on day boundary `[FULL]` |
| Vendor catalog from content | Pawn shop sell pipeline `[FULL]` |
| `CurrencyChanged` event | Financial history audit `[FULL]` |
| Persistence via `EconomyServiceState` | Dynamic/priced-by-relationship vendors `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Economy`
- **References:** Runtime, Content, Simulation.Time (scheduled rent)
- **Tick phase:** **Events** (day-end rent tick) · **Content** (script grants/deducts)

Registered as `IEconomyService`.

---

## Public API

See [contracts.md](contracts.md) — `IEconomyService`.

| Method | Semantics |
|---|---|
| `int GetCurrency(string currencyId)` | Current balance; missing ID → 0 |
| `void AddCurrency(string currencyId, int amount)` | Credit; negative amount rejected |
| `bool TrySpend(string currencyId, int amount)` | Debit if sufficient; no throw |
| `bool TryPurchase(string vendorId, string itemId)` | Price check + spend + inventory grant |

Internal (not contracted): `RegisterVendor`, `GetVendorPrice` for tests.

---

## Internal implementation

### EconomyService

```csharp
sealed class EconomyService : IEconomyService, IStatefulService<EconomyServiceState>
{
    readonly Dictionary<string, int> _balances = new();
    readonly VendorCatalog _vendors = new();
    readonly IContentStore _content;
    readonly IEventBus _events;
    readonly IInventoryService _inventory;

    public int GetCurrency(string currencyId)
        => _balances.TryGetValue(currencyId, out var v) ? v : 0;

    public void AddCurrency(string currencyId, int amount)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount));
        _balances[currencyId] = GetCurrency(currencyId) + amount;
        PublishCurrencyChanged(currencyId, _balances[currencyId]);
    }

    public bool TrySpend(string currencyId, int amount)
    {
        if (amount < 0 || GetCurrency(currencyId) < amount) return false;
        _balances[currencyId] -= amount;
        PublishCurrencyChanged(currencyId, _balances[currencyId]);
        return true;
    }

    public bool TryPurchase(string vendorId, string itemId)
    {
        var vendor = _vendors.Get(vendorId);
        if (!vendor.TryGetListing(itemId, out var listing)) return false;
        if (!TrySpend(listing.CurrencyId, listing.BuyPrice)) return false;
        _inventory.AddItem(itemId, 1);
        _events.Publish(new SimEvent {
            Type = EventTypes.ItemPurchased,
            Payload = { ["vendorId"] = vendorId, ["itemId"] = itemId }
        });
        return true;
    }
}
```

### VendorCatalog (internal)

```csharp
sealed class VendorCatalog
{
    readonly Dictionary<string, VendorInstance> _vendors = new();

    public void Load(VendorDefinition def)
        => _vendors[def.Id] = VendorInstance.FromDefinition(def);

    public VendorInstance Get(string vendorId) => _vendors[vendorId];
}

class VendorInstance
{
    public bool TryGetListing(string itemId, out VendorListing listing);
}
```

### RentProcessor (internal) `[FULL]`

Subscribes to `NewDayStarted`. Reads pending `RentObligation` entries; attempts auto-spend; on failure publishes `RentUnpaid` for story fail-forward hooks.

---

## Definition assets

```csharp
class VendorDefinition : ContentDefinition
{
    public string DisplayNameKey;
    public string ScheduleId;           // optional ITimeService schedule
    public List<VendorListing> Listings;
}

class VendorListing
{
    public string ItemId;
    public string CurrencyId;
    public int BuyPrice;
    public int SellPrice;               // [FULL] pawn
    public string StockGateId;          // conditional availability
}

class CurrencyDefinition : ContentDefinition
{
    public string DisplayNameKey;
    public int StartingBalance;         // new game default
}
```

Content pipeline validates item IDs exist in inventory definitions and prices are non-negative.

---

## Runtime state

```csharp
class EconomyServiceState
{
    public Dictionary<string, int> Balances;
    public List<DebtEntry> Debts;              // [FULL]
    public List<RentObligation> RentLedger;    // [FULL]
}

class DebtEntry   // [FULL]
{
    public string CreditorActorId;
    public int Amount;
    public GameTime DueAt;
}
```

MVP state: `Balances` only. Envelope key stable for migration v1 → v2 when debt added.

---

## Core algorithms

### TrySpend idempotency

Failed spend leaves balance unchanged — no partial transactions. Tests assert no event on failure.

### TryPurchase flow

```text
resolve vendor + listing
  → TrySpend(currency, buyPrice)?
      no → return false
      yes → inventory.AddItem
          → publish ItemPurchased
          → return true
```

Inventory failure after spend rolls back in debug builds; MVP assumes AddItem always succeeds for catalog items.

### Starting balance

On new game bootstrap, content `CurrencyDefinition.StartingBalance` seeds `_balances`. Restore from save overrides.

### Day-end rent `[FULL]`

1. `NewDayStarted` handler iterates due rent
2. Auto `TrySpend` for configured amount
3. Success → `RentPaid` event; failure → story flag + `RentUnpaid` event
4. Never hard-stops kernel — fail-forward to alternate lodging narrative

### Gate integration

Rules reference `economy_currency_*` thresholds via custom rule operands calling `GetCurrency`. Evaluated in **Gating** phase only.

---

## Event contracts + tick phase

| Event | When | Subscribers |
|---|---|---|
| `CurrencyChanged` | Add/Spend success | UI bridge, chronicle |
| `ItemPurchased` | TryPurchase success | Conduct, relationship (generosity) |
| `RentDue` / `RentPaid` / `RentUnpaid` | `[FULL]` day boundary | Story state, dialogue |
| `DebtReminder` | `[FULL]` scheduled | Voice, pacing |

Economy mutations allowed Content/Facts phases; financial events flushed Events phase.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IContentStore | Vendor/currency definitions |
| IInventoryService | Item grant on purchase |
| IEventBus | Notifications |
| ITimeService | Rent schedules `[FULL]` |

| Used by | Reason |
|---|---|
| IDialogueService | Bribe/payment branches |
| IGateService | Affordability checks |
| IPersistenceService | Balance snapshot |
| IStoryStateService | Rent failure flags |
| ScriptCompiler actions | Grant/deduct currency |

**Forbidden:** EconomyService → Presentation.

---

## MVP scope (Phase 10)

- [ ] `IEconomyService` + `EconomyService`
- [ ] Single currency `currency_cash` + extensible dictionary
- [ ] `GetCurrency` / `AddCurrency` / `TrySpend`
- [ ] `TryPurchase` with vendor definition stub
- [ ] `CurrencyChanged`, `ItemPurchased` events
- [ ] `EconomyServiceState` capture/restore
- [ ] Tests: spend fail/success, purchase, persistence

---

## Full scope

- `[FULL]` Debt ledger + creditor dialogue hooks
- `[FULL]` Rent obligation loop with fail-forward
- `[FULL]` Pawn sell at `SellPrice` with inventory remove
- `[FULL]` Time-conditional vendor stock
- `[FULL]` Economic history for chronicle projection

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Economy/IEconomyService.cs
Packages/NarrativeFramework/Simulation/Economy/EconomyService.cs
Packages/NarrativeFramework/Simulation/Economy/VendorCatalog.cs
Packages/NarrativeFramework/Simulation/Economy/VendorDefinition.cs
Packages/NarrativeFramework/Simulation/Economy/CurrencyDefinition.cs
Packages/NarrativeFramework/Simulation/Economy/EconomyServiceState.cs
Packages/NarrativeFramework/Simulation/Economy/RentProcessor.cs
Packages/NarrativeFramework/Tests/EditMode/Economy/EconomyBalanceTests.cs
Packages/NarrativeFramework/Tests/EditMode/Economy/EconomyPurchaseTests.cs
Packages/NarrativeFramework/Tests/EditMode/Economy/EconomyPersistenceTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `AddCurrency_IncreasesBalance` | GetCurrency |
| `TrySpend_Insufficient_ReturnsFalse` | Balance unchanged |
| `TrySpend_Sufficient_Deducts` | Balance + event |
| `TryPurchase_ValidListing_GrantsItem` | Inventory mock |
| `TryPurchase_UnknownItem_ReturnsFalse` | No spend |
| `CaptureRestore_PreservesBalances` | Hash-stable JSON |
| `NegativeAdd_Throws` | Validation |

Contributes to Phase 10 ≥15 tests with info flow and persistence.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) ECO-01 (full `systems/sim-economy.md` scope).

---

## Bootstrap and inventory boundary

`EconomyService` depends on `IInventoryService` (Phase 12) for `TryPurchase`. Phase 10 MVP registers a **null inventory stub** in test fixtures that records `AddItem` calls without equipment modifiers. Production composition replaces stub once inventory lands — economy interface unchanged.

Currency seeding runs once at new game:

```csharp
void SeedEconomy(IEconomyService economy, IContentStore content)
{
    foreach (var id in content.GetAllIds<CurrencyDefinition>())
    {
        var def = content.GetDefinition<CurrencyDefinition>(id);
        economy.AddCurrency(id, def.StartingBalance);
    }
}
```

Restore from save **skips** seeding — balances come solely from `EconomyServiceState`. Script actions `GrantCurrency` / `SpendCurrency` compile to `AddCurrency` / `TrySpend` in Phase 12 script subset.

Pawn sell flow `[FULL]` lives in economy but calls `IInventoryService.RemoveItem` + `AddCurrency` at `SellPrice` — never bypasses inventory ownership checks.

---

## Related documents

- [sim-persistence.md](sim-persistence.md), [sim-time.md](sim-time.md)
- [content-inventory.md](../systems/content-inventory.md)
- [story-fail-forward.md](story-fail-forward.md) (rent failure branches)
