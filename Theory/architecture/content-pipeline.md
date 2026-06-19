# Content Pipeline — Implementation Architecture

- Spec: [systems/content-pipeline.md](../systems/content-pipeline.md)
- Glossary: [Content IDs](../terminology-glossary.md)
- Roadmap: [Phase 2](../development-roadmap.md) MVP · Phase 12 script compile integration
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [content-store.md](content-store.md) · [content-script.md](content-script.md)

---

## Design rationale

### Why a pipeline separate from runtime services

Runtime services assume content is **valid and complete**. The pipeline is manufacturing infrastructure: it catches broken references, schema drift, and missing localization before a build ships. Letting `IContentStore` self-validate at load time duplicates logic and fails too late for CI.

### Why Validate, Build, and TryFix as three entry points

- **Validate** — fast feedback for authors and Edit Mode tests; no side effects.
- **Build** — produces load-ready definitions for `ContentBootstrapper`.
- **TryFix** — optional auto-repair (missing fallback locale, orphan pruning) for agent workflows; never silent in MVP — always returns issues in `PipelineReport`.

### Why dependency graph validation

Narrative content is a graph (dialogue → actor → faculty → belief). Missing edges fail silently at runtime as null refs. Topological validation during Build catches `dialogue_kim_intro` referencing `actor_missing` before Play Mode.

### Why pipeline does not affect gameplay

The pipeline never registers facts, mutates story flags, or runs the simulation kernel. It only emits validated assets. This boundary keeps Edit Mode pipeline tests independent of tick order and kernel phases.

### Why Newtonsoft JSON round-trip in Phase 2

Agent-generated FrameworkTestPack uses JSON fixtures for deterministic CI. Same validators run on ScriptableObject imports and JSON deserializers — one validation path, two authoring surfaces.

---

## Implementation summary

| MVP (Phase 2) | Full (Phase 12+) |
|---|---|
| Schema + ID validation | Graph connectivity (dialogue dead ends) |
| Reference integrity graph | Localization completeness scan |
| `PipelineReport` with severities | Incremental build cache `[FULL]` |
| JSON import/export for test pack | `.nsf` compile stage via `ScriptCompiler` |
| Build populates in-memory definitions | Asset bundle packaging `[FULL]` |
| Reject broken pack (exit non-zero in CI) | Simulation smoke test stage `[FULL]` |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Content` (runtime validators) + `NarrativeFramework.Editor` (CLI/menu) |
| **References** | Runtime, Content definitions, Newtonsoft JSON |
| **Public namespace** | `NarrativeFramework.Content.Pipeline` |
| **Editor namespace** | `NarrativeFramework.Editor.Pipeline` |
| **Tick phase** | None — invoked at edit time and bootstrap |

---

## Public API

See [contracts.md](contracts.md) — `IContentPipeline`.

**Module notes:**

- `Validate` — read-only; safe in Edit Mode tests every frame.
- `Build` — may deserialize SOs / JSON into transient `ContentDefinition` instances attached to `PipelineReport.LoadedDefinitions` `[internal accessor]`.
- `TryFix` — MVP returns `false` for unfixable issues; may apply safe fixes (trim whitespace on IDs) when `Severity == Warning` only.

`ScriptCompiler` is a separate concrete orchestrator — pipeline **calls** it in Phase 12 for `.nsf` assets listed in manifest.

---

## Internal implementation

### ContentPipeline (orchestrator)

```csharp
namespace NarrativeFramework.Content.Pipeline
{
    sealed class ContentPipeline : IContentPipeline
    {
        readonly IReadOnlyList<IPipelineStage> _stages;

        public PipelineReport Validate(ContentPackManifest manifest)
            => Run(manifest, build: false);

        public PipelineReport Build(ContentPackManifest manifest)
            => Run(manifest, build: true);

        public bool TryFix(ContentPackManifest manifest, out PipelineReport report)
        {
            report = Validate(manifest);
            if (report.Success) return true;

            var fixer = new PipelineAutoFixer();
            var fixedManifest = fixer.ApplySafeFixes(manifest, report);
            report = Validate(fixedManifest);
            return report.Success;
        }

