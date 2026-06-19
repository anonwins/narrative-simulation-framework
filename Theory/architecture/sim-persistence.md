# Sim Persistence Service — Implementation Architecture

- Spec: [systems/sim-persistence.md](../systems/sim-persistence.md)
- Roadmap: [Phase 10](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why one persistence facade

Without `IPersistenceService`, each module invents JSON shapes and load order bugs appear silently. A versioned `SaveSnapshot` aggregates all `IStatefulService` envelopes so restore replays one canonical document with hash verification.

### Why schema version + hash

Roadmap exit criteria require **bit-identical restore**. SHA256 over canonical JSON catches serializer drift and field reorder bugs in integration tests before players lose saves.

### Why services own their state DTOs

Persistence orchestrates capture/restore; it does not embed domain logic. Each `*ServiceState` type lives with its service; persistence only serializes envelopes keyed by stable service type names.

### Why persistence is Simulation, not Platform

Save files are game state, not UI layout. File I/O adapter (`ISaveStorage`) can live in a thin Platform layer `[FULL]`; core `Capture`/`Restore` stays Edit Mode testable with in-memory bytes.

---

## Implementation summary

| MVP (Phase 10) | Full |
|---|---|
| `Capture()` → `SaveSnapshot` | Delta/incremental saves `[FULL]` |
| `Restore(SaveSnapshot)` | Cloud sync abstraction `[FULL]` |
| Discover `IStatefulService` from registry | Autosave rotation `[FULL]` |
| Schema v1 + single migration stub | Migration chain v1→vN `[FULL]` |
| State hash in snapshot | Compression `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Persistence`
- **References:** Runtime (registry), all Simulation stateful services
- **Tick phase:** **None** — invoked outside kernel tick (load screen, manual save)

Registered as `IPersistenceService`. Must not reference Presentation.

---

## Public API

See [contracts.md](contracts.md) — `IPersistenceService`.

| Member | Contract |
|---|---|
| `int CurrentSchemaVersion { get; }` | Matches `NsfVersion.SaveSchemaVersion` |
| `SaveSnapshot Capture()` | Aggregate all registered stateful services |
| `void Restore(SaveSnapshot snapshot)` | Validate version/hash; restore in dependency order |

---

## Internal implementation

### PersistenceService

```csharp
sealed class PersistenceService : IPersistenceService
{
    readonly INarrativeServiceRegistry _registry;
    readonly IReadOnlyList<ISaveMigration> _migrations;
    readonly ITimeService _time;

    public int CurrentSchemaVersion => NsfVersion.SaveSchemaVersion;

    public SaveSnapshot Capture()
    {
        var root = new SaveGameRoot {
            SavedAt = _time.Now,
            Services = CollectEnvelopes()
        };
        var json = JsonSerializer.Serialize(root, SaveJsonOptions.Canonical);
        var payload = Encoding.UTF8.GetBytes(json);
        return new SaveSnapshot {
            SchemaVersion = CurrentSchemaVersion,
            Payload = payload,
            StateHash = HashUtil.Sha256Hex(payload)
        };
    }

    public void Restore(SaveSnapshot snapshot)
    {
        snapshot = MigrateToCurrent(snapshot);
        ValidateHash(snapshot);
        var root = JsonSerializer.Deserialize<SaveGameRoot>(
            Encoding.UTF8.GetString(snapshot.Payload), SaveJsonOptions.Canonical);

        RestoreInOrder(root.Services);
    }

    List<ServiceStateEnvelope> CollectEnvelopes()
    {
        var list = new List<ServiceStateEnvelope>();
        foreach (var svc in _registry.GetAllStatefulServices())
        {
            var state = svc.CaptureState();
            list.Add(new ServiceStateEnvelope {
                ServiceType = svc.StateKey,
                JsonState = JsonSerializer.Serialize(state, SaveJsonOptions.Canonical)
            });
        }
        return list;
    }
}
```

### Restore ordering

Fixed topological order prevents FK-like failures:

```text
1. TimeService
2. FactService
3. LocationService
4. ActorService
5. InfoFlowService
6. EconomyService
7. StoryStateService (Story asmdef)
8. Relationship / Faction / Companion / Ideology (Social)
9. Cognition services [Phase 8+]
10. Thread / Chronicle projection caches (rebuild after restore) [Phase 9+]
```

`RestoreInOrder` sorts envelopes by `ServiceRestoreOrder` attribute; unknown keys log warning in debug.

### StatefulServiceRegistry extension (internal)

```csharp
interface IStatefulService
{
    string StateKey { get; }
    object CaptureState();
    void RestoreState(object state);
}
```

Generic `IStatefulService<TState>` adapters implement non-generic interface for persistence discovery.

### Migration pipeline

```csharp
SaveSnapshot MigrateToCurrent(SaveSnapshot input)
{
    var v = input.SchemaVersion;
    while (v < CurrentSchemaVersion)
    {
        var mig = _migrations.First(m => m.FromVersion == v);
        input = mig.Migrate(input);
        v = mig.ToVersion;
    }
    return input;
}
```

MVP: empty migration list when only v1 exists; stub `NoOpMigration` for tests.

---

## Definition assets

None in content store. Optional `SaveSlotMetadata` for UI `[FULL]`:

```csharp
class SaveSlotDisplayInfo
{
    public GameTime SavedAt;
    public string ChapterLabelKey;
}
```

Authored strings only — not loaded by core restore path.

---

## Runtime state

Persistence service itself is stateless between operations. Transient during restore:

```csharp
class RestoreContext
{
    public bool IsRestoring;
    public Queue<Action> PostRestoreActions;
}
```

Services suppress event publish during restore when `RestoreContext.IsRestoring` — replay events `[FULL]` optional.

---

## Core algorithms

### Capture

1. Pause event flush `[FULL]` or capture at tick boundary
2. Iterate all `IStatefulService` registrations
3. Serialize each to JSON with canonical options (sorted keys, invariant culture)
4. Build `SaveGameRoot`, UTF-8 encode, hash, wrap in `SaveSnapshot`

### Restore

1. Run migrations until `SchemaVersion == Current`
2. Recompute hash — mismatch throws `SaveCorruptException`
3. Deserialize root
4. Set `RestoreContext.IsRestoring = true`
5. For each envelope in sort order: deserialize to typed state, `RestoreState`
6. Clear restoring flag; invoke `IChronicleService.RefreshProjection()` and similar read models
7. Optional: single `GameLoaded` event after full restore

### Hash validation

```csharp
static void ValidateHash(SaveSnapshot s)
{
    var computed = HashUtil.Sha256Hex(s.Payload);
    if (!string.Equals(computed, s.StateHash, StringComparison.OrdinalIgnoreCase))
        throw new SaveCorruptException("State hash mismatch");
}
```

### Integration test: full registry roundtrip

Register real or test doubles for all Phase 0–10 services → mutate → `Capture` → new registry → `Restore` → assert identical hash on second capture.

---

## Event contracts + tick phase

| Event | When |
|---|---|
| `GameSaved` | After successful Capture `[FULL]` |
| `GameLoaded` | After Restore completes |
| `SaveMigrationApplied` | Debug only during migrate |

Persistence does not participate in simulation tick phases. Chronicle refresh runs **after** restore, not during kernel tick.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| INarrativeServiceRegistry | Service discovery |
| All IStatefulService impls | State sources |
| ITimeService | SavedAt timestamp |
| ISaveMigration chain | Version upgrades |

| Used by | Reason |
|---|---|
| Game bootstrap | New game vs continue |
| Platform save UI `[FULL]` | Slot read/write |
| Debug menu | Quick save/load Phase 14 |

**Forbidden:** Persistence → Presentation (UI calls persistence through game facade).

### Stateful services in Phase 10 scope

| Service | State type |
|---|---|
| IFactService | FactServiceState |
| ITimeService | TimeServiceState |
| ILocationService | LocationServiceState |
| IActorService | ActorServiceState |
| IInfoFlowService | InfoFlowServiceState |
| IEconomyService | EconomyServiceState |
| IStoryStateService | StoryStateServiceState |
| Social services | respective `*State` |

---

## MVP scope (Phase 10)

- [ ] `IPersistenceService` + `PersistenceService`
- [ ] `SaveSnapshot`, `SaveGameRoot`, `ServiceStateEnvelope` per data-model
- [ ] Registry discovery of stateful services
- [ ] Canonical JSON + SHA256 hash
- [ ] Restore order attribute
- [ ] Integration test: full registry roundtrip identical hash
- [ ] Unit tests: corrupt hash throws, empty capture

---

## Full scope

- `[FULL]` File storage adapter (PC path, console API)
- `[FULL]` Migration implementations per schema bump
- `[FULL]` Autosave + quicksave slots
- `[FULL]` Selective restore (debug time travel)
- `[FULL]` Compression (gzip payload)

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Persistence/IPersistenceService.cs
Packages/NarrativeFramework/Simulation/Persistence/PersistenceService.cs
Packages/NarrativeFramework/Simulation/Persistence/SaveSnapshot.cs
Packages/NarrativeFramework/Simulation/Persistence/SaveGameRoot.cs
Packages/NarrativeFramework/Simulation/Persistence/ServiceStateEnvelope.cs
Packages/NarrativeFramework/Simulation/Persistence/IStatefulService.cs
Packages/NarrativeFramework/Simulation/Persistence/ISaveMigration.cs
Packages/NarrativeFramework/Simulation/Persistence/SaveJsonOptions.cs
Packages/NarrativeFramework/Simulation/Persistence/HashUtil.cs
Packages/NarrativeFramework/Simulation/Persistence/ServiceRestoreOrderAttribute.cs
Packages/NarrativeFramework/Simulation/Persistence/SaveCorruptException.cs
Packages/NarrativeFramework/Tests/EditMode/Persistence/PersistenceRoundtripTests.cs
Packages/NarrativeFramework/Tests/EditMode/Persistence/PersistenceHashTests.cs
Packages/NarrativeFramework/Tests/EditMode/Persistence/PersistenceMigrationTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `CaptureRestore_IdenticalHash` | Phase 10 exit criterion |
| `CorruptHash_Throws` | SaveCorruptException |
| `RestoreOrder_TimeBeforeActor` | Actor sees restored time |
| `UnknownEnvelope_SkippedWithWarning` | Forward compatibility |
| `EmptyRegistry_CaptureMinimalRoot` | Valid snapshot |
| `Migration_v0_to_v1` | Stub migration chain |
| `Restore_SuppressesEventsDuringLoad` | No duplicate chronicle entries |

Combined with info flow dialogue scenario for ≥15 Phase 10 tests.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) P-09, P-10, SIM-03.

---

## Related documents

- [data-model.md](data-model.md) — SaveSnapshot schema v1
- [sim-fact.md](sim-fact.md), [sim-economy.md](sim-economy.md), [sim-info-flow.md](sim-info-flow.md)
- [foundation.md](foundation.md) — registry patterns
