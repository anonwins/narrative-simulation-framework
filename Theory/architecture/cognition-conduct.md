# Cognition Conduct — Implementation Architecture

- Spec: [systems/cognition-conduct.md](../systems/cognition-conduct.md)
- Glossary: [conduct_* IDs](../terminology-glossary.md) · [Standing vs Conduct](../terminology-glossary.md)
- Roadmap: [Phase 8](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [social-faction.md](../systems/social-faction.md) · [story-outcome.md](../systems/story-outcome.md)

---

## Design rationale

### Why conduct observes patterns, not moral judgment

Conduct tracks **recurring player behavior patterns** (`conduct_humble`, `conduct_bold`, …) — emergent identity, not good/evil scoring. The engine asks “what does this player tend to do?” to unlock recognition dialogue, ending synthesis, and actor reactions.

### Why separate from faction standing

**Standing** (`IFactionService`) is relational reputation with external groups. **Conduct** is internal character profile. Glossary boundary: gates may check either, but services stay separate to avoid conflating “gang trusts you” with “you act boldly.”

### Why cumulative scores with thresholds

Weighted `RecordAction` deltas accumulate. Threshold milestones (5 / 15 / 30 in spec examples) derive recognition flags consumed by `IGateService` and dialogue conditions — without hardcoding every branch to raw action history.

### Why GetDominantConductProfile

`IOutcomeService` (Phase 11) and voice/narration need a single emergent label for ending synthesis. Dominant = highest score among `conduct_*` IDs; ties broken by recency `[FULL]`.

### Why event-driven updates

Conduct subscribes to dialogue choices, story beats, item use — via explicit `RecordAction` calls from Story/Social layers, not by scraping logs. Keeps causality traceable for debug (Phase 14).

---

## Implementation summary

| MVP (Phase 8) | Full |
|---|---|
| `IConductService` full contract | Weighted event map from content |
| `RecordAction(conductId, delta)` | Automatic subscription table |
| `GetScore` / `GetAllScores` | Recognition flag derivation |
| `GetDominantConductProfile` | Actor awareness barks |
| `conduct_*` content IDs | Conduct decay over time `[FULL]` |
| Tests: accumulation, dominant | Dialogue conduct conditions (Phase 5 gates) |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Cognition` |
| **References** | Runtime, Content, Rules |
| **Namespace** | `NarrativeFramework.Cognition.Conduct` |
| **Tick phase** | **Events** (record after player action) · **Gating** (threshold checks) |

---

## Public API

See [contracts.md](contracts.md) — `IConductService`.

**Module notes:**

- `RecordAction(conductId, delta)` — clamps score ≥ 0; publishes `ConductScoreChanged`.
- `GetScore(conductId)` — 0 if never recorded.
- `GetAllScores()` — snapshot for outcome UI.
- `GetDominantConductProfile()` — highest conduct ID string.

---

## Internal implementation

### ConductService

```csharp
namespace NarrativeFramework.Cognition.Conduct
{
    sealed class ConductService : IConductService, IStatefulService<ConductServiceState>
    {
        readonly Dictionary<string, int> _scores = new();
        readonly ConductThresholdTable _thresholds;

        public int GetScore(string conductId)
            => _scores.GetValueOrDefault(conductId);

        public void RecordAction(string conductId, int delta)
        {
            ContentIdValidator.IsValid(conductId, "conduct_");
            var next = Math.Max(0, GetScore(conductId) + delta);
            _scores[conductId] = next;
            Publish(ConductEvents.ScoreChanged, conductId, next);
            CheckThresholds(conductId, next);
        }

        public string GetDominantConductProfile()
        {
            if (_scores.Count == 0) return null;
            return _scores.OrderByDescending(kv => kv.Value).First().Key;
        }

        public IReadOnlyDictionary<string, int> GetAllScores()
            => new Dictionary<string, int>(_scores);

        void CheckThresholds(string conductId, int score)
        {
            foreach (var t in _thresholds.GetCrossed(conductId, score))
                Publish(ConductEvents.ThresholdCrossed, conductId, t);
        }
    }
}
```

### ConductScore / StandingScore (domain)

```csharp
class ConductScore { public string ConductId; public int Value; }
class StandingScore { public string StandingId; public int Value; }  // Social module — not stored here
```

`StandingScore` lives in Social architecture — listed in spec for glossary clarity only.

### ConductThresholdTable

```csharp
sealed class ConductThresholdTable
{
    readonly Dictionary<string, int[]> _thresholdsByConduct = new()
    {
        ["conduct_humble"] = new[] { 5, 15, 30 },
        ["conduct_bold"] = new[] { 5, 15, 30 },
    };

    public IEnumerable<int> GetCrossed(string conductId, int newScore) { /* ... */ }
}
```

MVP: hardcoded defaults; `[FULL]` load from content pack.

### ConductRecognition `[FULL]`

```csharp
class ConductRecognition
{
    public HashSet<string> RecognizedConducts;
}
```

Derived at threshold cross; stored in `ConductServiceState` for save.

---

## Definition assets

Phase 8 MVP: conduct IDs validated by prefix only — no separate ScriptableObject required.

`[FULL]` `ConductDefinition` (`conduct_profile_*`) with threshold arrays and display labels.

Dialogue/gate content references `conduct_*` in rule conditions (Phase 5).

---

## Runtime state / persistence

```csharp
class ConductServiceState
{
    public Dictionary<string, int> Scores;
    public HashSet<string> RecognizedConducts;
}
```

Phase 10 save envelope.

---

## Core algorithms

### RecordAction flow

1. Validate `conduct_*` ID.
2. Apply delta, floor at zero.
3. Publish score changed event.
4. Compare against threshold table; fire recognition events on first cross.

### Dominant profile

1. If no scores → null.
2. Order by score descending.
3. MVP: first wins; `[FULL]` tie-break by last RecordAction timestamp.

### Integration call sites

Story/dialogue effect executor:

```csharp
_conduct.RecordAction("conduct_humble", delta: 3);
```

Not automatic in MVP — explicit in dialogue graph actions; `[FULL]` event subscription map.

---

## Event contracts + kernel tick phase

| Event | When | Phase |
|---|---|---|
| `ConductScoreChanged` | RecordAction | Events |
| `ConductThresholdCrossed` | Milestone first hit | Events |
| `ConductProfileChanged` | Dominant changes `[FULL]` | Events |

**Kernel:** Recording happens **Events** or **Content** immediately after player choice resolves. Gate checks use scores during **Gating**.

Subscribers: `IGateService`, `IDialogueService`, `IOutcomeService`, debug trace.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `ContentIdValidator` | conduct_* prefix |

| Used by | Reason |
|---|---|
| `IGateService` | requiresConduct conditions |
| `IDialogueService` | Choice effects RecordAction |
| `IOutcomeService` | Ending synthesis |
| `IRelationshipService` | Complementary not duplicate |
| `IFactionService` | Standing separate |

**Forbidden:** ConductService calling IFactionService directly — parallel scores.

---

## MVP scope checklist (Phase 8)

- [ ] `IConductService` implementation
- [ ] Score accumulation + clamp
- [ ] GetAllScores / GetDominantConductProfile
- [ ] Threshold crossing events (minimal table)
- [ ] ConductServiceState persistence
- [ ] Tests: delta sum, dominant selection, threshold once
- [ ] Sample conduct IDs in FrameworkTestPack / NoirSample prep

---

## Full scope

- `[FULL]` Content-driven ConductDefinition thresholds
- `[FULL]` Event subscription auto-map (PLAYER_APOLOGIZED → conduct_humble)
- `[FULL]` Conduct decay / seasonal reset
- `[FULL]` NPC conduct awareness dialogue pack
- `[FULL]` Weighted action registry in content store

---

## File tree

```text
Packages/NarrativeFramework/Cognition/Conduct/IConductService.cs
Packages/NarrativeFramework/Cognition/Conduct/ConductService.cs
Packages/NarrativeFramework/Cognition/Conduct/ConductScore.cs
Packages/NarrativeFramework/Cognition/Conduct/ConductServiceState.cs
Packages/NarrativeFramework/Cognition/Conduct/ConductThresholdTable.cs
Packages/NarrativeFramework/Cognition/Conduct/ConductEvents.cs
Packages/NarrativeFramework/Cognition/Conduct/ConductRecognition.cs
Packages/NarrativeFramework/Tests/EditMode/Conduct/ConductScoreTests.cs
Packages/NarrativeFramework/Tests/EditMode/Conduct/ConductDominantProfileTests.cs
Packages/NarrativeFramework/Tests/EditMode/Conduct/ConductThresholdTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `RecordAction_IncrementsScore` | GetScore |
| `RecordAction_NeverNegative` | Clamp at 0 |
| `GetDominant_ReturnsHighest` | conduct_bold vs humble |
| `ThresholdCrossed_FiresOnce` | Event count |
| `GetAllScores_Snapshot` | Immutable copy |
| `RestoreState_PreservesScores` | Persistence |

**Exit criteria:** Conduct emergence tests in Phase 8 suite.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) C-01 only (explicit `RecordAction`; auto map Phase 11+ per conduct spec).

