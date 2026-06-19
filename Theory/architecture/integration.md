# Full Integration — Implementation Architecture

- Spec: [runtime-kernel.md](../runtime-kernel.md)
- Roadmap: [Phase 14](../development-roadmap.md)
- Kernel: [runtime-kernel.md](runtime-kernel.md) (architecture)
- Unity host: [unity-host.md](unity-host.md) — `NsfSession`, tick driver, platform boundaries
- Presentation: [present-ui.md](present-ui.md), [present-interaction.md](present-interaction.md)
- Editor: [editor.md](editor.md) — `setup-nsf-sample.ps1`
- Debug: [debug.md](debug.md)

---

## Design rationale

### Why Phase 14 is the closure gate

Phases 0–13 prove modules in isolation. Phase 14 proves the **closed kernel loop** — the causal chain the spec describes:

```text
Fact → Dialogue → Relationship → Event → Gate → Chronicle
```

If integration fails, the fault is wiring or phase order, not a single module's unit tests.

### Why Edit Mode integration tests precede human play

`FullLoopIntegrationTests` simulate a complete investigation beat without Canvas clicks or audio hardware. They bind headless presenters, drive exploration/interaction APIs directly, and assert facts, chronicle entries, and dialogue models after each kernel tick.

### Why setup automation is part of integration

A green unit suite with a broken generated scene still blocks real validation. `setup-nsf-sample.ps1` proves the Editor pipeline, FrameworkTestPack, and Presentation bootstrap compose into loadable `NsfSampleScene.unity` — same registry wiring integration tests use, plus minimal scene hierarchy.

### Why core must not reference Presentation

Integration tests **wire** Presentation from the test assembly (allowed to reference all modules). Production kernel bootstrap for headless Edit Mode may register Presentation services — but Story/Simulation/Ledger code paths never `using NarrativeFramework.Presentation`. Tests verify asmdef guards explicitly.

---

## Implementation summary

| MVP (Phase 14) | Full (Phase 17+) |
|---|---|
| `FullLoopIntegrationTests` ≥10 scenarios | Both sample playthrough tests |
| Closed loop kernel with real services | Performance budget assertions |
| `NsfIntegrationBootstrap` test registry | CI matrix Unity versions |
| `setup-nsf-sample.ps1` exit 0 | Noir + SciFi setup scripts |
| Total Edit Mode tests ≥200 | Play Mode nightly `[FULL]` |
| UPM `package.json` distribution | Published registry `[FULL]` |

---

## Assembly and namespace

- **Tests assembly:** `NarrativeFramework.Tests` (EditMode)
- **Namespace:** `NarrativeFramework.Tests.Integration`
- **Bootstrap helper:** `NarrativeFramework.Tests.Integration.NsfIntegrationBootstrap`
- **References:** All runtime + Presentation + Debug (Editor tests only)

Integration code lives in **Tests**, not Runtime — keeps player surface clean.

**Composition root split:** Production and samples use `NsfSession` in `Runtime.Host` ([unity-host.md](unity-host.md)). Integration tests use `NsfIntegrationBootstrap`, which **must delegate** to the same factory graph as `NsfSession.Create` — recommended shared helper: `NsfSampleCompositionRoot.BuildRegistry(manifest, options)` called from both `NsfSession` and `NsfIntegrationBootstrap`. Tests never duplicate wiring logic.

---

## Public API

No new player-facing API. Integration exposes test infrastructure:

```csharp
static class NsfIntegrationBootstrap
{
    /// <summary>
    /// Builds Phase-14 registry: real kernel, facts, dialogue, social, gates, chronicle, presentation bridges.
    /// </summary>
    public static INarrativeServiceRegistry CreateFullLoopRegistry(
        ContentPackManifest manifest,
        IntegrationTestOptions options = null);

    public static void BindHeadlessPresenters(INarrativeServiceRegistry registry, HeadlessPresenterBundle bundle);
}

class HeadlessPresenterBundle
{
    public HeadlessDialoguePresenter Dialogue { get; } = new();
    public HeadlessChroniclePresenter Chronicle { get; } = new();
    public HeadlessBeliefView Belief { get; } = new();
}
```

