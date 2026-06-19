# Sim Location Service — Implementation Architecture

- Spec: [systems/sim-location.md](../systems/sim-location.md)
- Roadmap: [Phase 6](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why locations are narrative containers

NSF movement is not map traversal for its own sake — each area carries access rules, ambient mood, investigation hooks, and time-gated availability. `ILocationService` models a **connectivity graph** of narrative spaces (areas/scenes) with travel costs and gate integration, not Unity scene loading (that is Presentation/Exploration in Phase 13+).

### Why travel is simulation-first

`Travel(actorId, toLocationId)` mutates authoritative position, consumes time via `ITimeService`, and publishes `ActorMoved`. Exploration UI calls the same API through `IExplorationService` adapter later — core logic stays headless-testable.

### Why CanTravel is separate from Travel

Gates, story flags, and relationships deny travel before side effects. `CanTravel` evaluates connection graph + `IGateService` rules without advancing time or firing events. UI and AI planners query safely.

### Why location ≠ interaction

Spec mandates separation: locations contain interaction points; merging them creates unmaintainable mega-nodes. Location service knows graph and actor positions; `IInteractionService` (Presentation bridge) resolves world objects inside a scene.

---

## Implementation summary

| MVP (Phase 6) | Full |
|---|---|
| Area graph + bidirectional edges | Scene hierarchy (district → area → scene) `[FULL]` |
| `CanTravel` / `Travel` / `GetConnectedLocations` | Fast travel with time cost `[FULL]` |
| `GetActorLocation` | Per-scene local state flags `[FULL]` |
| Travel advances time from edge definition | Discovery states (hidden/known/visited) `[FULL]` |
| `ActorMoved` + `LocationEntered` events | Ambient profile hooks `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Locations`
- **References:** Runtime, Content, Simulation.Time (advance on travel)
- **Tick phase:** **Facts** (position updates) · **Events** (movement notifications)

Registered as `ILocationService`. Does not reference Presentation or Unity scene APIs.

---

## Public API

See [contracts.md](contracts.md) — `ILocationService`.

| Method | Behavior |
|---|---|
| `bool CanTravel(from, to)` | Graph edge exists and gate rules pass |
| `void Travel(actorId, toLocationId)` | Validates, advances time, updates position, publishes events |
| `IReadOnlyList<string> GetConnectedLocations(locationId)` | Outbound edges from node |
| `string GetActorLocation(actorId)` | Current location ID for actor |

Player travel uses `actor_player` or companion-accompanied travel rules `[FULL]`.

---

## Internal implementation

### LocationService

```csharp
sealed class LocationService : ILocationService, IStatefulService<LocationServiceState>
{
    readonly LocationGraph _graph = new();
    readonly Dictionary<string, string> _actorPositions = new();
    readonly IContentStore _content;
    readonly ITimeService _time;
    readonly IEventBus _events;
    readonly IGateService _gates;

    public bool CanTravel(string fromLocationId, string toLocationId)
    {
        if (!_graph.HasEdge(fromLocationId, toLocationId)) return false;
        var edge = _graph.GetEdge(fromLocationId, toLocationId);
        var ctx = new GateContext { SourceLocationId = fromLocationId, TargetLocationId = toLocationId };
        return _gates.IsAllowed(edge.GateId, ctx);
    }

    public void Travel(string actorId, string toLocationId)
    {
        var from = GetActorLocation(actorId);
        if (!CanTravel(from, toLocationId))
            throw new InvalidOperationException($"Travel denied: {from} → {toLocationId}");

        var edge = _graph.GetEdge(from, toLocationId);
        _time.Advance(edge.TravelCost);
        _actorPositions[actorId] = toLocationId;

        _events.Publish(new SimEvent {
            Type = EventTypes.ActorMoved,
            Timestamp = _time.Now,
            Payload = { ["actorId"] = actorId, ["from"] = from, ["to"] = toLocationId }
        });
        _events.Publish(new SimEvent {
            Type = EventTypes.LocationEntered,
            Timestamp = _time.Now,
            Payload = { ["actorId"] = actorId, ["locationId"] = toLocationId }
        });
    }

    public IReadOnlyList<string> GetConnectedLocations(string locationId)
        => _graph.GetNeighbors(locationId);

    public string GetActorLocation(string actorId)
        => _actorPositions.TryGetValue(actorId, out var loc) ? loc : GetDefaultSpawn(actorId);
}
```

### LocationGraph (internal)

```csharp
sealed class LocationGraph
{
    readonly Dictionary<string, List<LocationEdge>> _adjacency = new();

    public void LoadFromContent(IReadOnlyList<AreaDefinition> areas) { /* build edges */ }
    public bool HasEdge(string from, string to) => /* directed or bidirectional */;
    public LocationEdge GetEdge(string from, string to);
    public IReadOnlyList<string> GetNeighbors(string locationId);
}

class LocationEdge
{
    public string FromId;
    public string ToId;
    public GameTimeDelta TravelCost;
    public string GateId;           // optional rule gate
}
```

### LocationRegistry (internal)

Indexes `AreaDefinition` / `SceneDefinition` by `location_*` / `area_*` IDs. MVP uses flat area list; `[FULL]` nested hierarchy for scene loading metadata consumed by Exploration bridge only.

---

## Definition assets

```csharp
class AreaDefinition : ContentDefinition
{
    public string DisplayNameKey;
    public string[] ConnectedAreaIds;
    public GameTimeDelta DefaultTravelCost;
    public string AccessGateId;         // optional area-level gate
}

class SceneDefinition : ContentDefinition   // [FULL]
{
    public string AreaId;
    public string UnityScenePath;       // Presentation only — not read by LocationService MVP
    public NarrativeProfileRef Narrative;
}

class ConnectionDefinition : ContentDefinition
{
    public string FromAreaId;
    public string ToAreaId;
    public GameTimeDelta TravelCost;
    public string GateId;
    public bool Bidirectional;
}
```

IDs: `area_*`, `location_*`, `scene_*` per glossary. Graph built at content pipeline build time or lazy on first `CanTravel`.

---

## Runtime state

```csharp
class LocationServiceState
{
    public Dictionary<string, string> ActorPositions;
    public HashSet<string> VisitedAreaIds;      // MVP track for gates
    public Dictionary<string, SceneLocalState> LocalStates;  // [FULL]
}

class SceneLocalState   // [FULL]
{
    public Dictionary<string, string> FlagValues;  // DoorOpen, CorpsePresent, etc.
}
```

Visited set updated on `LocationEntered` for player actor. Serialized in Phase 10 envelope.

---

## Core algorithms

### Graph construction

1. Load all `AreaDefinition` and `ConnectionDefinition` from content store
2. Add nodes for each area ID
3. Add edges; duplicate reverse edge when `Bidirectional`
4. Validate no orphan required story areas in pipeline `Validate()`

### Travel sequence

```text
CanTravel(from, to)?
  → Advance time by edge.TravelCost
  → Set actor position
  → Publish ActorMoved, LocationEntered
  → [FULL] Mark visited, apply scene entry hooks
```

Failure before time advance: no partial state.

### Default spawn resolution

On first `GetActorLocation`, if actor missing from map, use `ActorDefinition.DefaultLocationId` from content via `IActorService` bootstrap or content default `area_*`.

### Gate integration

Edge and area `GateId` passed to `IGateService` with `GateContext` carrying source/target locations, actor ID, time slice. Soft gating preferred (guard dialogue) — implemented as gate denial reason consumed by dialogue `[FULL]`.

---

## Event contracts + tick phase

| Event | Publisher | Subscribers | Phase |
|---|---|---|---|
| `ActorMoved` | ILocationService | IActorService, IInfoFlowService (witness range) | Events |
| `LocationEntered` | ILocationService | IDiscoveryService bridge, chronicle, companion commentary | Events |
| `LocationExited` | `[FULL]` exploration bridge | Ambient, pacing | Events |

Mutations occur synchronously in `Travel` during caller's phase (usually Content or Interpretation); events queued for Events flush.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IContentStore | Area/connection definitions |
| ITimeService | Travel cost |
| IEventBus | Movement notifications |
| IGateService | Access rules (Phase 5+) |

| Used by | Reason |
|---|---|
| IActorService | Location mirror + schedule targets |
| IDialogueService | Location-context dialogue |
| IInfoFlowService | Witness proximity on move |
| ICompanionService | Follow travel `[Phase 7]` |
| IPersistenceService | Positions + visited |
| IExplorationService | `[Phase 13]` Unity scene bind |

**Forbidden:** LocationService → Presentation, UnityEngine.SceneManagement in Simulation asmdef.

---

## MVP scope (Phase 6)

- [ ] `ILocationService` + `LocationService` + `LocationGraph`
- [ ] Load areas/connections from content stubs
- [ ] `CanTravel` / `Travel` / `GetConnectedLocations` / `GetActorLocation`
- [ ] Time advance on travel
- [ ] `ActorMoved`, `LocationEntered` events
- [ ] `LocationServiceState` with positions + visited
- [ ] Tests: graph connectivity, denied travel, time cost, events

---

## Full scope

- `[FULL]` Scene local state mutations (doors, evidence taken)
- `[FULL]` Fast travel unlocked by discovery
- `[FULL]` Global world state affecting multiple areas
- `[FULL]` Faculty-reactive ambient metadata (content only; read by voice/story)
- `[FULL]` Location memory for chronicle projection

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Locations/ILocationService.cs
Packages/NarrativeFramework/Simulation/Locations/LocationService.cs
Packages/NarrativeFramework/Simulation/Locations/LocationGraph.cs
Packages/NarrativeFramework/Simulation/Locations/LocationEdge.cs
Packages/NarrativeFramework/Simulation/Locations/LocationRegistry.cs
Packages/NarrativeFramework/Simulation/Locations/AreaDefinition.cs
Packages/NarrativeFramework/Simulation/Locations/ConnectionDefinition.cs
Packages/NarrativeFramework/Simulation/Locations/LocationServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Locations/LocationGraphTests.cs
Packages/NarrativeFramework/Tests/EditMode/Locations/LocationTravelTests.cs
Packages/NarrativeFramework/Tests/EditMode/Locations/LocationGateTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `GetConnectedLocations_ReturnsNeighbors` | Graph adjacency |
| `CanTravel_NoEdge_ReturnsFalse` | Missing edge |
| `Travel_AdvancesTime` | ITimeService mock received delta |
| `Travel_PublishesActorMoved` | Payload from/to |
| `Travel_DeniedGate_NoStateChange` | Position and time unchanged |
| `GetActorLocation_DefaultSpawn` | First query uses definition default |
| `CaptureRestore_PreservesPositions` | Stateful roundtrip |

Part of Phase 6 ≥18 test budget with time and actor suites.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-07, SIM-04 (bidirectional edges; one hop per tick; scenes via `IExplorationService`).

---

## Bootstrap and registry wiring

`LocationService` registers after `ITimeService` and `IGateService` (Phase 5). Content pipeline `Build()` validates every `ConnectionDefinition.FromAreaId` and `ToAreaId` resolves to loaded `AreaDefinition` — broken graphs fail build, not runtime.

Player bootstrap sets `actor_player` position to content `GameStartAreaId`. NPC positions come from `ActorDefinition.DefaultLocationId` on first query; explicit `Travel` or schedule hooks update `_actorPositions`.

```csharp
// Composition root excerpt (Phase 6 sample scene / test fixture)
registry.Register<ITimeService>(new TimeService(eventBus));
registry.Register<IGateService>(gateService);
registry.Register<ILocationService>(new LocationService(content, time, events, gates));
registry.Register<IActorService>(new ActorService(content, events, registry.GetRequired<ILocationService>()));
```

Exploration bridge (Phase 13) implements `IExplorationService.Enter` by calling `Travel(actor_player, locationId)` then loading Unity scene path from `SceneDefinition` — path never read inside `LocationService`.

---

## Related documents

- [sim-actor.md](sim-actor.md), [sim-time.md](sim-time.md)
- [present-exploration.md](../systems/present-exploration.md) (Phase 13 bridge)
- [rules-gate.md](rules-gate.md), [runtime-kernel.md](runtime-kernel.md)
