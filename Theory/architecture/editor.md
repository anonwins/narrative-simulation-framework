# Editor Module — Implementation Architecture

- Spec: [content-pipeline.md](../systems/content-pipeline.md), [content-store.md](../systems/content-store.md)
- Roadmap: [Phase 2](../development-roadmap.md) (pipeline), [Phase 14](../development-roadmap.md) (setup automation)
- Foundation: [foundation.md](foundation.md)
- Integration: [integration.md](integration.md)

---

## Design rationale

### Why Editor is a separate assembly

Content validation, batch scene generation, and importer menus require `UnityEditor` APIs. Keeping Editor code out of Runtime ensures player builds stay lean and Edit Mode tests do not accidentally load Editor assemblies.

### Why Phase 2 starts Editor tooling early

Without pipeline menus, agents cannot validate `FrameworkTestPack` during Phase 2 exit gates. Early Editor support — validate pack, import JSON, show pipeline report — unblocks every subsequent phase that depends on content definitions.

### Why Phase 14 adds setup automation

Manual Unity scene YAML is fragile for agents. `Tools → NSF → Setup Sample Scene` and `setup-nsf-sample.ps1` generate `NsfSampleScene.unity` **from code** (Greywater SetupPipeline pattern), producing deterministic scenes for integration tests and human playthrough.

### Why core must not reference Editor

Runtime services (`IContentPipeline`) declare behavior; Editor provides **menus and batch runners** that call those interfaces. Same inversion as Debug: Runtime knows pipeline contract, Editor knows Unity batch APIs.

---

## Implementation summary

| MVP Phase 2 | MVP Phase 14 |
|---|---|
| `Tools/NSF/Validate Content Pack` | `Tools/NSF/Setup Sample Scene` |
| Pipeline report window | `NsfSampleSceneBuilder` |
| JSON import/export menu | `setup-nsf-sample.ps1` wrapper |
| Definition inspectors (basic) | `SetupReport.json` emission |
| | Batchmode entry for CI |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Editor`
- **Platform:** Editor
- **Namespace:** `NarrativeFramework.Editor`
- **References:** All runtime assemblies, `UnityEditor`, optional `UnityEditor.SceneManagement`

---

## Public API

Editor exposes menus and batch static entry points — not player services.

```csharp
// Batch entry — invoked by setup-nsf-sample.ps1 via -executeMethod
static class NsfSampleSetupBatch
{
    public static void RunFromCommandLine()
    {
        var report = NsfSampleSceneBuilder.Build();
        SetupReportWriter.Write(report);
        EditorApplication.Exit(report.AllPassed ? 0 : 1);
    }
}
```

Runtime pipeline API (called by Editor):

```csharp
interface IContentPipeline
{
    PipelineReport Validate(ContentPackManifest manifest);
    PipelineReport Build(ContentPackManifest manifest);
    bool TryFix(ContentPackManifest manifest, out PipelineReport report);
}
```

See [contracts.md](contracts.md).

---

## Internal implementation

### Phase 2 — ContentPipelineMenu

```csharp
class ContentPipelineMenu
{
    [MenuItem("Tools/NSF/Validate Content Pack")]
    static void ValidateSelectedPack()
    {
        var manifest = ContentPackManifestLoader.FromSelection();
        var pipeline = new ContentPipeline(new ContentRegistry());
        var report = pipeline.Validate(manifest);
        PipelineReportWindow.Show(report);
    }

    [MenuItem("Tools/NSF/Import Pack JSON")]
    static void ImportJson() { /* Newtonsoft deserialize → ScriptableObjects */ }

