# Story Pacing Service — Implementation Architecture

- Spec: [story-pacing.md](../systems/story-pacing.md)
- Glossary: [Pacing vs Rule Engine](../terminology-glossary.md)
- Roadmap: [Phase 11](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Kernel: [runtime-kernel.md](runtime-kernel.md)

---

## Design rationale

### Why pacing is separate from rules

`IRuleEngine` answers **can** this happen — legality given world state. `IPacingService` answers **when** it should fire — emotional rhythm, cooldowns, density. A confession scene may be rule-legal but pacing-blocked until a window opens. Kernel resolves conflict: rules decide truth; pacing delays firing to next tick (see [runtime-kernel.md](runtime-kernel.md)).

### Why pacing protects fail-forward

Without cooldowns, players spam failed rolls to harvest comedy branches and collapse narrative tension. Pacing registers content windows per `contentId` including failure branches tagged `fail_forward_*`.

### Why pacing reads beats but does not advance them

**Story beats** signal narrative phase; pacing windows may key off `StoryBeatStage.Complete` via rules. Pacing never calls `AdvanceBeat` — that remains story state + dialogue/roll actions.

### Term split

| Question | Service |
|---|---|
| Is beat X complete? | `IStoryStateService` |
| Is thread subject resolved? | `IThreadService` |
| Should we show chronicle reveal now? | Pacing gates **content fire**, chronicle **projects** after fire |

Chronicle sections do not have pacing timers — entries appear when underlying events fire and projection refreshes.

---

## Implementation summary

| MVP (Phase 11) | Full (Phase 14+) |
|---|---|
| `CanFireContent` / `RegisterContentWindow` | Intensity curves per act `[FULL]` |
| Cooldown after content fire | Cross-content spacing rules `[FULL]` |
| `TickPacing` in Gating phase | Beat-linked window templates `[FULL]` |
| Blocks early reveal test | Mystery unfold sequencing `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Story`
- **Namespace:** `NarrativeFramework.Story.Pacing`
- **Tick phase:** **Gating** (evaluate), decrement timers in **Content** or end of tick

---

## Public API

See [contracts.md](contracts.md) — `IPacingService`.

```csharp
interface IPacingService
{
    bool CanFireContent(string contentId);
    void RegisterContentWindow(string contentId, PacingWindow window);
    void TickPacing();
}

class PacingWindow
{
    public GameTime Earliest;
    public GameTime Latest;          // optional
    public int CooldownTicks;
    public int MaxFiresPerChapter;   // `[FULL]`
}
```

---

## Internal implementation

### PacingService

```csharp
sealed class PacingService : IPacingService, IStatefulService<PacingServiceState>
{
    readonly Dictionary<string, PacingWindow> _windows = new();
    readonly Dictionary<string, ContentPacingRecord> _records = new();
    readonly ITimeService _time;

    public bool CanFireContent(string contentId)
    {
        if (!_windows.TryGetValue(contentId, out var window))
            return true; // unregistered content: no pacing constraint

        if (_time.Now < window.Earliest) return false;

        if (_records.TryGetValue(contentId, out var rec))
        {
            if (rec.CooldownRemaining > 0) return false;
            if (window.MaxFiresPerChapter > 0 && rec.FireCount >= window.MaxFiresPerChapter)
                return false;
        }
        return true;
    }

    public void RegisterContentWindow(string contentId, PacingWindow window)
        => _windows[contentId] = window;

    public void TickPacing()
    {
        foreach (var rec in _records.Values)
            if (rec.CooldownRemaining > 0) rec.CooldownRemaining--;
    }

    internal void RecordFire(string contentId)
    {
        if (!_records.TryGetValue(contentId, out var rec))
            _records[contentId] = rec = new ContentPacingRecord();
        rec.FireCount++;
        if (_windows.TryGetValue(contentId, out var w))
            rec.CooldownRemaining = w.CooldownTicks;
    }
}
```

### PacingGateAdapter (internal)

Dialogue and script runners query pacing during Gating:

```csharp
static class PacingGateAdapter
{
    public static bool TryFire(string contentId, IPacingService pacing, Action fire)
    {
        if (!pacing.CanFireContent(contentId)) return false;
        fire();
        ((PacingService)pacing).RecordFire(contentId); // internal friend or interface `[FULL]`
        return true;
    }
}
```

---

## Definition assets

| Asset | Purpose |
|---|---|
| `PacingWindowDefinition` | Authored per scene/dialogue/beat content ID |
| Content ID linkage | Same IDs as dialogue graphs, voice keys, fail-forward branches |

Windows reference **content IDs**, not `beat_*` directly — beats unlock content via story state; pacing gates the content ID.

---

## Runtime state

```csharp
class PacingServiceState
{
    public Dictionary<string, ContentPacingRecord> Records;
}

class ContentPacingRecord
{
    public int FireCount;
    public int CooldownRemaining;
}
```

---

## Core algorithms

### CanFireContent (Gating phase)

1. Lookup window for `contentId`
2. Compare `ITimeService.Now` to `Earliest`/`Latest`
3. Check cooldown and max fires
4. Return boolean — no side effects

### Record fire (Content phase)

1. After dialogue node, voice line, or fail-forward branch actually executes
2. Increment fire count; set cooldown ticks
3. Publish `ContentFired` event `[FULL]` for chronicle projection timing

### TickPacing

1. Decrement all `CooldownRemaining` at end of Gating or start of next tick
2. Called from kernel **Gating** handler

### Rule + pacing interaction

```text
1. IRuleEngine.Evaluate → legal
2. IPacingService.CanFireContent → timed
3. If legal && !timed → queue retry next tick (do not invalidate rule)
4. If legal && timed → fire in Content phase
```

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `ContentPacingBlocked` | CanFire false after rule pass `[FULL]` | Debug |
| `ContentFired` | After RecordFire | Analytics, chronicle |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `ITimeService` | Earliest/latest windows |
| `IContentStore` | Window definitions Phase 12 |

| Used by | Reason |
|---|---|
| `IDialogueService` | Gate heavy scenes |
| Fail-forward pattern | Throttle failure branches |
| `IVoiceService` | Optional narration delay `[FULL]` |
| Kernel Gating phase | Ordered evaluation |

| Does not use | Reason |
|---|---|
| `IChronicleService` | Projection follows content fire, not vice versa |

---

## MVP scope (Phase 11)

- [ ] `PacingService` with windows and cooldowns
- [ ] Kernel Gating integration hook
- [ ] Test: early content blocked; fires after window
- [ ] Test: rule-legal but pacing-delayed retries next tick
- [ ] Fail-forward branch respects cooldown

---

## Full scope

- `[FULL]` Intensity curves and chapter budgets
- `[FULL]` Automatic window registration from beat definitions
- `[FULL]` PacingBlocked debug events

---

## File tree

```text
Packages/NarrativeFramework/Story/Pacing/IPacingService.cs
Packages/NarrativeFramework/Story/Pacing/PacingService.cs
Packages/NarrativeFramework/Story/Pacing/PacingWindow.cs
Packages/NarrativeFramework/Story/Pacing/PacingServiceState.cs
Packages/NarrativeFramework/Content/Definitions/PacingWindowDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Story/PacingWindowTests.cs
Packages/NarrativeFramework/Tests/EditMode/Story/PacingRuleInteractionTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `CanFire_BeforeEarliest_False` | Time gate |
| `CanFire_AfterCooldown_True` | Cooldown decay via TickPacing |
| `RuleLegal_PacingBlocks_NoMutation` | No dialogue advance same tick |
| `RuleLegal_PacingAllows_Fires` | Content executes |
| `FailForward_RespectsCooldown` | Spam failure blocked |

---

## Deferred decisions

**Resolved** — [decisions-log.md](../decisions-log.md) ST-03, ST-04: pacing **configurable** at bootstrap + per-beat content; default policy **Mixed**.

---

## Related documents

- [runtime-kernel.md](runtime-kernel.md), [rules-engine.md](rules-engine.md)
- [story-fail-forward.md](story-fail-forward.md), [story-outcome.md](story-outcome.md)
- [contracts.md](contracts.md)
