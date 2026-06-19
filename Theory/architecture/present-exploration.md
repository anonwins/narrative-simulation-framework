# Presentation Exploration — Implementation Architecture

- Spec: [present-exploration.md](../systems/present-exploration.md)
- Glossary: [ExplorationService](../terminology-glossary.md)
- Roadmap: [Phase 13](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) — `IExplorationService`
- Related: [sim-location.md](sim-location.md), [present-discovery.md](present-discovery.md)

---

## Design rationale

### Why exploration bridges Simulation location truth and player navigation

`ILocationService` (Simulation) tracks where actors are in the world graph — data for rules, AI, and time schedules. **Exploration** adds player-centric concerns: which areas appear on the map, travel transitions, visit memory, and pacing of area unlocks. Mixing map UI state into Simulation would force every location test to carry presentation flags.

### Why point-and-click is the default contract

NSF investigation games prioritize observation over locomotion skill. `IExplorationService.Enter` / `Exit` express **authorial area transitions** independent of whether the sample uses NavMesh, teleport, or 2D hotspot graphs. Core simulation only needs `GetCurrentLocationId()` for gates.

### Why area unlocking is not the same as gating

`IGateService` decides legality of content. Exploration decides **what the player knows exists**. A gated-unlocked alley and a discovered alley on the map are related but separable: player may learn of an alley through dialogue before the gate allows entry.

### Why core must not reference Presentation

Story scripts call `ILocationService` for NPC placement. They must never call exploration map code. Presentation exploration service **reads** location state and **writes** visit/unlock memory plus player enter/exit intents that delegate to Simulation.

---

## Implementation summary

| MVP (Phase 13) | Full (Phase 15+) |
|---|---|
| `IExplorationService` enter/exit/can-enter | NavMesh movement controller in samples |
| Location graph from content | Travel cinematics `[FULL]` |
| `ExplorationState` visit/unlock lists | Fog-of-war rendering |
| Enter → discovery eval hook | Companion follow paths `[FULL]` |
| Headless location walk tests | Click-to-walk Play Mode |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Presentation`
- **Namespace:** `NarrativeFramework.Presentation.Exploration`
- **References:** Runtime, Content, Simulation, Rules, Story

Simulation provides `ILocationService`; Presentation wraps player navigation policy.

---

## Public API

See [contracts.md](contracts.md):

```csharp
interface IExplorationService
{
    bool CanEnter(string locationId);
    void Enter(string locationId);
    void Exit(string locationId);
    string GetCurrentLocationId();
}
```

Presentation extensions (internal, for samples):

```csharp
interface IExplorationMapView
{
    void ShowUnlockedAreas(IReadOnlyList<string> locationIds);
    void HighlightCurrentLocation(string locationId);
}
```

Map view is sample Canvas — not core contract.

---

## Internal implementation

### ExplorationService

```csharp
sealed class ExplorationService : IExplorationService
{
    readonly INarrativeServiceRegistry _registry;
    readonly ExplorationState _state = new();
    readonly IContentStore _content;

    public string GetCurrentLocationId()
        => _registry.GetRequired<ILocationService>().GetActorLocation("actor_player");

    public bool CanEnter(string locationId)
    {
        if (!_state.UnlockedAreas.Contains(locationId)) return false;
        var gate = _registry.GetRequired<IGateService>();
        return gate.EvaluateLocationEntry(locationId);
    }

    public void Enter(string locationId)
    {
        if (!CanEnter(locationId))
            throw new InvalidOperationException($"Cannot enter {locationId}");

        var location = _registry.GetRequired<ILocationService>();
        var previous = GetCurrentLocationId();
        location.SetActorLocation("actor_player", locationId);

        _state.VisitedLocations.Add(locationId);
        _registry.GetRequired<IEventBus>().Publish(new LocationEnteredEvent(locationId, previous));

        TriggerDiscoveryEvaluation(locationId);
        TriggerExplorationEffects(locationId);
    }

    public void Exit(string locationId)
    {
        var current = GetCurrentLocationId();
        if (current != locationId) return;
        _registry.GetRequired<IEventBus>().Publish(new LocationExitedEvent(locationId));
        // Exit typically transitions to hub — content-driven default exit target
    }

    void TriggerDiscoveryEvaluation(string locationId)
    {
        var discovery = _registry.GetRequired<IDiscoveryService>();
        var ctx = RevealContext.ForLocation(locationId, _registry);
        ((DiscoveryService)discovery).EvaluateReveals(ctx);  // internal test seam; prefer public Evaluate API
    }
}
```

### ExplorationState

```csharp
class ExplorationState
{
    public HashSet<string> VisitedLocations { get; } = new();
    public HashSet<string> UnlockedAreas { get; } = new();
    public HashSet<string> TriggeredExplorationEvents { get; } = new();
}
```

Initial unlock set seeded from content `LocationGraphDefinition.StartingLocations`.

### AreaUnlockController

Listens for story/discovery events:

```csharp
sealed class AreaUnlockController
{
    public void OnStoryFlagChanged(string flagId)
    {
        foreach (var rule in _content.GetAll<AreaUnlockRuleDefinition>())
            if (rule.RequiredFlagId == flagId)
                _exploration.UnlockArea(rule.LocationId);
    }
}
```

Unlock updates `ExplorationState.UnlockedAreas` and publishes `AreaUnlockedEvent`.

### ExplorationEffectRunner

Location enter scripts: start ambient audio, fire one-shot dialogue barks, advance time — via `IAudioNarrativeService`, `ITimeService`, without embedding in Simulation.

### Headless exploration driver

Integration tests simulate travel without geometry:

```csharp
sealed class ExplorationTestDriver
{
    public void WalkTo(IExplorationService exploration, string locationId)
    {
        if (!exploration.CanEnter(locationId))
            throw new InvalidOperationException("Blocked");
        exploration.Enter(locationId);
    }
}
```

---

## Definition assets

```csharp
class LocationGraphDefinition : ContentDefinition
{
    public string[] StartingLocations;
    public LocationNodeDefinition[] Nodes;
}