---

## Internal implementation

### Closed kernel loop

Orchestration matches [runtime-kernel.md](runtime-kernel.md) six phases:

| Step | Phase | Integration assertion |
|---|---|---|
| 1 | Setup | Load FrameworkTestPack; seed flags/facts |
| 2 | Facts | Register investigation facts from interaction |
| 3 | Interpretation | Dialogue node active; voice hooks fire |
| 4 | Social | Relationship delta after dialogue choice |
| 5 | Events | Location/interaction events flushed |
| 6 | Gating | Blocked/allowed content verified |
| 7 | Content | Script effects; new facts queued |
| 8 | Presentation | Headless models updated post-tick |
| 9 | Chronicle | Projection contains clue/task entries |

Single investigation beat (canonical scenario):

```text
Player at location_test_office
→ Execute interaction inspect desk
→ Discovery reveals discoverable_test_note
→ Fact registered
→ Dialogue branch unlocked (gate passes)
→ Choose dialogue option affecting relationship
→ Kernel tick(s)
→ Chronicle shows new clue section
→ Headless dialogue/chronicle models match expected strings
```

### FullLoopIntegrationTests structure

```csharp
[TestFixture]
class FullLoopIntegrationTests
{
    INarrativeServiceRegistry _registry;
    HeadlessPresenterBundle _presenters;
    ISimulationKernel _kernel;

    [SetUp]
    public void SetUp()
    {
        var manifest = ContentPackManifestLoader.LoadFrameworkTestPack();
        _registry = NsfIntegrationBootstrap.CreateFullLoopRegistry(manifest);
        _presenters = new HeadlessPresenterBundle();
        NsfIntegrationBootstrap.BindHeadlessPresenters(_registry, _presenters);
        _kernel = _registry.GetRequired<ISimulationKernel>();
        _kernel.Initialize(_registry);
    }

    [Test]
    public void InvestigationBeat_DeskInspect_RegistersFactAndUpdatesChronicle()
    {
        var exploration = _registry.GetRequired<IExplorationService>();
        var interaction = _registry.GetRequired<IInteractionService>();

        exploration.Enter("location_test_office");
        interaction.ExecuteInteraction("object_test_desk", "interaction_inspect");

        _kernel.Tick();  // full loop

        Assert.That(_registry.GetRequired<IFactService>().HasFact("fact_test_clue"), Is.True);
        Assert.That(_presenters.Chronicle.LastModel.Sections, Is.Not.Empty);
        Assert.That(_presenters.Dialogue.ShowCallCount, Is.GreaterThanOrEqualTo(0));
    }
}
```

### Scenario catalog (≥10)

| # | Scenario | Proves |
|---|---|---|
| 1 | Desk inspect → fact → chronicle | Interaction + fact + ledger |
| 2 | Gate blocks dialogue until flag | Rules + story state |
| 3 | Gate opens after fact | Fact → gate chain |
| 4 | Relationship shift on choice | Social phase |
| 5 | Fail-forward roll branch | Cognition + story |
| 6 | Discovery faculty reveal | Discovery + presentation |
| 7 | Location travel unlock | Exploration + gate |
| 8 | Belief phase UI model | Belief + headless view |
| 9 | Audio call log sequence | Audio coordinator |
| 10 | Multi-tick pacing delay | Pacing + retry next tick |
| 11 | Persistence round-trip `[FULL]` | Save/load mid-beat |
| 12 | Thread milestone → chronicle section | Thread + chronicle |

Each scenario uses deterministic roll seeds (`IntegrationTestOptions.FixedSeed`).

### NsfIntegrationBootstrap service graph

Delegates to `NsfSampleCompositionRoot.BuildRegistry` (shared with `NsfSession`) — see [unity-host.md](unity-host.md). Replaces Phase 0 `Null*` stubs with real implementations through Phase 13:

