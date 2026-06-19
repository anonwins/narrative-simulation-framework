# Cognition Faculty — Implementation Architecture

- Spec: [systems/cognition-faculty.md](../systems/cognition-faculty.md)
- Glossary: [Faculty IDs](../terminology-glossary.md) · [Vitality / Morale](../terminology-glossary.md)
- Roadmap: [Phase 3](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [cognition-roll.md](cognition-roll.md) · [content-store.md](content-store.md)

---

## Design rationale

### Why faculties are not flat RPG stats

Faculties are **interpretive agents** — they speak in dialogue (Phase 13 interjections), gate passive perception, and define character identity. A flat integer stat system cannot express NSF-style internal voices, group caps, or narrative discovery tied to faculty thresholds.

### Why base vs modified values

`GetValue` returns authored base; `GetModifiedValue` applies equipment, emotion, belief, conduct, and script modifiers with auditable `ModifierSource`. Saves persist base + modifier ledger or recomputed cache — MVP stores base in state, modifiers rebuilt on equip/emotion tick.

### Why Vitality and Morale pools

Roadmap Phase 3 deliverables require **Vitality / Morale** as first-class pools on `IFacultyService`. They constrain repeated rolls, health-like narrative pressure, and morale-driven dialogue — distinct from individual faculty values but queried through the same service for roll gating.

### Why faculty groups

Groups (`group_physical`, `group_intellect`, …) enable passive checks against cluster totals, UI grouping, and content pack queries via `GetFacultyIdsInGroup`. Definitions carry `GroupId`; runtime aggregates member modified values `[FULL]` or max/sum policy.

### Why definitions in Content, service in Cognition

Faculty caps, voice profiles, and group membership are authored data. Mutable values are simulation state owned by Cognition assembly — asmdef rule: Cognition references Content, not Presentation.

---

## Implementation summary

| MVP (Phase 3) | Full |
|---|---|
| `IFacultyService` all contract methods | FacultyInterjection UI (Phase 13) |
| `FacultyDefinition` + `FacultyState` | Group aggregate formulas (sum vs max) |
| Modifier stack with `ModifierSource` | Temporary timed modifiers |
| Vitality / Morale pools | Damage-type narrative injuries `[FULL]` |
| Caps from definition | Dynamic cap shifts from story `[FULL]` |
| Tests: modifier order, caps, groups | XP/progression curve `[FULL]` |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Cognition` |
| **References** | Runtime, Content, Rules |
| **Namespace** | `NarrativeFramework.Cognition.Faculty` |
| **Definitions** | `NarrativeFramework.Content.Definitions.FacultyDefinition` |
| **Tick phase** | **Interpretation** — passive threshold checks; modifier decay hooks |

---

## Public API

See [contracts.md](contracts.md) — `IFacultyService`.

**Module notes:**

- `GetValue(facultyId)` — base from state; missing ID throws or returns 0 per test policy (MVP: throw `FacultyNotFoundException`).
- `GetModifiedValue(facultyId)` — base + sum of active modifiers, clamped to definition min/max.
- `ApplyModifier(facultyId, delta, source)` — idempotent per source key `[FULL]`; MVP stacks deltas per source category.
- `GetVitality()` / `GetMorale()` — pool structs with Current/Max.
- `GetFacultyIdsInGroup(groupId)` — from content definitions index at bootstrap.

---

## Internal implementation

### FacultyService

```csharp
namespace NarrativeFramework.Cognition.Faculty
{
    sealed class FacultyService : IFacultyService, IStatefulService<FacultyServiceState>
    {
        readonly IContentStore _content;
        readonly FacultyRegistry _registry = new();
        Vitality _vitality;
        Morale _morale;

        public int GetValue(string facultyId)
            => _registry.GetBase(facultyId);

        public int GetModifiedValue(string facultyId)
        {
            var def = _content.GetDefinition<FacultyDefinition>(facultyId);
            var raw = _registry.GetBase(facultyId) + _registry.GetModifierSum(facultyId);
            return Math.Clamp(raw, def.MinValue, def.MaxValue);
        }

        public void ApplyModifier(string facultyId, int delta, ModifierSource source)
            => _registry.AddModifier(facultyId, source, delta);

        public Vitality GetVitality() => _vitality;
        public Morale GetMorale() => _morale;

        public IReadOnlyList<string> GetFacultyIdsInGroup(string groupId)
            => _registry.GetIdsInGroup(groupId);

        public FacultyServiceState CaptureState() => _registry.ToState(_vitality, _morale);
        public void RestoreState(FacultyServiceState state) => _registry.FromState(state, out _vitality, out _morale);
    }
}
```

### FacultyRegistry (internal)

```csharp
sealed class FacultyRegistry
{
    readonly Dictionary<string, int> _base = new();
    readonly Dictionary<string, Dictionary<ModifierSource, int>> _modifiers = new();
    readonly Dictionary<string, List<string>> _groupIndex = new();

    public void InitializeFromContent(IContentStore store)
    {
        foreach (var id in store.GetAllIds<FacultyDefinition>())
        {
            var def = store.GetDefinition<FacultyDefinition>(id);
            _base[id] = def.StartingValue;
            IndexGroup(def.GroupId, id);
        }
    }

    public int GetModifierSum(string facultyId)
    {
        if (!_modifiers.TryGetValue(facultyId, out var bySource)) return 0;
        return bySource.Values.Sum();
    }
}
```

### FacultyDefinition

```csharp
class FacultyDefinition : ContentDefinition
{
    [SerializeField] string groupId;
    [SerializeField] string voiceProfileId;
    [SerializeField] int startingValue;
    [SerializeField] int minValue;
    [SerializeField] int maxValue;

    public string GroupId => groupId;
    public string VoiceProfileId => voiceProfileId;
    public int StartingValue => startingValue;
    public int MinValue => minValue;
    public int MaxValue => maxValue;
}
```

### Domain types (spec + data-model)

```csharp
class FacultyState { public string FacultyId; public int BaseValue; public int ModifiedValue; }
class FacultyGroupState { public string GroupId; public int BaseValue; public int ModifiedValue; }
class Vitality { public int Current; public int Max; }
class Morale { public int Current; public int Max; }
```

`ModifiedValue` in state DTO is snapshot for persistence; live query uses `GetModifiedValue`.

---

## Definition assets

| Asset | ID prefix | Key fields |
|---|---|---|
| `FacultyDefinition` | `faculty_` | GroupId, VoiceProfileId, StartingValue, Min/Max |

Loaded via `IContentStore`; registry built at service init.

---

## Runtime state / persistence

```csharp
class FacultyServiceState
{
    public Dictionary<string, int> BaseValues;
    public Dictionary<string, Dictionary<ModifierSource, int>> Modifiers;
    public Vitality Vitality;
    public Morale Morale;
}
```

Phase 10: `IPersistenceService` envelope. On restore, re-apply equipment modifiers from inventory if policy is recompute-from-sources.

---

## Core algorithms

### Initialize

1. Load all `FacultyDefinition` IDs from store.
2. Set base values from `StartingValue`.
3. Build group index map.
4. Initialize Vitality/Morale from pack defaults or character template `[FULL]`.

### GetModifiedValue

1. Sum modifiers by source for facultyId.
2. Add base.
3. Clamp to definition min/max.
4. `[FULL]` Apply global debuff caps.

### Passive check support (for IRollService)

Passive mode compares `GetModifiedValue(facultyId) >= difficulty` — no random roll (see cognition-roll.md).

### Modifier from external systems

| Source | Producer |
|---|---|
| Equipment | `IInventoryService` |
| Emotion | `IEmotionService.GetRollModifier` |
| Belief | `IBeliefService` assimilating effects |
| Conduct | `[FULL]` threshold buffs |
| Script | Content effects |

---

## Event contracts + kernel tick phase

| Event | When | Phase |
|---|---|---|
| `FacultyModifierApplied` | ApplyModifier | Interpretation |
| `VitalityChanged` | Damage/heal `[FULL]` | Interpretation |
| `MoraleChanged` | Story beat `[FULL]` | Social / Interpretation |

**Kernel:** Faculty passive checks run **Interpretation** after Facts published. Interjections (Phase 13) same phase.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | FacultyDefinition |
| `ContentIdValidator` | faculty_* |

| Used by | Reason |
|---|---|
| `IRollService` | Roll resolution values |
| `IInventoryService` | Equipment modifiers |
| `IEmotionService` | Roll penalties/bonuses |
| `IBeliefService` | Assimilating temp modifiers |
| `IGateService` | Faculty threshold gates |
| `IDialogueService` | Faculty-gated lines |
| `IVoiceService` | VoiceProfileId mapping Phase 13 |

---

## MVP scope checklist (Phase 3)

- [ ] `FacultyDefinition` in FrameworkTestPack
- [ ] `FacultyService` implements `IFacultyService`
- [ ] Base + modified values with caps
- [ ] `ApplyModifier` with `ModifierSource`
- [ ] Vitality / Morale getters
- [ ] `GetFacultyIdsInGroup`
- [ ] `FacultyServiceState` capture/restore stub
- [ ] Tests: cap clamp, modifier sum, group listing

---

## Full scope

- `[FULL]` FacultyInterjection trigger pipeline (Phase 13)
- `[FULL]` Timed modifier expiration
- `[FULL]` Group aggregate passive checks
- `[FULL]` Faculty discovery/unlock progression
- `[FULL]` Voice line catalog per faculty

---

## File tree

```text
Packages/NarrativeFramework/Cognition/Faculty/IFacultyService.cs
Packages/NarrativeFramework/Cognition/Faculty/FacultyService.cs
Packages/NarrativeFramework/Cognition/Faculty/FacultyRegistry.cs
Packages/NarrativeFramework/Cognition/Faculty/FacultyServiceState.cs
Packages/NarrativeFramework/Cognition/Faculty/Vitality.cs
Packages/NarrativeFramework/Cognition/Faculty/Morale.cs
Packages/NarrativeFramework/Cognition/Faculty/FacultyNotFoundException.cs
Packages/NarrativeFramework/Content/Definitions/FacultyDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Faculty/FacultyValueTests.cs
Packages/NarrativeFramework/Tests/EditMode/Faculty/FacultyModifierCapTests.cs
Packages/NarrativeFramework/Tests/EditMode/Faculty/FacultyGroupTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `GetValue_ReturnsStartingValue` | From definition |
| `ApplyModifier_IncreasesModifiedValue` | +2 authority |
| `ModifiedValue_ClampsToMax` | Cap enforced |
| `GetFacultyIdsInGroup_ReturnsMembers` | Count + IDs |
| `Vitality_Morale_InitialState` | Non-zero max |
| `RestoreState_PreservesBase` | Persistence stub |

**Exit criteria:** Phase 3 faculty tests part of ≥25 cognition core tests.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) FAC-01, FAC-02.

