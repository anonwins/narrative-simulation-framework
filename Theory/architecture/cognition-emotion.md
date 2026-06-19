# Cognition Emotion — Implementation Architecture

- Spec: [systems/cognition-emotion.md](../systems/cognition-emotion.md)
- Glossary: [Emotion axes](../terminology-glossary.md)
- Roadmap: [Phase 8](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [cognition-faculty.md](cognition-faculty.md) · [cognition-roll.md](cognition-roll.md) · [cognition-belief.md](cognition-belief.md)

---

## Design rationale

### Why emotion is stateful mind simulation

Emotion is not narration text — it is **continuous internal state** (intensity, stress, confidence) that modifies rolls, unlocks beliefs, and drives presentation audio/UI. Characters react like people because emotion persists and decays across beats.

### Why float axes vs integer faculties

Facilities are build identity; emotions are volatile psychological signals. Float 0–1 (or unbounded stress) supports gradual decay and threshold rules (`stress > 0.9 → panic dialogue`) without polluting faculty caps.

### Why GetRollModifier bridges to IRollService

Roll spec modifier stack includes “Current Mood / Damage State.” `IEmotionService.GetRollModifier(facultyId)` returns aggregated penalty/bonus — e.g. high stress → logic penalty, high confidence → authority bonus — keeping roll math centralized in one query.

### Why TickEmotionDecay in service contract

Decay is simulation logic, not presentation. Kernel calls `TickEmotionDecay` during **Interpretation** so passive checks and dialogue in the same tick see post-decay values after time advances.

### Why separate from Belief

Beliefs are discrete phased thoughts; emotions are continuous axes. Emotion can **unlock** beliefs (threshold rules); beliefs can modify emotion `[FULL]` — integration via events, not merged services.

---

## Implementation summary

| MVP (Phase 8) | Full |
|---|---|
| `IEmotionService` full contract | Per-character emotion (NPC) |
| `EmotionState` intensity/stress/confidence | Personality profiles from content |
| `ApplyModifier` + `GetRollModifier` | Emotional memory traces |
| `TickEmotionDecay` | Rule engine emotion conditions |
| Player-only default emotion set | Audio/presentation mapping (Phase 13) |
| Tests: decay, roll penalty | Belief unlock on shame threshold |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Cognition` |
| **References** | Runtime, Content, Rules |
| **Namespace** | `NarrativeFramework.Cognition.Emotion` |
| **Tick phase** | **Interpretation** — decay + threshold checks |

---

## Public API

See [contracts.md](contracts.md) — `IEmotionService`.

**Module notes:**

- `GetState(emotionId)` — returns snapshot struct; missing ID returns zeroed state.
- `ApplyModifier(emotionId, delta)` — adds to intensity/stress/confidence fields per emotion definition mapping.
- `GetRollModifier(facultyId)` — aggregated float cast to int for roll stack.
- `TickEmotionDecay()` — called once per kernel tick Interpretation.

---

## Internal implementation

### EmotionService

```csharp
namespace NarrativeFramework.Cognition.Emotion
{
    sealed class EmotionService : IEmotionService, IStatefulService<EmotionServiceState>
    {
        readonly EmotionRegistry _registry = new();
        readonly EmotionRollModifierTable _rollTable;
        readonly IContentStore _content;

        public EmotionState GetState(string emotionId)
            => _registry.GetOrDefault(emotionId);

        public void ApplyModifier(string emotionId, float delta)
        {
            _registry.Add(emotionId, delta);
            Publish(EmotionEvents.Changed, emotionId);
            CheckThresholds(emotionId);
        }

        public float GetRollModifier(string facultyId)
        {
            var stress = _registry.GetAggregateStress();
            var confidence = _registry.GetAggregateConfidence();
            return _rollTable.Compute(facultyId, stress, confidence);
        }

        public void TickEmotionDecay()
        {
            foreach (var def in _content.GetAllIds<EmotionDefinition>())
            {
                var d = _content.GetDefinition<EmotionDefinition>(def);
                _registry.Decay(def, d.DecayRate);
            }
            Publish(EmotionEvents.DecayTick, null);
        }

        void CheckThresholds(string emotionId)
        {
            var state = GetState(emotionId);
            if (state.Stress > 0.9f)
                Publish(EmotionEvents.StressPanicThreshold, emotionId);
            // belief unlock hooks `[FULL]` via IBeliefService
        }
    }
}
```

### EmotionRegistry (internal)

```csharp
sealed class EmotionRegistry
{
    readonly Dictionary<string, EmotionState> _states = new();

    public void Add(string emotionId, float delta)
    {
        var s = GetOrDefault(emotionId);
        s.Intensity = Clamp01(s.Intensity + delta);
        _states[emotionId] = s;
    }

    public void Decay(string emotionId, float rate)
    {
        if (!_states.TryGetValue(emotionId, out var s)) return;
        s.Stress = ApproachBaseline(s.Stress, rate);
        s.Confidence = ApproachBaseline(s.Confidence, rate);
        s.Intensity = ApproachBaseline(s.Intensity, rate);
    }
}
```

### EmotionState

```csharp
class EmotionState
{
    public string EmotionId;
    public float Intensity;
    public float Stress;
    public float Confidence;
}
```

### EmotionRollModifierTable

```csharp
sealed class EmotionRollModifierTable
{
    public float Compute(string facultyId, float stress, float confidence)
    {
        float mod = 0f;
        if (facultyId.Contains("logic") && stress > 0.5f)
            mod -= (stress - 0.5f) * 2f;
        if (facultyId.Contains("authority") && confidence > 0.5f)
            mod += (confidence - 0.5f) * 2f;
        return mod;
    }
}
```

MVP: simplified mapping from spec examples; `[FULL]` content-driven `EmotionDefinition.RollModifiers`.

### EmotionDefinition `[Phase 8 content optional]`

```csharp
class EmotionDefinition : ContentDefinition
{
    public float DecayRate;
    public FacultyRollModifier[] RollModifiers;
}
```

---

## Definition assets

| Asset | ID prefix | Purpose |
|---|---|---|
| `EmotionDefinition` | `emotion_` | Decay rates, roll modifier rules `[FULL]` |

MVP may use hardcoded player emotion axes (`emotion_stress`, `emotion_confidence`) without full definition assets until content pack pass.

---

## Runtime state / persistence

```csharp
class EmotionServiceState
{
    public List<EmotionState> States;
}
```

Restore preserves float values; one decay tick may run after load `[FULL]` policy.

---

## Core algorithms

### ApplyModifier

1. Load or create `EmotionState` for ID.
2. Apply delta to Intensity (MVP primary axis); specialized mappings for stress/confidence via content `[FULL]`.
3. Clamp 0–1 where applicable.
4. Evaluate threshold side effects.

### TickEmotionDecay

1. For each tracked emotion, reduce toward baseline (0.5 neutral or 0 calm — tunable).
2. `newValue = Lerp(current, baseline, decayRate)`.
3. Called once per full kernel tick in Interpretation **after** Events time advance.

### GetRollModifier

1. Aggregate stress/confidence across relevant emotions or read global axes.
2. Apply faculty-specific table.
3. `IRollService` casts to int: `(int)Math.Round(mod)`.

Example from spec:

```text
Logic check: base 6, stress 0.8 → -1 modifier
```

---

## Event contracts + kernel tick phase

| Event | When | Phase |
|---|---|---|
| `EmotionChanged` | ApplyModifier | Events |
| `EmotionDecayTick` | TickEmotionDecay | Interpretation |
| `StressPanicThreshold` | stress > 0.9 | Interpretation |
| `EmotionBeliefUnlock` | threshold `[FULL]` | Interpretation |

**Kernel registration:** SimulationKernel Interpretation phase invokes `_emotion.TickEmotionDecay()` after time service update.

Subscribers: `IBeliefService`, `IGateService`, `IAudioNarrativeService` (Phase 13), debug trace.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | EmotionDefinition `[FULL]` |
| `ITimeService` | Decay scaling per hour `[FULL]` |

| Used by | Reason |
|---|---|
| `IRollService` | GetRollModifier in stack |
| `IBeliefService` | Threshold unlocks |
| `IGateService` | Emotion conditions |
| `IDialogueService` | Mood-gated lines |
| `IVoiceService` / `IAudioNarrativeService` | Tone selection Phase 13 |

---

## MVP scope checklist (Phase 8)

- [ ] `IEmotionService` implementation
- [ ] EmotionState get/apply
- [ ] GetRollModifier for logic/authority faculty IDs
- [ ] TickEmotionDecay toward baseline
- [ ] EmotionServiceState persistence stub
- [ ] Kernel hook for decay tick
- [ ] Tests: stress reduces logic roll, decay lowers stress
- [ ] Integration: roll with high stress fails more often (seeded)

---

## Full scope

- `[FULL]` NPC per-actor emotion maps
- `[FULL]` Personality profiles (Kim low anger gain, etc.)
- `[FULL]` Emotional memory traces
- `[FULL]` Presentation audio filters by emotion
- `[FULL]` Rule engine `IF stress > 0.7` conditions
- `[FULL]` Misinterpretation under stress (info flow)

---

## File tree

```text
Packages/NarrativeFramework/Cognition/Emotion/IEmotionService.cs
Packages/NarrativeFramework/Cognition/Emotion/EmotionService.cs
Packages/NarrativeFramework/Cognition/Emotion/EmotionRegistry.cs
Packages/NarrativeFramework/Cognition/Emotion/EmotionState.cs
Packages/NarrativeFramework/Cognition/Emotion/EmotionServiceState.cs
Packages/NarrativeFramework/Cognition/Emotion/EmotionRollModifierTable.cs
Packages/NarrativeFramework/Cognition/Emotion/EmotionEvents.cs
Packages/NarrativeFramework/Content/Definitions/EmotionDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Emotion/EmotionApplyTests.cs
Packages/NarrativeFramework/Tests/EditMode/Emotion/EmotionDecayTests.cs
Packages/NarrativeFramework/Tests/EditMode/Emotion/EmotionRollModifierTests.cs
Packages/NarrativeFramework/Tests/EditMode/Emotion/EmotionRollIntegrationTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `ApplyModifier_IncreasesIntensity` | GetState |
| `TickEmotionDecay_ReducesStress` | Lower after tick |
| `GetRollModifier_HighStress_NegativeLogic` | <= -1 |
| `GetRollModifier_HighConfidence_BonusAuthority` | >= +1 |
| `RollIntegration_StressAffectsOutcome` | With fixed seed |
| `RestoreState_PreservesFloats` | Persistence |

**Exit criteria:** Emotion roll penalty test in Phase 8 roadmap.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) emotion baselines (0 intensity / 0.5 stress); per-tick decay; game-time scaling when `ITimeService` active.

---

## Related documents

- [cognition-roll.md](cognition-roll.md) — modifier consumer
- [cognition-faculty.md](cognition-faculty.md) — distinct from faculties
- [cognition-belief.md](cognition-belief.md) — unlock integration
- [runtime-kernel.md](runtime-kernel.md) — Interpretation decay hook