    [MenuItem("Tools/NSF/Export Pack JSON")]
    static void ExportJson() { /* reverse */ }
}
```

### PipelineReportWindow

Human-readable + machine-parseable:

```text
✓ actor_test_detective — OK
✗ gate_test_missing_ref — missing target flag_story_unknown
Summary: 42 passed, 1 failed
```

Agents read exit code; humans read window.

### Definition inspectors (Phase 2 baseline)

Custom `Editor` for `ContentDefinition` subclasses:

- Show ID with prefix validation hint
- Draw reference fields as dropdowns from registry
- Red highlight broken refs (local pre-validate before full pipeline)

### Phase 14 — NsfSampleSceneBuilder

**Why code-generated scenes:** Avoid hand-editing `.unity` YAML; version control friendly; reproducible on CI.

```csharp
static class NsfSampleSceneBuilder
{
    public static SetupReport Build()
    {
        var report = new SetupReport();
        try
        {
            var scene = EditorSceneManager.NewScene(NewSceneSetup.EmptyScene, NewSceneMode.Single);
            report.Step("CreateScene", true);

            BootstrapSampleRegistry(report);
            report.Step("BootstrapRegistry", true);

            CreateSampleEnvironment(report);      // planes, lighting — minimal
            CreateSampleInteractables(report);  // test desk, NPC marker
            WirePresentationHarness(report);    // headless or placeholder UI roots

            var path = "Assets/Scenes/NsfSampleScene.unity";
            EditorSceneManager.SaveScene(scene, path);
            report.Step("SaveScene", true);
        }
        catch (Exception ex)
        {
            report.Fail(ex.Message);
        }
        return report;
    }
}
```

Steps mirror roadmap closed loop: load FrameworkTestPack → register services → place interactables → bind headless presenters for Edit Mode hybrid tests.

### SetupReport

```csharp
class SetupReport
{
    public bool AllPassed => Steps.All(s => s.Passed);
    public List<SetupStep> Steps { get; } = new();
    public string ErrorMessage;

    public void Step(string name, bool passed) => Steps.Add(new SetupStep(name, passed));
    public void Fail(string message) { ErrorMessage = message; Step("Fatal", false); }
}

static class SetupReportWriter
{
    public static void Write(SetupReport report)
    {
        var json = JsonConvert.SerializeObject(new {
            allPassed = report.AllPassed,
            steps = report.Steps,
            error = report.ErrorMessage
        }, Formatting.Indented);
        File.WriteAllText("SetupReport.json", json);
    }
}
```

### setup-nsf-sample.ps1

Repository root script (agent gate Phase 14+):

```powershell
# Invokes Unity batchmode — never Unity.exe directly (workspace rule: invoke-unity.ps1)
& "$PSScriptRoot\invoke-unity.ps1" `
    -executeMethod NarrativeFramework.Editor.NsfSampleSetupBatch.RunFromCommandLine `
    -quit -batchmode -nographics

