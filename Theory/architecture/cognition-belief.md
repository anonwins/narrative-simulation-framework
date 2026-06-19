# Cognition Belief — Implementation Architecture

- Spec: [systems/cognition-belief.md](../systems/cognition-belief.md)
- Glossary: [Belief phases](../terminology-glossary.md)
- Roadmap: [Phase 8](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [cognition-faculty.md](cognition-faculty.md) · [present-ui.md](../systems/present-ui.md) (BeliefView Phase 13)

---

## Design rationale

### Why beliefs are a psychological operating layer

Beliefs are not perk trees or collectible buffs. They represent thoughts the character is **processing** — discovered, assimilating with side effects, resolved into permanent identity shifts, or forgotten. Phase machine (`BeliefPhase`) makes internalization a first-class simulation concern.

### Why four phases

| Phase | Meaning |
|---|---|
| **Discovered** | Known but not internalized |
| **Assimilating** | Timer running; temporary faculty effects |
| **Resolved** | Fully integrated; stable modifiers/unlocks |
| **Forgotten** | Removed from active mind; slot freed |

This matches spec internalization timer and completion events — measured in **game time**, not real time.

### Why IBeliefService separate from Story flags

Story flags drive plot progression (`IStoryStateService`). Beliefs drive **character psychology** and faculty modifiers. Conflating them breaks save semantics and confuses pipeline validation (`belief_*` vs `flag_*`).

### Why slot capacity matters

Limited belief slots create meaningful build choices. MVP tracks active beliefs list; `[FULL]` enforces max slot count from content.

### Why assimilating applies temporary faculty modifiers

Spec: beliefs produce side effects during internalization (e.g. -1 Logic, +1 Electrochemistry). `IBeliefService` calls `IFacultyService.ApplyModifier` on `BeginAssimilating` and reverses on `Resolve`/`Forget`.

---

## Implementation summary

| MVP (Phase 8) | Full |
|---|---|
| Full `IBeliefService` phase transitions | Slot capacity enforcement |
| `BeliefDefinition` + `BeliefState` | Internalization timer via `ITimeService` |
| Discover → Assimilating → Resolved → Forget | Thought cabinet UI (Phase 13) |
| Temporary modifiers during Assimilating | Belief conflict rules `[FULL]` |
| `GetActiveBeliefIds` | Emotion-triggered belief unlock |
| Tests: phase transitions | Chronicle projection entries |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Cognition` |
| **References** | Runtime, Content, Rules, Simulation (time Phase 6+) |
| **Namespace** | `NarrativeFramework.Cognition.Belief` |
| **Tick phase** | **Interpretation** (timer advance) · **Events** (phase change notifications) |

---

## Public API

See [contracts.md](contracts.md) — `IBeliefService`.

**Module notes:**

- `Discover(beliefId)` — sets Discovered if not known; idempotent if already active.
- `BeginAssimilating(beliefId)` — Discovered → Assimilating; applies temp modifiers.
- `Resolve(beliefId)` — Assimilating → Resolved; removes temp mods, applies permanent `[FULL]`.
- `Forget(beliefId)` — any active → Forgotten; clears modifiers.
- `GetActiveBeliefIds()` — excludes Forgotten.

---

## Internal implementation

### BeliefService

```csharp
namespace NarrativeFramework.Cognition.Belief
{
    sealed class BeliefService : IBeliefService, IStatefulService<BeliefServiceState>
    {
        readonly IContentStore _content;
        readonly IFacultyService _faculty;
        readonly ITimeService _time;  // Phase 6+; stub for Phase 8 unit tests
        readonly BeliefRegistry _registry = new();

        public BeliefPhase GetPhase(string beliefId)
            => _registry.GetPhase(beliefId);

        public void Discover(string beliefId)
        {
            ValidateBelief(beliefId);
            _registry.SetPhase(beliefId, BeliefPhase.Discovered);
            Publish(BeliefEvents.Discovered, beliefId);
        }

        public void BeginAssimilating(string beliefId)
        {
            EnsurePhase(beliefId, BeliefPhase.Discovered);
            _registry.SetPhase(beliefId, BeliefPhase.Assimilating);
            ApplyTemporaryEffects(beliefId, sign: +1);
            _registry.StartTimer(beliefId, _time.Now);
            Publish(BeliefEvents.AssimilatingStarted, beliefId);
        }

        public void Resolve(string beliefId)
        {
            EnsurePhase(beliefId, BeliefPhase.Assimilating);
            ApplyTemporaryEffects(beliefId, sign: -1);
            _registry.SetPhase(beliefId, BeliefPhase.Resolved);
            Publish(BeliefEvents.Resolved, beliefId);
        }

        public void Forget(string beliefId)
        {
            var phase = GetPhase(beliefId);
            if (phase == BeliefPhase.Assimilating)
                ApplyTemporaryEffects(beliefId, sign: -1);
            _registry.SetPhase(beliefId, BeliefPhase.Forgotten);
            Publish(BeliefEvents.Forgotten, beliefId);
        }

        public IReadOnlyList<string> GetActiveBeliefIds()
            => _registry.GetAllInPhases(
                BeliefPhase.Discovered,
                BeliefPhase.Assimilating,
                BeliefPhase.Resolved);

        public void TickAssimilation()  // called from kernel Interpretation
        {
            foreach (var id in _registry.GetAllInPhase(BeliefPhase.Assimilating))
            {
                var def = _content.GetDefinition<BeliefDefinition>(id);
                if (_registry.IsTimerComplete(id, def.InternalizationHours, _time.Now))
                    Resolve(id);
            }
        }
    }
}
```

### BeliefRegistry (internal)

```csharp
sealed class BeliefRegistry
{
    readonly Dictionary<string, BeliefState> _states = new();

    public void SetPhase(string beliefId, BeliefPhase phase)
    {
        if (!_states.TryGetValue(beliefId, out var s))
            _states[beliefId] = s = new BeliefState { BeliefId = beliefId };
        s.Phase = phase;
    }
}
```

### BeliefDefinition

```csharp
class BeliefDefinition : ContentDefinition
{
    [SerializeField] string titleKey;
    [SerializeField] BeliefPhase initialPhase;
    [SerializeField] int internalizationHours;
    [SerializeField] FacultyModifier[] temporaryEffects;
    [SerializeField] FacultyModifier[] resolvedEffects;

    public string TitleKey => titleKey;
    public BeliefPhase InitialPhase => initialPhase;
    public int InternalizationHours => internalizationHours;
}
```

### Domain types

```csharp
enum BeliefPhase { Discovered, Assimilating, Resolved, Forgotten }

class BeliefState
{
    public string BeliefId;
    public BeliefPhase Phase;
    public float AssimilationProgress;
    public GameTime AssimilationStartedAt;
}
```

---

## Definition assets

| Asset | ID prefix | Key fields |
|---|---|---|
| `BeliefDefinition` | `belief_` | TitleKey, InternalizationHours, temp/resolved effects |

Title resolves via `ILocaleService` at presentation time.

---

## Runtime state / persistence

```csharp
class BeliefServiceState
{
    public List<BeliefState> Beliefs;
}
```

Restore replays modifier application for Assimilating beliefs to keep faculty consistent.

---

## Core algorithms

### Phase transition guards

Illegal transitions throw `InvalidBeliefTransitionException` in tests; log + ignore in release `[FULL]`.

Allowed:

```text
(none) → Discovered (Discover)
Discovered → Assimilating (BeginAssimilating)
Assimilating → Resolved (Resolve or timer)
* → Forgotten (Forget)
```

### Timer completion

1. On each Interpretation tick, `TickAssimilation()`.
2. `elapsed = Now - AssimilationStartedAt`.
3. If `elapsed >= InternalizationHours` → auto `Resolve`.

MVP tests: manually advance `ITimeService` mock.

### Temporary effects

For each `temporaryEffects` entry while Assimilating:

```csharp
_faculty.ApplyModifier(effect.FacultyId, sign * effect.Delta, ModifierSource.Belief);
```

---

## Event contracts + kernel tick phase

| Event | When | Phase |
|---|---|---|
| `BeliefDiscovered` | Discover | Events |
| `BeliefAssimilatingStarted` | BeginAssimilating | Events |
| `BeliefResolved` | Resolve / timer | Events |
| `BeliefForgotten` | Forget | Events |

**Kernel:** `TickAssimilation` in **Interpretation** after time advanced in **Events** (same or next tick per test doc).

Subscribers: `IBeliefView` (Phase 13), `IChronicleService` projection `[FULL]`, `IGateService` belief conditions.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | BeliefDefinition |
| `IFacultyService` | Temp/permanent modifiers |
| `ITimeService` | Internalization timer |
| `ILocaleService` | Title keys (Phase 12) |

| Used by | Reason |
|---|---|
| `IRollService` | Indirect via faculty mods |
| `IGateService` | Belief phase conditions |
| `IDialogueService` | Discover/resolve effects |
| `IEmotionService` | Emotion unlocks beliefs `[FULL]` |
| `IOutcomeService` | Ending synthesis (Phase 11) |

**Standing vs Conduct:** Faction standing is social; beliefs are cognition — glossary boundary.

---

## MVP scope checklist (Phase 8)

- [ ] `BeliefDefinition` in content pack
- [ ] `IBeliefService` all methods
- [ ] Phase guards + GetActiveBeliefIds
- [ ] Temporary modifiers on Assimilating
- [ ] Timer auto-resolve with ITimeService
- [ ] Belief events published
- [ ] `BeliefServiceState` persistence stub
- [ ] Tests: full phase cycle, forget clears mods

---

## Full scope

- `[FULL]` Slot capacity + slot unlock progression
- `[FULL]` Belief conflict (mutually exclusive beliefs)
- `[FULL]` Emotion threshold auto-discover
- `[FULL]` Resolved permanent effects distinct from temp
- `[FULL]` Chronicle “Thought Cabinet” entries

---

## File tree

```text
Packages/NarrativeFramework/Cognition/Belief/IBeliefService.cs
Packages/NarrativeFramework/Cognition/Belief/BeliefService.cs
Packages/NarrativeFramework/Cognition/Belief/BeliefRegistry.cs
Packages/NarrativeFramework/Cognition/Belief/BeliefPhase.cs
Packages/NarrativeFramework/Cognition/Belief/BeliefState.cs
Packages/NarrativeFramework/Cognition/Belief/BeliefServiceState.cs
Packages/NarrativeFramework/Cognition/Belief/BeliefEvents.cs
Packages/NarrativeFramework/Content/Definitions/BeliefDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Belief/BeliefPhaseTransitionTests.cs
Packages/NarrativeFramework/Tests/EditMode/Belief/BeliefAssimilationTimerTests.cs
Packages/NarrativeFramework/Tests/EditMode/Belief/BeliefFacultyModifierTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Discover_SetsDiscoveredPhase` | GetPhase |
| `BeginAssimilating_AppliesTempModifiers` | Faculty delta |
| `Resolve_RemovesTempModifiers` | Back to baseline |
| `Forget_FromResolved` | Phase Forgotten |
| `Timer_AutoResolves` | After time advance |
| `GetActiveBeliefIds_ExcludesForgotten` | Count |
| `IllegalTransition_Throws` | Invalid path |

**Exit criteria:** Phase 8 ≥25 tests combined with conduct/emotion.

---

## Deferred decisions

**Resolved** — [decisions-log.md](../decisions-log.md) C-05: **no hard belief slot cap**.

---

## Related documents

- [cognition-faculty.md](cognition-faculty.md) — modifiers
- [cognition-emotion.md](cognition-emotion.md) — unlock hooks
- [content-store.md](content-store.md) — BeliefDefinition
- [story-state.md](story-state.md) — flag boundary
