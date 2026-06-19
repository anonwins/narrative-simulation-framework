# Presentation Interaction — Implementation Architecture

- Spec: [present-interaction.md](../systems/present-interaction.md)
- Glossary: [InteractionService](../terminology-glossary.md)
- Roadmap: [Phase 13](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) — `IInteractionService`
- UI bridge: [present-ui.md](present-ui.md)

---

## Design rationale

### Why interaction lives in Presentation, not Simulation

World click handling is player-facing affordance wiring. Simulation owns **truth** (facts, actor positions, inventory contents); Presentation owns **what the player can click and what label they see**. Moving interaction into Simulation would drag gate copy, locale strings, and highlight logic into kernel tests.

### Why data-driven interaction nodes

A corpse, locker, and window share no C# subtype — they share a **graph of interaction nodes** authored in content. Genre games define inspect/pry/smell nodes in JSON; framework evaluates requirements uniformly.

### Why core never calls Presentation

`IGateService` and `IFactService` answer "is this legal?" and "what is true?" — they do not know about cursors or highlight shaders. `IInteractionService` in Presentation aggregates gate results + locale + discovery state into `InteractionOption` lists for UI or headless tests.

### Why interaction history matters narratively

Repeat clicks, exhausted nodes, and failed tool attempts are story signals. Presentation tracks **interaction memory** separately from facts so designers can show different copy on second inspect without polluting the fact registry.

---

## Implementation summary

| MVP (Phase 13) | Full (Phase 15+) |
|---|---|
| `IInteractionService` + `InteractionService` | Scene proxy components in samples |
| Content-driven `InteractionDefinition` | Highlight/outline rendering `[FULL]` |
| Requirement evaluation via registry | Animation triggers on execute `[FULL]` |
| Headless option list capture | Point-and-click raycast wiring in samples |
| Execute → effects → facts/events | Tool-combination chains `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Presentation`
- **Namespace:** `NarrativeFramework.Presentation.Interaction`
- **References:** Runtime, Content, Rules, Cognition, Story, Simulation, Ledger

**Forbidden:** Simulation → Presentation.

---

## Public API

See [contracts.md](contracts.md):

```csharp
interface IInteractionService
{
    IReadOnlyList<InteractionOption> GetAvailableInteractions(string worldObjectId);
    void ExecuteInteraction(string worldObjectId, string interactionId);
}
```

Supporting DTOs (Presentation module):

```csharp
class InteractionOption
{
    public string InteractionId;
    public string Label;           // locale-resolved
    public bool Visible;
    public bool Enabled;
    public InteractionAffordance Affordance;  // Visible, Hidden, Suggested, Forbidden
}

enum InteractionAffordance { Visible, Hidden, Suggested, Forbidden }
```

---

## Internal implementation

### InteractionService

```csharp
sealed class InteractionService : IInteractionService
{
    readonly INarrativeServiceRegistry _registry;
    readonly IContentStore _content;
    readonly InteractionHistory _history;
    readonly InteractionRequirementEvaluator _evaluator;

    public IReadOnlyList<InteractionOption> GetAvailableInteractions(string worldObjectId)
    {
        var def = _content.GetRequired<WorldObjectDefinition>(worldObjectId);
        var locale = _registry.GetRequired<ILocaleService>();
        var options = new List<InteractionOption>();

        foreach (var node in def.InteractionNodes)
        {
            var eval = _evaluator.Evaluate(node.Requirements);
            if (!eval.Visible) continue;

            options.Add(new InteractionOption
            {
                InteractionId = node.Id,
                Label = locale.Resolve(node.LabelKey, locale.GetFallbackLocale(), null),
                Visible = eval.Visible,
                Enabled = eval.Enabled,
                Affordance = eval.Affordance
            });
        }
        return options;
    }

    public void ExecuteInteraction(string worldObjectId, string interactionId)
    {
        var def = _content.GetRequired<WorldObjectDefinition>(worldObjectId);
        var node = def.GetNode(interactionId);
        var eval = _evaluator.Evaluate(node.Requirements);

        if (!eval.Enabled)
        {
            ApplyResult(node.FailureResult);
            _history.RecordFailure(worldObjectId, interactionId);
            return;
        }

        ApplyResult(node.SuccessResult);
        _history.RecordSuccess(worldObjectId, interactionId);
    }

    void ApplyResult(InteractionResult result)
    {
        // Delegates to IEffectRunner — registers facts, sets flags, starts dialogue
        var runner = _registry.GetRequired<IInteractionEffectRunner>();
        runner.Apply(result);
    }
}
```

