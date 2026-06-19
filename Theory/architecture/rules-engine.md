# Rules Engine — Implementation Architecture

- Spec: [rules-engine.md](../systems/rules-engine.md)
- Glossary: [RuleEngine](../terminology-glossary.md)
- Roadmap: [Phase 5](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Gates: [rules-gate.md](rules-gate.md)

---

## Design rationale

### Why one evaluator replaces scattered condition checks

Without `IRuleEngine`, dialogue, gates, scripts, and beats each embed bespoke `if` logic. One IF/THEN evaluator keeps content data-driven and testable. `RuleEngine` is a **concrete orchestrator** implementing `IRuleEngine` — registered explicitly, not hidden inside dialogue.

### Why evaluate and execute are separate steps

`Evaluate` returns legality without side effects — safe during **Gating** phase. `ExecuteActions` mutates via delegated services during **Content** phase. Mixing them breaks kernel ordering and causes double-fire bugs.

### Why rules reference story flags, not chronicle entries

Conditions query **authoritative** stores: facts, flags, beat stages, conduct scores, thread resolution snapshots (read-only). Chronicle text and section visibility are never rule inputs — they lag authoritative state.

### Term split in rule conditions

| Condition type | Queries | Never queries |
|---|---|---|
| Progression | `IStoryStateService` beat stage, flags | Chronicle task completion |
| Investigation | `IThreadService` resolution, evidence count | Story beat ID as proxy for theory |
| World truth | `IFactService.IsActive` | Story flag as fact |

---

## Implementation summary

| MVP (Phase 5) | Full (Phase 12+) |
|---|---|
| Boolean condition trees | Script-compiled rules `[FULL]` |
| Action dispatch to services | Rule profiling / debug trace `[FULL]` |
| Dialogue delegates conditions | Derived fact rules `[FULL]` |
| Fact + flag + beat conditions | Complex temporal conditions `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Rules`
- **Namespace:** `NarrativeFramework.Rules.Engine`
- **Tick phases:** **Gating** (Evaluate), **Content** (ExecuteActions)

---

## Public API

See [contracts.md](contracts.md) — `IRuleEngine`, `RuleEngine`.

```csharp
interface IRuleEngine
{
    bool Evaluate(Rule rule, RuleContext context);
    void ExecuteActions(IReadOnlyList<Action> actions, RuleContext context);
}

class RuleEngine : IRuleEngine
{
    readonly ConditionEvaluator _conditions;
    readonly ActionExecutor _actions;
    /* ctor injects service registry */
}
```

---

## Internal implementation

### RuleEngine

```csharp
sealed class RuleEngine : IRuleEngine
{
    readonly ConditionEvaluator _evaluator;
    readonly ActionExecutor _executor;

    public RuleEngine(INarrativeServiceRegistry registry)
    {
        _evaluator = new ConditionEvaluator(registry);
        _executor = new ActionExecutor(registry);
    }

    public bool Evaluate(Rule rule, RuleContext context)
        => _evaluator.Evaluate(rule.RootCondition, context);

    public void ExecuteActions(IReadOnlyList<Action> actions, RuleContext context)
    {
        foreach (var action in actions)
            _executor.Execute(action, context);
    }
}
```

### ConditionEvaluator (internal)

```csharp
sealed class ConditionEvaluator
{
    readonly INarrativeServiceRegistry _registry;

    public bool Evaluate(Condition node, RuleContext ctx)
    {
        return node.Kind switch
        {
            ConditionKind.All => node.Children.All(c => Evaluate(c, ctx)),
            ConditionKind.Any => node.Children.Any(c => Evaluate(c, ctx)),
            ConditionKind.Not => !Evaluate(node.Children[0], ctx),
            ConditionKind.FactActive => _registry.GetRequired<IFactService>()
                .IsActive(node.TargetId),
            ConditionKind.StoryFlag => _registry.GetRequired<IStoryStateService>()
                .GetFlag(node.TargetId),
            ConditionKind.BeatStage => CompareBeatStage(node, ctx),
            ConditionKind.ConductScore => CompareConduct(node, ctx),
            ConditionKind.ThreadResolved => CompareThread(node, ctx),
            _ => throw new ArgumentOutOfRangeException()
        };
    }
}
```

### ActionExecutor (internal)

```csharp
sealed class ActionExecutor
{
    readonly INarrativeServiceRegistry _registry;

    public void Execute(Action action, RuleContext ctx)
    {
        switch (action.Kind)
        {
            case ActionKind.SetStoryFlag:
                _registry.GetRequired<IStoryStateService>()
                    .SetFlag(action.TargetId, action.BoolValue);
                break;
            case ActionKind.AdvanceBeat:
                _registry.GetRequired<IStoryStateService>()
                    .AdvanceBeat(action.TargetId);
                break;
            case ActionKind.RegisterFact:
                _registry.GetRequired<IFactService>()
                    .Register(action.FactPayload);
                break;
            case ActionKind.AddThreadEvidence:
                _registry.GetRequired<IThreadService>()
                    .AddEvidence(action.ThreadId, action.Evidence);
                break;
            // `[FULL]` economy, relationship, etc.
        }
    }
}
```

### RuleContext

```csharp
class RuleContext
{
    public string ActorId;
    public string DialogueGraphId;
    public string CurrentNodeId;
    public SimulationSnapshot Snapshot; // read-only aggregate `[FULL]`
}
```

---

## Definition assets

| Asset | Source |
|---|---|
| `Rule` / `Condition` / `Action` | Embedded in dialogue, gates, beats |
| `RuleDefinition` | Standalone content `[FULL]` |

MVP: inline rule trees in dialogue graph definitions. Phase 12: script compiler emits rule AST.

---

## Runtime state

`RuleEngine` is stateless. All mutations go to target services' state DTOs. Optional evaluation cache **disabled** in MVP to avoid stale Gating reads.

---

## Core algorithms

### Evaluate (Gating)

1. Walk condition tree depth-first
2. Leaf nodes query single service (no cross-service writes)
3. Short-circuit `All`/`Any`/`Not`
4. Return boolean — **no actions**

### ExecuteActions (Content)

1. Iterate actions in order
2. Dispatch to `ActionExecutor`
3. Each action publishes domain events via target service
4. Do not call `IChronicleService` — chronicle reacts to events

### Dialogue integration

```csharp
// DialogueService — Gating
if (!_rules.Evaluate(choice.VisibilityRule, ctx)) return false;

// DialogueService — Content
_rules.ExecuteActions(choice.Actions, ctx);
```

### Thread vs beat actions

- `AdvanceBeat` → story progression unit
- `AddThreadEvidence` → inquiry object
- Never a single action that conflates both

---

## Event contracts

Rule engine does not publish events directly. Target services publish:

| Action | Event |
|---|---|
| SetStoryFlag | `StoryFlagChanged` |
| AdvanceBeat | `StoryBeatAdvanced` |
| RegisterFact | `FactRegistered` |
| AddThreadEvidence | `ThreadEvidenceAdded` |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `INarrativeServiceRegistry` | Service dispatch |
| All condition target services | Leaf evaluation |

| Used by | Reason |
|---|---|
| `IDialogueService` | Choice visibility + effects |
| `IGateService` | Wraps rules with gate metadata |
| `IStoryStateService` | Beat triggers `[FULL]` |
| Content scripts | Compiled rules Phase 12 |

---

## MVP scope (Phase 5)

- [ ] `RuleEngine` + evaluator + executor
- [ ] Conditions: FactActive, StoryFlag, BeatStage, All/Any/Not
- [ ] Actions: SetStoryFlag, AdvanceBeat, RegisterFact
- [ ] Dialogue integration
- [ ] ≥15 tests including fact-driven gate slice

---

## Full scope

- `[FULL]` Full action catalog (economy, social, emotion)
- `[FULL]` ScriptCompiler rule emission
- `[FULL]` Evaluation trace for debug Phase 14
- `[FULL]` Thread theory conditions

---

## File tree

```text
Packages/NarrativeFramework/Rules/Engine/IRuleEngine.cs
Packages/NarrativeFramework/Rules/Engine/RuleEngine.cs
Packages/NarrativeFramework/Rules/Engine/ConditionEvaluator.cs
Packages/NarrativeFramework/Rules/Engine/ActionExecutor.cs
Packages/NarrativeFramework/Rules/Engine/Rule.cs
Packages/NarrativeFramework/Rules/Engine/Condition.cs
Packages/NarrativeFramework/Rules/Engine/Action.cs
Packages/NarrativeFramework/Rules/Engine/RuleContext.cs
Packages/NarrativeFramework/Tests/EditMode/Rules/RuleEngineEvaluateTests.cs
Packages/NarrativeFramework/Tests/EditMode/Rules/RuleEngineActionTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Evaluate_AllAndAny` | Boolean composition |
| `Evaluate_FactActive` | IFactService integration |
| `Evaluate_StoryFlag` | Flag condition |
| `Evaluate_BeatStage` | Stage comparison |
| `Execute_SetFlag` | Mutation + event |
| `Execute_AdvanceBeat` | Beat stage change |
| `Evaluate_NoSideEffects` | Flag unchanged after Evaluate-only |
| `Dialogue_DelegatesToRuleEngine` | Integration slice |

**Exit:** ≥15 tests; gate blocks until fact registered (kernel test).

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) ST-05, SCR-01 (custom plugins Phase 12; RuleContext snapshot Phase 14).

---

## Related documents

- [rules-gate.md](rules-gate.md), [story-dialogue.md](story-dialogue.md)
- [runtime-kernel.md](runtime-kernel.md), [sim-fact.md](sim-fact.md)
- [contracts.md](contracts.md)