---

## Vitality and Morale semantics

Roadmap Phase 3 requires both pools on `IFacultyService`. They are **narrative pressure resources**, not duplicate HP/MP systems:

| Pool | Typical spend | Typical restore |
|---|---|---|
| **Vitality** | Failed physical rolls, injury beats, exhaustion | Rest, story beats, items `[FULL]` |
| **Morale** | Social failure, companion conflict, horror content | Success moments, companion trust |

MVP exposes getters and stores Current/Max in state. Spend APIs defer to `[FULL]` — rolls do not deduct vitality in Phase 3 unless content explicitly calls a future `TrySpendVitality` helper.

Initialization defaults (FrameworkTestPack):

```csharp
_vitality = new Vitality { Current = 10, Max = 10 };
_morale = new Morale { Current = 6, Max = 6 };
```

### Faculty voice profile linkage

`FacultyDefinition.VoiceProfileId` maps to `IVoiceService` channels in Phase 13. Architecture stores the ID only — no audio references in Cognition assembly. Passive interjection eligibility uses `GetModifiedValue >= threshold` plus content trigger tables `[FULL]`.

---

## Related documents

- [cognition-roll.md](cognition-roll.md) — consumes modified values
- [content-store.md](content-store.md) — definitions
- [content-inventory.md](content-inventory.md) — equipment modifiers
- [cognition-emotion.md](cognition-emotion.md) — roll modifiers
