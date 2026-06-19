# Presentation Discovery — Implementation Architecture

- Spec: [present-discovery.md](../systems/present-discovery.md)
- Glossary: [DiscoveryService](../terminology-glossary.md)
- Roadmap: [Phase 13](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) — `IDiscoveryService`
- Siblings: [present-interaction.md](present-interaction.md), [present-exploration.md](present-exploration.md)

---

## Design rationale

### Why discovery is revelation, not a stat check

Standard RPG hidden-object rolls treat perception as binary spot/hide. NSF discovery models **layers of meaning**: an object can be visible but not understood, hinted but not examinable, discovered but not exhausted. That state machine belongs at the player-facing boundary where visibility rules meet faculty thresholds — not inside `IFactService`.

### Why facts and discovery are separate

A fact (`fact_blood_on_knife`) is simulation truth. Discovery (`discoverable_bloodstain`) is **what the player has been shown**. Registering a fact before discovery would spoil chronicle pacing; showing a discoverable without a fact would create orphan UI. Discovery service marks reveal; effect runners register facts when narrative demands.

### Why Presentation owns IDiscoveryService

Core modules consume discovery **state** through the interface at runtime via registry, but nothing in Story/Ledition asmdefs references Presentation types. `IDiscoveryService` is declared in contracts and implemented in Presentation — same bridge pattern as interaction.

### Why discovery memory enables return visits

Investigation games reward backtracking with new faculties. `DiscoveryMemory` records source and timestamp so UI can show "new since last visit" and tests can assert layered reveals without replaying entire threads.

---

## Implementation summary

| MVP (Phase 13) | Full (Phase 15+) |
|---|---|
| `IDiscoveryService` state machine | Environmental aura reveals `[FULL]` |
| `DiscoverableDefinition` content schema | Particle/highlight spawn in samples |
| Faculty + context reveal rules | Passive scan on room enter `[FULL]` |
| `MarkDiscovered` → fact effects | Discovery exhaustion copy variants |
| Headless state assertions | Minimap fog-of-war `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Presentation`
- **Namespace:** `NarrativeFramework.Presentation.Discovery`
- **References:** Runtime, Content, Rules, Cognition, Simulation, Story

---

## Public API

See [contracts.md](contracts.md):

```csharp
interface IDiscoveryService
{
    bool IsDiscovered(string discoverableId);
    void MarkDiscovered(string discoverableId, DiscoverySource source);
    IReadOnlyList<string> GetDiscoveredIds();
}
```

Extended internal types (not on public contract):

```csharp
enum DiscoveryState { Hidden, Hinted, Visible, Discovered, Exhausted }
enum DiscoverySource { Interaction, Faculty, Dialogue, Exploration, Script, Debug }

class DiscoverableState
{
    public string Id;
    public DiscoveryState State;
    public DiscoverySource LastSource;
    public int RevealGeneration;  // increments on layer peel
}
```

---

## Internal implementation

### DiscoveryService

```csharp
sealed class DiscoveryService : IDiscoveryService
{
    readonly IContentStore _content;
    readonly INarrativeServiceRegistry _registry;
    readonly Dictionary<string, DiscoverableState> _states = new();
    readonly DiscoveryRevealEngine _revealEngine;

    public bool IsDiscovered(string discoverableId)
    {
        return _states.TryGetValue(discoverableId, out var s)
            && s.State >= DiscoveryState.Discovered;
    }

    public void MarkDiscovered(string discoverableId, DiscoverySource source)
    {
        var def = _content.GetRequired<DiscoverableDefinition>(discoverableId);
        ref var state = ref GetOrCreateState(discoverableId);
        if (state.State >= DiscoveryState.Discovered) return;

        state.State = DiscoveryState.Discovered;
        state.LastSource = source;
        ApplyDiscoveryEffects(def, source);
        _registry.GetRequired<IEventBus>().Publish(new DiscoverableRevealedEvent(discoverableId, source));
    }

    public IReadOnlyList<string> GetDiscoveredIds()
        => _states.Where(kv => kv.Value.State >= DiscoveryState.Discovered)
                  .Select(kv => kv.Key).ToList();

    internal void EvaluateReveals(RevealContext context)
        => _revealEngine.EvaluateAll(_content.GetAll<DiscoverableDefinition>(), _states, context);
}
```

### DiscoveryRevealEngine

Evaluates `RevealCondition` arrays on definitions:

| Condition kind | Evaluator calls |
|---|---|
| Faculty threshold | `IFacultyService`, optional `IRollService` |
| Story flag | `IStoryStateService` |
| Fact present | `IFactService` |
| Location | `IExplorationService.GetCurrentLocationId()` |
| Time window | `ITimeService` |
| Belief phase | `IBeliefService` |

Outcomes:

- `Hidden` → `Hinted`: UI may show environmental text (exploration bridge)
- `Hinted` → `Visible`: object appears in scene proxy / interaction list
- `Visible` → `Discovered`: player acknowledged; facts fire

Engine never renders — updates state and publishes events.

### DiscoveryEffectApplier

```csharp
sealed class DiscoveryEffectApplier
{
    public void Apply(DiscoverableDefinition def, DiscoverySource source)
    {
        foreach (var effect in def.Effects)
        {
            switch (effect.Kind)
            {
                case DiscoveryEffectKind.RegisterFact:
                    _facts.Register(effect.FactId);
                    break;
                case DiscoveryEffectKind.UnlockInteraction:
                    // Interaction nodes keyed by discoverable visibility — see present-interaction
                    break;
                case DiscoveryEffectKind.StartDialogue:
                    _dialogue.StartDialogue(effect.DialogueId);
                    break;
            }
        }
    }
}
```

### Headless discovery probe