if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
$report = Get-Content "SetupReport.json" | ConvertFrom-Json
if (-not $report.allPassed) { exit 1 }
exit 0
```

**Not required before Phase 14** per roadmap — Phase 2–13 use Edit Mode tests only.

### NsfEditorSettings

Optional `ScriptableObject` for default pack path, sample scene output path, test seed — loaded by menus and batch runner.

---

## Definition assets

Editor creates/updates ScriptableObjects in:

```text
Samples~/FrameworkTestPack/
Assets/Scenes/NsfSampleScene.unity   (generated Phase 14)
```

Does not author new definition **schemas** — those live in Content assembly ([data-model.md](data-model.md)).

---

## Runtime state

Editor tooling is stateless between runs except:

| State | Storage |
|---|---|
| Last pipeline report | Session memory / window |
| Default paths | `NsfEditorSettings` asset |
| SetupReport.json | Repo root or `Logs/` (CI artifact) |

---

## Core algorithms

### Validate content pack (Phase 2)

1. Load `ContentPackManifest` from selected folder
2. Build dependency graph of all definitions
3. Run schema validators + ID prefix checks
4. Fail on missing references, duplicate IDs, wrong schema version
5. Display `PipelineReport`; return non-zero on failure for batch `[FULL]`

### Setup sample scene (Phase 14)

1. Clean or overwrite target scene path
2. Create registry with **real** services (not all Null*) through phase-appropriate bootstrap
3. Import/load FrameworkTestPack into registry
4. Instantiate sample geometry placeholders
5. Attach sample `NsfSampleBootstrap` MonoBehaviour wiring presenters
6. Save scene; write SetupReport.json
7. Exit code 0 iff `allPassed`

### JSON round-trip (Phase 2)

1. Export selected definitions to JSON fixture
2. Import into empty registry
3. Re-export; Editor test or menu confirms deep equality

---

## Event contracts

Editor does not publish `SimEvent`s during validate. Setup scene may run one kernel tick in Edit Mode to verify wiring — optional smoke `[FULL]`.

---

## Integration matrix

| Calls | Usage |
|---|---|
| `IContentPipeline` | Validate/build pack |
| `NsfBootstrap` | Sample registry creation |
| `NsfSampleSceneBuilder` | Scene codegen |
| `invoke-unity.ps1` | Batchmode wrapper |

| Consumed by | Usage |
|---|---|
| Agents Phase 2 | Validate FrameworkTestPack |
| Agents Phase 14 | setup-nsf-sample.ps1 gate |
| CI Phase 17 | Both samples + setup |

**Forbidden:** Runtime → Editor; player build excludes Editor asmdef.

---

## MVP scope

### Phase 2

- [ ] `NarrativeFramework.Editor` asmdef
- [ ] Validate / Import / Export menus
- [ ] `PipelineReportWindow`
- [ ] Basic definition custom inspectors
- [ ] JSON round-trip menu action

### Phase 14

- [ ] `NsfSampleSceneBuilder` + 12-step report (aligned with Greywater pattern)
- [ ] `NsfSampleSetupBatch.RunFromCommandLine`
- [ ] `setup-nsf-sample.ps1` → `SetupReport.json` `allPassed: true`
- [ ] Menu `Tools/NSF/Setup Sample Scene`
- [ ] Generated `NsfSampleScene.unity` loads without missing refs

---

## Full scope

- `[FULL]` NoirSample / SciFiSample setup pipelines (Phase 15–16)
- `[FULL]` Graph visualizers for dialogue/thread `[FULL]`
- `[FULL]` Batch fix (`TryFix`) with preview diff
- `[FULL]` CI uploads SetupReport as artifact

---

## File tree

```text
Packages/NarrativeFramework/Editor/NarrativeFramework.Editor.asmdef
Packages/NarrativeFramework/Editor/Menus/ContentPipelineMenu.cs
Packages/NarrativeFramework/Editor/Windows/PipelineReportWindow.cs
Packages/NarrativeFramework/Editor/Inspectors/ContentDefinitionInspector.cs
Packages/NarrativeFramework/Editor/Setup/NsfSampleSceneBuilder.cs
Packages/NarrativeFramework/Editor/Setup/NsfSampleSetupBatch.cs
Packages/NarrativeFramework/Editor/Setup/SetupReport.cs
Packages/NarrativeFramework/Editor/Setup/SetupReportWriter.cs
Packages/NarrativeFramework/Editor/NsfEditorSettings.cs
setup-nsf-sample.ps1
SetupReport.json                    (generated, gitignored)
Assets/Scenes/NsfSampleScene.unity  (generated)
```

---

## Test plan

| Test | Phase | Asserts |
|---|---|---|
| `Pipeline_ValidatesFrameworkTestPack` | 2 | Report all green |
| `Pipeline_RejectsBrokenRef` | 2 | Failure message |
| `JsonRoundTrip_DeepEquals` | 2 | Import/export |
| `SetupBatch_WritesReport` | 14 | SetupReport.json exists |
| `SetupBatch_AllStepsPass` | 14 | allPassed true |
| `GeneratedScene_HasBootstrap` | 14 | Component present |

Phase 14: `setup-nsf-sample.ps1` exit 0 is roadmap exit gate.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) editor scene path `Assets/Scenes/`; placeholder UI roots in samples.

---

## Related documents

- [content-pipeline.md](content-pipeline.md) — validation rules
- [integration.md](integration.md) — full loop + setup script
- [samples.md](samples.md) — FrameworkTestPack layout
- [debug.md](debug.md) — debug dashboard menu sibling