---

## Example content integration

NoirSample prep (Phase 15) uses four conduct axes from the appendix mapping. Architecture expects IDs like:

```text
conduct_humble
conduct_bold
conduct_analytical
conduct_empathic
```

Dialogue graph action (Phase 4 executor calls Conduct):

```csharp
// Effect: player chose deferential apology branch
_conduct.RecordAction("conduct_humble", delta: 3);
```

Gate rule (Phase 5) reads score without mutation:

```json
{
  "requiresScore": { "conduct_bold": 15 }
}
```

Outcome synthesis (Phase 11) queries dominant:

```csharp
var profile = _conduct.GetDominantConductProfile();
// "conduct_analytical" → ending_branch_cold_case
```

### Standing vs Conduct quick reference

| Question | Service | ID prefix |
|---|---|---|
| Does the gang trust you? | `IFactionService` | `faction_*`, standing metrics |
| Does the player act boldly? | `IConductService` | `conduct_*` |
| Does Kim trust you? | `ICompanionService` | `metric_trust_companion` |

Never map faction standing deltas into conduct scores in the same event handler — parallel updates only when design explicitly records both.

### Debug and trace (Phase 14)

Debug module should display:

- Live conduct score table sorted descending
- Last 10 `RecordAction` call sites (stack via optional debug wrapper)
- Recognized conduct flags from threshold crosses

---

## Related documents

- [cognition-belief.md](cognition-belief.md) — psychological layer sibling
- [cognition-emotion.md](cognition-emotion.md) — parallel mind simulation
- [rules-gate.md](rules-gate.md) — conduct conditions
- [story-outcome.md](story-outcome.md) — dominant conduct in endings