### InteractionRequirementEvaluator

Calls core services only — never UI:

| Requirement type | Core dependency |
|---|---|
| Fact present | `IFactService` |
| Story flag | `IStoryStateService` |
| Faculty threshold | `IFacultyService` |
| Inventory item | `IInventoryService` |
| Discovery state | `IDiscoveryService` (Presentation, sibling module) |
| Gate rule | `IGateService` |

Returns `RequirementEvalResult { Visible, Enabled, Affordance }`.

Hidden-but-disabled options use `Visible=false` until faculty reveals them (`Affordance=Hidden` → `Suggested`).

### InteractionEffectRunner

Bridges interaction results to simulation/story **without** Presentation types leaking backward:

```csharp
interface IInteractionEffectRunner
{
    void Apply(InteractionResult result);
}
```

Implementation registers facts via `IFactService`, publishes `SimEvent`s, calls `IDialogueService.StartDialogue`, marks discoverables via `IDiscoveryService`.

### InteractionHistory

```csharp
sealed class InteractionHistory
{
    readonly HashSet<(string objectId, string interactionId)> _success = new();
    readonly HashSet<(string objectId, string interactionId)> _failure = new();

    public bool WasSuccessful(string objectId, string interactionId) => _success.Contains((objectId, interactionId));
    public int GetAttemptCount(string objectId, string interactionId);
}
```

Persisted in Phase 10 via envelope on `IPersistenceService` — `[FULL]` MVP may be session-only.

### Headless interaction capture

Tests never raycast. They call `GetAvailableInteractions` / `ExecuteInteraction` directly:

```csharp
sealed class InteractionTestHarness
{
    public IReadOnlyList<InteractionOption> LastOptions { get; private set; }

    public void Refresh(IInteractionService service, string objectId)
        => LastOptions = service.GetAvailableInteractions(objectId);
}
```

---

## Definition assets

Content pack authored (`Content` assembly schemas, loaded via `IContentStore`):

```csharp
class WorldObjectDefinition : ContentDefinition
{
    public string DisplayNameKey;
    public InteractionNodeDefinition[] InteractionNodes;
}

class InteractionNodeDefinition
{
    public string Id;
    public string LabelKey;
    public RequirementDefinition[] Requirements;
    public InteractionResultDefinition SuccessResult;
    public InteractionResultDefinition FailureResult;
}
```

`FrameworkTestPack` includes `object_test_desk` with inspect + take nodes for pipeline and interaction tests.

---

## Runtime state

| State | Owner | Notes |
|---|---|---|
| Interaction history | `InteractionService` | Per save slot |
| World object defs | `IContentStore` | Immutable at runtime |
| Facts/flags from effects | Core services | Authoritative |

---

## Core algorithms

### Get available interactions

1. Load `WorldObjectDefinition` by ID
2. For each node, evaluate requirements
3. Filter non-visible nodes
4. Resolve labels through `ILocaleService`
5. Return ordered list (content-defined priority)

### Execute interaction

1. Re-evaluate requirements (race-safe if state changed same frame)
2. If disabled → apply failure result (fail-forward copy, optional bark via audio bridge)
3. If enabled → apply success result:
   - Register facts
   - Set story flags
   - Start dialogue graph
   - Mark discovery
   - Queue roll if node specifies `RollSpec`
