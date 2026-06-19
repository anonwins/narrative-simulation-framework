# Sim Fact Service — Implementation Architecture

- Spec: [systems/sim-fact.md](../systems/sim-fact.md)
- Roadmap: [Phase 1](../development-roadmap.md) MVP · Phase 10 persistence
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why facts are first-class

Everything in NSF ultimately grounds in **what is true in the world**. Dialogue interprets facts; gates check facts; threads accumulate evidence linked to facts. Without a central fact store, modules invent parallel “truth” and consistency breaks.

### Why revoke, not delete

Investigation games need history. Revocation marks a fact inactive while preserving audit trail for debug and `[FULL]` replay narratives.

### Why internal FactRegistry

External code uses `IFactService` only. Index structure (by ID, by subject) is an implementation detail that can change without breaking content packs.

---

## Implementation summary

| MVP (Phase 1) | Full |
|---|---|
| Register / TryGet / Revoke / GetAll | Subject/predicate queries `[FULL]` |
| Publish `FactRegistered` events | Fact derivation rules `[FULL]` |
| Snapshot in persistence envelope | Temporal validity windows `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Facts`
- **Tick phase:** **Facts** (first phase — gates depend on this)

---

## Public API

[contracts.md](contracts.md) — `IFactService`.

---

## Internal implementation

### FactService

```csharp
sealed class FactService : IFactService, IStatefulService<FactServiceState>
{
    readonly FactRegistry _registry = new();
    readonly IEventBus _events;
    readonly ITimeService _time;

    public void Register(FactRecord fact)
    {
        ContentIdValidator.IsValid(fact.Id, "fact_");
        fact.RegisteredAt = _time.Now;
        fact.IsRevoked = false;
        _registry.Upsert(fact);
        _events.Publish(new SimEvent {
            Type = EventTypes.FactRegistered,
            Payload = { ["factId"] = fact.Id }
        });
    }

    public void Revoke(string factId)
    {
        if (!_registry.TryGet(factId, out var fact)) return;
        fact.IsRevoked = true;
        _events.Publish(new SimEvent { Type = EventTypes.FactRevoked, /* ... */ });
    }

    public bool IsActive(string factId)
        => _registry.TryGet(factId, out var f) && !f.IsRevoked;

    // CaptureState / RestoreState for persistence Phase 10
}
```

### FactRegistry (internal)

```csharp
sealed class FactRegistry
{
    readonly Dictionary<string, FactRecord> _byId = new();

    public void Upsert(FactRecord fact) => _byId[fact.Id] = fact;
    public bool TryGet(string id, out FactRecord fact) => _byId.TryGetValue(id, out fact);
    public IReadOnlyList<FactRecord> All => _byId.Values.ToList();
}
```

---

## Definition assets

Facts are runtime records, not ScriptableObjects. Content may author **fact templates** in content store `[FULL]` that script expands into `FactRecord` instances.

---

## Runtime state

```csharp
class FactServiceState
{
    public List<FactRecord> Facts;
}
```

Owned by `IFactService`; serialized via [data-model.md](data-model.md) envelope in Phase 10.

---

## Core algorithms

### Register

1. Validate `fact_*` ID
2. Reject duplicate active ID (or upsert policy — MVP: upsert overwrite with log)
3. Store in registry
4. Queue `FactRegistered` event (flushed Events phase)

### Gate integration (Phase 5)

`IGateService` calls `IFactService.IsActive(factId)` during **Gating** phase — never during Facts phase of same tick for facts registered same tick (test documents ordering).

---

## Event contracts

| Event | When |
|---|---|
| `FactRegistered` | After successful register |
| `FactRevoked` | After revoke |

**Kernel phase:** Facts (write), Events (notify).

Subscribers: `IInfoFlowService`, `IChronicleService.RefreshProjection`, debug trace.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IEventBus | Notifications |
| ITimeService | Timestamps |
| ContentIdValidator | ID policy |

| Used by | Reason |
|---|---|
| IRuleEngine / IGateService | Conditions |
| IThreadService | Evidence links |
| IDialogueService | Condition checks |
| IPersistenceService | Save/load |

---

## MVP scope (Phase 1)

- [ ] `FactService` + `FactRegistry`
- [ ] CRUD + `IsActive`
- [ ] Event publish on register/revoke
- [ ] Persistence snapshot capture stub
- [ ] Tests: CRUD, revoke, event fired, gate slice test with kernel

---

## Full scope

- `[FULL]` Query by subject/predicate
- `[FULL]` Fact conflict detection (two exclusive truths)

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Facts/IFactService.cs
Packages/NarrativeFramework/Simulation/Facts/FactService.cs
Packages/NarrativeFramework/Simulation/Facts/FactRegistry.cs
Packages/NarrativeFramework/Simulation/Facts/FactRecord.cs
Packages/NarrativeFramework/Simulation/Facts/FactServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Facts/FactServiceTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Register_TryGet_RoundTrip` | Equality on fields |
| `Revoke_IsActiveFalse` | IsActive false, record exists |
| `Register_PublishesEvent` | Handler sees FactRegistered |
| `KernelFactsBeforeGating` | Gate sees fact same tick (with kernel test) |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-01, SIM-02, SIM-05, SIM-06.

---

## Related documents

- [sim-event.md](sim-event.md), [runtime-kernel.md](runtime-kernel.md)
- [rules-gate.md](rules-gate.md), [ledger-thread.md](ledger-thread.md)
