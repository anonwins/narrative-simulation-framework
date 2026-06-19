# Story Fail-Forward — Implementation Architecture

- Spec: [story-fail-forward.md](../systems/story-fail-forward.md)
- Glossary: [Fail-forward pattern](../terminology-glossary.md) · [Roll modes](../terminology-glossary.md)
- Roadmap: [Phase 4](../development-roadmap.md) stub · [Phase 11](../development-roadmap.md) full pattern
- Contracts: [contracts.md](contracts.md) — **no `I*Service`**; integration pattern only

---

## Design rationale

### Why fail-forward is a pattern, not a service

Fail-forward is how NSF avoids hard stops. It coordinates existing services — `IRollService`, `IDialogueService`, `IStoryStateService`, `IPacingService` — without a monolithic `IFailForwardService` that duplicates their responsibilities. [contracts.md](contracts.md) lists it explicitly as a **pattern**.

### Why failure must be content

Traditional RPGs block on failed checks. NSF treats failure as a branch with reactions, consequences, and new possibilities. Architecture ensures every failure path mutates at least one authoritative store (flag, fact, beat stage, or thread evidence) — never “try again” as the only option.

### Why Phase 4 stubs the pattern early

Phase 4 dialogue tests require **fail branches that still advance state** before rolls are fully integrated (Phase 3 cognition). MVP: explicit failure edges in dialogue graphs + flag mutations. Phase 11: roll-coordinated orchestration with pacing cooldowns.

### Term split under failure

| Layer | On failure | Example |
|---|---|---|
| **Story beat** | May advance to `Failed` or `AltResolved` stage | Beat moves forward; not blocked |
| **Thread** | May add contradicting evidence or lock subject | Wrong theory still progresses inquiry |
| **Chronicle** | Projects new Lead/Clue from events | Read-only; never rolls back thread |

Failure never means “hide chronicle task until success.”

---

## Implementation summary

| MVP (Phase 4) | Full (Phase 11) |
|---|---|
| Dialogue failure node + flag set | Roll failure → routed content pipeline |
| Beat `Failed` stage optional | Repeatable roll recovery via `IRollService` |
| Test: fail path ≠ hard stop | Pacing prevents failure spam |
| — | Conduct/emotion modifiers on failure tone `[FULL]` |

---

## Assembly and namespace

Fail-forward has **no dedicated assembly**. Orchestration lives in:

- `NarrativeFramework.Story.Dialogue` — failure edges
- `NarrativeFramework.Cognition.Rolls` — failure results
- `NarrativeFramework.Story` — optional `FailForwardCoordinator` static helper (not registered)

---

## Public API

**None.** Consumers use existing contracts:

- `IRollService.Resolve` → `RollResult` with `OutcomeKind.Failure`
- `IDialogueService.TrySelectChoice` → failure branch target
- `IStoryStateService.SetFlag` / `AdvanceBeat`
- `IPacingService.CanFireContent` — throttle repeated failures

Document authoring convention: content packs tag failure branches with `fail_forward_*` metadata for pipeline validation Phase 12.

---

## Internal implementation

### FailForwardCoordinator (optional helper, not in registry)

```csharp
static class FailForwardCoordinator
{
    public static void ApplyFailurePath(
        RollResult roll,
        FailForwardDefinition def,
        IStoryStateService story,
        IDialogueService dialogue,
        IEventBus events)
    {
        if (roll.OutcomeKind != RollOutcomeKind.Failure)
            throw new InvalidOperationException("Not a failure outcome");

        foreach (var action in def.ConsequenceActions)
            /* delegate to IRuleEngine.ExecuteActions */;

        if (!string.IsNullOrEmpty(def.FailureDialogueNodeId))
            /* dialogue jumps via internal session API or authored graph edge */;

        events.Publish(new SimEvent {
            Type = EventTypes.FailForwardTaken,
            Payload = { ["rollId"] = roll.RollId, ["branchId"] = def.Id }
        });
    }
}
```

### FailForwardDefinition (content)

```csharp
class FailForwardDefinition
{
    public string Id;                    // fail_forward_*
    public string RollId;                // optional link
    public string FailureDialogueNodeId;  // or graph edge
    public List<Action> ConsequenceActions;
    public bool AdvanceBeatOnFailure;
    public string BeatId;
}
```

### Phase 4 dialogue-only path

