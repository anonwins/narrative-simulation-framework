# NSF Shared Data Model

**Cross-module data shapes** — definitions, runtime state, persistence, validation.

- Glossary: [Content IDs](../terminology-glossary.md)
- Roadmap: Phases 2 (content), 10 (persistence)
- Contracts: [contracts.md](contracts.md)

---

## Design rationale

### Why separate Definition from State

Content packs author **definitions** (ScriptableObjects / JSON): stable IDs, narrative text keys, rules references. Runtime holds **mutable state**: flag values, belief phases, actor memory. Mixing them causes save bugs and prevents hot-reloading content packs.

### Why content IDs are snake_case with prefixes

Prefix namespaces (`fact_`, `flag_`, `actor_`) let the pipeline validate references without loading full graphs. Setting nouns stay in content pack *values*, not IDs (glossary SSOT).

### Why a single save envelope

If each service serializes independently with ad hoc JSON, schema migration becomes impossible. `IPersistenceService` owns one versioned blob aggregating per-service `*State` DTOs.

---

## Implementation summary

| MVP (Phase 2) | Full (Phase 10+) |
|---|---|
| `ContentDefinition` base | Migration pipeline v1→v2 |
| ID format validation | Cross-pack reference graphs |
| JSON import/export for test pack | Binary snapshot option |
| `SaveSnapshot` schema v1 | Hash verification + delta saves |

---

## Assembly and namespace

| Type family | Assembly | Namespace |
|---|---|---|
| `ContentDefinition`, validators | Content | `NarrativeFramework.Content.Definitions` |
| `*State` DTOs | owning module | `NarrativeFramework.{Module}.State` |
| `SaveSnapshot`, envelopes | Simulation | `NarrativeFramework.Simulation.Persistence` |
| Shared primitives (`GameTime`) | Simulation | `NarrativeFramework.Simulation` |

Content definitions never reference Presentation types.

---

## Definition layer (authored content)

### ContentDefinition base

```csharp
namespace NarrativeFramework.Content.Definitions
{
    abstract class ContentDefinition : ScriptableObject
    {
        [SerializeField] string id;
        [SerializeField] string schemaVersion = "1.0";

        public string Id => id;
        public string SchemaVersion => schemaVersion;

        public virtual void Validate(IValidationContext ctx) { }
    }
}
```

**Why ScriptableObject:** Unity-native authoring, diff-friendly in VCS, pipeline can load without Play Mode.

### Common definition types (Phase 2+)

| Type | ID prefix | Module doc |
|---|---|---|
| `FacultyDefinition` | `faculty_` | [cognition-faculty.md](cognition-faculty.md) |
| `ActorDefinition` | `actor_` | [sim-actor.md](sim-actor.md) |
| `BeliefDefinition` | `belief_` | [cognition-belief.md](cognition-belief.md) |
| `StoryFlagDefinition` | `flag_` | [story-state.md](story-state.md) |
| `GateRuleDefinition` | `gate_` | [rules-gate.md](rules-gate.md) |
| `DialogueGraphDefinition` | content-store IDs | [story-dialogue.md](story-dialogue.md) |

### ContentRegistry (internal)

**Why internal:** `IContentStore` is the only public entry. Registry allows module-local indexes without exposing mutation to game code.

```csharp
sealed class ContentRegistry
{
    readonly Dictionary<(Type type, string id), ContentDefinition> _map = new();
    public void Register(ContentDefinition def) { /* index by type + id */ }
    public bool TryGet<T>(string id, out T def) where T : ContentDefinition { /* ... */ }
}
```

---

## Runtime state layer (mutable simulation)

### Conventions

| Suffix | Serializable | Owned by |
|---|---|---|
| `*State` | yes | service listed in architecture doc |
| Plain DTO structs | yes | passed across services / events |

Each service exposing mutable state implements:

```csharp
interface IStatefulService<TState> where TState : class, new()
{
    TState CaptureState();
    void RestoreState(TState state);
}
```

Phase 10 `IPersistenceService` discovers all `IStatefulService` registrations via registry metadata or explicit register list.

### Shared primitives

```csharp
struct GameTime { public int Day; public int Hour; public int Minute; }

struct GameTimeDelta { public int Days; public int Hours; public int Minutes; }

enum ModifierSource { Equipment, Emotion, Conduct, Script, Debug }
```

### SimulationSnapshot (read-only aggregate)

**Why:** Event handlers need consistent read view during a tick without locking every service.

```csharp
class SimulationSnapshot
{
    public IReadOnlyFactView Facts { get; }
    public IReadOnlyStoryStateView Story { get; }
    // built at tick start; immutable for handler duration
}
```

---

## Fact model

```csharp
class FactRecord
{
    public string Id;           // fact_*
    public string SubjectId;    // actor_ or location_
    public string Predicate;    // content-defined verb key
    public string Value;
    public GameTime RegisteredAt;
    public bool IsRevoked;
}
```

**Fact vs story flag:** Facts are simulation truth (`IFactService`). Flags are narrative progression switches (`IStoryStateService`). Architecture docs must not conflate them.

