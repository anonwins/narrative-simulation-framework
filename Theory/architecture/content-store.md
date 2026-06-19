# Content Store — Implementation Architecture

- Spec: [systems/content-store.md](../systems/content-store.md)
- Glossary: [Content IDs](../terminology-glossary.md) · [Registry naming](../terminology-glossary.md)
- Roadmap: [Phase 2](../development-roadmap.md) MVP · Phase 12 extended definitions
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [content-pipeline.md](content-pipeline.md)

---

## Design rationale

### Why a typed content store instead of scattered ScriptableObjects

Narrative games accumulate thousands of authored records — dialogue graphs, actor profiles, belief definitions, gate rules. Without a single lookup facade, every service invents its own loading path, reference validation diverges, and content pack boundaries blur. `IContentStore` is the **only public read surface** for definitions at runtime.

### Why internal ContentRegistry

The glossary locks naming: `IContentStore` (public) vs `ContentRegistry` (internal index). External game code must not mutate the registry after bootstrap; only the pipeline and Editor import paths register definitions. This prevents runtime content injection bugs and keeps Edit Mode tests deterministic (load pack once, query many times).

### Why generic GetDefinition&lt;T&gt;

Strong typing catches category mistakes at compile time (`GetDefinition<BeliefDefinition>("actor_kim")` fails at validation, not in dialogue). The generic constraint `where T : ContentDefinition` ties every definition to shared ID and schema version policy from [data-model.md](data-model.md).

### Why separate content from narrative logic

The Rule Engine decides **when** content is available. The Content Store holds **what** the content is. Mixing availability logic into definition assets creates circular dependencies (a dialogue node that self-gates) and breaks pipeline validation (graphs that reference missing flags).

### Why snake_case prefixed IDs

Prefix namespaces (`faculty_`, `belief_`, `actor_`) let `ContentIdValidator` reject cross-category references before loading full object graphs. Setting nouns stay in content pack *values*, not IDs — glossary SSOT.

---

## Implementation summary

| MVP (Phase 2) | Full (Phase 12+) |
|---|---|
| `ContentDefinition` base + 5 seed types | All 37-system definition types |
| `IContentStore` + internal `ContentRegistry` | Hot-reload pack swap in Editor `[FULL]` |
| Register at bootstrap from FrameworkTestPack | Multi-pack merge with conflict policy `[FULL]` |
| `GetDefinition` / `TryGetDefinition` / `GetAllIds` | Cross-type query indexes (by tag, group) `[FULL]` |
| ID validation on register | Lazy load + addressables bridge `[FULL]` |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Content` |
| **References** | `NarrativeFramework.Runtime` |
| **Public namespace** | `NarrativeFramework.Content` |
| **Definitions** | `NarrativeFramework.Content.Definitions` |
| **Internal** | `NarrativeFramework.Content.Internal` |
| **Tick phase** | None — loaded at bootstrap; queried anytime read-only |

**Asmdef rule:** Content never references Presentation, Story runtime services, or Cognition orchestrators. Cognition/Story/Rules assemblies reference Content for definition types only.

---

## Public API

See [contracts.md](contracts.md) — `IContentStore`.

**Module notes:**

- `GetDefinition<T>` throws `ContentNotFoundException` when ID missing — use in code paths that assume pack integrity (post-pipeline).
- `TryGetDefinition<T>` for optional lookups (debug, modding hooks).
- `GetAllIds<T>` powers Editor browsers and pipeline reference scans — not for per-frame hot paths.

`ContentRegistry` is **not** registered in `INarrativeServiceRegistry`. Only `IContentStore` facade is registered.

---

## Internal implementation

### ContentStore (facade)

```csharp
namespace NarrativeFramework.Content
{
    sealed class ContentStore : IContentStore
    {
        readonly ContentRegistry _registry;

        public ContentStore(ContentRegistry registry)
            => _registry = registry ?? throw new ArgumentNullException(nameof(registry));

        public T GetDefinition<T>(string id) where T : ContentDefinition
        {
            if (!_registry.TryGet(id, out T def))
                throw new ContentNotFoundException(typeof(T).Name, id);
            return def;
        }

        public bool TryGetDefinition<T>(string id, out T definition) where T : ContentDefinition
            => _registry.TryGet(id, out definition);