        PipelineReport Run(ContentPackManifest manifest, bool build)
        {
            var ctx = new PipelineContext(manifest, build);
            foreach (var stage in _stages)
                stage.Execute(ctx);
            return ctx.ToReport();
        }
    }
}
```

### Pipeline stages (MVP order)

```csharp
interface IPipelineStage
{
    void Execute(PipelineContext ctx);
}

sealed class ManifestStage : IPipelineStage { }           // manifest schema
sealed class AssetLoadStage : IPipelineStage { }          // SO + JSON paths
sealed class SchemaValidationStage : IPipelineStage { }   // per-type Validate()
sealed class IdValidationStage : IPipelineStage { }       // ContentIdValidator
sealed class ReferenceGraphStage : IPipelineStage { }     // missing refs
sealed class BuildMaterializeStage : IPipelineStage { }   // build-only finalize
```

Phase 12 adds: `ScriptCompileStage`, `LocaleCoverageStage`, `DialogueGraphStage`.

### PipelineContext

```csharp
sealed class PipelineContext
{
    public ContentPackManifest Manifest { get; }
    public bool IsBuild { get; }
    public List<PipelineIssue> Issues { get; } = new();
    public List<ContentDefinition> LoadedDefinitions { get; } = new();
    public DependencyGraph Graph { get; } = new();

    public void AddError(string message, string assetPath = null)
        => Issues.Add(new PipelineIssue { Severity = "Error", Message = message, AssetPath = assetPath });

    public PipelineReport ToReport()
        => new PipelineReport {
            Success = Issues.All(i => i.Severity != "Error"),
            Issues = Issues.ToArray(),
            LoadedDefinitions = IsBuild ? LoadedDefinitions : null
        };
}
```

### DependencyGraph

```csharp
sealed class DependencyGraph
{
    readonly Dictionary<string, HashSet<string>> _edges = new();

    public void AddEdge(string fromId, string toId) { /* ... */ }
    public bool HasMissingTargets(IContentIndex index, out string missingId) { /* ... */ }
    public IReadOnlyList<string> DetectCycles() { /* DFS; MVP optional warn */ }
}
```

### Domain types (aligned with spec + data-model)

```csharp
class ContentPackManifest
{
    public string PackId;
    public string SchemaVersion;
    public string[] AssetPaths;
    public string[] ScriptPaths;  // Phase 12
}

class PipelineReport
{
    public bool Success;
    public PipelineIssue[] Issues;
    internal List<ContentDefinition> LoadedDefinitions;
}

