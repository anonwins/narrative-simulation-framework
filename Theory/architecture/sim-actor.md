# Sim Actor Service — Implementation Architecture

- Spec: [systems/sim-actor.md](../systems/sim-actor.md)
- Roadmap: [Phase 6](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why actors are stateful agents, not dialogue containers

NSF investigation depends on **who knows what, remembers what, and reacts how**. An actor that only stores a dialogue graph ID cannot support withheld testimony, mood shifts, or schedule-driven absence. `IActorService` is the authoritative runtime store for per-actor simulation state; dialogue, gates, and info flow query it rather than duplicating memory.

### Why definitions are separate from instances

Content packs author `ActorDefinition` assets (identity, default faction, dialogue profile). Runtime `ActorInstance` holds mutable mood, availability, location binding, and memory. Separation keeps save payloads small and allows the same definition to spawn multiple instances in `[FULL]` without rewriting content.

### Why memory lives on the actor, not only in info flow

`IInfoFlowService` tracks **propagation** (who learned a fact, from whom, when). `IActorService.KnowsFact` is the fast per-actor query surface; info flow writes into actor memory when propagation completes. Actors also remember **events** (insults, promises) that are not facts — `RememberEvent` stores narrative memory distinct from fact ownership.

### Why no Presentation references

Actor portraits and mood UI are Presentation concerns. Core `ActorService` exposes IDs and state DTOs only; headless tests never load Canvas or `NarrativeFramework.Presentation`.

---

## Implementation summary

| MVP (Phase 6) | Full |
|---|---|
| `GetState` / `SetState` / `GetLocation` | Full schedule engine `[FULL]` |
| `KnowsFact` backed by memory bank | Actor-to-actor relationship mirror `[FULL]` |
| `RememberEvent` + event subscription | Recognition / appearance reactivity `[FULL]` |
| Definition lookup via `IContentStore` | Multi-instance spawn from one definition `[FULL]` |
| Persistence via `ActorServiceState` | Memory decay tiers `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Actors`
- **References:** Runtime, Content (definitions only — no Presentation)
- **Tick phase:** **Facts** (location sync) · **Social** (availability reactions) · **Events** (memory writes from bus)

Registered as `IActorService` in `INarrativeServiceRegistry`.

---

## Public API

See [contracts.md](contracts.md) — `IActorService`.

| Method | Purpose |
|---|---|
| `GetState(actorId)` | Availability, mood, dialogue flags |
| `SetState(actorId, state)` | Scripted or schedule-driven updates |
| `KnowsFact(actorId, factId)` | Per-actor knowledge query |
| `RememberEvent(actorId, e)` | Append narrative memory from `SimEvent` |
| `GetLocation(actorId)` | Current location ID (delegates to instance) |

Domain types (`ActorState`, `MemoryBank`, `ActorDefinition`) are internal to Simulation; only the interface crosses asmdef boundaries.

---

## Internal implementation

### ActorService

```csharp
sealed class ActorService : IActorService, IStatefulService<ActorServiceState>
{
    readonly ActorInstanceRegistry _instances = new();
    readonly IContentStore _content;
    readonly IEventBus _events;
    readonly ILocationService _locations;

    public ActorState GetState(string actorId)
    {
        EnsureInstance(actorId);
        return _instances.Get(actorId).State;
    }

    public void SetState(string actorId, ActorState state)
    {
        ContentIdValidator.IsValid(actorId, "actor_");
        var inst = EnsureInstance(actorId);
        var prior = inst.State.Availability;
        inst.State = state;
        if (prior != state.Availability)
            _events.Publish(new SimEvent {
                Type = EventTypes.ActorAvailabilityChanged,
                Payload = { ["actorId"] = actorId, ["availability"] = state.Availability.ToString() }
            });
    }

    public bool KnowsFact(string actorId, string factId)
        => EnsureInstance(actorId).Memory.Knows(factId);

    public void RememberEvent(string actorId, SimEvent e)
    {
        EnsureInstance(actorId).Memory.Remember(e);
        _events.Publish(new SimEvent {
            Type = EventTypes.ActorMemoryUpdated,
            Payload = { ["actorId"] = actorId, ["sourceEventId"] = e.Id }
        });
    }

    public string GetLocation(string actorId)
        => EnsureInstance(actorId).LocationId;

    ActorInstance EnsureInstance(string actorId)
    {
        if (_instances.TryGet(actorId, out var inst)) return inst;
        var def = _content.GetDefinition<ActorDefinition>(actorId);
        inst = ActorInstance.FromDefinition(def);
        _instances.Add(inst);
        return inst;
    }
}
```

### ActorInstanceRegistry (internal)

```csharp
sealed class ActorInstanceRegistry
{
    readonly Dictionary<string, ActorInstance> _byId = new();

    public bool TryGet(string id, out ActorInstance inst) => _byId.TryGetValue(id, out inst);
    public ActorInstance Get(string id) => _byId[id];
    public void Add(ActorInstance inst) => _byId[inst.Id] = inst;
    public IReadOnlyCollection<ActorInstance> All => _byId.Values;
}
```

### ActorInstance + MemoryBank

```csharp
sealed class ActorInstance
{
    public string Id;
    public ActorState State;
    public MemoryBank Memory = new();
    public string LocationId;

    public static ActorInstance FromDefinition(ActorDefinition def) => new() {
        Id = def.Id,
        LocationId = def.DefaultLocationId,
        State = new ActorState { MoodId = def.DefaultMoodId, Availability = ActorAvailability.Idle }
    };
}

sealed class MemoryBank
{
    readonly HashSet<string> _knownFacts = new();
    readonly List<MemoryEntry> _events = new();

    public bool Knows(string factId) => _knownFacts.Contains(factId);
    public void LearnFact(string factId) => _knownFacts.Add(factId);
    public void Remember(SimEvent e) => _events.Add(MemoryEntry.FromEvent(e));
}
```

### ActorEventHandler (internal)

Subscribes to `FactRegistered`, `DialogueChoiceSelected`, `ActorMoved` during kernel init. Routes relevant payloads to `RememberEvent` or `Memory.LearnFact` when info flow confirms propagation.

---

## Definition assets

Content store types (ScriptableObject or JSON pack):

```csharp
class ActorDefinition : ContentDefinition
{
    public string DefaultFactionId;
    public string DefaultLocationId;
    public string DefaultMoodId;
    public string DisplayNameKey;       // locale key
    public DialogueProfileRef DialogueProfile;
    public BehaviorProfileRef BehaviorProfile;
}
```

IDs use `actor_*` prefix per glossary. Schedules reference `schedule_*` definitions `[FULL]`.

---

## Runtime state

```csharp
class ActorServiceState
{
    public List<ActorInstanceSnapshot> Instances;
}

class ActorInstanceSnapshot
{
    public string Id;
    public ActorState State;
    public string LocationId;
    public MemoryBankSnapshot Memory;
}

class MemoryBankSnapshot
{
    public List<string> KnownFactIds;
    public List<MemoryEntry> Events;
}

enum ActorAvailability { Idle, Talking, Observing, Moving, Unavailable, Hostile, Withholding }
```

Captured/restored via `IStatefulService<ActorServiceState>`; envelope key `NarrativeFramework.Simulation.Actors.ActorService` in Phase 10 persistence.

---

## Core algorithms

### Lazy instance materialization

1. First query for `actor_*` loads `ActorDefinition` from `IContentStore`
2. Instance created with definition defaults
3. Subsequent calls mutate instance only — definition is read-only at runtime

### KnowsFact vs info flow

- **Direct learn:** script or dialogue action calls internal `ActorMemory.LearnFact` after gate passes
- **Propagation:** `IInfoFlowService.RecordPropagation` → handler calls `LearnFact` on target actor
- `KnowsFact` never queries `IFactService` — knowing ≠ truth

### Location synchronization

When `ILocationService.Travel(actorId, to)` succeeds, location service updates authoritative position; actor service listens for `ActorMoved` and mirrors `LocationId` on instance. `GetLocation` reads instance cache for O(1) dialogue checks.

### Availability transitions

Schedule hooks (Phase 6 MVP: manual/scripted) call `SetState` with new `ActorAvailability`. `Talking` set by `IDialogueService.StartConversation`; cleared on `EndConversation`. Gates query availability during **Gating** phase.

---

## Event contracts + tick phase

| Event | Publisher | Subscribers | Phase |
|---|---|---|---|
| `ActorMoved` | ILocationService | IActorService, IInfoFlowService | Events |
| `ActorAvailabilityChanged` | IActorService | IDialogueService, IGateService | Events |
| `ActorMemoryUpdated` | IActorService | IChronicleService (projection), debug | Events |
| `DialogueChoiceSelected` | IDialogueService | ActorEventHandler → RememberEvent | Events |
| `FactRegistered` | IFactService | Info flow (conditional actor learn) | Events |

**Write timing:** state mutations during Facts/Social; event publish queued, flushed Events phase.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IContentStore | Actor definitions |
| IEventBus | Memory + availability notifications |
| ILocationService | Travel sync (optional at init — resolved from registry) |
| ContentIdValidator | ID policy |

| Used by | Reason |
|---|---|
| IDialogueService | Speaker availability, topic gating |
| IInfoFlowService | Propagation targets |
| IRelationshipService | Per-actor metric keys (Phase 7) |
| IFactionService | Member lookup |
| ICompanionService | Wraps one actor ID |
| IGateService | `KnowsFact`, availability conditions |
| IPersistenceService | Instance snapshots |

**Forbidden:** `NarrativeFramework.Simulation.Actors` → `NarrativeFramework.Presentation`.

---

## MVP scope (Phase 6)

- [ ] `IActorService` + `ActorService` + registries
- [ ] `ActorDefinition` content type stub
- [ ] `GetState` / `SetState` / `KnowsFact` / `RememberEvent` / `GetLocation`
- [ ] Subscribe to `ActorMoved`, `DialogueChoiceSelected`
- [ ] `ActorServiceState` capture/restore stub
- [ ] Tests: lazy init, memory roundtrip, availability event, location mirror

---

## Full scope

- `[FULL]` Schedule engine driving location + availability by `ITimeService`
- `[FULL]` Mood decay and reaction profiles
- `[FULL]` Actor-to-actor memory and relationship queries
- `[FULL]` Recognition layers (equipment, conduct appearance)
- `[FULL]` Death/removal irreversible state transitions

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Actors/IActorService.cs
Packages/NarrativeFramework/Simulation/Actors/ActorService.cs
Packages/NarrativeFramework/Simulation/Actors/ActorInstanceRegistry.cs
Packages/NarrativeFramework/Simulation/Actors/ActorInstance.cs
Packages/NarrativeFramework/Simulation/Actors/ActorState.cs
Packages/NarrativeFramework/Simulation/Actors/MemoryBank.cs
Packages/NarrativeFramework/Simulation/Actors/MemoryEntry.cs
Packages/NarrativeFramework/Simulation/Actors/ActorDefinition.cs
Packages/NarrativeFramework/Simulation/Actors/ActorServiceState.cs
Packages/NarrativeFramework/Simulation/Actors/ActorEventHandler.cs
Packages/NarrativeFramework/Tests/EditMode/Actors/ActorServiceTests.cs
Packages/NarrativeFramework/Tests/EditMode/Actors/ActorMemoryTests.cs
Packages/NarrativeFramework/Tests/EditMode/Actors/ActorLocationSyncTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `GetState_LazyCreatesFromDefinition` | Defaults match definition |
| `KnowsFact_AfterLearn_ReturnsTrue` | Memory bank query |
| `RememberEvent_AppendsMemoryEntry` | Count + event ID stored |
| `SetState_AvailabilityChange_PublishesEvent` | Handler sees `ActorAvailabilityChanged` |
| `GetLocation_MirrorsAfterActorMoved` | Location ID updated from bus |
| `CaptureRestore_PreservesMemoryAndState` | Stateful roundtrip |
| `InvalidActorId_ThrowsOnSetState` | Validator rejects bad prefix |

Target: contribute to Phase 6 exit ≥18 tests across time/location/actor suite.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-02, SIM-09, SIM-10.

---

## Related documents

- [sim-time.md](sim-time.md), [sim-location.md](sim-location.md)
- [sim-info-flow.md](sim-info-flow.md), [social-relationship.md](social-relationship.md)
- [runtime-kernel.md](runtime-kernel.md), [contracts.md](contracts.md)
