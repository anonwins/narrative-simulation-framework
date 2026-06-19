# Story Outcome Service — Implementation Architecture

- Spec: [story-outcome.md](../systems/story-outcome.md)
- Glossary: [Outcome](../terminology-glossary.md) · [Thread vs Story beat](../terminology-glossary.md#when-to-use-which-term)
- Roadmap: [Phase 11](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why endings are synthesized, not single scripts

NSF endings aggregate conduct, relationships, ideology, thread resolutions, and story beat stages — not one authored finale node. `IOutcomeService` reads authoritative state at epilogue time; it does not mutate progression during play.

### Why outcome reads thread resolution separately from beats

A **thread subject** may resolve as `Incorrect` while the related **story beat** completes as `AltResolved`. Outcome weights both signals. Chronicle sections summarize for the player; outcome service computes ending ID from state snapshots.

### Why outcome never writes chronicle or thread

Epilogue generation is a **query + synthesis** pass. Chronicle may display outcome summary via projection refresh after `EndingResolved` event — chronicle remains read-only.

### Term split in ending synthesis

| Input | Source | Outcome use |
|---|---|---|
| Thread resolution | `ThreadEngine` / `IThreadService` | Did player solve inquiry correctly |
| Story beat stage | `IStoryStateService` | Which narrative branches completed |
| Chronicle section labels | Not used directly | UI only; outcome uses authoritative IDs |

---

## Implementation summary

| MVP (Phase 11) | Full (Phase 14+) |
|---|---|
| `Synthesize` from flags + conduct + thread | Ideology/emotion weighting `[FULL]` |
| `GetEndingId` weighted table | Multi-axis ending matrix `[FULL]` |
| Epilogue text key selection | Character fate summaries `[FULL]` |
| Tests: conduct changes ending | Pacing history tone modifier `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Story`
- **Namespace:** `NarrativeFramework.Story.Outcome`
- **Tick phase:** Invoked on demand (game end trigger), not every tick

---

## Public API

See [contracts.md](contracts.md) — `IOutcomeService`.

```csharp
interface IOutcomeService
{
    OutcomeSynthesis Synthesize(OutcomeContext context);
    string GetEndingId(OutcomeContext context);
}

class OutcomeContext
{
    public IStoryStateService Story;
    public IConductService Conduct;
    public IThreadService Thread;
    public IRelationshipService Relationships; // `[FULL]`
    public IIdeologyService Ideology;         // `[FULL]`
}

class OutcomeSynthesis
{
    public string EndingId;
    public Dictionary<string, float> AxisScores;
    public List<string> EpilogueTextKeys;
    public List<string> UnresolvedThreadIds; // `[FULL]`
}
```

---

## Internal implementation

### OutcomeService

```csharp
sealed class OutcomeService : IOutcomeService
{
    readonly IContentStore _content;

    public string GetEndingId(OutcomeContext context)
    {
        var weights = ScoreEndingCandidates(context);
        return weights.OrderByDescending(kv => kv.Value).First().Key;
    }

    public OutcomeSynthesis Synthesize(OutcomeContext context)
    {
        var endingId = GetEndingId(context);
        var def = _content.GetDefinition<EndingDefinition>(endingId);
        return new OutcomeSynthesis {
            EndingId = endingId,
            EpilogueTextKeys = def.EpilogueKeys,
            AxisScores = BuildAxisReport(context),
            UnresolvedThreadIds = FindUnresolvedThreads(context)
        };
    }

    Dictionary<string, float> ScoreEndingCandidates(OutcomeContext ctx)
    {
        var candidates = _content.GetAllIds<EndingDefinition>();
        var scores = new Dictionary<string, float>();
        foreach (var id in candidates)
        {
            var def = _content.GetDefinition<EndingDefinition>(id);
            scores[id] = def.Requirements
                .Sum(req => req.Evaluate(ctx) ? req.Weight : 0f);
        }
        return scores;
    }
}
```

### EndingRequirement (internal)

```csharp
class EndingRequirement
{
    public string FlagId;           // story flag
    public string BeatId;           // beat stage check
    public string ThreadId;         // resolution kind
    public string ConductId;        // conduct threshold
    public float Weight;
    public bool Evaluate(OutcomeContext ctx) { /* ... */ }
}
```

---

## Definition assets

| Asset | Purpose |
|---|---|
| `EndingDefinition` | Weighted requirements, epilogue keys |
| `EndingRequirement` | Maps authoritative IDs to scores |

Requirements reference `beat_*`, `flag_*`, `thread_*`, `conduct_*` — never `section_*` chronicle IDs.

---

## Runtime state

Outcome service is **stateless** in MVP. Optional cache of last synthesis in `OutcomeServiceState` for UI replay `[FULL]`. No persistence requirement beyond referenced services' save data.

---

## Core algorithms

### GetEndingId

1. Load all ending definitions from content
2. Build `OutcomeContext` from registry services (read-only)
3. Score each candidate by summing satisfied requirement weights
4. Tie-break: content-defined priority index
5. Return highest scoring `ending_*` ID

### Synthesize

1. Call `GetEndingId`
2. Collect epilogue text keys from winning definition
3. Build axis report: conduct profile, dominant relationship metrics `[FULL]`
4. List unresolved thread subjects (resolution null or `Unresolved`)
5. Return `OutcomeSynthesis` — no mutations

### Thread vs beat in scoring

```csharp
bool BeatSatisfied(OutcomeContext ctx, string beatId, StoryBeatStage minStage)
    => ctx.Story.GetBeatState(beatId).Stage >= minStage;

bool ThreadSatisfied(OutcomeContext ctx, string threadId, ThreadResolutionKind kind)
    => ctx.Thread.TryGetResolution(threadId, out var r) && r.Kind == kind;
```

Separate checks — completing a beat does not imply correct thread resolution.

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `EndingResolved` | After game calls Synthesize | Chronicle projection, UI |
| `EpilogueStarted` | Presentation `[FULL]` | Audio |

Outcome service may publish `EndingResolved` once; subscribers refresh chronicle **projection** only.

---

## Integration matrix

| Reads (read-only) | Purpose |
|---|---|
| `IStoryStateService` | Flags, beat stages |
| `IThreadService` | Subject resolutions |
| `IConductService` | Identity classification |
| `IRelationshipService` | Companion/faction fates `[FULL]` |
| `IIdeologyService` | Political endings `[FULL]` |
| `IPacingService` | Optional tone modifier `[FULL]` |

| Does not write | Reason |
|---|---|
| Story state | Endings are summaries |
| Thread state | Inquiry closed at play time |
| Chronicle | Projection only |

---

## MVP scope (Phase 11)

- [ ] `OutcomeService` with weighted ending table
- [ ] Requirements: flags, beat stage, conduct threshold, thread resolution
- [ ] `Synthesize` returns epilogue keys
- [ ] Test: different conduct → different ending ID
- [ ] Test: wrong thread resolution → alternate ending

---

## Full scope

- `[FULL]` Multi-axis ending matrix with partial victories
- `[FULL]` Unresolved thread listing in synthesis
- `[FULL]` Relationship and ideology weights
- `[FULL]` Pacing history affects epilogue tone

---

## File tree

```text
Packages/NarrativeFramework/Story/Outcome/IOutcomeService.cs
Packages/NarrativeFramework/Story/Outcome/OutcomeService.cs
Packages/NarrativeFramework/Story/Outcome/OutcomeContext.cs
Packages/NarrativeFramework/Story/Outcome/OutcomeSynthesis.cs
Packages/NarrativeFramework/Content/Definitions/EndingDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Story/OutcomeSynthesisTests.cs
Packages/NarrativeFramework/Tests/EditMode/Story/OutcomeThreadVsBeatTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `GetEndingId_HighestWeightWins` | Scoring table |
| `ConductThreshold_SelectsEnding` | Different conduct profiles |
| `ThreadWrongResolution_AltEnding` | Thread signal independent of beat |
| `BeatComplete_ThreadUnresolved` | Both reported in synthesis |
| `Synthesize_NoStateMutation` | Story/thread unchanged after call |
| `Synthesize_NoChronicleWrite` | Chronicle service not mutated |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) outcome preview = Presentation; deterministic tie-break.

---

## Related documents

- [story-state.md](story-state.md), [ledger-thread.md](ledger-thread.md)
- [ledger-chronicle.md](ledger-chronicle.md), [story-pacing.md](story-pacing.md)
- [contracts.md](contracts.md)
