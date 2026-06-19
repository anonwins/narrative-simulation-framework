# Story State Service — Implementation Architecture

- Spec: [story-state.md](../systems/story-state.md)
- Glossary: [Story beat](../terminology-glossary.md) · [Fact vs Story flag](../terminology-glossary.md)
- Roadmap: [Phase 4](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why story state is separate from facts

Facts are **simulation truth** — what objectively happened in the world. Story flags and beats are **narrative progression switches** — what the story machine considers active, offered, or complete. Conflating them breaks investigation games where the player can be wrong about truth while the story still advances.

### Why story beats are not chronicle tasks

A **story beat** is an atomic progression unit in `IStoryStateService` (start → advance → complete). A **Task** in the chronicle is a projected UI entry (“talk to the clerk”) derived from thread + story + facts. Authors advance beats; players read tasks. Never store beat completion only in chronicle.

### Why story state does not own threads

**Thread** inquiry logic (evidence, theories, subject resolution) lives in `ThreadEngine`. Story state **references** thread outcomes via flags and variables (`thread_homicide_resolved = true`), not by embedding `ThreadEvidence` objects. This keeps Phase 4 deliverable independent of Phase 9 ledger.

### Why rich beat states, not boolean quest flags

NSF stories need offered, in progress, failed, abandoned, resolved differently — not just active/complete. `StoryBeatState` encodes a stage enum plus optional metadata for outcome synthesis Phase 11.

### Term split reminder

| You mean… | API | Not |
|---|---|---|
| Advance narrative step | `AdvanceBeat(beatId)` | `AddEvidence`, chronicle task |
| Player journal grouping | Chronicle `section_*` | Beat ID |
| Investigation resolution | `IThreadService.TryResolveSubject` | `AdvanceBeat` alone |

---

## Implementation summary

| MVP (Phase 4) | Full (Phase 11+) |
|---|---|
| Flags, typed variables | Beat dependency graph `[FULL]` |
| Beat stage machine | Auto-transitions from rule triggers `[FULL]` |
| Manual `AdvanceBeat` + rule actions | Outcome weights on beat resolution `[FULL]` |
| Persistence snapshot stub | Cross-beat consequence propagation `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Story`
- **Namespace:** `NarrativeFramework.Story.State`
- **Tick phases:** **Gating** (beat gates), **Content** (scripted beat advances)

---

## Public API

See [contracts.md](contracts.md) — `IStoryStateService`.

```csharp
interface IStoryStateService
{
    bool GetFlag(string flagId);
    void SetFlag(string flagId, bool value);
    T GetVariable<T>(string key);
    void SetVariable<T>(string key, T value);
    void AdvanceBeat(string beatId);
    StoryBeatState GetBeatState(string beatId);
}
```

---

## Internal implementation

### StoryStateService

```csharp
sealed class StoryStateService : IStoryStateService, IStatefulService<StoryStateServiceState>
{
    readonly StoryFlagRegistry _flags = new();
    readonly StoryBeatRegistry _beats = new();
    readonly VariableStore _variables = new();
    readonly IContentStore _content;
    readonly IEventBus _events;

    public bool GetFlag(string flagId) => _flags.Get(flagId);

    public void SetFlag(string flagId, bool value)
    {
        ContentIdValidator.IsValid(flagId, "flag_");
        _flags.Set(flagId, value);
        _events.Publish(new SimEvent {
            Type = EventTypes.StoryFlagChanged,
            Payload = { ["flagId"] = flagId, ["value"] = value }
        });
    }

    public void AdvanceBeat(string beatId)
    {
        var def = _content.GetDefinition<StoryBeatDefinition>(beatId);
        var current = _beats.GetOrCreate(beatId);
        var nextStage = BeatStageMachine.Next(current.Stage, def);
        current.Stage = nextStage;
        current.LastAdvancedAt = /* ITimeService.Now */;
        _events.Publish(new SimEvent {
            Type = EventTypes.StoryBeatAdvanced,
            Payload = { ["beatId"] = beatId, ["stage"] = nextStage.ToString() }
        });
    }

    public StoryBeatState GetBeatState(string beatId) => _beats.GetOrCreate(beatId);
}
```

### StoryFlagRegistry (internal)

```csharp
sealed class StoryFlagRegistry
{
    readonly Dictionary<string, bool> _flags = new();
    public bool Get(string id) => _flags.TryGetValue(id, out var v) && v;
    public void Set(string id, bool value) => _flags[id] = value;
}
```

### StoryBeatRegistry (internal)

```csharp
sealed class StoryBeatRegistry
{
    readonly Dictionary<string, StoryBeatState> _beats = new();
    public StoryBeatState GetOrCreate(string id) =>
        _beats.TryGetValue(id, out var s) ? s : (_beats[id] = new StoryBeatState { BeatId = id });
}
```

### BeatStageMachine (internal)

```csharp
static class BeatStageMachine
{
    public static StoryBeatStage Next(StoryBeatStage current, StoryBeatDefinition def)
    {
        // MVP: linear Hidden → Offered → Active → Complete
        // `[FULL]` def.Transitions table for branch stages (Failed, Abandoned, AltResolved)
        return current switch {
            StoryBeatStage.Hidden => StoryBeatStage.Offered,
            StoryBeatStage.Offered => StoryBeatStage.Active,
            StoryBeatStage.Active => StoryBeatStage.Complete,
            _ => current
        };
    }
}
```

---

## Definition assets

| Asset | ID prefix | Purpose |
|---|---|---|
| `StoryBeatDefinition` | `beat_*` | Stage graph, linked content IDs |
| Flag declarations | `flag_*` | Documented in content; runtime bool only |
| Variable schema | `var_*` | Typed keys for scripting |

Beats link to **content** (dialogue graphs, scenes), not to chronicle section IDs. Sections map in chronicle projection rules Phase 9.

---

## Runtime state

```csharp
class StoryStateServiceState
{
    public Dictionary<string, bool> Flags;
    public Dictionary<string, StoryBeatState> Beats;
    public Dictionary<string, object> Variables; // typed envelope in serializer
}

class StoryBeatState
{
    public string BeatId;
    public StoryBeatStage Stage;
    public GameTime LastAdvancedAt;
    public Dictionary<string, string> Metadata; // `[FULL]` outcome tags
}

enum StoryBeatStage
{
    Hidden, Offered, Active, InProgress, Complete,
    Failed, Abandoned, AltResolved // `[FULL]` fully wired
}
```

---

## Core algorithms

### Set flag

1. Validate `flag_*` ID
2. Write registry; publish `StoryFlagChanged`
3. Chronicle subscriber may refresh projection — never write back to story state

### Advance beat

1. Load beat definition from content store
2. Evaluate optional preconditions via `IRuleEngine` (Phase 5)
3. Compute next stage from stage machine
4. Execute definition `OnAdvance` actions (flags, variables, unlock content)
5. Publish `StoryBeatAdvanced`

### Query beat for gates

`IGateService` reads `GetBeatState(beatId).Stage` during **Gating** — same tick ordering as facts: beats advanced in **Content** visible next tick unless same-tick rule explicitly allows.

### Thread outcome sync (Phase 9 integration)

When `ThreadEngine` resolves a subject, a rule action sets `flag_thread_{id}_resolved` — story beat may then advance via separate authored rule. Do not call `AdvanceBeat` from `ThreadEngine` directly (cross-module coupling).

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `StoryFlagChanged` | After SetFlag | Rules, chronicle, debug |
| `StoryBeatAdvanced` | After AdvanceBeat | Pacing, outcome, chronicle |
| `StoryVariableChanged` | After SetVariable | Scripts, gates |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | Beat definitions |
| `IEventBus` | Downstream projection |
| `ITimeService` | Beat timestamps |
| `IRuleEngine` | Preconditions and consequences Phase 5 |

| Used by | Reason |
|---|---|
| `IDialogueService` | Choice actions |
| `IGateService` | Beat stage gates |
| `IOutcomeService` | Ending synthesis Phase 11 |
| `IChronicleService` | Read-only projection Phase 9 |

---

## MVP scope (Phase 4)

- [ ] `StoryStateService` with flags, variables, beats
- [ ] Linear beat stage machine (Hidden → Complete)
- [ ] Rule engine actions: `SetFlag`, `AdvanceBeat`
- [ ] Event publish on flag/beat change
- [ ] Persistence capture stub
- [ ] Tests: beat advance, flag gate, branch via flag

---

## Full scope

- `[FULL]` Full stage enum with failure/abandon paths
- `[FULL]` Rule-triggered auto advances
- `[FULL]` Beat dependency DAG validation in content pipeline
- `[FULL]` Outcome weight accumulation on beat metadata

---

## File tree

```text
Packages/NarrativeFramework/Story/State/IStoryStateService.cs
Packages/NarrativeFramework/Story/State/StoryStateService.cs
Packages/NarrativeFramework/Story/State/StoryFlagRegistry.cs
Packages/NarrativeFramework/Story/State/StoryBeatRegistry.cs
Packages/NarrativeFramework/Story/State/StoryBeatState.cs
Packages/NarrativeFramework/Story/State/BeatStageMachine.cs
Packages/NarrativeFramework/Story/State/StoryStateServiceState.cs
Packages/NarrativeFramework/Content/Definitions/StoryBeatDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Story/StoryBeatAdvanceTests.cs
Packages/NarrativeFramework/Tests/EditMode/Story/StoryFlagGateTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `SetFlag_GetFlag_RoundTrip` | Boolean persistence |
| `AdvanceBeat_StageProgression` | Hidden → Offered → Active → Complete |
| `AdvanceBeat_PublishesEvent` | Handler receives beat ID |
| `Gate_BlocksUntilBeatActive` | Gate deny until stage Active |
| `Variable_TypedRoundTrip` | int/float/string storage |
| `Beat_NotThreadEvidence` | AdvanceBeat does not add thread evidence |

**Exit:** Part of Phase 4 ≥20 tests with dialogue.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) `beat_*` IDs; kernel ordering tests for same-tick visibility.

---

## Related documents

- [story-dialogue.md](story-dialogue.md), [ledger-thread.md](ledger-thread.md)
- [ledger-chronicle.md](ledger-chronicle.md), [story-outcome.md](story-outcome.md)
- [contracts.md](contracts.md)