        public IReadOnlyList<string> GetAllIds<T>() where T : ContentDefinition
            => _registry.GetAllIds<T>();
    }
}
```

### ContentRegistry (internal)

```csharp
namespace NarrativeFramework.Content.Internal
{
    sealed class ContentRegistry
    {
        readonly Dictionary<(Type type, string id), ContentDefinition> _map = new();
        readonly Dictionary<Type, HashSet<string>> _idsByType = new();

        public void Register(ContentDefinition def)
        {
            var type = def.GetType();
            var key = (type, def.Id);
            _map[key] = def;
            if (!_idsByType.TryGetValue(type, out var set))
                _idsByType[type] = set = new HashSet<string>();
            set.Add(def.Id);
        }

        public bool TryGet<T>(string id, out T def) where T : ContentDefinition
        {
            def = null;
            if (!_map.TryGetValue((typeof(T), id), out var raw)) return false;
            def = raw as T;
            return def != null;
        }

        public IReadOnlyList<string> GetAllIds<T>() where T : ContentDefinition
            => _idsByType.TryGetValue(typeof(T), out var set)
                ? set.ToList()
                : Array.Empty<string>();
    }
}
```

### ContentBootstrapper

```csharp
sealed class ContentBootstrapper
{
    public static ContentStore LoadFromManifest(
        ContentPackManifest manifest,
        IContentPipeline pipeline)
    {
        var report = pipeline.Build(manifest);
        if (!report.Success)
            throw new ContentPackLoadException(report);

        var registry = new ContentRegistry();
        foreach (var def in report.LoadedDefinitions)
        {
            def.Validate(new ValidationContext(registry));
            registry.Register(def);
        }
        return new ContentStore(registry);
    }
}
```

### ContentNotFoundException

Typed exception carrying `DefinitionType` and `ContentId` for pipeline error UX and test assertions.

---

## Definition assets

Phase 2 seed types (ScriptableObjects inheriting `ContentDefinition`):

| Type | ID prefix | Used by |
|---|---|---|
| `FacultyDefinition` | `faculty_` | [cognition-faculty.md](cognition-faculty.md) |
| `ActorDefinition` | `actor_` | sim-actor (Phase 6) |
| `BeliefDefinition` | `belief_` | [cognition-belief.md](cognition-belief.md) |
| `StoryFlagDefinition` | `flag_` | story-state (Phase 4) |
| `GateRuleDefinition` | `gate_` | rules-gate (Phase 5) |

Base shape from [data-model.md](data-model.md):

```csharp
abstract class ContentDefinition : ScriptableObject
{
    [SerializeField] string id;
    [SerializeField] string schemaVersion = "1.0";
    public string Id => id;
    public string SchemaVersion => schemaVersion;
    public virtual void Validate(IValidationContext ctx) { }
}
```

Phase 12+ adds `ItemDefinition`, `LocaleTableDefinition`, `DialogueGraphDefinition`, etc. — each registered in the same registry.

---

## Runtime state / persistence

**Definitions are not saved.** They ship with the content pack and reload from disk on boot.

Runtime mutable state lives in owning services (`FacultyState`, `BeliefState`, …) — see respective cognition architecture docs and [data-model.md](data-model.md) persistence envelopes (Phase 10).

`IContentStore` implements no `IStatefulService` — store is immutable after bootstrap in MVP.

---

## Core algorithms

### Bootstrap load sequence

1. Read `ContentPackManifest` (pack ID, schema version, asset paths).
2. Delegate to `IContentPipeline.Build` — validates, resolves refs, materializes definitions.
3. For each definition: run `Validate(IValidationContext)` with registry-backed ref checks.
4. `ContentRegistry.Register` — duplicate ID same type → pipeline should have failed; runtime logs error and overwrites in debug builds only.
5. Register `IContentStore` facade in `INarrativeServiceRegistry`.

### Lookup

1. Hash key `(typeof(T), id)` — O(1).
2. No fallback casting across types — wrong type returns not found.

### Validation context

```csharp
interface IValidationContext
{
    bool DefinitionExists<T>(string id) where T : ContentDefinition;
    void ReportError(string message);
}
```

Validators call `DefinitionExists` for every outbound reference field.

---

## Event contracts + kernel tick phase

**No kernel tick participation.** Content Store is read-only infrastructure.

| Event | Producer | When |
|---|---|---|
| `ContentPackLoaded` | Bootstrap | After successful pipeline build `[FULL]` debug |
| `ContentDefinitionMissing` | Services | When `TryGet` fails in optional paths `[FULL]` |

Pipeline failures are synchronous exceptions during bootstrap, not sim events.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentPipeline` | Validated load path |
| `ContentIdValidator` | ID policy |
| `ContentDefinition` hierarchy | Typed storage |