class LocationNodeDefinition
{
    public string LocationId;
    public string DisplayNameKey;
    public string[] ConnectedLocationIds;
    public string[] RequiredUnlockFlags;
}

class AreaUnlockRuleDefinition : ContentDefinition
{
    public string LocationId;
    public string RequiredFlagId;
    public UnlockSource Source;  // Dialogue, Thread, Discovery
}
```

`FrameworkTestPack`: two-node graph `location_test_office` ↔ `location_test_alley` with alley locked until flag.

---

## Runtime state

| State | Owner | Persisted |
|---|---|---|
| Current location (truth) | `ILocationService` | Yes |
| Visited / unlocked sets | `ExplorationState` | Yes |
| Graph definitions | Content | No |

Save envelope: `ExplorationSaveState { string[] Visited, string[] Unlocked, string[] TriggeredEvents }`.

---

## Core algorithms

### Enter location

1. Validate unlock + gate
2. Update `ILocationService` actor location
3. Record visit
4. Publish `LocationEnteredEvent`
5. Run discovery reveal evaluation for new location context
6. Apply location enter effects (audio, time, pacing)
7. Refresh exploration map view if bound in sample

### Unlock area

1. Match unlock rule to story flag / thread milestone / discovery
2. Add locationId to `UnlockedAreas`
3. Publish `AreaUnlockedEvent` — chronicle may add lead
4. Do **not** auto-enter; player chooses travel

### Can enter vs is unlocked

| Check | Service | Failure reason |
|---|---|---|
| Player knows area exists | `ExplorationState.UnlockedAreas` | Hidden on map |
| Narrative allows entry | `IGateService` | Blocked by story |
| Physical connection | Location graph `[FULL]` | Disconnected node |

MVP: unlock + gate sufficient; graph connectivity optional.

---

## Event contracts

| Event | Publisher | Subscribers |
|---|---|---|
| `LocationEnteredEvent` | `ExplorationService` | Discovery, audio, debug |
| `LocationExitedEvent` | `ExplorationService` | Ambient stop |
| `AreaUnlockedEvent` | `AreaUnlockController` | Map view, chronicle |

Events defined in Simulation/Events; payloads are IDs only — no Presentation types.

---

## Integration matrix

| Depends on | Usage |
|---|---|
| `ILocationService` | Authoritative position |
| `IGateService` | Entry legality |
| `IStoryStateService` | Unlock flags |
| `IDiscoveryService` | Reveal on enter |
| `ITimeService` | Travel cost `[FULL]` |
| `IAudioNarrativeService` | Ambient on enter |

| Consumed by | Usage |
|---|---|
| Sample movement controllers | Click hotspot → Enter |
| `FullLoopIntegrationTests` | Travel between test locations |
| Debug | Location timeline |

**Forbidden:** Simulation → Presentation; Location service does not call Exploration.

---

## MVP scope (Phase 13)

- [ ] `IExplorationService`, `ExplorationService`
- [ ] `ExplorationState` + save envelope stub
- [ ] `LocationGraphDefinition` in FrameworkTestPack
- [ ] `AreaUnlockController` flag-driven unlock
- [ ] `LocationEnteredEvent` → discovery eval hook
- [ ] Headless walk tests across two locations
- [ ] `CanEnter` respects gate + unlock

---

## Full scope

- `[FULL]` NavMesh `MovementController` in samples
- `[FULL]` Travel time advancement
- `[FULL]` Dynamic blockers (actor crowds)
- `[FULL]` Exploration pacing caps (`IPacingService`)

---

## File tree

```text
Packages/NarrativeFramework/Presentation/Exploration/IExplorationService.cs
Packages/NarrativeFramework/Presentation/Exploration/ExplorationService.cs
Packages/NarrativeFramework/Presentation/Exploration/ExplorationState.cs
Packages/NarrativeFramework/Presentation/Exploration/AreaUnlockController.cs
Packages/NarrativeFramework/Presentation/Exploration/ExplorationEffectRunner.cs
Packages/NarrativeFramework/Content/Definitions/LocationGraphDefinition.cs
Packages/NarrativeFramework/Content/Definitions/AreaUnlockRuleDefinition.cs
Packages/NarrativeFramework/Simulation/Events/LocationEnteredEvent.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/ExplorationServiceTests.cs
Samples~/FrameworkTestPack/Locations/location_graph_test.asset
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Enter_UpdatesLocationService` | Player location ID |
| `CanEnter_FalseWhenLocked` | Gate/unlock |
| `UnlockArea_AddsToUnlockedSet` | State contains ID |
| `Enter_TriggersDiscoveryEval` | Discoverable state change |
| `VisitRecorded_OnFirstEnter` | Visited set |
| `LocationEnteredEvent_Published` | Event bus capture |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) PR-01, PR-02.

---

## Related documents

- [sim-location.md](sim-location.md) — simulation location graph
- [present-discovery.md](present-discovery.md) — enter-triggered reveals
- [present-audio.md](present-audio.md) — ambient on location
- [integration.md](integration.md) — travel in investigation loop
