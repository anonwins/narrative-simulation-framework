# Runtime Kernel — Implementation Architecture

- Spec: [runtime-kernel.md](../runtime-kernel.md)
- Glossary: [SimulationTickPhase](../terminology-glossary.md)
- Roadmap: [Phase 1](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Registry: [foundation.md](foundation.md)

---

## Design rationale

### Why a kernel at all

NSF modules are interdependent. Without orchestration, dialogue could mutate factions before facts exist, or gates could run before social state updates. The kernel encodes **causal order** so the world evolves consistently and tests can reproduce bugs with a single `Tick()` call.

### Why six phases, not nine layers

The spec describes nine conceptual layers for authors; code uses **six tick phases** to avoid micro-ticking overhead while preserving order. Mapping is documented in the spec and glossary — architecture implements the six-phase model only.

### Why single-phase tick for tests

`Tick(SimulationTickPhase? singlePhase)` lets Edit Mode tests assert “Gating never runs before Facts” without simulating an entire frame of narrative content.

---

## Implementation summary

| MVP (Phase 1) | Full (Phase 14+) |
|---|---|
| `SimulationKernel` runs 6 phases in order | Phase handlers delegate to all registered services |
| Phase guards (Facts before Gating) | Performance budgets per phase |
| `SimulationSnapshot` built at tick start | Parallel read-only snapshot `[FULL]` |
| Event publish during Events phase | Cross-phase deferred events queue `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Runtime`
- **Namespace:** `NarrativeFramework.Runtime.Kernel`
- **References:** Simulation (for `IEventBus`, `IFactService` interfaces — use registry at runtime to avoid circular compile deps; Runtime asmdef references Simulation per roadmap)

**Note:** Kernel *implementation* lives in Runtime; event bus *implementation* lives in Simulation but is invoked from kernel during `Events` phase.

---

## Public API

See [contracts.md](contracts.md) — `ISimulationKernel`, `INarrativeServiceRegistry`.

---

## Internal implementation

### SimulationKernel

```csharp
sealed class SimulationKernel : ISimulationKernel
{
    INarrativeServiceRegistry _registry;
    SimulationTickPhase _current = SimulationTickPhase.Facts;

    public SimulationTickPhase CurrentPhase => _current;

    public void Initialize(INarrativeServiceRegistry registry)
        => _registry = registry ?? throw new ArgumentNullException(nameof(registry));

    public void Tick(SimulationTickPhase? singlePhase = null)
    {
        var phases = singlePhase.HasValue
            ? new[] { singlePhase.Value }
            : Enum.GetValues(typeof(SimulationTickPhase)).Cast<SimulationTickPhase>();

        foreach (var phase in phases)
        {
            _current = phase;
            ExecutePhase(phase);
        }
    }

    void ExecutePhase(SimulationTickPhase phase)
    {
        switch (phase)
        {
            case SimulationTickPhase.Facts:
                // IFactService: reconcile pending fact registrations
                break;
            case SimulationTickPhase.Interpretation:
                // IDialogueService / IVoiceService passive interpretation hooks
                break;
            case SimulationTickPhase.Social:
                // Relationship + faction drift from interpretation outputs
                break;
            case SimulationTickPhase.Events:
                // IEventBus flush queued events; ITimeService schedule due
                break;
            case SimulationTickPhase.Gating:
                // IGateService / IRuleEngine / IPacingService evaluate constraints
                break;
            case SimulationTickPhase.Content:
                // Script effects, new content unlock → may Register facts for next tick
                break;
        }
    }
}
```

### Phase ordering guard (tests)

```csharp
static class TickPhaseGuard
{
    static SimulationTickPhase _last = SimulationTickPhase.Facts;
    static readonly SimulationTickPhase[] Order = { Facts, Interpretation, Social, Events, Gating, Content };

    public static void Enter(SimulationTickPhase phase)
    {
        if (Array.IndexOf(Order, phase) < Array.IndexOf(Order, _last) && phase != Facts)
            throw new InvalidOperationException($"Tick phase {phase} after {_last}");
        _last = phase;
    }
}
```

MVP: guard used in tests; production kernel relies on ordered `Tick()` loop.

### SimulationSnapshot factory

Built at tick start from registry services — see [data-model.md](data-model.md). Handlers receive immutable snapshot during `Events` phase.

---

## Definition assets

None — kernel is pure runtime coordination.

---

## Runtime state

Kernel holds no save state in MVP. `[FULL]` optional tick counter / paused flag on kernel for debug.

---

## Core algorithms

### Full tick (game loop)

1. Build `SimulationSnapshot` from fact + story + social services
2. **Facts** — apply queued fact registrations/revocations
3. **Interpretation** — dialogue/voice systems consume new facts (passive lines, faculty interjections Phase 13+)
4. **Social** — apply relationship/faction deltas from interpretation results
5. **Events** — flush deferred typed events; advance time schedules
6. **Gating** — evaluate what content is legal; pacing windows
7. **Content** — fire unlocked scripts/dialogue; emit new facts → picked up next tick

### Conflict resolution (spec hierarchy)

When pacing delays a legal rule outcome: **RuleEngine** decides legality; **IPacingService** delays firing; kernel does not override rule truth — it schedules retry next tick.

---

## Event contracts

Kernel does not define domain events; it **schedules** the phase when `IEventBus.Publish` runs (Events phase). See [sim-event.md](sim-event.md).

| Phase | May publish |
|---|---|
| Facts | `FactRegistered`, `FactRevoked` |
| Social | `RelationshipChanged`, `FactionStandingChanged` |
| Events | `ActorMoved`, `TimeAdvanced`, custom |
| Content | `ContentUnlocked`, `DialogueStarted` |

---

## Integration matrix

| Depends on | Via |
|---|---|
| All services | `INarrativeServiceRegistry` |
| Simulation | Event bus, facts (phase handlers) |

| Consumed by | Usage |
|---|---|
| Game bootstrap | Calls `Tick()` each narrative frame |
| Debug | Traces phase + causal chain Phase 14 |
| Integration tests | Full loop Phase 14 |

**Forbidden:** Kernel → Presentation.

---

## MVP scope (Phase 1)

- [ ] `SimulationKernel` with 6 ordered phases
- [ ] `Initialize(registry)` wires registry
- [ ] Facts phase invokes `IFactService` CRUD hooks
- [ ] Events phase invokes `IEventBus` flush
- [ ] Tests: phase order, fact before gate slice, ≥15 tests (roadmap)

---

## Full scope

- `[FULL]` Tick timing telemetry (Phase 17 benchmarks)
- `[FULL]` Pause / step-one-phase debug API (Editor — DBG-04)
- `[FULL]` Event history ring buffer for debug replay

---

## File tree

```text
Packages/NarrativeFramework/Runtime/Kernel/ISimulationKernel.cs
Packages/NarrativeFramework/Runtime/Kernel/SimulationKernel.cs
Packages/NarrativeFramework/Runtime/Kernel/SimulationTickPhase.cs
Packages/NarrativeFramework/Runtime/Kernel/SimulationSnapshot.cs
Packages/NarrativeFramework/Tests/EditMode/Kernel/SimulationKernelTickOrderTests.cs
Packages/NarrativeFramework/Tests/EditMode/Kernel/FactsBeforeGatingTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Tick_RunsAllPhases_InOrder` | CurrentPhase sequence |
| `Tick_SinglePhase_OnlyRunsOne` | Partial tick |
| `FactsBeforeGating_FactVisibleToGate` | Register fact in Facts; gate reads in Gating same tick |
| `EventBus_SubscriberCalled_InEventsPhase` | Handler invocation count |

**Exit:** ≥15 Edit Mode tests (roadmap Phase 1).

---

## Deferred decisions

**Resolved** — [decisions-log.md](../decisions-log.md) EVT-01–EVT-05. Kernel uses same code path in player and editor; debug overlays optional.

---

## Related documents

- [sim-event.md](sim-event.md), [sim-fact.md](sim-fact.md)
- [integration.md](integration.md) — full closed loop Phase 14