| Used by | Reason |
|---|---|
| `IFacultyService` | `FacultyDefinition` caps, groups |
| `IRollService` | Roll definitions, DC tables |
| `IBeliefService` | `BeliefDefinition` metadata |
| `IRuleEngine` / `IGateService` | `GateRuleDefinition` |
| `IDialogueService` | Dialogue graph definitions (Phase 4) |
| `ILocaleService` | Locale table definitions (Phase 12) |
| `IInventoryService` | `ItemDefinition` (Phase 12) |
| `ScriptCompiler` | Emits definitions into pack (Phase 12) |

**Forbidden:** Content → Presentation, Content → Story runtime (except definition types consumed by Story asmdef).

---

## MVP scope checklist (Phase 2)

- [ ] `ContentDefinition` base class + `Validate` hook
- [ ] `ContentRegistry` internal index
- [ ] `ContentStore` implements `IContentStore`
- [ ] `ContentBootstrapper` from FrameworkTestPack manifest
- [ ] Five seed definition ScriptableObject types
- [ ] Register store in service registry at game init
- [ ] Tests: lookup hit/miss, type safety, duplicate ID rejected by pipeline

---

## Full scope

- `[FULL]` Multi-pack overlay (DLC merges base pack)
- `[FULL]` Addressables async load with staging registry
- `[FULL]` Definition versioning + in-place migration on load
- `[FULL]` Tag/category indexes for Editor search
- `[FULL]` Runtime definition hot-swap (Editor only)

---

## File tree

```text
Packages/NarrativeFramework/Content/IContentStore.cs
Packages/NarrativeFramework/Content/ContentStore.cs
Packages/NarrativeFramework/Content/ContentBootstrapper.cs
Packages/NarrativeFramework/Content/ContentNotFoundException.cs
Packages/NarrativeFramework/Content/Definitions/ContentDefinition.cs
Packages/NarrativeFramework/Content/Definitions/FacultyDefinition.cs
Packages/NarrativeFramework/Content/Definitions/ActorDefinition.cs
Packages/NarrativeFramework/Content/Definitions/BeliefDefinition.cs
Packages/NarrativeFramework/Content/Definitions/StoryFlagDefinition.cs
Packages/NarrativeFramework/Content/Definitions/GateRuleDefinition.cs
Packages/NarrativeFramework/Content/Internal/ContentRegistry.cs
Packages/NarrativeFramework/Content/Validation/IValidationContext.cs
Packages/NarrativeFramework/Content/Validation/ValidationContext.cs
Packages/NarrativeFramework/Tests/EditMode/Content/ContentStoreLookupTests.cs
Packages/NarrativeFramework/Tests/EditMode/Content/ContentBootstrapTests.cs
Samples~/FrameworkTestPack/Manifest.json
Samples~/FrameworkTestPack/Definitions/
```

---

## Test plan

| Test | Phase | Asserts |
|---|---|---|
| `GetDefinition_ReturnsRegisteredAsset` | 2 | Field equality on FacultyDefinition |
| `GetDefinition_ThrowsWhenMissing` | 2 | ContentNotFoundException type + id |
| `TryGetDefinition_WrongType_ReturnsFalse` | 2 | Same ID, different T |
| `GetAllIds_ReturnsOnlyMatchingType` | 2 | Count matches pack |
| `Bootstrap_RejectsBrokenPack` | 2 | Pipeline report !Success |
| `Validate_ReportsMissingReference` | 2 | ValidationContext error count |
| `JsonRoundTrip_DefinitionPreserved` | 2 | See data-model ContentJson test |

**Exit criteria (roadmap):** Pipeline validates good pack, rejects broken pack; round-trip JSON test.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SCR-02, SCR-03, SCR-04, CP-03.

---

## Related documents

- [content-pipeline.md](content-pipeline.md) — load path into registry
- [data-model.md](data-model.md) — definition base, ID rules
- [cognition-faculty.md](cognition-faculty.md), [cognition-belief.md](cognition-belief.md) — primary Phase 2 consumers