```csharp
static INarrativeServiceRegistry CreateFullLoopRegistry(ContentPackManifest manifest, IntegrationTestOptions options)
{
    var registry = new NarrativeServiceRegistry();
    var content = ContentRegistry.FromManifest(manifest);

    registry.Register<IContentStore>(content);
    registry.Register<IFactService>(new FactService());
    registry.Register<IEventBus>(new EventBus(registry));
    registry.Register<IDialogueService>(new DialogueService(registry));
    registry.Register<IStoryStateService>(new StoryStateService());
    registry.Register<IRelationshipService>(new RelationshipService());
    registry.Register<IGateService>(new GateService(registry));
    registry.Register<IRuleEngine>(new RuleEngine(registry));
    registry.Register<IChronicleService>(new ChronicleService(registry));
    registry.Register<IThreadService>(new ThreadService(registry));
    registry.Register<IRollService>(new RollService(options?.FixedSeed ?? 42));
    registry.Register<IBeliefService>(new BeliefService(registry));
    registry.Register<ILocationService>(new LocationService());
    registry.Register<ILocaleService>(new LocaleService(content));

    // Presentation bridges — wired in test assembly, not referenced by Story internals
    registry.Register<IUIShell>(new UIShell());
    registry.Register<IInteractionService>(new InteractionService(registry, content));
    registry.Register<IDiscoveryService>(new DiscoveryService(registry, content));
    registry.Register<IExplorationService>(new ExplorationService(registry, content));
    registry.Register<IAudioNarrativeService>(new AudioNarrativeService(content));

    var kernel = new SimulationKernel();
    registry.Register<ISimulationKernel>(kernel);

    WirePresentationCoordinators(registry);
    return registry;
}
```

### WirePresentationCoordinators

Subscribes `UIPresentationCoordinator`, `AudioNarrativeCoordinator`, `AreaUnlockController` to `IEventBus` after all services registered — single composition root for integration.

### Asmdef guard tests

```csharp
[Test]
public void CoreAssemblies_DoNotReferencePresentation()
{
    Assert.That(AssemblyReferences.HasReference(typeof(StoryStateService).Assembly, typeof(UIShell).Assembly), Is.False);
    // Repeat for Cognition, Ledger, Simulation, Rules, Story
}
```

---

## setup-nsf-sample.ps1 integration

Phase 14 automated gate (sequential after Edit Mode tests):

```powershell
.\run-nsf-edit-mode-tests.ps1    # Phase 0–14: must pass first
.\setup-nsf-sample.ps1             # Phase 14+: scene generation
```

Script flow ([editor.md](editor.md)):

1. Invoke Unity batchmode via `invoke-unity.ps1`
2. `NsfSampleSetupBatch.RunFromCommandLine`
3. `NsfSampleSceneBuilder.Build()` using same bootstrap logic as `NsfIntegrationBootstrap` (shared helper `NsfSampleCompositionRoot` recommended)
4. Write `SetupReport.json`:

```json
{
  "allPassed": true,
  "steps": [
    { "name": "CreateScene", "passed": true },
    { "name": "BootstrapRegistry", "passed": true },
    { "name": "LoadFrameworkTestPack", "passed": true },
    { "name": "WirePresenters", "passed": true },
    { "name": "SaveScene", "passed": true }
  ],
  "error": null
}
```

5. Exit 0 iff `allPassed: true`

**Roadmap:** Do not require this script before Phase 14.

---

## Definition assets

Integration tests load exclusively from `Samples~/FrameworkTestPack/` for Phase 14 MVP. Noir/SciFi samples added Phase 15–16 ([samples.md](samples.md)).

---

## Runtime state

Integration tests use fresh registry per test (`[SetUp]`). No shared static simulation state.

Optional `[TearDown]` dumps `IDebugTraceSink` buffer on failure.

---

## Core algorithms

### Full loop tick after player action

1. Presentation service handles input API (interaction/exploration/dialogue choice)
2. Effects queue fact registrations, flag changes, events
3. `_kernel.Tick()` runs Facts → Content
4. Coordinators push headless view models
5. `IChronicleService.RefreshProjection()`
6. Test asserts registry + presenter capture state

### Multi-tick pacing scenario

