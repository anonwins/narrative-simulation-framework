# Content Inventory — Implementation Architecture

- Spec: [systems/content-inventory.md](../systems/content-inventory.md)
- Glossary: [Item modifiers](../terminology-glossary.md)
- Roadmap: [Phase 12](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [cognition-faculty.md](cognition-faculty.md) · [cognition-roll.md](cognition-roll.md)

---

## Design rationale

### Why inventory is a narrative state modifier system

NSF items are not primarily stat sticks for combat. They alter **faculty values**, dialogue availability, investigation capability, and identity expression. Equipment modifiers feed `IFacultyService` and `IRollService` through a single `GetEquippedModifiers()` query — keeping modifier stack order deterministic (see roll spec resolution order).

### Why separate from Economy (Phase 10)

Currency and vendor purchase live in `IEconomyService`. Inventory owns **possession, equip slots, and narrative modifiers**. Overlap at `AddItem` from loot/economy is via explicit calls, not merged services.

### Why equip triggers faculty refresh

Modified faculty values must update when loadout changes without requiring save reload. Equip/Unequip applies `ModifierSource.Equipment` deltas through `IFacultyService.ApplyModifier` and notifies roll service on next resolution.

### Why content-driven ItemDefinition

Item behavior is data: which slot, which faculty deltas, optional gate/dialogue hooks. Hardcoded item classes do not scale across NoirSample and SciFiSample content packs.

### Why not duplicate modifier math in RollService

Roll service aggregates modifiers from inventory, emotion, belief, conduct in defined order. Inventory only ** exposes** equipped modifiers; it does not compute final roll totals.

---

## Implementation summary

| MVP (Phase 12) | Full |
|---|---|
| `IInventoryService` CRUD + equip | Consumable use effects |
| `ItemDefinition` with `ItemModifier[]` | Dialogue unlock flags on items |
| Slot-based equip (one item per slot) | Multi-slot items `[FULL]` |
| `GetEquippedModifiers()` for rolls | Item durability / narrative decay `[FULL]` |
| Integration test: equip → roll change | Crafting / combine `[FULL]` |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Content` (definitions) + `NarrativeFramework.Simulation` (service) **or** Content-only if inventory stays narrative-light — **MVP: Simulation** for `IInventoryService` impl alongside economy hooks |
| **Decision MVP** | `NarrativeFramework.Simulation.Inventory` for service; definitions stay in Content |
| **References** | Runtime, Content, Cognition (faculty apply) |
| **Tick phase** | **Content** — equip/loot often from interactions; passive modifiers read anytime |

**Asmdef:** Simulation references Cognition for `IFacultyService` modifier apply on equip. Cognition does not reference Simulation.

---

## Public API

See [contracts.md](contracts.md) — `IInventoryService`.

**Module notes:**

- `HasItem` / `AddItem` / `RemoveItem` — quantity stack by item ID.
- `Equip` / `Unequip` — slot-based; replaces incumbent in slot.
- `GetEquippedModifiers()` — flattened list for roll modifier stack step 3 (equipment per roll spec).

Registered in `INarrativeServiceRegistry`.

---

## Internal implementation

### InventoryService

```csharp
namespace NarrativeFramework.Simulation.Inventory
{
    sealed class InventoryService : IInventoryService, IStatefulService<InventoryServiceState>
    {
        readonly IContentStore _content;
        readonly IFacultyService _faculty;
        readonly Dictionary<string, int> _stacks = new();
        readonly Dictionary<string, string> _slotToItem = new(); // slotId -> itemId

        public bool HasItem(string itemId) =>
            _stacks.TryGetValue(itemId, out var qty) && qty > 0;

        public void AddItem(string itemId, int quantity)
        {
            ValidateItemExists(itemId);
            _stacks[itemId] = _stacks.GetValueOrDefault(itemId) + quantity;
        }

        public void RemoveItem(string itemId, int quantity)
        {
            if (!HasItem(itemId) || _stacks[itemId] < quantity)
                throw new InvalidOperationException($"Insufficient {itemId}");
            _stacks[itemId] -= quantity;
            if (_stacks[itemId] == 0) _stacks.Remove(itemId);
        }

        public void Equip(string itemId, string slotId)
        {
            var def = _content.GetDefinition<ItemDefinition>(itemId);
            if (def.SlotId != slotId)
                throw new InvalidOperationException($"Item {itemId} not valid for slot {slotId}");

            if (_slotToItem.TryGetValue(slotId, out var prev))
                Unequip(slotId);

            _slotToItem[slotId] = itemId;
            ApplyModifiers(def, sign: +1);
        }

        public void Unequip(string slotId)
        {
            if (!_slotToItem.TryGetValue(slotId, out var itemId)) return;
            var def = _content.GetDefinition<ItemDefinition>(itemId);
            ApplyModifiers(def, sign: -1);
            _slotToItem.Remove(slotId);
        }

        public IReadOnlyList<ItemModifier> GetEquippedModifiers()
        {
            var list = new List<ItemModifier>();
            foreach (var itemId in _slotToItem.Values)
            {
                var def = _content.GetDefinition<ItemDefinition>(itemId);
                list.AddRange(def.Modifiers);
            }
            return list;
        }

        void ApplyModifiers(ItemDefinition def, int sign)
        {
            foreach (var m in def.Modifiers)
                _faculty.ApplyModifier(m.FacultyId, sign * m.Delta, ModifierSource.Equipment);
        }
    }
}
```

### ItemDefinition

```csharp
namespace NarrativeFramework.Content.Definitions
{
    class ItemDefinition : ContentDefinition
    {
        [SerializeField] string slotId;
        [SerializeField] ItemModifier[] modifiers;

        public string SlotId => slotId;
        public IReadOnlyList<ItemModifier> Modifiers => modifiers;

        public override void Validate(IValidationContext ctx)
        {
            ContentIdValidator.IsValid(Id, "item_");
            foreach (var m in modifiers)
                if (!ctx.DefinitionExists<FacultyDefinition>(m.FacultyId))
                    ctx.ReportError($"Unknown faculty {m.FacultyId} on item {Id}");
        }
    }
}

[Serializable]
class ItemModifier
{
    public string FacultyId;
    public int Delta;
}
```

Spec interfaces (internal DTOs):

```csharp
interface IItem { string Id; int Quantity; }
interface IEquipable { string SlotId; }
```

MVP uses concrete stacks + slot map; `[FULL]` explicit `IItem` instances per stack entry.

---

## Definition assets

| Asset | ID prefix | Fields |
|---|---|---|
| `ItemDefinition` | `item_` | `SlotId`, `ItemModifier[]` |
| Slot IDs | `slot_*` | Content convention, not separate SO in MVP |

Examples:

- `item_trenchcoat` → `faculty_authority` +1
- `item_magnifying_glass` → `faculty_perception` +2

---

## Runtime state / persistence

```csharp
class InventoryServiceState
{
    public Dictionary<string, int> Stacks;       // itemId -> qty
    public Dictionary<string, string> Equipped;  // slotId -> itemId
}
```

Phase 10: envelope in `SaveSnapshot` via `IStatefulService<InventoryServiceState>`.

On restore: re-apply equip modifiers idempotently (clear modifiers then equip from state).

---

## Core algorithms

### Equip

1. Validate item in inventory (optional MVP — allow equip from inventory only).
2. Validate slot compatibility.
3. Unequip incumbent if any.
4. Apply positive faculty modifiers.
5. `[FULL]` Publish `ItemEquipped` event.

### Roll integration

`RollService` modifier aggregation (simplified):

```csharp
int equipmentBonus = 0;
foreach (var m in _inventory.GetEquippedModifiers())
    if (m.FacultyId == roll.FacultyId)
        equipmentBonus += m.Delta;
```

Order: base faculty → caps → **equipment** → belief → emotion → … (per cognition-roll spec).

### Remove equipped item

If item removed while equipped → force `Unequip` first.

---

## Event contracts + kernel tick phase

| Event | When | Phase |
|---|---|---|
| `ItemAdded` | AddItem | Content |
| `ItemRemoved` | RemoveItem | Content |
| `ItemEquipped` | Equip | Content |
| `ItemUnequipped` | Unequip | Content |

**Kernel:** Mutations typically **Content** phase after interaction/gate approval. Modifier reads occur during **Interpretation** (passive) and roll resolution (any phase invoke — prefer synchronous on attempt).

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | ItemDefinition |
| `IFacultyService` | Apply equipment modifiers |
| `ContentIdValidator` | item_* IDs |

| Used by | Reason |
|---|---|
| `IRollService` | Equipment modifier stack |
| `IGateService` | Requires item conditions |
| `IDialogueService` | Item-gated lines |
| `IEconomyService` | Purchase → AddItem |
| `IInteractionService` | Loot/pickup (Phase 13) |

---

## MVP scope checklist (Phase 12)

- [ ] `ItemDefinition` in content store
- [ ] `IInventoryService` implementation
- [ ] Stack CRUD + slot equip/unequip
- [ ] `GetEquippedModifiers()` feeds roll tests
- [ ] `InventoryServiceState` capture/restore stub
- [ ] Test: equip item changes `GetModifiedValue` / roll outcome
- [ ] FrameworkTestPack sample items

---

## Full scope

- `[FULL]` Consumables with timed modifiers
- `[FULL]` Items unlocking dialogue/belief content
- `[FULL]` Narrative item decay (evidence degrades)
- `[FULL]` Container / stash UI state
- `[FULL]` Item-set bonuses

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Inventory/IInventoryService.cs
Packages/NarrativeFramework/Simulation/Inventory/InventoryService.cs
Packages/NarrativeFramework/Simulation/Inventory/InventoryServiceState.cs
Packages/NarrativeFramework/Simulation/Inventory/ItemModifier.cs
Packages/NarrativeFramework/Content/Definitions/ItemDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Inventory/InventoryCrudTests.cs
Packages/NarrativeFramework/Tests/EditMode/Inventory/EquipModifierRollTests.cs
Samples~/FrameworkTestPack/Definitions/item_sample_coat.asset
```

---

## Test plan

| Test | Asserts |
|---|---|
| `AddItem_HasItemTrue` | Quantity stack |
| `RemoveItem_Insufficient_Throws` | |
| `Equip_AppliesFacultyModifier` | GetModifiedValue delta |
| `Unequip_RevertsModifier` | Back to base |
| `GetEquippedModifiers_AggregatesSlots` | Sum across slots |
| `Roll_WithEquippedItem_ChangesOutcome` | Integration with IRollService |
| `RestoreState_ReappliesEquip` | Persistence stub |

**Exit criteria:** Equip modifies roll — Phase 12 roadmap.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) inventory in Simulation asmdef; equip requires ownership; one item per slot.

---

## Related documents

- [cognition-faculty.md](cognition-faculty.md) — modifier application
- [cognition-roll.md](cognition-roll.md) — resolution order
- [content-store.md](content-store.md) — ItemDefinition registration
- sim-economy (Phase 10) — purchase integration
