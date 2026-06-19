# Presentation UI — Implementation Architecture

- Spec: [present-ui.md](../systems/present-ui.md)
- Glossary: [Dialogue presenter](../terminology-glossary.md), [Chronicle presenter](../terminology-glossary.md)
- Roadmap: [Phase 13](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) — `IUIShell`, `IDialoguePresenter`, `IChroniclePresenter`, `IBeliefView`
- Shared types: [data-model.md](data-model.md)

---

## Design rationale

### Why Presentation is a bridge, not a core dependency

Story, Cognition, and Ledger modules must compile and run in Edit Mode without Canvas, TextMeshPro, or scene objects. If `IDialogueService` referenced `UnityEngine.UI`, every kernel test would require Play Mode. Presentation therefore **pulls** narrative state and **pushes** view models to bound presenters — never the reverse.

**Forbidden:** any core asmdef (`Story`, `Cognition`, `Ledger`, `Simulation`, `Rules`, `Social`, `Content`) referencing `NarrativeFramework.Presentation`.

### Why headless presenters

Human eyes are a poor regression gate. Phase 13 requires ≥20 Edit Mode tests that assert dialogue text, choice enablement, belief phase, and chronicle sections **without** rendering. Headless presenters implement the same contracts as real UI widgets but store the last `Show()` payload in memory for assertions.

### Why three presenter interfaces instead of one mega-UI

Dialogue, chronicle, and belief screens have different refresh cadences and test surfaces. Splitting contracts keeps tests focused and allows samples to skin each screen independently (NoirSample detective notebook vs SciFiSample mission log).

### Why view models are immutable snapshots

Core services mutate state during kernel ticks. Presenters receive **copies** at presentation time so UI does not observe half-applied social or gating changes mid-tick.

---

## Implementation summary

| MVP (Phase 13) | Full (Phase 15+) |
|---|---|
| `IUIShell` + three presenter bindings | Screen stack with modal overlays |
| `UIShell` orchestrator in Presentation | Animation/tween hooks `[FULL]` |
| `HeadlessDialoguePresenter`, etc. | Sample Canvas implementations |
| View model mappers from Story/Ledger/Cognition | Roll UI, tooltips, interrupt layer `[FULL]` |
| Edit Mode capture tests | Play Mode smoke (optional scripted) |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Presentation`
- **Namespace:** `NarrativeFramework.Presentation.UI`
- **References:** Runtime, Content, Rules, Cognition, Social, Story, Ledger, Simulation

**Does not reference:** Debug (Debug references Presentation for overlays).

### Dependency direction

```text
Story / Ledger / Cognition  →  (via registry, at runtime)  →  Presentation mappers
Presentation  →  IDialoguePresenter / IChroniclePresenter / IBeliefView  →  Canvas (samples)
```

Core services expose data through `I*Service` contracts only. Presentation subscribes to `IEventBus` or polls registry **after** tick phases — never injected into Story constructors.

---

## Public API

Signatures are SSOT in [contracts.md](contracts.md):

```csharp
interface IUIShell
{
    void BindDialoguePresenter(IDialoguePresenter presenter);
    void BindChroniclePresenter(IChroniclePresenter presenter);
    void BindBeliefView(IBeliefView beliefView);
    void PushScreen(string screenId);
    void PopScreen();
}

interface IDialoguePresenter { void Show(DialogueViewModel model); }
interface IChroniclePresenter { void Show(ChronicleView model); }
interface IBeliefView { void Show(BeliefViewModel model); }
```

Module-specific notes:

| Interface | View model source | Refresh trigger |
|---|---|---|
| `IDialoguePresenter` | `IDialogueService` active node + `ILocaleService` | `DialogueStarted`, `DialogueAdvanced`, choice selected |
| `IChroniclePresenter` | `IChronicleService.GetSections()` projection | `ChronicleUpdated`, thread/clue mutations |
| `IBeliefView` | `IBeliefService` active belief slots | `BeliefPhaseChanged`, belief internalized |

---

## Internal implementation

### UIShell

Central binder and screen stack. Does not render — delegates to bound presenters.

```csharp
namespace NarrativeFramework.Presentation.UI
{
    sealed class UIShell : IUIShell
    {
        IDialoguePresenter _dialogue;
        IChroniclePresenter _chronicle;
        IBeliefView _belief;
        readonly Stack<string> _screenStack = new();

        public void BindDialoguePresenter(IDialoguePresenter presenter)
            => _dialogue = presenter ?? throw new ArgumentNullException(nameof(presenter));

        public void BindChroniclePresenter(IChroniclePresenter presenter)
            => _chronicle = presenter ?? throw new ArgumentNullException(nameof(presenter));

        public void BindBeliefView(IBeliefView beliefView)
            => _belief = beliefView ?? throw new ArgumentNullException(nameof(beliefView));