Internal: `FactRegistry` indexes active facts by ID — see [sim-fact.md](sim-fact.md).

---

## Event model

```csharp
abstract class SimEventBase
{
    public string Id;
    public GameTime Timestamp;
    public int Priority;  // lower = earlier; EVT-05 ordering
    public abstract string EventType { get; }
}

[AttributeUsage(AttributeTargets.Class)]
sealed class SimEventTypeAttribute : Attribute
{
    public SimEventTypeAttribute(string typeId) => TypeId = typeId;
    public string TypeId { get; }
}

// Example:
[SimEventType("FactRegistered")]
sealed class FactRegisteredEvent : SimEventBase
{
    public override string EventType => "FactRegistered";
    public string FactId;
    public string SourceActorId;
}
```

Concrete event types live in `NarrativeFramework.Simulation.Events`. Content script `TriggerEvent` resolves through the event type registry ([decisions-log.md](../decisions-log.md) EVT-01).

---

## Persistence model

### SaveSnapshot (schema v1)

```csharp
class SaveSnapshot
{
    public int SchemaVersion;   // matches NsfVersion.SaveSchemaVersion
    public string StateHash;    // SHA256 of canonical JSON payload
    public byte[] Payload;      // UTF-8 JSON: SaveGameRoot
}

class SaveGameRoot
{
    public List<ServiceStateEnvelope> Services;
    public GameTime SavedAt;
}

class ServiceStateEnvelope
{
    public string ServiceType;  // typeof(IFactService).FullName or stable key
    public string JsonState;
}
```

**Why hash:** Integration tests assert bit-identical restore (roadmap Phase 10 exit criteria).

### Migration

```csharp
interface ISaveMigration
{
    int FromVersion { get; }
    int ToVersion { get; }
    SaveSnapshot Migrate(SaveSnapshot input);
}
```

Migrations live in Simulation assembly; register by version chain.

---

## Validation

### Content ID rules

```csharp
static class ContentIdValidator
{
    static readonly Regex Pattern = new(@"^[a-z][a-z0-9_]+$");

    public static bool IsValid(string id, string requiredPrefix)
        => id.StartsWith(requiredPrefix) && Pattern.IsMatch(id);
}
```

Pipeline ([content-pipeline.md](content-pipeline.md)) calls validators on every definition reference.

### Reference integrity

Build dependency graph: definition A references B → edge A→B. Pipeline fails on missing targets before runtime.

---

## JSON round-trip (Phase 2)

**Why Newtonsoft:** Roadmap locks `com.unity.nuget.newtonsoft-json` for Edit Mode pack import/export and test fixtures.

```csharp
static class ContentJson
{
    public static string SerializeDefinition(ContentDefinition def);
    public static T DeserializeDefinition<T>(string json) where T : ContentDefinition;
}
```

Tests: author minimal pack in JSON → import → export → deep equals.

---

## Integration matrix

| Producer | Consumer | Data |
|---|---|---|
| Content pipeline | ContentRegistry | validated definitions |
| All `*Service` | IPersistenceService | `*State` envelopes |
| IFactService | IRuleEngine, IThreadService | `FactRecord` views |
| IStoryStateService | IGateService | flag snapshots |

---

## MVP scope

Phase 2:

- [ ] `ContentDefinition` base + validator
- [ ] `ContentIdValidator`
- [ ] `SaveSnapshot` / envelope types (stub capture)
- [ ] JSON round-trip test for `FrameworkTestPack`

Phase 10:

- [ ] Full capture/restore all stateful services
- [ ] Hash-stable integration test

---

## Full scope

- `[FULL]` Binary MessagePack snapshots for large saves
- `[FULL]` Content definition source generators from `.nsf` scripts

---

## File tree

```text
Packages/NarrativeFramework/Content/Definitions/ContentDefinition.cs
Packages/NarrativeFramework/Content/Validation/ContentIdValidator.cs
Packages/NarrativeFramework/Content/Internal/ContentRegistry.cs
Packages/NarrativeFramework/Simulation/GameTime.cs
Packages/NarrativeFramework/Simulation/Events/SimEventBase.cs
Packages/NarrativeFramework/Simulation/Events/FactRegisteredEvent.cs
Packages/NarrativeFramework/Simulation/Facts/FactRecord.cs
Packages/NarrativeFramework/Simulation/Persistence/SaveSnapshot.cs
Packages/NarrativeFramework/Simulation/Persistence/ServiceStateEnvelope.cs
```

---

## Test plan

| Test | Phase |
|---|---|
| `ContentIdValidator_RejectsInvalidPrefix` | 2 |
| `ContentJson_RoundTrip_FacultyDefinition` | 2 |
| `SaveSnapshot_Restore_IdenticalHash` | 10 |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-05 (fact audit in saves); story state typing expands as content requires.

---

## Related documents

- [content-store.md](content-store.md), [content-pipeline.md](content-pipeline.md)
- [sim-fact.md](sim-fact.md), [sim-persistence.md](sim-persistence.md)