4. Publish `InteractionExecuted` on `IEventBus` (Events phase safe)
5. Update history; optionally refresh UI via coordinator

### Faculty-gated reveal

Passive faculty check (Phase 13 MVP): when player enters location or clicks object, `InteractionRequirementEvaluator` runs passive roll via `IRollService`. Success adds visibility to hidden nodes without separate discovery pass — coordinated with [present-discovery.md](present-discovery.md).

---

## Event contracts

| Event | Publisher | Subscribers |
|---|---|---|
| `InteractionExecuted` | `InteractionService` | Audio, UI refresh, debug trace |
| `InteractionFailed` | `InteractionService` | Fail-forward dialogue hooks |
| `FactRegistered` | Effect runner | Discovery, chronicle (core) |

Presentation publishes `InteractionExecuted`; core handlers must not reference Presentation types.

---

## Integration matrix

| Depends on | Usage |
|---|---|
| `IContentStore` | World object + node defs |
| `IGateService`, `IFactService`, `IStoryStateService` | Requirements |
| `IDiscoveryService` | Hidden interactables |
| `IDialogueService` | Effect: start conversation |
| `IRollService` | Gated / faculty nodes |
| `ILocaleService` | Labels |

| Consumed by | Usage |
|---|---|
| Sample scene controllers | Click → `ExecuteInteraction` |
| `FullLoopIntegrationTests` | Headless investigate object |
| Debug module | Interaction history panel |

**Forbidden:** `Story` → `IInteractionService`.

---

## MVP scope (Phase 13)

- [ ] `IInteractionService`, `InteractionService`
- [ ] `WorldObjectDefinition`, `InteractionNodeDefinition` in Content
- [ ] `InteractionRequirementEvaluator` (fact, flag, faculty, inventory)
- [ ] `IInteractionEffectRunner` wiring to facts/dialogue
- [ ] `InteractionExecuted` event
- [ ] Headless tests on `FrameworkTestPack` objects
- [ ] ≥5 interaction-specific Edit Mode tests (part of Phase 13 ≥20 total)

---

## Full scope

- `[FULL]` Tool-on-target combination matrix
- `[FULL]` Repeat cooldowns and bark suppression
- `[FULL]` Scene `InteractableProxy` MonoBehaviour in samples only
- `[FULL]` Highlight/outline affordance rendering

---

## File tree

```text
Packages/NarrativeFramework/Presentation/Interaction/IInteractionService.cs
Packages/NarrativeFramework/Presentation/Interaction/InteractionService.cs
Packages/NarrativeFramework/Presentation/Interaction/InteractionRequirementEvaluator.cs
Packages/NarrativeFramework/Presentation/Interaction/InteractionHistory.cs
Packages/NarrativeFramework/Presentation/Interaction/InteractionEffectRunner.cs
Packages/NarrativeFramework/Content/Definitions/WorldObjectDefinition.cs
Packages/NarrativeFramework/Content/Definitions/InteractionNodeDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/InteractionServiceTests.cs
Samples~/FrameworkTestPack/Objects/object_test_desk.asset
```

---

## Test plan

| Test | Asserts |
|---|---|
| `GetAvailable_ReturnsInspect_WhenNoRequirements` | Option count, label |
| `Execute_RegistersFact_OnSuccess` | `IFactService` contains fact |
| `HiddenNode_NotVisible_UntilFacultyPass` | Visibility flip after roll seed |
| `DisabledNode_AppliesFailureResult` | Flag set, history failure |
| `Locale_ResolvesLabelKey` | Label string |
| `Core_DoesNotReferenceInteractionService` | Asmdef guard |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) PR-03 (interaction history in save Phase 10).

---

## Related documents

- [present-discovery.md](present-discovery.md) — hidden object reveals
- [present-ui.md](present-ui.md) — dialogue after interaction effect
- [rules-gate.md](rules-gate.md) — requirement evaluation
- [integration.md](integration.md) — full investigation beat