        public void PushScreen(string screenId) => _screenStack.Push(screenId);
        public void PopScreen() { if (_screenStack.Count > 0) _screenStack.Pop(); }

        internal void PresentDialogue(DialogueViewModel model) => _dialogue?.Show(model);
        internal void PresentChronicle(ChronicleView model) => _chronicle?.Show(model);
        internal void PresentBelief(BeliefViewModel model) => _belief?.Show(model);
    }
}
```

### UI presentation coordinator

Subscribes to domain events; maps service state → view models; calls `UIShell` internal present methods.

```csharp
sealed class UIPresentationCoordinator
{
    readonly INarrativeServiceRegistry _registry;
    readonly UIShell _shell;
    readonly DialogueViewModelMapper _dialogueMapper;
    readonly ChronicleViewMapper _chronicleMapper;
    readonly BeliefViewModelMapper _beliefMapper;

    public void OnDialogueAdvanced()
    {
        var dialogue = _registry.GetRequired<IDialogueService>();
        var locale = _registry.GetRequired<ILocaleService>();
        _shell.PresentDialogue(_dialogueMapper.Map(dialogue, locale));
    }

    public void OnChronicleUpdated()
    {
        var chronicle = _registry.GetRequired<IChronicleService>();
        chronicle.RefreshProjection();
        _shell.PresentChronicle(_chronicleMapper.Map(chronicle));
    }
}
```

### View model mappers

Pure functions / small classes in `Presentation/Mappers/`:

```csharp
static class DialogueViewModelMapper
{
    public static DialogueViewModel Map(IDialogueService dialogue, ILocaleService locale)
    {
        var node = dialogue.GetActiveNode();
        return new DialogueViewModel
        {
            Speaker = locale.Resolve(node.SpeakerKey, locale.GetFallbackLocale(), null),
            Text = locale.Resolve(node.TextKey, locale.GetFallbackLocale(), null),
            Choices = MapChoices(node.Choices, dialogue, locale)
        };
    }
}
```

View model shapes (Presentation-owned DTOs, not in core):

```csharp
class DialogueViewModel { public string Speaker; public string Text; public ChoiceViewModel[] Choices; }
class ChoiceViewModel { public string Id; public string Label; public bool Enabled; }
class BeliefViewModel { public string BeliefId; public string Title; public BeliefPhase Phase; }
// ChronicleView defined with IChronicleService — see ledger-chronicle spec
```

### Headless presenters (test doubles)

```csharp
sealed class HeadlessDialoguePresenter : IDialoguePresenter
{
    public DialogueViewModel LastModel { get; private set; }
    public int ShowCallCount { get; private set; }

