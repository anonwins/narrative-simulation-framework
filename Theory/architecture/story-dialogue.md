# Story Dialogue Service — Implementation Architecture

- Spec: [story-dialogue.md](../systems/story-dialogue.md)
- Glossary: [Dialogue](../terminology-glossary.md) · [Thread vs Story beat vs Chronicle](../terminology-glossary.md#when-to-use-which-term)
- Roadmap: [Phase 4](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why dialogue is primary gameplay

In NSF, dialogue is not a cutscene overlay — it is the hub where investigation, rolls, relationships, and story progression meet. `IDialogueService` must orchestrate graph traversal, choice resolution, and world effects without owning simulation truth (facts) or investigation logic (threads).

### Why a directed graph, not a tree

Reconverging nodes keep content combinatorics manageable. Architecture stores edges explicitly; `DialogueNode.Choices` reference target node IDs that may appear in multiple paths.

### Why conditions delegate to rules (Phase 5+)

Dialogue must not embed ad-hoc `if (flag)` checks. `IRuleEngine` evaluates visibility and choice legality during **Gating** phase; dialogue service consumes results during **Content** phase. This keeps one condition language across dialogue, gates, and scripts.

### Why dialogue never writes chronicle entries

Chronicle is a **read-only projection**. Dialogue may set story flags, register facts, or add thread evidence — chronicle updates via event subscribers in Phase 9. Never call `IChronicleService` mutators from dialogue (there are none by design).

### Term split reminder

| Concept | Owner | Dialogue role |
|---|---|---|
| **Story beat** | `IStoryStateService` | Actions may `AdvanceBeat(beatId)` |
| **Thread** | `IThreadService` / `ThreadEngine` | Actions may `AddEvidence` — not “complete quest” |
| **Chronicle section** | `IChronicleService` (projection) | No direct write; UI refreshes from events |

---

## Implementation summary

| MVP (Phase 4) | Full (Phase 11+) |
|---|---|
| Start/end conversation, graph walk | Faculty interruption queue `[FULL]` |
| Choice select with rule-gated visibility | Roll-integrated choice branches |
| Node actions: flags, variables, facts | Cross-graph memory / revisit rules `[FULL]` |
| Fail branch advances state (fail-forward hook) | Dynamic bark injection from social state `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Story`
- **Namespace:** `NarrativeFramework.Story.Dialogue`
- **Tick phases:** **Interpretation** (passive hooks), **Gating** (choice legality), **Content** (advance + effects)

**Forbidden:** Dialogue → Presentation types. Presenters bind via view models in Phase 13+.

---

## Public API

See [contracts.md](contracts.md) — `IDialogueService`.

```csharp
interface IDialogueService
{
    void StartConversation(string dialogueGraphId, string speakerActorId);
    DialogueNode GetCurrentNode();
    bool TrySelectChoice(string choiceId, out DialogueAdvanceResult result);
    void EndConversation();
}
```

Supporting types live in `NarrativeFramework.Story.Dialogue` and serialize via `DialogueServiceState` in persistence Phase 10.

---

## Internal implementation

### DialogueService

```csharp
sealed class DialogueService : IDialogueService, IStatefulService<DialogueServiceState>
{
    readonly IContentStore _content;
    readonly IRuleEngine _rules;
    readonly IStoryStateService _story;
    readonly IFactService _facts;
    readonly IEventBus _events;

    DialogueSession _session;

    public void StartConversation(string dialogueGraphId, string speakerActorId)
    {
        var graph = _content.GetDefinition<DialogueGraphDefinition>(dialogueGraphId);
        _session = new DialogueSession {
            GraphId = dialogueGraphId,
            SpeakerActorId = speakerActorId,
            CurrentNodeId = graph.EntryNodeId
        };
        _events.Publish(new SimEvent {
            Type = EventTypes.DialogueStarted,
            Payload = { ["graphId"] = dialogueGraphId, ["speakerId"] = speakerActorId }
        });
    }

    public bool TrySelectChoice(string choiceId, out DialogueAdvanceResult result)
    {
        result = default;
        if (_session == null) return false;

        var node = GetCurrentNode();
        var choice = node.Choices.FirstOrDefault(c => c.Id == choiceId);
        if (choice == null) return false;

        var ctx = BuildRuleContext(_session);
        if (!_rules.Evaluate(choice.VisibilityRule, ctx))
            return false;

        foreach (var action in choice.Actions)
            _rules.ExecuteActions(new[] { action }, ctx);

        _session.CurrentNodeId = choice.TargetNodeId;
        result = new DialogueAdvanceResult {
            NewNodeId = choice.TargetNodeId,
            Ended = choice.EndsConversation
        };
        if (choice.EndsConversation) EndConversation();
        return true;
    }

    public DialogueNode GetCurrentNode()
    {
        var graph = _content.GetDefinition<DialogueGraphDefinition>(_session.GraphId);
        return graph.Nodes[_session.CurrentNodeId];
    }

    public void EndConversation()
    {
        _events.Publish(new SimEvent { Type = EventTypes.DialogueEnded, /* ... */ });
        _session = null;
    }
}
```

### DialogueGraphRegistry (internal)

Content-backed lookup only — not registered in `INarrativeServiceRegistry`. Graphs load from `IContentStore`; runtime holds active session state.

### DialogueSession (internal)

```csharp
sealed class DialogueSession
{
    public string GraphId;
    public string SpeakerActorId;
    public string CurrentNodeId;
    public List<string> VisitedNodeIds = new();
}
```

---

## Definition assets

| Asset | Source | Notes |
|---|---|---|
| `DialogueGraphDefinition` | Content pack | Nodes, choices, embedded rule refs |
| `DialogueNodeDefinition` | Per graph | Speaker, text key, choice list |
| `ChoiceDefinition` | Per node | `VisibilityRule`, `Actions`, target node |

Authoring IDs: `dialogue_*`, `node_*`, `choice_*`. Text keys resolve via `ILocaleService` in Presentation layer.

---

## Runtime state

```csharp
class DialogueServiceState
{
    public DialogueSession ActiveSession; // null if not in conversation
    public List<string> CompletedGraphIds; // `[FULL]` replay gating
}
```

Dialogue state is **session truth** for the current conversation, not story progression truth — beats and flags live in `IStoryStateService`.

---

## Core algorithms

### Start conversation

1. Load graph from content store; validate entry node exists
2. Evaluate entry node entry rules (optional `OnEnter` actions via rule engine)
3. Publish `DialogueStarted`; presentation subscribes in Phase 13

### Select choice (Gating + Content)

1. Resolve choice on current node
2. Build `RuleContext` with actor, story flags, facts, session
3. **Gating:** evaluate visibility rule — deny if false
4. **Content:** execute choice actions (set flag, register fact, advance beat, start roll)
5. Advance `CurrentNodeId`; record visit for fail-forward analytics `[FULL]`
6. If node is terminal or choice ends conversation, call `EndConversation`

### Fail branch (Phase 4 stub, Phase 11 pattern)

When a roll-linked choice fails, dialogue must still advance to a **failure node** — never block graph progress. See [story-fail-forward.md](story-fail-forward.md). MVP: content authors wire explicit failure edges; Phase 11 adds roll service coordination.

### Integration with story beats

`AdvanceBeat` actions in choice payloads call `IStoryStateService.AdvanceBeat(beatId)`. This is distinct from thread resolution — a beat may unlock dialogue without resolving a thread subject.

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `DialogueStarted` | After session created | Voice, UI, debug |
| `DialogueNodeEntered` | After node advance `[FULL]` | Pacing, chronicle projection |
| `DialogueEnded` | Session cleared | Social, pacing cooldown |
| `DialogueChoiceSelected` | After successful choice | Info flow, conduct `[FULL]` |

**Kernel phase:** publish during **Content** (or **Events** if deferred queue used in Phase 14).

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | Graph definitions |
| `IRuleEngine` | Conditions and actions |
| `IStoryStateService` | Flag/beat mutations from actions |
| `IFactService` | World effects from dialogue |
| `IEventBus` | Decouple chronicle and UI |

| Used by | Reason |
|---|---|
| `IGateService` | Dialogue availability gates |
| `IRollService` | Skill-check choices Phase 3+ |
| `IChronicleService` | Read-only refresh on dialogue events Phase 9 |
| `IPacingService` | Cooldown after heavy dialogue Phase 11 |

**Never:** Dialogue → `IChronicleService` write path (none exists).

---

## MVP scope (Phase 4)

- [ ] `DialogueService` + session model
- [ ] Graph load from content store stubs
- [ ] `TrySelectChoice` with rule delegation
- [ ] Actions: `SetFlag`, `AdvanceBeat`, `RegisterFact` via rule engine
- [ ] Fail branch test: failed path still sets flag / advances beat
- [ ] ≥20 Edit Mode tests (3-node graph simulation)

---

## Full scope

- `[FULL]` Faculty interruption injection during Interpretation phase
- `[FULL]` Dynamic choice generation from thread state (read-only query)
- `[FULL]` Dialogue memory — hide/show nodes based on prior visits
- `[FULL]` Roll-integrated choices with fail-forward orchestration

---

## File tree

```text
Packages/NarrativeFramework/Story/Dialogue/IDialogueService.cs
Packages/NarrativeFramework/Story/Dialogue/DialogueService.cs
Packages/NarrativeFramework/Story/Dialogue/DialogueSession.cs
Packages/NarrativeFramework/Story/Dialogue/DialogueAdvanceResult.cs
Packages/NarrativeFramework/Story/Dialogue/DialogueServiceState.cs
Packages/NarrativeFramework/Content/Definitions/DialogueGraphDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Story/DialogueGraphTraversalTests.cs
Packages/NarrativeFramework/Tests/EditMode/Story/DialogueFailBranchTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `StartConversation_ReturnsEntryNode` | Current node matches graph entry |
| `SelectChoice_AdvancesToTarget` | Node ID updates |
| `SelectChoice_BlockedByRule` | Returns false when visibility fails |
| `ChoiceAction_SetsStoryFlag` | Flag visible to story state service |
| `ChoiceAction_AdvancesBeat` | Beat state transitions |
| `FailBranch_StillMutatesState` | Fail path sets flag; not hard stop |
| `EndConversation_ClearsSession` | GetCurrentNode throws or null session |
| `DialogueStarted_PublishesEvent` | Event bus handler invoked |

**Exit:** ≥20 tests; full 3-node graph in Edit Mode without Unity scene.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) ST-01, ST-02.

---

## Related documents

- [story-state.md](story-state.md), [rules-engine.md](rules-engine.md)
- [story-fail-forward.md](story-fail-forward.md), [ledger-chronicle.md](ledger-chronicle.md)
- [contracts.md](contracts.md)
