# Sim Info Flow Service — Implementation Architecture

- Spec: [systems/sim-info-flow.md](../systems/sim-info-flow.md)
- Roadmap: [Phase 10](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why truth ≠ knowledge

`IFactService` records what is **true in the world**. `IInfoFlowService` records **who learned what, when, and from whom**. Investigation gameplay depends on witnesses not omnisciently knowing player actions — without propagation, actors behave as if facts teleport globally.

### Why propagation is explicit

`RecordPropagation` creates an auditable edge in the knowledge graph. Dialogue can grant direct knowledge; rumors and leaks use the same API so chronicle and debug traces share one history format.

### Why DidActorLearn wraps actor memory

Fast path: delegate to `IActorService.KnowsFact`. Authoritative history: `GetHistory(factId)` returns ordered `InfoPropagation` records for thread/evidence UI and `[FULL]` reliability scoring.

### Why info flow sits in Simulation

Knowledge state is save-critical and gate-relevant ("witness must know fact X"). Presentation discovery UI reads projections; it does not own propagation logic.

---

## Implementation summary

| MVP (Phase 10) | Full |
|---|---|
| `RecordPropagation` | Automatic rumor spread tick `[FULL]` |
| `DidActorLearn` | Confidence levels per acquisition `[FULL]` |
| `GetHistory(factId)` | Faction/org knowledge buckets `[FULL]` |
| Subscribe `FactRegistered`, `DialogueChoiceSelected` | Leakage + false information `[FULL]` |
| Sync to `IActorService` memory on record | Public knowledge thresholds `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.InfoFlow`
- **References:** Runtime, Simulation.Actors, Simulation.Facts, Simulation.Time
- **Tick phase:** **Events** (propagation handlers) · **Social** (rumor batch `[FULL]`)

Registered as `IInfoFlowService`.

---

## Public API

See [contracts.md](contracts.md) — `IInfoFlowService`.

| Method | Purpose |
|---|---|
| `RecordPropagation(factId, fromActorId, toActorId, at)` | Append learn edge; update target memory |
| `bool DidActorLearn(actorId, factId)` | Query knowledge (not truth) |
| `IReadOnlyList<InfoPropagation> GetHistory(factId)` | Audit trail for fact |

Domain type:

```csharp
class InfoPropagation
{
    public string FactId;
    public string FromActorId;    // empty = world/system
    public string ToActorId;
    public GameTime At;
    public float Confidence;      // MVP: 1.0f default
}
```

---

## Internal implementation

### InfoFlowService

```csharp
sealed class InfoFlowService : IInfoFlowService, IStatefulService<InfoFlowServiceState>
{
    readonly InfoPropagationRegistry _registry = new();
    readonly IActorService _actors;
    readonly IFactService _facts;
    readonly IEventBus _events;

    public void RecordPropagation(string factId, string fromActorId, string toActorId, GameTime at)
    {
        if (!_facts.IsActive(factId)) return;

        var record = new InfoPropagation {
            FactId = factId,
            FromActorId = fromActorId,
            ToActorId = toActorId,
            At = at,
            Confidence = 1f
        };
        _registry.Add(record);
        TeachActor(toActorId, factId);

        _events.Publish(new SimEvent {
            Type = EventTypes.InfoPropagated,
            Timestamp = at,
            Payload = {
                ["factId"] = factId,
                ["from"] = fromActorId ?? "",
                ["to"] = toActorId
            }
        });
    }

    public bool DidActorLearn(string actorId, string factId)
        => _actors.KnowsFact(actorId, factId);

    public IReadOnlyList<InfoPropagation> GetHistory(string factId)
        => _registry.GetByFact(factId);

    void TeachActor(string actorId, string factId)
    {
        // Internal bridge: ActorService exposes internal LearnFact or handler syncs
        _actors.RememberEvent(actorId, new SimEvent {
            Type = EventTypes.FactLearned,
            Payload = { ["factId"] = factId }
        });
    }
}
```

### InfoPropagationRegistry (internal)

```csharp
sealed class InfoPropagationRegistry
{
    readonly List<InfoPropagation> _all = new();
    readonly Dictionary<string, List<InfoPropagation>> _byFact = new();

    public void Add(InfoPropagation p)
    {
        _all.Add(p);
        if (!_byFact.TryGetValue(p.FactId, out var list))
            _byFact[p.FactId] = list = new List<InfoPropagation>();
        list.Add(p);
    }

    public IReadOnlyList<InfoPropagation> GetByFact(string factId)
        => _byFact.TryGetValue(factId, out var l) ? l : Array.Empty<InfoPropagation>();
}
```

### InfoFlowEventHandler

| Source event | Action |
|---|---|
| `DialogueChoiceSelected` with `revealsFact` payload | RecordPropagation(player → speaker) or reverse per content |
| `FactRegistered` with `witnessActorIds` | Propagate to listed witnesses at fact timestamp |
| `ActorMoved` | `[FULL]` proximity witness checks |

---

## Definition assets

```csharp
class PropagationRuleDefinition : ContentDefinition   // [FULL]
{
    public string FactIdPattern;
    public string SourceActorTag;
    public float DailySpreadChance;
    public string[] ValidLocationTags;
}

class WitnessProfileDefinition : ContentDefinition
{
    public string ActorId;
    public float Reliability;           // affects confidence [FULL]
    public string[] ObservationTags;    // theft, violence, etc.
}
```

MVP: propagation driven by script/dialogue payloads only; no autonomous rumor rules.

---

## Runtime state

```csharp
class InfoFlowServiceState
{
    public List<InfoPropagation> Propagations;
}
```

Actor memory duplicates known fact IDs for fast query — restore must reconcile: load propagations then replay `TeachActor` or restore actor memory from its own envelope (preferred — single source in actor state).

---

## Core algorithms

### RecordPropagation invariants

1. Fact must be active (`IFactService.IsActive`)
2. No duplicate identical edge (same fact, from, to) in MVP — log and skip
3. Target actor receives memory update immediately (same tick, before Events flush)
4. Revoked facts: `DidActorLearn` may still true (they learned it was believed) — `[FULL]` policy flag `ForgetOnRevoke`

### Witness on player action `[FULL]`

```text
On ActorMoved / interaction event:
  find actors in same locationId
  filter by WitnessProfile
  for matching observation → Register fact + RecordPropagation(player → witness)
```

MVP: explicit content triggers only.

### Dialogue reveal path

1. Gate passes on dialogue choice
2. Dialogue service publishes choice event with `factId`, `targetActorId`
3. InfoFlow handler calls `RecordPropagation(playerActorId, target, Now)`
4. Gate for later topic uses `DidActorLearn`

### History query for threads

`IThreadService` evidence nodes may reference `factId`; thread UI calls `GetHistory` for "who told you" chronicle lines — read-only projection.

---

## Event contracts + tick phase

| Event | Publisher | Subscribers |
|---|---|---|
| `InfoPropagated` | IInfoFlowService | Chronicle, debug trace |
| `FactLearned` | Internal/teaching bridge | Actor memory index |
| `RumorSpread` | `[FULL]` batch processor | Social, faction standing |

Handlers run Events phase after fact registrations from same tick are visible in snapshot.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IFactService | Active fact validation |
| IActorService | Knowledge storage |
| ITimeService | Propagation timestamps |
| IEventBus | Notifications |
| ILocationService | Witness proximity `[FULL]` |

| Used by | Reason |
|---|---|
| IDialogueService | Topic unlock prechecks |
| IGateService | `DidActorLearn` conditions |
| IThreadService | Evidence provenance |
| IChronicleService | Projection entries |
| IPersistenceService | Propagation log |

**Forbidden:** InfoFlow → Presentation.

---

## MVP scope (Phase 10)

- [ ] `IInfoFlowService` + `InfoFlowService` + registry
- [ ] `RecordPropagation` / `DidActorLearn` / `GetHistory`
- [ ] Handler for dialogue reveal payload
- [ ] `InfoPropagated` event
- [ ] `InfoFlowServiceState` capture/restore
- [ ] Tests: propagation, history order, inactive fact rejected, dialogue integration

---

## Full scope

- `[FULL]` Confidence decay and contradictory rumors
- `[FULL]` Faction-level knowledge aggregates
- `[FULL]` Daily rumor propagation tick (Social phase)
- `[FULL]` Leakage from secret factions
- `[FULL]` False fact IDs with separate truth link

---

## File tree

```text
Packages/NarrativeFramework/Simulation/InfoFlow/IInfoFlowService.cs
Packages/NarrativeFramework/Simulation/InfoFlow/InfoFlowService.cs
Packages/NarrativeFramework/Simulation/InfoFlow/InfoPropagation.cs
Packages/NarrativeFramework/Simulation/InfoFlow/InfoPropagationRegistry.cs
Packages/NarrativeFramework/Simulation/InfoFlow/InfoFlowServiceState.cs
Packages/NarrativeFramework/Simulation/InfoFlow/InfoFlowEventHandler.cs
Packages/NarrativeFramework/Tests/EditMode/InfoFlow/InfoPropagationTests.cs
Packages/NarrativeFramework/Tests/EditMode/InfoFlow/InfoFlowDialogueTests.cs
Packages/NarrativeFramework/Tests/EditMode/InfoFlow/InfoFlowPersistenceTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `RecordPropagation_DidActorLearnTrue` | Actor knows fact |
| `GetHistory_ReturnsOrderedEdges` | Count + from/to |
| `InactiveFact_SkipsPropagation` | No memory change |
| `DuplicateEdge_SkippedOnce` | Single history entry |
| `DialogueReveal_CreatesPropagation` | Integration handler |
| `CaptureRestore_PreservesHistory` | List equality |
| `InfoPropagated_EventFired` | Bus delivery |

Phase 10 exit: includes save/load + propagation after dialogue scenario.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-11, SIM-02 (info-flow forget Phase 8; optional `actor_player`).

---

## Bootstrap and actor memory bridge

Phase 10 adds internal `IActorMemoryBridge` so `InfoFlowService` does not call private `ActorService` methods:

```csharp
interface IActorMemoryBridge
{
    void TeachFact(string actorId, string factId);
}
```

`ActorService` implements bridge; registered alongside actor service. Alternative MVP: `InfoFlowEventHandler` lives in Simulation assembly and calls package-internal APIs — prefer bridge for test doubles.

Dialogue content authors attach propagation metadata to choice nodes:

```json
{
  "choiceId": "choice_reveal_alibi",
  "propagateFact": "fact_player_alibi",
  "propagateTo": "actor_witness"
}
```

`IDialogueService` includes these keys in `DialogueChoiceSelected` payload; handler resolves player actor ID from story config constant `actor_player`.

Chronicle projection reads `GetHistory` read-only — never mutates propagation log. Thread evidence may store `factId` only; provenance UI calls info flow at display time.

---

## Related documents

- [sim-fact.md](sim-fact.md), [sim-actor.md](sim-actor.md)
- [ledger-thread.md](ledger-thread.md), [sim-persistence.md](sim-persistence.md)
- [story-dialogue.md](../systems/story-dialogue.md)
