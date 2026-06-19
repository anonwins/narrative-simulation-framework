# Sim Event Bus — Implementation Architecture

- Spec: [systems/sim-event.md](../systems/sim-event.md)
- Roadmap: [Phase 1](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Events: [data-model.md](data-model.md)

---

## Design rationale

### Why an event bus instead of direct calls

NSF has strict asmdef boundaries. Story cannot reference Ledger directly, yet chronicle must react to dialogue outcomes. **Events** carry facts-about-changes; subscribers stay decoupled. This also powers debug causal tracing (Phase 14) without instrumenting every service.

### Why typed events from Phase 1

Stringly `Type` + dictionary payloads would require a second migration pass — violates the no-temporary-builds rule ([decisions-log.md](../decisions-log.md) EVT-01). Typed `SimEventBase` subclasses give compile-time safety, deterministic replay, and causal tracing. Content script `TriggerEvent` resolves through a **registry of known event types** (attribute or factory map), not ad-hoc strings.

### Why sync + deferred queue from Phase 1

`systems/sim-event.md` requires both immediate and deferred dispatch. Re-entrant chains (inspect → fact → chronicle → UI) need a queue from day one — not a Phase 14 afterthought ([decisions-log.md](../decisions-log.md) EVT-03).

### Why handlers receive SimulationSnapshot

Handlers must not re-query mutable services mid-dispatch — race conditions produce flaky tests. Snapshot is frozen for the dispatch loop.

---

## Implementation summary

| Phase 1 (full spec alignment) | Phase 17+ polish |
|---|---|
| Typed `SimEventBase` + sync publish | Roslyn analyzers for event contracts |
| Deferred queue + priority ordering | Weak refs / MonoBehaviour adapter helpers |
| Handlers receive `SimulationSnapshot` | Event history ring buffer for debug |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Events`
- **References:** Runtime, Content (for timestamp types only)

Registered as `IEventBus` in registry. Kernel invokes flush during **Events** tick phase.

---

## Public API

[contracts.md](contracts.md) — `IEventBus`, `IEventHandler`.

---

## Internal implementation

### EventBus

```csharp
sealed class EventBus : IEventBus
{
    readonly Dictionary<Type, List<object>> _handlers = new();
    readonly List<SimEventBase> _immediate = new();
    readonly List<SimEventBase> _deferred = new();

    public void Publish(SimEventBase e) => _deferred.Add(e);

    public void PublishImmediate(SimEventBase e, SimulationSnapshot snapshot)
    {
        if (_flushing) throw new InvalidOperationException("Publish during Flush");
        Dispatch(e, snapshot);
    }

    bool _flushing;

    public void Flush(SimulationSnapshot snapshot)
    {
        _flushing = true;
        try
        {
            while (_deferred.Count > 0)
            {
                var batch = _deferred.OrderBy(e => e.Priority).ToList();
                _deferred.Clear();
                foreach (var e in batch)
                    Dispatch(e, snapshot);
            }
        }
        finally { _flushing = false; }
    }

    void Dispatch(SimEventBase e, SimulationSnapshot snapshot) { /* invoke typed handlers */ }

    public void Subscribe<T>(IEventHandler<T> handler) where T : SimEventBase { /* add */ }
    public void Unsubscribe<T>(IEventHandler<T> handler) where T : SimEventBase { /* remove */ }
}
```

**Why immediate + deferred:** Per `systems/sim-event.md` — faculty/dialogue updates immediate; chronicle/UI batch deferred. Kernel **Flush** during Events phase (EVT-03).

### Event type registry

Content script `TriggerEvent("FactRegistered", …)` maps to concrete `SimEventBase` subclasses via `[SimEventType("FactRegistered")]` registry — not ad-hoc string payloads.

---

## Definition assets

None. Optional ScriptableObject event catalogs `[FULL]` for content-driven event metadata.

---

## Runtime state

`EventBus` holds handler registry + queue (transient). No save state — replay from service states if needed.

---

## Core algorithms

### Publish flow

1. Service constructs typed `SimEventBase` subclass with `Timestamp` from `ITimeService.Now`
2. `Publish` (deferred) or `PublishImmediate` (sync) per event kind
3. Kernel Events phase calls `Flush(snapshot)` — priority-ordered batch (EVT-05)
4. Handlers mutate their services or schedule follow-up work (never block on Presentation)

### Subscription lifetime

Edit Mode tests: plain `IEventHandler` instances. `[FULL]` Unity: `MonoBehaviour` adapters unsubscribe OnDestroy.

---

## Event contracts

| Event | Typical publisher | Typical subscribers |
|---|---|---|
| `FactRegistered` | IFactService | IInfoFlowService, debug trace |
| `ActorMoved` | ILocationService | IActorService, exploration |
| `RollResolved` | IRollService | Story fail-forward, chronicle projection |
| `DialogueChoiceSelected` | IDialogueService | IConductService, IRelationshipService |

**Kernel phase:** Events (phase 4).

---

## Integration matrix

| Depends on | Reason |
|---|---|
| Runtime snapshot types | Handler signature |
| GameTime | Event timestamps |

| Consumed by | Reason |
|---|---|
| ISimulationKernel | Flush timing |
| IChronicleService | Projection refresh |
| Debug module | Trace log Phase 14 |

**Forbidden:** EventBus implementation → Presentation.

---

## MVP scope (Phase 1)

- [ ] `EventBus` implements `IEventBus`
- [ ] Queue + flush pattern
- [ ] Subscribe/unsubscribe
- [ ] Tests: delivery, order, unsubscribe, ≥5 event tests as part of Phase 1 total

---

## Full scope

- Event history ring buffer for debug replay (Phase 17)

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Events/IEventBus.cs
Packages/NarrativeFramework/Simulation/Events/IEventHandler.cs
Packages/NarrativeFramework/Simulation/Events/EventBus.cs
Packages/NarrativeFramework/Simulation/Events/SimEventBase.cs
Packages/NarrativeFramework/Simulation/Events/FactRegisteredEvent.cs
Packages/NarrativeFramework/Simulation/Events/EventBus.cs
Packages/NarrativeFramework/Simulation/Events/EventTypes.cs
Packages/NarrativeFramework/Tests/EditMode/Events/EventBusDeliveryTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `PublishFlush_DeliversToSubscriber` | Handler called once |
| `Unsubscribe_StopsDelivery` | Zero calls |
| `MultipleHandlers_SameType_AllCalled` | Count = N |
| `Flush_ClearsQueue` | Second flush empty |

---

## Deferred decisions

**Resolved** — [decisions-log.md](../decisions-log.md) EVT-01–EVT-05.

---

## Related documents

- [runtime-kernel.md](runtime-kernel.md)
- [sim-fact.md](sim-fact.md)
