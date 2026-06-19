# Samples — Implementation Architecture

- Roadmap: [Phase 2](../development-roadmap.md) (FrameworkTestPack), [Phase 15](../development-roadmap.md) (NoirSample), [Phase 16](../development-roadmap.md) (SciFiSample)
- Foundation: [foundation.md](foundation.md) — `Samples~/` layout
- Content: [data-model.md](data-model.md), [content-pipeline.md](content-pipeline.md)
- Integration: [integration.md](integration.md)
- Genre mapping: [appendix-detective-noir-mapping.md](../appendix-detective-noir-mapping.md)

---

## Design rationale

### Why three sample tiers exist

| Sample | Purpose | Phase |
|---|---|---|
| **FrameworkTestPack** | Data-only minimal defs for pipeline + unit tests | 2 |
| **NoirSample** | First validation game — detective noir, **3D** | 15 |
| **SciFiSample** | Reusability proof — sci-fi, **2D**; with Noir, covers all NSF features | 16 |

Framework code must never branch on genre. Samples prove that **content alone** differentiates games.

### Why FrameworkTestPack is not a game

Phase 2–14 tests need stable IDs (`actor_test_*`, `fact_test_*`) without narrative prose churn. FrameworkTestPack is **agent-authored fixture data** — smallest graph that exercises pipeline validation, JSON round-trip, and integration beats.

### Why Noir and SciFi are sibling repos

**Locked:** [decisions-log.md](../decisions-log.md) P-07 — `NoirSample/` and `SciFiSample/` as **sibling repos** referencing local `"file:Packages/NarrativeFramework"`. FrameworkTestPack stays in-package for Phase 2–14 tests.

UPM optional import copies may mirror sample content; canonical sample **games** live in sibling repos for full scenes, art, and CI.

### Sample presentation strategy (decided)

| Sample | Dimension | Phase | Role |
|---|---|---|---|
| **NoirSample** | **3D** + **uGUI** HUD | 15 | Walkable crime scenes; relationship **abstract labels**; locale **en + fr** |
| **SciFiSample** | **2D** + **UITK** terminal | 16 | Hotspot map; relationship **numeric meters**; locale **en** |

**NSF core stays dimension-agnostic.** The two samples differ in **Presentation bridges only** (how `IExplorationService`, `IInteractionService`, etc. map to Unity).

**Combined coverage rule:** Noir + SciFi together must exercise **every NSF module and Presentation bridge** at least once. Neither sample needs every feature at maximum depth alone — the pair is the completeness proof (showcase + reference usage). See § Feature coverage matrix.

### Why zero genre logic in NarrativeFramework

Exit criteria Phase 15–16: no `NarrativeFramework` code references noir/scifi nouns. Samples hold faculty labels, conduct axes, and UI skin strings. Architecture tests scan assemblies for forbidden genre tokens.

---

## Implementation summary

| FrameworkTestPack | NoirSample (3D) | SciFiSample (2D) |
|---|---|---|
| Data only, no scene | Full playable 3D scene | Full playable 2D scene |
| ~15–30 definitions | Content caps per roadmap | Different faculty grid |
| Pipeline + integration | Playthrough Edit Mode test | Cross-pack compile + coverage test |
| `Samples~/FrameworkTestPack/` | `Samples~/NoirSample/` | `Samples~/SciFiSample/` |

---

## Feature coverage matrix (Noir + SciFi = full NSF)

Primary showcase sample for each area. Both samples still run kernel, dialogue, gates, chronicle, and thread resolution.

| Area | Primary sample | How it is exercised |
|---|---|---|
| **Exploration (3D scenes)** | Noir | Walkable locations, scene load on `Enter(locationId)` |
| **Exploration (2D map / hotspots)** | SciFi | Deck plan, click-to-travel, no NavMesh |
| **Interaction (world objects)** | Noir | 3D inspect / talk hotspots |
| **Interaction (UI targets)** | SciFi | Terminal panels, comms buttons |
| **Discovery** | Noir | Clue objects in 3D space |
| **Discovery** | SciFi | Document / log unlock flow |
| **Dialogue + voice text** | Both | Noir: face-to-face; SciFi: comms transcript |
| **Faculty + rolls** | Both | Noir 4×6 grid; SciFi 3×4 grid |
| **Belief + conduct** | Both | Noir cop axes; SciFi diplomatic/ruthless |
| **Emotion** | Noir | Stress during interrogation beats |
| **Relationship + companion** | Noir | Partner NPC in world |
| **Faction + ideology** | SciFi | Station factions, pressure UI |
| **Info-flow** | SciFi | Rumor / intel propagation thread |
| **Economy** | SciFi | Resource / bribe gate (minimal) |
| **Time + pacing** | Both | Noir night deadline; SciFi alert countdown |
| **Facts + gates + rules** | Both | Evidence gates (Noir); clearance gates (SciFi) |
| **Thread + chronicle** | Both | Murder thread vs station alert thread |
| **Fail-forward outcomes** | Both | At least one failed roll path per sample |
| **Persistence save/load** | Both | Playthrough or integration test round-trip |
| **Locale** | SciFi | Secondary locale strings in pack |
| **Inventory** | SciFi | Keycard / tool gating |
| **Content script (`.nsf`)** | SciFi | Scripted gate or beat hook |
| **Audio narrative** | Noir | Ambient + VO stub playback bridge |
| **UI shell + presenters** | Both | Noir detective skin (3D HUD); SciFi terminal skin (2D) |
| **Presentation headless path** | FrameworkTestPack + integration | Edit Mode without Canvas |