1. Action triggers legal but paced content
2. First tick: gate delays; chronicle unchanged
3. Second tick: pacing window open; content fires
4. Assert single fire via event count

### Failure diagnostics

On assertion failure, test helper dumps:

- Last 20 event trace entries
- Active facts list
- Current dialogue node ID
- Roll log last entry

Uses `IDebugTraceSink` when Editor present.

---

## Event contracts

Full loop validates ordering:

| Event | Must occur before |
|---|---|
| `FactRegistered` | Gate evaluation same tick (Facts phase) |
| `DialogueAdvanced` | Relationship change same or next Social phase |
| `InteractionExecuted` | Fact effect same tick |
| `ChronicleUpdated` | After thread/fact projection refresh |

Violations indicate kernel phase regression — catch with dedicated ordering tests from Phase 1 plus integration scenarios.

---

## Integration matrix

| Module | Role in closed loop |
|---|---|
| Simulation | Facts, events, location, time |
| Story | Dialogue, flags |
| Social | Relationships |
| Cognition | Rolls, beliefs |
| Rules | Gates, rule engine |
| Ledger | Chronicle, threads |
| Presentation | Bridges only — headless in tests |
| Content | FrameworkTestPack defs |
| Debug | Trace on failure |
| Editor | Scene setup script |

**Dependency rule:** Story → Presentation forbidden; Tests → all allowed.

---

## MVP scope (Phase 14)

- [ ] `NsfIntegrationBootstrap` + `HeadlessPresenterBundle`
- [ ] `FullLoopIntegrationTests` ≥10 scenarios
- [ ] `NsfSampleCompositionRoot` shared with scene builder
- [ ] `setup-nsf-sample.ps1` + `SetupReport.json`
- [ ] Asmdef guard tests
- [ ] Total Edit Mode tests ≥200 (roadmap)
- [ ] UPM `package.json` for local file package
- [ ] Roadmap Phase 14 status updated

---

## Full scope

- `[FULL]` Play Mode smoke loading `NsfSampleScene.unity`
- `[FULL]` Persistence mid-beat scenario
- `[FULL]` CI runs setup + both sample packs (Phase 17)
- `[FULL]` Package tarball export

---

## File tree

```text
Packages/NarrativeFramework/Tests/EditMode/Integration/NsfIntegrationBootstrap.cs
Packages/NarrativeFramework/Tests/EditMode/Integration/HeadlessPresenterBundle.cs
Packages/NarrativeFramework/Tests/EditMode/Integration/IntegrationTestOptions.cs
Packages/NarrativeFramework/Tests/EditMode/Integration/FullLoopIntegrationTests.cs
Packages/NarrativeFramework/Tests/EditMode/Integration/AsmdefGuardTests.cs
Packages/NarrativeFramework/Tests/EditMode/Integration/IntegrationFailureDiagnostics.cs
Packages/NarrativeFramework/Runtime/Composition/NsfSampleCompositionRoot.cs
setup-nsf-sample.ps1
SetupReport.json
run-nsf-edit-mode-tests.ps1
Assets/Scenes/NsfSampleScene.unity
```

---

## Test plan

| Test / Gate | Asserts |
|---|---|
| `FullLoop_*` (≥10) | End-to-end beat outcomes |
| `AsmdefGuardTests` | No core → Presentation |
| `TickOrderRegression` | Facts before gating with real services |
| `run-nsf-edit-mode-tests.ps1` | ≥200 tests pass |
| `setup-nsf-sample.ps1` | exit 0, allPassed true |

**Exit criteria (roadmap Phase 14):**

- Integration suite ≥10 scenarios green
- setup script exit 0 + SetupReport.json `allPassed: true`
- Total Edit Mode tests ≥200

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) P-04, P-08.

---

## Related documents

- [runtime-kernel.md](runtime-kernel.md) — tick phases
- [present-ui.md](present-ui.md) — headless presenters
- [editor.md](editor.md) — setup batch
- [samples.md](samples.md) — FrameworkTestPack
- [debug.md](debug.md) — trace tooling