class PipelineIssue
{
    public string Severity;   // Error | Warning | Info
    public string Message;
    public string AssetPath;
}
```

---

## Definition assets

Pipeline does not author definitions — it **processes** them. Manifest points at:

- ScriptableObject assets under `Samples~/FrameworkTestPack/`
- JSON equivalents for round-trip tests
- Phase 12: `.nsf` files compiled to intermediate definition JSON

Each definition type implements `Validate(IValidationContext)` — pipeline constructs context backed by in-progress index.

---

## Runtime state / persistence

Pipeline is stateless between runs. `[FULL]` incremental cache stores file hashes under `Library/NSF/PipelineCache/` — not in save games.

No `IStatefulService` — pipeline is not registered in gameplay registry (optional Editor singleton for menu commands).

---

## Core algorithms

### Reference integrity

1. Load all definitions into temporary index (ID → definition).
2. For each definition, reflect/reference-walk serialized fields matching `*_` ID patterns OR explicit `[ContentRef(typeof(T))]` attributes.
3. Add graph edge `from → to` for each reference.
4. Fail if `to` not in index (Error) or wrong type (Error).

### ID validation

```csharp
ContentIdValidator.IsValid(id, requiredPrefix)
```

Called per definition in `IdValidationStage`; prefix derived from definition type registry map (same as [data-model.md](data-model.md) table).

### Build materialize

When `IsBuild`:

1. All stages pass.
2. Clone definitions into immutable build set (avoid accidental SO mutation).
3. Attach to report for `ContentBootstrapper`.

### TryFix safe fixes (MVP minimal)

- Trim whitespace on IDs (Warning → fixed).
- Does **not** invent missing targets or delete orphan nodes in MVP.

---

## Event contracts + kernel tick phase

**No kernel tick.** Editor may log to console; `[FULL]` `PipelineCompleted` ScriptableObject event for CI reporters.

| Hook | When |
|---|---|
| `OnPipelineValidate` | Editor menu / CI script |
| `OnPipelineBuild` | Pre-play bootstrap |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `ContentDefinition` types | Schema validation |
| `ContentIdValidator` | ID policy |
| `ContentJson` | JSON load path |
| `ScriptCompiler` (Phase 12) | `.nsf` → definitions |

| Used by | Reason |
|---|---|
| `ContentBootstrapper` | Build → registry |
| Edit Mode tests | Good/bad pack fixtures |
| `ILocaleService` setup (Phase 12) | Locale tables pre-validated |
| CI / `run-nsf-edit-mode-tests.ps1` | Fail build on Error severity |

---

## MVP scope checklist (Phase 2)

- [ ] `ContentPackManifest` + `PipelineReport` + `PipelineIssue`
- [ ] `IContentPipeline` with Validate / Build / TryFix stubs
- [ ] Manifest, load, schema, ID, reference stages
- [ ] JSON import for FrameworkTestPack
- [ ] Good pack passes; broken ref pack fails with actionable message
- [ ] Round-trip JSON test (definition → JSON → definition)
- [ ] Editor menu: **NSF → Validate Content Pack** (optional Phase 2)

---

## Full scope

- `[FULL]` Dialogue graph orphan/dead-end detection
- `[FULL]` Localization key coverage vs manifest locale list
- `[FULL]` Simulation smoke: load pack + 1 kernel tick headless
- `[FULL]` Incremental build with file hash cache
- `[FULL]` Parallel stage execution for large packs
- `[FULL]` Auto-fix missing fallback locale keys

---

## File tree

```text
Packages/NarrativeFramework/Content/Pipeline/IContentPipeline.cs
Packages/NarrativeFramework/Content/Pipeline/ContentPipeline.cs
Packages/NarrativeFramework/Content/Pipeline/ContentPackManifest.cs
Packages/NarrativeFramework/Content/Pipeline/PipelineReport.cs
Packages/NarrativeFramework/Content/Pipeline/PipelineIssue.cs
Packages/NarrativeFramework/Content/Pipeline/PipelineContext.cs
Packages/NarrativeFramework/Content/Pipeline/DependencyGraph.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/ManifestStage.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/AssetLoadStage.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/SchemaValidationStage.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/IdValidationStage.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/ReferenceGraphStage.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/BuildMaterializeStage.cs
Packages/NarrativeFramework/Content/Serialization/ContentJson.cs
Packages/NarrativeFramework/Editor/Pipeline/ContentPipelineMenu.cs
Packages/NarrativeFramework/Tests/EditMode/Pipeline/PipelineGoodPackTests.cs
Packages/NarrativeFramework/Tests/EditMode/Pipeline/PipelineBrokenRefTests.cs
Packages/NarrativeFramework/Tests/EditMode/Pipeline/ContentJsonRoundTripTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Validate_GoodPack_Success` | No Error issues |
| `Validate_MissingRef_Fails` | Error mentions target ID + asset path |
| `Validate_InvalidId_Fails` | Prefix violation caught |
| `Build_PopulatesDefinitions` | LoadedDefinitions count |
| `TryFix_Unfixable_ReturnsFalse` | Missing ref still Error |
| `JsonRoundTrip_FacultyDefinition` | Deep equals per data-model |
| `DependencyGraph_DetectsCycle` | Warning or Error `[FULL]` |

**Exit criteria:** Pipeline validates good pack, rejects broken pack; round-trip JSON test.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) CP-01, CP-02, S-04.

---

## Related documents

- [content-store.md](content-store.md) — consumer of Build output
- [content-script.md](content-script.md) — ScriptCompileStage Phase 12
- [data-model.md](data-model.md) — validators, JSON