```csharp
// In DialogueService — choice with explicit IsFailureBranch
void ExecuteChoice(ChoiceDefinition choice, RuleContext ctx)
{
    _rules.ExecuteActions(choice.Actions, ctx);
    if (choice.IsFailureBranch)
    {
        // Must still mutate state — enforced by content pipeline `[FULL]`
        _events.Publish(new SimEvent { Type = EventTypes.FailForwardTaken, /* ... */ });
    }
}
```

---

## Definition assets

| Asset | Purpose |
|---|---|
| `FailForwardDefinition` | Maps roll/dialogue failure to consequences |
| Dialogue `ChoiceDefinition.IsFailureBranch` | Phase 4 explicit marking |
| Roll definition `FailureBranchId` | Phase 11 roll integration |

Validation rule (Phase 12 pipeline): every `IsFailureBranch` choice must include ≥1 mutating action (flag, beat, fact, evidence).

---

## Runtime state

No dedicated `FailForwardState`. Failure memory lives in:

- `StoryStateServiceState` — flags and beat stages
- `RollServiceState` — repeatable roll cooldowns
- `DialogueServiceState` — visited failure nodes `[FULL]`

---

## Core algorithms

### Dialogue fail branch (Phase 4)

1. Player selects choice marked failure (or roll-linked failure edge)
2. Execute consequence actions (set flag, register fact, advance beat to Failed/Alt)
3. Advance to failure target node — conversation continues or ends with new state
4. Publish `FailForwardTaken`
5. Chronicle refreshes projection — does not store authoritative failure state

### Roll fail forward (Phase 11)

1. `IRollService.Resolve` returns `Failure` with `FailForwardId`
2. Load `FailForwardDefinition` from content
3. `IPacingService.CanFireContent` — if false, queue for next tick (failure still recorded in roll state)
4. `FailForwardCoordinator.ApplyFailurePath`
5. Dialogue or voice presents failure content
6. `IRollService.RecoverRepeatable` if mode is Repeatable

### Anti-patterns (forbidden)

```text
Failure → return false with no state change
Failure → infinite retry loop without new content
Failure → chronicle-only update (projection without authority)
```

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `FailForwardTaken` | After failure path applied | Chronicle, conduct, debug |
| `RollFailed` | From roll service | Fail-forward routing |
| `StoryBeatAdvanced` | Beat moved to Failed/Alt | Outcome weights Phase 11 |

---

## Integration matrix

| Coordinates | Role on failure |
|---|---|
| `IRollService` | Produces failure outcome |
| `IDialogueService` | Presents failure branch |
| `IStoryStateService` | Flags and beat stages |
| `IThreadService` | Optional wrong-theory evidence |
| `IPacingService` | Prevents fatigue from repeated fails |
| `IRuleEngine` | Executes consequence actions |
| `IChronicleService` | **Read-only** refresh only |

---

## MVP scope (Phase 4)

- [ ] Dialogue failure edges with mandatory state mutation
- [ ] `FailForwardTaken` event stub
- [ ] Test: failure choice sets flag and reaches terminal node
- [ ] Test: failure does not block beat progression (Failed stage)
- [ ] Document pattern in architecture (this file)

---

## Full scope (Phase 11)

- [ ] `FailForwardCoordinator` + content definitions
- [ ] Roll → fail-forward routing
- [ ] Pacing integration for failure content windows
- [ ] Outcome service reads failure history for epilogue tone
- [ ] ≥5 dedicated fail-forward integration tests

---

## File tree

```text
Packages/NarrativeFramework/Story/FailForward/FailForwardCoordinator.cs
Packages/NarrativeFramework/Story/FailForward/FailForwardDefinition.cs
Packages/NarrativeFramework/Content/Definitions/FailForwardDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Story/DialogueFailBranchTests.cs
Packages/NarrativeFramework/Tests/EditMode/Story/RollFailForwardTests.cs
```

No `IFailForwardService.cs` — intentional.

---

## Test plan

| Test | Asserts |
|---|---|
| `FailureChoice_SetsFlag` | Authoritative mutation |
| `FailureChoice_AdvancesGraph` | Not hard stop |
| `FailureChoice_NoChronicleWrite` | Chronicle service has no mutator called |
| `RollFailure_RoutesToDialogue` | Phase 11 |
| `Pacing_BlocksRepeatedFailureSpam` | Phase 11 |
| `ThreadWrongTheory_StillAddsEvidence` | Thread progresses on failure path |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) fail-forward via rules/pacing; no runtime comedy generator.

---

## Related documents

- [story-dialogue.md](story-dialogue.md), [story-state.md](story-state.md)
- [story-pacing.md](story-pacing.md), [cognition-roll](../systems/cognition-roll.md)
- [contracts.md](contracts.md)