**Exit gate (Phase 16):** Automated test or checklist asserts every row has a green path in at least one sample playthrough or integration scenario.

---

## Assembly and namespace

Samples do **not** ship runtime code in core asmdefs.

```text
Samples~/FrameworkTestPack/          → assets + JSON only (Addressables group)
../NoirSample/
  Runtime/NoirSample.Runtime.asmdef  → thin bootstrap, references NSF only
  Editor/NoirSample.Editor.asmdef    → setup pipeline
../SciFiSample/
  Runtime/SciFiSample.Runtime.asmdef
  Editor/SciFiSample.Editor.asmdef
```

Sample assemblies may reference Presentation for Canvas presenters; NSF core may not reference sample assemblies.

---

## FrameworkTestPack (Phase 2+)

### Purpose

- Validate `IContentPipeline` good/broken packs
- JSON import/export round-trip ([data-model.md](data-model.md))
- Feed `FullLoopIntegrationTests` ([integration.md](integration.md))
- Minimal defs for interaction/discovery/exploration architecture tests

### Layout

```text
Samples~/FrameworkTestPack/
  manifest.json
  Actors/actor_test_detective.asset
  Actors/actor_test_witness.asset
  Facts/
  Flags/
  Faculties/
  Locations/location_graph_test.asset
  Locations/location_test_office.asset
  Locations/location_test_alley.asset
  Objects/object_test_desk.asset
  Discovery/discoverable_test_note.asset
  Dialogue/dialogue_test_intro.asset
  Gates/
  Threads/thread_test_main.asset
  Locale/en_test_strings.asset
  Audio/vo_test_greeting.asset
  Audio/ambient_test_office.asset
```

### manifest.json

```json
{
  "packId": "framework_test_pack",
  "schemaVersion": 1,
  "displayName": "Framework Test Pack",
  "localeIds": ["en_test"],
  "entryThreadId": "thread_test_main",
  "startingLocationId": "location_test_office"
}
```

### Content ID conventions

All IDs use glossary prefixes with `test` segment — never `noir_*` or `scifi_*`:

| Prefix | Example |
|---|---|
| `actor_` | `actor_test_detective` |
| `fact_` | `fact_test_clue` |
| `location_` | `location_test_office` |
| `object_` | `object_test_desk` |
| `discoverable_` | `discoverable_test_note` |
| `thread_` | `thread_test_main` |
| `dialogue_` | `dialogue_test_intro` |

### Minimal narrative graph

```text
thread_test_main
  └─ section_test_primary
       ├─ task: inspect desk → fact_test_clue
       ├─ gate: requires fact_test_clue → unlock dialogue_test_followup
       └─ chronicle: clue entry on fact register
```

Placeholder prose acceptable — tests assert IDs and state, not literary quality.

### Pipeline tests

| Test | Input | Expected |
|---|---|---|
| `Validate_GoodPack` | FrameworkTestPack | 0 errors |
| `Validate_MissingRef` | Broken copy | Error with target ID |
| `JsonRoundTrip` | Export → import | Deep equal |

---

## NoirSample (Phase 15)

### Purpose

First **validation game** using [appendix-detective-noir-mapping.md](../appendix-detective-noir-mapping.md) — detective noir vocabulary in **content only**.

**Presentation:** **3D** — walkable crime scenes, spatial interactables, third-person or first-person investigation camera (sample-owned; not NSF core).

### Roadmap content caps (hard limits)

| Asset type | Count |
|---|---|
| Primary thread | 1 (`thread_main`) |
| Primary section | 1 |
| Actors | 3 |
| Factions | 2 |
| Faculties | 6 |
| Conduct profiles | 4 |
| Beliefs | 3 |
| Dialogue graphs | 8 |
| Location graphs | 1 |
| Automated setup scene | 1 |

### Deliberate content choices