    public void Show(DialogueViewModel model)
    {
        LastModel = model;
        ShowCallCount++;
    }
}
```

Mirror pattern for `HeadlessChroniclePresenter` and `HeadlessBeliefView`. Tests bind these via `IUIShell` before triggering dialogue/chronicle/belief flows.

### Null presenters (Phase 0 stub)

`NullDialoguePresenter` etc. satisfy registry wiring when Presentation is not registered; no-op `Show()`.

---

## Definition assets

Presentation does not own content definitions. Optional **UI skin** ScriptableObjects live in content packs:

```csharp
class UISkinDefinition : ContentDefinition
{
    public string BeliefScreenTitleKey;  // content pack: "Detective Notes" vs "Mission Brief"
    public string ChronicleSectionStyleId;
}
```

Framework reads skin keys through `IContentStore`; no genre strings in Presentation code.

---

## Runtime state

| State | Owner | Persisted |
|---|---|---|
| Screen stack | `UIShell` | No (session UI) |
| Last view models | Headless presenters (tests) | No |
| Dialogue/chronicle/belief truth | Core services | Yes via `IPersistenceService` |

Presentation holds **no authoritative simulation state**.

---

## Core algorithms

### Dialogue present flow

1. `IDialogueService` advances graph (Content phase or player input handler)
2. `DialogueAdvanced` event published on `IEventBus`
3. `UIPresentationCoordinator.OnDialogueAdvanced()` runs (after tick or input frame)
4. Mapper resolves locale keys, evaluates choice `Enabled` from `IGateService` / faculty gates
5. `IDialoguePresenter.Show(model)` — headless or Canvas

### Chronicle present flow

1. Thread/clue/fact changes update ledger projection via `IChronicleService`
2. `ChronicleUpdated` event
3. `RefreshProjection()` rebuilds read-only sections (chronicle is **projection**, not source of truth)
4. `IChroniclePresenter.Show(ChronicleView)`

### Belief present flow

1. `IBeliefService` transitions phase (Hint → Internalized, etc.)
2. `BeliefPhaseChanged` event
3. Mapper builds `BeliefViewModel` with localized title
4. `IBeliefView.Show(model)`

### Choice enablement (why in Presentation mapper, not Story)

Story exposes raw choice nodes with condition IDs. Presentation mapper calls `IGateService` / `IRollService` preview APIs to set `ChoiceViewModel.Enabled`. Story never knows about UI gray-out styling.

---

## Event contracts

Presentation **subscribes**; it does not publish simulation events.

| Event | Handler | Presenter |
|---|---|---|
| `DialogueStarted` | Map + show | `IDialoguePresenter` |
| `DialogueAdvanced` | Map + show | `IDialoguePresenter` |
| `ChronicleUpdated` | Map + show | `IChroniclePresenter` |
| `BeliefPhaseChanged` | Map + show | `IBeliefView` |
| `RollResolved` | `[FULL]` roll popup | Roll presenter Phase 14+ |

Tick phase: presentation reactions run **after** `ISimulationKernel.Tick()` completes for the frame, or on dedicated input events — not inside Gating phase handlers.

---

## Integration matrix

| Depends on | Usage |
|---|---|
| `IDialogueService` | Active node, choices |
| `IChronicleService` | Section/entry projection |
| `IBeliefService` | Slot state, phases |
| `ILocaleService` | Resolved strings |
| `IGateService` | Choice/interaction enabled state |
| `IEventBus` | Refresh triggers |

| Consumed by | Usage |
|---|---|
| Samples (Noir/SciFi) | Canvas presenter implementations |
| Integration tests | Headless bindings |
| Debug (Phase 14) | UI event trace overlay |

**Forbidden couplings:**

| From | To | Reason |
|---|---|---|
| Story | Presentation | Breaks headless kernel tests |
| Cognition | Presentation | Same |
| Ledger | Presentation | Chronicle service stays core; view is bridge |
| Presentation | Canvas types in Runtime | Keep Presentation free of scene refs in mappers |

---

## MVP scope (Phase 13)

- [ ] `IUIShell`, `UIShell`, three presenter interfaces per contracts.md
- [ ] `DialogueViewModelMapper`, `ChronicleViewMapper`, `BeliefViewModelMapper`
- [ ] `UIPresentationCoordinator` + event subscriptions
- [ ] `HeadlessDialoguePresenter`, `HeadlessChroniclePresenter`, `HeadlessBeliefView`
- [ ] Register `IUIShell` in bootstrap; samples bind headless in tests
- [ ] ≥20 Edit Mode tests via headless capture (roadmap exit)

---

## Full scope

- `[FULL]` Roll UI presenter + `IRollResultView`
- `[FULL]` Tooltip and faculty interjection popups
- `[FULL]` Interrupt stacking and priority queue
- `[FULL]` Accessibility settings adapter (font scale, reduced motion)
- `[FULL]` UI element pooling for large chronicles

---

## File tree

```text
Packages/NarrativeFramework/Presentation/UI/IUIShell.cs
Packages/NarrativeFramework/Presentation/UI/UIShell.cs
Packages/NarrativeFramework/Presentation/UI/IDialoguePresenter.cs
Packages/NarrativeFramework/Presentation/UI/IChroniclePresenter.cs
Packages/NarrativeFramework/Presentation/UI/IBeliefView.cs
Packages/NarrativeFramework/Presentation/Mappers/DialogueViewModelMapper.cs
Packages/NarrativeFramework/Presentation/Mappers/ChronicleViewMapper.cs
Packages/NarrativeFramework/Presentation/Mappers/BeliefViewModelMapper.cs
Packages/NarrativeFramework/Presentation/UI/UIPresentationCoordinator.cs
Packages/NarrativeFramework/Presentation/UI/ViewModels/DialogueViewModel.cs
Packages/NarrativeFramework/Presentation/UI/ViewModels/BeliefViewModel.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/HeadlessDialoguePresenterTests.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/HeadlessChroniclePresenterTests.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/HeadlessBeliefViewTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `BindDialoguePresenter_InvokesShow_OnDialogueAdvanced` | Headless `LastModel.Speaker` |
| `ChoiceMapper_DisabledWhenGateBlocks` | `Enabled == false` |
| `ChronicleMapper_ReflectsNewClueEntry` | Section count, entry title |
| `BeliefMapper_PhaseTransition` | `BeliefPhase` on model |
| `LocaleFallback_UsedWhenKeyMissing` | Fallback string in model |
| `UIShell_PushPopScreen_StackOrder` | Screen stack depth |
| `CoreAssembly_DoesNotReferencePresentation` | Asmdef reflection test |

**Exit:** ≥20 Edit Mode tests (roadmap Phase 13); Play Mode smoke optional.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) PR-04, PR-05, NSF-H01 (event-driven presenters; roll UI Phase 14; text scale hooks).

---

## Related documents

- [present-interaction.md](present-interaction.md) — world click affordances
- [present-audio.md](present-audio.md) — voice sync with dialogue presenter
- [ledger-chronicle.md](ledger-chronicle.md) — `IChronicleService` projection
- [integration.md](integration.md) — full loop with headless UI assertions