```csharp
sealed class DiscoveryTestProbe
{
    readonly IDiscoveryService _discovery;
    readonly DiscoveryService _concrete;  // internal test accessor

    public void RunRevealEvaluation(RevealContext ctx) => _concrete.EvaluateReveals(ctx);
    public DiscoveryState GetState(string id) => _concrete.GetStateForTest(id);
}
```

---

## Definition assets

```csharp
class DiscoverableDefinition : ContentDefinition
{
    public string DisplayNameKey;
    public DiscoveryState InitialState;
    public RevealConditionDefinition[] RevealConditions;
    public DiscoveryEffectDefinition[] Effects;
    public string LinkedWorldObjectId;  // optional bind to interaction
}

class RevealConditionDefinition
{
    public RevealConditionKind Kind;
    public string TargetId;
    public int Threshold;
    public string FacultyId;  // when Kind == FacultyRoll
}
```

`FrameworkTestPack`: `discoverable_test_note` — Hidden until `fact_met_witness` or Perception passive pass.

---

## Runtime state

| State | Persisted | Owner |
|---|---|---|
| Per-discoverable `DiscoveryState` | Yes (Phase 10 envelope) | `DiscoveryService` |
| Reveal generation counters | Yes | `DiscoveryService` |
| Definition catalog | No (content) | `IContentStore` |

Serialization DTO:

```csharp
class DiscoverySaveState
{
    public DiscoverableState[] States;
}
```

---

## Core algorithms

### Layered reveal evaluation

1. Build `RevealContext` from registry (location, time, flags, faculties)
2. For each discoverable not `Exhausted`, evaluate next applicable condition tier
3. Promote state one step maximum per evaluation pass (prevents skip-ahead)
4. On transition to `Discovered`, call `MarkDiscovered` path
5. Notify `IInteractionService` refresh and exploration memory

### Passive faculty scan (MVP: manual trigger)

Phase 13 MVP: tests call `EvaluateReveals` explicitly after faculty change. `[FULL]` hooks on `LocationEntered` event from exploration.

### Discovery vs interaction visibility

| Layer | Service | Question answered |
|---|---|---|
| Discovery | `IDiscoveryService` | Has player **noticed** this detail? |
| Interaction | `IInteractionService` | What can player **do** with this object? |

An object may be `Visible` in discovery but have zero interaction nodes until a dialogue fact unlocks inspect.

---

## Event contracts

| Event | Phase | Payload |
|---|---|---|
| `DiscoverableHinted` | After reveal eval | discoverableId |
| `DiscoverableRevealed` | On MarkDiscovered | discoverableId, source |
| `DiscoverableExhausted` | All layers consumed | discoverableId |

Subscribers: `UIPresentationCoordinator` (notifications), `IChronicleService` (clue entries), Debug trace.

Core chronicle updates through `IChronicleService` — not by reading Presentation internals.

---

## Integration matrix

| Depends on | Usage |
|---|---|
| `IContentStore` | Discoverable defs |
| `IFactService`, `IStoryStateService` | Conditions + effects |
| `IFacultyService`, `IRollService` | Faculty gates |
| `IExplorationService` | Location context |
| `IBeliefService` | Belief-gated reveals |

| Consumed by | Usage |
|---|---|
| `IInteractionService` | Visibility of nodes |
| `FullLoopIntegrationTests` | Clue find simulation |
| Samples | Scene object enable/disable |

**Forbidden:** Ledger → Presentation direct reference; use events + chronicle refresh.

---

## MVP scope (Phase 13)

- [ ] `IDiscoveryService`, `DiscoveryService`
- [ ] `DiscoverableDefinition` + pipeline validation
- [ ] State machine Hidden → Discovered (Exhausted stub)
- [ ] Faculty + fact + flag reveal conditions
- [ ] `DiscoverableRevealedEvent`
- [ ] Headless tests with `FrameworkTestPack` discoverable
- [ ] Persistence envelope stub (empty restore OK)

---

## Full scope

- `[FULL]` Environmental non-object reveals (aura text)
- `[FULL]` Automatic reveal on location enter
- `[FULL]` Exhausted state + repeat inspect copy
- `[FULL]` Discovery pacing integration with `IPacingService`

---

## File tree

```text
Packages/NarrativeFramework/Presentation/Discovery/IDiscoveryService.cs
Packages/NarrativeFramework/Presentation/Discovery/DiscoveryService.cs
Packages/NarrativeFramework/Presentation/Discovery/DiscoveryRevealEngine.cs
Packages/NarrativeFramework/Presentation/Discovery/DiscoveryEffectApplier.cs
Packages/NarrativeFramework/Presentation/Discovery/DiscoverySaveState.cs
Packages/NarrativeFramework/Content/Definitions/DiscoverableDefinition.cs
Packages/NarrativeFramework/Simulation/Events/DiscoverableRevealedEvent.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/DiscoveryServiceTests.cs
Samples~/FrameworkTestPack/Discovery/discoverable_test_note.asset
```

---

## Test plan

| Test | Asserts |
|---|---|
| `MarkDiscovered_Idempotent` | Second call no duplicate facts |
| `FacultyReveal_HiddenToVisible` | State transition |
| `FactCondition_BlocksUntilRegistered` | Stays Hidden |
| `GetDiscoveredIds_FiltersPartialStates` | Only Discovered+ |
| `Effect_RegistersFact` | Fact in registry |
| `InteractionRefresh_AfterReveal` | New option visible |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) PR-02 (global registry, per-location visibility).

---

## Related documents

- [present-exploration.md](present-exploration.md) — location enter triggers
- [present-interaction.md](present-interaction.md) — post-discovery affordances
- [sim-fact.md](sim-fact.md) — fact registration
- [integration.md](integration.md) — discovery in closed loop