- Faculty groups: 4×6 noir-style (mapping appendix)
- Conduct: cop/investigator axes from appendix
- UI skin: "Detective" theme via `UISkinDefinition` — not hardcoded in NSF
- Thread: murder/evidence investigation pattern

### NoirSampleSetupPipeline

12-step Editor batch mirroring Greywater / NSF sample pattern:

```text
 1. Validate manifest
 2. Import locale
 3. Build faculty registry
 4. Build actor/faction defs
 5. Compile dialogue graphs
 6. Build location graph
 7. Place interactables
 8. Wire thread + chronicle
 9. Create scene
10. Bind Canvas presenters (detective skin)
11. Run pack playthrough test
12. Write SetupReport.json
```

Menu: `Tools/Noir Sample/Setup Scene`

### NoirSamplePlaythroughTests

Edit Mode — **no human input**:

```csharp
[TestFixture]
class NoirSamplePlaythroughTests
{
    [Test]
    public void FullThread_ResolvesWithoutPlayerInput()
    {
        var registry = NoirSampleBootstrap.CreateRegistry();
        var driver = new SamplePlaythroughDriver(registry);
        driver.RunUntilThreadResolved("thread_main");
        Assert.That(registry.GetRequired<IThreadService>().IsResolved("thread_main"), Is.True);
    }
}
```

Driver uses headless presenters + direct service APIs — same pattern as `FullLoopIntegrationTests`.

### Exit criteria

- Playthrough test green
- Content pipeline validates entire pack
- **Zero** `NarrativeFramework` code references noir nouns (grep gate)

---

## SciFiSample (Phase 16)

### Purpose

Prove NSF reusability with **different genre schema** and **2D presentation** on the same package version.

**Presentation:** **2D** — station deck map, terminal UI, screen-space interaction (hotspot graph, no NavMesh).

### Deliberate differences from NoirSample

| Aspect | NoirSample (3D) | SciFiSample (2D) |
|---|---|---|
| Presentation | 3D walkable scenes | 2D map + UI panels |
| Faculty grid | 4×6 noir faculties | 3×4 station crew skills |
| Conduct | Cop conduct profiles | `conduct_diplomatic`, `conduct_ruthless` |
| Thread emphasis | Murder evidence | Info-flow + faction pressure |
| Setting | City noir | Orbital station |
| Magic/noir faculties | Present in content | Absent |

### Shared with NoirSample

- Same NSF package version reference
- Same setup/test pattern (12-step pipeline)
- Same headless playthrough driver infrastructure
- Same prohibition: no genre strings in framework code

### SciFiSamplePlaythroughTests

```csharp
[Test]
public void FullThread_ResolvesWithoutPlayerInput()
{
    var registry = SciFiSampleBootstrap.CreateRegistry();
    var driver = new SamplePlaythroughDriver(registry);
    driver.RunUntilThreadResolved("thread_station_alert");
    Assert.That(registry.GetRequired<IThreadService>().IsResolved("thread_station_alert"), Is.True);
}
```

### Cross-pack test (Phase 16 exit)

```csharp
[Test]
public void CombinedSamples_CoverAllNsfModules()
{
    var coverage = SampleFeatureCoverageAudit.Run(NoirSampleBootstrap.CreateRegistry(), SciFiSampleBootstrap.CreateRegistry());
    Assert.That(coverage.UncoveredModules, Is.Empty, "Noir (3D) + SciFi (2D) must exercise every NSF module — see samples.md § Feature coverage matrix");
}

[Test]
public void BothSamples_CompileAgainstSameNsfRelease()
{
    Assert.That(NoirSampleBootstrap.NsfVersion, Is.EqualTo(SciFiSampleBootstrap.NsfVersion));
    Assert.That(NoirSampleBootstrap.CreateRegistry(), Is.Not.Null);
    Assert.That(SciFiSampleBootstrap.CreateRegistry(), Is.Not.Null);
}
```

Both green on **same commit** — framework contains zero genre conditionals.

---

## Presentation in samples

Samples wire **real Canvas presenters** (optional Phase 15+); tests use headless implementations.

| Presenter | FrameworkTestPack / integration | Noir / SciFi |
|---|---|---|
| `IDialoguePresenter` | Headless | `NoirDialogueCanvasPresenter` |
| `IChroniclePresenter` | Headless | Detective notebook / mission log skin |
| `IBeliefView` | Headless | Genre-themed belief panel |

Binding via sample bootstrap only:

```csharp
var shell = registry.GetRequired<IUIShell>();
shell.BindDialoguePresenter(presenter);  // sample-owned
```

Core `IDialogueService` never imports sample types.

---

## Definition assets

All sample content uses standard Content definitions ([data-model.md](data-model.md)). Sample-specific additions:

```csharp
class UISkinDefinition : ContentDefinition { /* genre labels */ }
class SampleBootstrapConfig : ScriptableObject { /* scene refs */ }
```

Live in sample folders — not in `NarrativeFramework.Content` core schemas unless promoted to generic.

---

## Runtime state

| Sample | Persisted state |
|---|---|
| FrameworkTestPack | Used in tests — ephemeral |
| NoirSample | Full save via `IPersistenceService` in playthrough test |
| SciFiSample | Same |

Playthrough tests may run without persistence; save/load scenario `[FULL]` integration.

---

## Core algorithms

### Load sample pack

1. Read `manifest.json`
2. `IContentPipeline.Validate(manifest)` — fail fast
3. `ContentRegistry.FromManifest(manifest)`
4. Register in `INarrativeServiceRegistry`
5. Sample bootstrap wires genre presenters + scene refs

### Playthrough driver (shared)

```csharp
class SamplePlaythroughDriver
{
    public void RunUntilThreadResolved(string threadId)
    {
        while (!_threads.IsResolved(threadId))
        {
            var next = _planner.GetNextAction();  // inspect, talk, travel
            next.Execute(_registry);
            _kernel.Tick();
        }
    }
}
```

Planner uses gate + thread state — not scripted coordinates.

---

## Event contracts

Samples do not define new framework events. Playthrough asserts on standard events:

- `FactRegistered`, `DialogueAdvanced`, `ChronicleUpdated`, `ThreadMilestoneReached`

---

## Integration matrix

| Sample | Used by |
|---|---|
| FrameworkTestPack | Phase 2–14 all tests, setup-nsf-sample.ps1 |
| NoirSample | Phase 15 playthrough; optional sample-repo QA ([decisions-log.md](../decisions-log.md) NSF-H04) |
| SciFiSample | Phase 16 cross-pack, CI Phase 17 |

| Dependency | Rule |
|---|---|
| NSF → Sample | **Forbidden** |
| Sample → NSF | Allowed (package reference) |
| Sample → Presentation | Allowed (Canvas) |

---

## MVP scope

### FrameworkTestPack (Phase 2)

- [ ] manifest + minimal defs listed above
- [ ] Pipeline validate pass
- [ ] JSON round-trip test
- [ ] Loaded by `FullLoopIntegrationTests`

### NoirSample (Phase 15)

- [ ] Content caps per roadmap
- [ ] 3D presentation wiring (walkable scenes + interactables)
- [ ] `NoirSampleSetupPipeline` 12-step
- [ ] `NoirSamplePlaythroughTests` green
- [ ] Appendix mapping IDs in content only

### SciFiSample (Phase 16)

- [ ] Different faculty/conduct schema
- [ ] 2D presentation wiring (map + UI panels)
- [ ] Setup + playthrough tests
- [ ] Cross-pack compile test same commit
- [ ] `SampleFeatureCoverageAudit` green with NoirSample

---

## Full scope

- Sample repo commercial art/audio — § Game-scoped polish in [decisions-log.md](../decisions-log.md) (not NSF package scope)
- `[FULL]` Sibling repos already locked (P-07)
- `[FULL]` Localized sample packs beyond en/fr — post-1.0 sample milestone

---

## File tree

```text
Packages/NarrativeFramework/Samples~/FrameworkTestPack/...   (in-package test fixtures)
../NoirSample/                    (sibling repo — Phase 15)
  manifest.json
  Runtime/NoirSampleBootstrap.cs
  Scenes/NoirSampleScene.unity
../SciFiSample/                   (sibling repo — Phase 16)
  manifest.json
  Runtime/SciFiSampleBootstrap.cs
  Scenes/SciFiSampleScene.unity
```

---

## Test plan

| Test | Sample | Phase |
|---|---|---|
| `Validate_GoodPack` | FrameworkTestPack | 2 |
| `JsonRoundTrip` | FrameworkTestPack | 2 |
| `FullLoopIntegrationTests` | FrameworkTestPack | 14 |
| `NoirSamplePlaythroughTests` | NoirSample | 15 |
| `SciFiSamplePlaythroughTests` | SciFiSample | 16 |
| `CrossPack_SameNsfVersion` | Both | 16 |
| `Framework_NoGenreTokens` | — | 15–16 |

---

## Deferred decisions

All sample forks **locked** — [decisions-log.md](../decisions-log.md). Literary/commercial polish → § Game-scoped (not NSF deferrals).

---

## Related documents

- [foundation.md](foundation.md) — package layout
- [integration.md](integration.md) — FrameworkTestPack in full loop
- [editor.md](editor.md) — setup pipelines
- [present-ui.md](present-ui.md) — sample Canvas presenters
- [appendix-detective-noir-mapping.md](../appendix-detective-noir-mapping.md) — Noir content IDs
