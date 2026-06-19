# Social Ideology Service — Implementation Architecture

- Spec: [systems/social-ideology.md](../systems/social-ideology.md)
- Roadmap: [Phase 7](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why ideology is emergent, not chosen at character creation

NSF political identity accumulates from dialogue, beliefs, and actions — not a class selection screen. `IIdeologyService` maintains **axis scores** (`axis_communist`, `axis_moralist`, etc.) that gates content and drives actor reactions without moralizing good/evil alignment.

### Why axes are floats with labels

Raw scores support weighted events (+1 minor comment, +3 speech). `GetDominantLabel` maps score ranges to author-facing labels for dialogue branching and faculty voice lines — players infer identity from behavior, not menus.

### Why ideology is Social, not Cognition

Beliefs (`IBeliefService`, Phase 8) internalize ideas; ideology aggregates **observable political posture** for social gates. Ideology service listens to belief and dialogue events but does not replace belief phase machinery.

### Why no Presentation in core

Ideology UI (political profile screen) is optional and late-phase. Core exposes query/shift APIs for gates, dialogue tags, and outcome synthesis only.

---

## Implementation summary

| MVP (Phase 7) | Full |
|---|---|
| `GetAxisValue` / `ShiftAxis` | Weighted event tables from content `[FULL]` |
| `GetDominantLabel(axisId)` | Cross-axis dominant profile `[FULL]` |
| Default axes from content manifest | Milestone unlock flags `[FULL]` |
| Dialogue choice ideology tags | Belief internalization hooks `[Phase 8]` |
| `IdeologyAxisShifted` event | Actor reaction catalog `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Social`
- **Namespace:** `NarrativeFramework.Social.Ideology`
- **References:** Runtime, Simulation.Events, Content, Story (dialogue events)
- **Tick phase:** **Social** (accumulate shifts) · **Gating** (read thresholds)

Registered as `IIdeologyService`.

---

## Public API

See [contracts.md](contracts.md) — `IIdeologyService`.

| Method | Behavior |
|---|---|
| `float GetAxisValue(string axisId)` | Current score on axis |
| `void ShiftAxis(string axisId, float delta)` | Add delta with clamp |
| `string GetDominantLabel(string axisId)` | Label for current score band |

Axis IDs use `axis_*` prefix per content convention (e.g. `axis_communist`, `axis_fascist`, `axis_moralist`, `axis_ultraliberal`).

---

## Internal implementation

### IdeologyService

```csharp
sealed class IdeologyService : IIdeologyService, IStatefulService<IdeologyServiceState>
{
    readonly Dictionary<string, float> _axes = new();
    readonly IContentStore _content;
    readonly IEventBus _events;

    public float GetAxisValue(string axisId)
        => _axes.TryGetValue(axisId, out var v) ? v : 0f;

    public void ShiftAxis(string axisId, float delta)
    {
        ContentIdValidator.IsValid(axisId, "axis_");
        var def = _content.GetDefinition<IdeologyAxisDefinition>(axisId);
        var prior = GetAxisValue(axisId);
        var next = Math.Clamp(prior + delta, def.Min, def.Max);
        _axes[axisId] = next;
        if (Math.Abs(next - prior) > float.Epsilon)
        {
            _events.Publish(new SimEvent {
                Type = EventTypes.IdeologyAxisShifted,
                Payload = {
                    ["axisId"] = axisId,
                    ["prior"] = prior.ToString(CultureInfo.InvariantCulture),
                    ["next"] = next.ToString(CultureInfo.InvariantCulture),
                    ["label"] = GetDominantLabel(axisId)
                }
            });
            CheckMilestoneCrossing(axisId, prior, next);
        }
    }

    public string GetDominantLabel(string axisId)
    {
        var score = GetAxisValue(axisId);
        var def = _content.GetDefinition<IdeologyAxisDefinition>(axisId);
        return LabelResolver.Resolve(score, def.Thresholds);
    }
}
```

### IdeologyEventHandler

Parses dialogue choice payload:

```json
{ "addsIdeology": { "axis_communist": 1, "axis_moralist": -1 } }
```

Applies during **Social** phase batch. `[FULL]` belief assimilated events add weighted shifts.

### LabelResolver (internal)

```csharp
static class LabelResolver
{
    public static string Resolve(float score, IList<IdeologyThreshold> thresholds)
    {
        foreach (var t in thresholds.OrderByDescending(x => x.MinScore))
            if (score >= t.MinScore) return t.LabelKey;
        return thresholds.First().LabelKey;
    }
}
```

Example bands: 0–4 "none", 5–14 "drift", 15–29 "strong", 30+ "locked_in".

### Milestone checker `[FULL]`

Emits `IdeologyMilestoneReached` when crossing content thresholds; sets story flags for beat unlocks.

---

## Definition assets

```csharp
class IdeologyAxisDefinition : ContentDefinition
{
    public float Min;
    public float Max;
    public List<IdeologyThreshold> Thresholds;
    public string DisplayNameKey;
}

class IdeologyThreshold
{
    public float MinScore;
    public string LabelKey;         // locale key
    public string MilestoneFlagId;  // [FULL]
}

class IdeologyShiftTable : ContentDefinition
{
    public string EventKey;
    public List<AxisDelta> Deltas;
}

class AxisDelta
{
    public string AxisId;
    public float Delta;
}
```

Content pipeline validates axis IDs referenced in dialogue tags exist in manifest.

---

## Runtime state

```csharp
class IdeologyServiceState
{
    public Dictionary<string, float> AxisValues;
    public HashSet<string> MilestonesReached;   // [FULL]
}
```

All axes initialized to 0 on new game unless content specifies `StartingValue`.

---

## Core algorithms

### ShiftAxis clamp

Same pattern as relationship metrics — no wraparound, debug log on clamp hit.

### Dominant label per axis

Each axis resolved independently in MVP. `[FULL]` `GetDominantProfile()` returns highest-scoring axis across set for ending synthesis.

### Dialogue gating

Rule operands: `ideology(axis_communist).value >= 15` or `ideology(axis_communist).label == strong_drift`.

Evaluated **Gating** phase — read-only.

### Contradictory axes

Implementation allows simultaneous high communist and ultraliberal scores; content/writing treats extreme combos as unstable. No engine enforcement in MVP.

### Integration with conduct/outcome `[Phase 11]`

`IOutcomeService` reads axis values for ending buckets — ideology service does not compute endings.

---

## Event contracts + tick phase

| Event | Publisher | Subscribers |
|---|---|---|
| `IdeologyAxisShifted` | IIdeologyService | Dialogue, faculty voice, chronicle |
| `IdeologyMilestoneReached` | `[FULL]` | IStoryStateService |
| `DialogueChoiceSelected` | IDialogueService | IdeologyEventHandler |

Writes: **Social**. Reads: **Gating**, **Content** (script conditions).

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IContentStore | Axis defs + shift tables |
| IEventBus | Shift notifications |
| IDialogueService events | Primary score source |

| Used by | Reason |
|---|---|
| IGateService | Political dialogue gates |
| IBeliefService | `[Phase 8]` reciprocal hooks |
| IFactionService | Actor ideological reactions `[FULL]` |
| IOutcomeService | Ending political profile |
| IPersistenceService | Axis snapshot |
| IVoiceService | Faculty political commentary |

**Forbidden:** IdeologyService → Presentation.

---

## MVP scope (Phase 7)

- [ ] `IIdeologyService` + `IdeologyService`
- [ ] Four default axes from spec (communist, fascist, moralist, ultraliberal)
- [ ] `GetAxisValue` / `ShiftAxis` / `GetDominantLabel`
- [ ] Dialogue choice ideology payload handler
- [ ] `IdeologyAxisShifted` event
- [ ] `IdeologyServiceState` capture/restore
- [ ] Tests: accumulation, clamp, label bands, dialogue tag integration

---

## Full scope

- `[FULL]` Milestone flags + story beat unlocks
- `[FULL]` Weighted tables by context intensity
- `[FULL]` Belief assimilated shift hooks
- `[FULL]` Actor-specific reaction content triggers
- `[FULL]` Global dominant profile for outcomes

---

## File tree

```text
Packages/NarrativeFramework/Social/Ideology/IIdeologyService.cs
Packages/NarrativeFramework/Social/Ideology/IdeologyService.cs
Packages/NarrativeFramework/Social/Ideology/IdeologyAxisDefinition.cs
Packages/NarrativeFramework/Social/Ideology/IdeologyThreshold.cs
Packages/NarrativeFramework/Social/Ideology/LabelResolver.cs
Packages/NarrativeFramework/Social/Ideology/IdeologyEventHandler.cs
Packages/NarrativeFramework/Social/Ideology/IdeologyShiftTable.cs
Packages/NarrativeFramework/Social/Ideology/IdeologyServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Social/IdeologyAxisTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/IdeologyLabelTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/IdeologyDialogueTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `ShiftAxis_Accumulates` | GetAxisValue |
| `ShiftAxis_ClampsToMax` | At Max |
| `GetDominantLabel_ReturnsCorrectBand` | Threshold boundaries |
| `DialogueChoice_AppliesTaggedShift` | Handler integration |
| `IdeologyAxisShifted_EventPublished` | Payload axisId |
| `CaptureRestore_PreservesAxes` | Roundtrip |
| `Gate_RequiresMinimumAxisValue` | With IGateService |

Part of Phase 7 ≥20 tests and ideology axis accumulation exit criterion.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SOC-02 (ideology decay, clamp at min).

---

## Bootstrap and dialogue tag schema

Dialogue choice nodes carry optional ideology blocks validated at content pipeline time:

```json
{
  "choiceId": "choice_defend_workers",
  "ideologyShift": [
    { "axisId": "axis_communist", "delta": 2 },
    { "axisId": "axis_ultraliberal", "delta": -1 }
  ],
  "requiresIdeology": {
    "axisId": "axis_communist",
    "minValue": 5
  }
}
```

`requiresIdeology` evaluated in **Gating** via `IIdeologyService.GetAxisValue`. `ideologyShift` applied in **Social** phase after choice event — same tick ordering as relationship deltas.

Sample axis thresholds for `axis_communist` (content-authored):

| Min score | Label key | Narrative use |
|---|---|---|
| 0 | `ideology_none` | No recognition |
| 5 | `ideology_mild_drift` | NPC offhand comments |
| 15 | `ideology_strong` | Unlocks tagged dialogue |
| 30 | `ideology_locked` | Ending-weighted identity |

Faculty voice reactions to ideology `[Phase 11]` subscribe to `IdeologyAxisShifted` — ideology service does not call `IVoiceService` directly (Social → Story direction only through events).

New game initializes all declared axes to 0. Content manifest `ideology_axes.json` lists required axis IDs — pipeline fails if dialogue references undeclared axis.

---

## Related documents

- [social-relationship.md](social-relationship.md), [social-faction.md](social-faction.md)
- [cognition-belief.md](../systems/cognition-belief.md) (Phase 8)
- [story-outcome.md](../systems/story-outcome.md), [contracts.md](contracts.md)
