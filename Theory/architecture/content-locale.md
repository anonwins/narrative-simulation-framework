# Content Locale — Implementation Architecture

- Spec: [systems/content-locale.md](../systems/content-locale.md)
- Glossary: [Localization keys](../terminology-glossary.md)
- Roadmap: [Phase 12](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [content-pipeline.md](content-pipeline.md) · [content-store.md](content-store.md)

---

## Design rationale

### Why language-agnostic core with keyed text

Dialogue definitions store **locale keys**, not inline prose. This separates translation workflow from simulation logic, enables pipeline verification of missing translations, and keeps AI-generated content safe (keys validated; strings swapped per locale).

### Why ILocaleService at runtime

Presentation and Story services need resolved strings at display time with variable substitution (`{actor_name}`, `{item_count}`). Centralizing lookup avoids each presenter implementing fallback chains and duplicate substitution rules.

### Why fallback locale chain

Ship builds cannot crash on missing translation. MVP: explicit fallback locale (typically `en`) with debug logging for missing keys. Pipeline **Validate** stage treats missing keys as Error in CI; runtime **Resolve** uses fallback to keep Play Mode alive.

### Why not embed strings in ScriptableObjects only

ScriptableObject-only strings block CSV/PO export for translators. Locale tables as definitions (`LocaleTableDefinition`) plus JSON import align with pipeline and external localization tools.

### Why pipeline integration matters

Runtime fallback hides authoring gaps; pipeline must fail builds when required locale coverage is incomplete for shipping locales.

---

## Implementation summary

| MVP (Phase 12) | Full |
|---|---|
| `ILocaleService.Resolve` + `HasKey` | Pluralization rules |
| `GetFallbackLocale` | Gender-aware substitution |
| String tables as content definitions | Voice line ID mapping |
| `{token}` substitution dictionary | RTL layout hints `[Presentation]` |
| Pipeline locale coverage stage | Unused key detection |
| Debug log on missing key | ICU message format `[FULL]` |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Content` |
| **References** | Runtime |
| **Namespace** | `NarrativeFramework.Content.Locale` |
| **Tick phase** | None — on-demand during Interpretation/Content when UI/dialogue resolves text |

---

## Public API

See [contracts.md](contracts.md) — `ILocaleService`.

**Module notes:**

- `Resolve(key, localeId, substitutions)` — never returns null; returns `[MISSING:key]` in debug builds if key and fallback missing.
- `GetFallbackLocale()` — from pack manifest default or `en`.
- `HasKey(key, localeId)` — pipeline and tests; O(1) hash lookup.

Registered in `INarrativeServiceRegistry` at bootstrap after locale tables loaded from content store.

---

## Internal implementation

### LocaleService

```csharp
namespace NarrativeFramework.Content.Locale
{
    sealed class LocaleService : ILocaleService
    {
        readonly LocaleRegistry _registry;
        readonly string _fallbackLocaleId;

        public LocaleService(IContentStore store, string fallbackLocaleId = "en")
        {
            _fallbackLocaleId = fallbackLocaleId;
            _registry = LocaleRegistry.BuildFromStore(store);
        }

        public string Resolve(
            string key,
            string localeId,
            IReadOnlyDictionary<string, string> substitutions)
        {
            if (!_registry.TryGet(key, localeId, out var text))
            {
                if (!_registry.TryGet(key, _fallbackLocaleId, out text))
                {
                    LocaleDiagnostics.LogMissing(key, localeId);
                    text = $"[{key}]";
                }
            }
            return SubstitutionEngine.Apply(text, substitutions);
        }

        public string GetFallbackLocale() => _fallbackLocaleId;

        public bool HasKey(string key, string localeId)
            => _registry.TryGet(key, localeId, out _);
    }
}
```

### LocaleRegistry (internal)

```csharp
sealed class LocaleRegistry
{
    readonly Dictionary<(string localeId, string key), string> _entries = new();

    public static LocaleRegistry BuildFromStore(IContentStore store)
    {
        var reg = new LocaleRegistry();
        foreach (var tableId in store.GetAllIds<LocaleTableDefinition>())
        {
            var table = store.GetDefinition<LocaleTableDefinition>(tableId);
            reg.Merge(table);
        }
        return reg;
    }

    public bool TryGet(string key, string localeId, out string text)
        => _entries.TryGetValue((localeId, key), out text);
}
```

### SubstitutionEngine

```csharp
static class SubstitutionEngine
{
    static readonly Regex TokenPattern = new(@"\{(\w+)\}", RegexOptions.Compiled);

    public static string Apply(string template, IReadOnlyDictionary<string, string> substitutions)
    {
        if (substitutions == null || substitutions.Count == 0) return template;
        return TokenPattern.Replace(template, m =>
            substitutions.TryGetValue(m.Groups[1].Value, out var val) ? val : m.Value);
    }
}
```

### LocaleTableDefinition

```csharp
class LocaleTableDefinition : ContentDefinition
{
    [SerializeField] string localeId;
    [SerializeField] LocaleEntry[] entries;

    public string LocaleId => localeId;
    public IReadOnlyList<LocaleEntry> Entries => entries;

    public override void Validate(IValidationContext ctx)
    {
        foreach (var e in entries)
            if (string.IsNullOrWhiteSpace(e.Key))
                ctx.ReportError($"Empty locale key in {Id}");
    }
}

[Serializable]
class LocaleEntry
{
    public string Key;
    public string Text;
}
```

Spec domain type:

```csharp
class LocaleTable { string LocaleId; Dictionary<string, string> Entries; }
```

Runtime registry flattens tables into hash map — `LocaleTable` is authoring/export shape.

---

## Definition assets

| Asset | ID pattern | Content |
|---|---|---|
| `LocaleTableDefinition` | `locale_table_*` | Per-locale key → string map |
| Dialogue definitions | text fields | Store keys only, e.g. `kim_intro_01_text` |

Pipeline `LocaleCoverageStage` (Phase 12):

1. Collect all keys referenced by dialogue/script definitions.
2. For each required shipping locale, assert `HasKey`.
3. Emit Warning for unused keys (optional MVP).

---

## Runtime state / persistence

**Locale strings are definitions** — not saved in game state.

Player-selected `localeId` may live in game settings `[Presentation]` — not in `ILocaleService` MVP (passed per `Resolve` call from caller).

No `IStatefulService` for locale in MVP.

---

## Core algorithms

### Resolve

1. Lookup `(localeId, key)`.
2. On miss → lookup `(fallbackLocaleId, key)`.
3. On miss → diagnostic + bracket placeholder.
4. Run substitution pass left-to-right; unreplaced tokens left as `{token}` for visibility.

### BuildFromStore

1. Iterate all `LocaleTableDefinition` IDs.
2. Merge entries; duplicate key same locale → pipeline Error (last-wins forbidden in MVP).

### Pipeline coverage

```csharp
foreach (var key in referencedKeys)
    foreach (var locale in manifest.RequiredLocales)
        if (!localeService.HasKey(key, locale))
            ctx.AddError($"Missing {locale} for {key}");
```

---

## Event contracts + kernel tick phase

**No dedicated tick.** Called synchronously when:

- `IDialogueService` builds node display text (**Interpretation** / **Content**)
- `IVoiceService` resolves line keys
- `IChronicleService` projects journal strings `[FULL]`

| Debug event | When |
|---|---|
| `LocaleKeyMissing` | Resolve fallback failed `[FULL]` |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IContentStore` | Locale table definitions |
| `IContentPipeline` | Coverage validation |

| Used by | Reason |
|---|---|
| `IDialogueService` | Node text resolution |
| `IVoiceService` | Spoken line keys |
| `IChroniclePresenter` | Journal strings (Phase 13) |
| `IAudioNarrativeService` | Voice line lookup |
| `ScriptCompiler` | TEXT keys validated at compile |

---

## MVP scope checklist (Phase 12)

- [ ] `ILocaleService` implementation
- [ ] `LocaleTableDefinition` + store registration
- [ ] `Resolve` with fallback + substitution
- [ ] `HasKey` for pipeline
- [ ] FrameworkTestPack includes `en` + one secondary locale fixture
- [ ] Pipeline fails when required locale missing
- [ ] Tests: resolve, fallback, substitution, missing key behavior

---

## Full scope

- `[FULL]` Pluralization (`{count, plural, ...}`)
- `[FULL]` Gender-aware keys (`key_male`, `key_female` selector)
- `[FULL]` Voice profile → audio asset map per locale
- `[FULL]` ICU MessageFormat
- `[FULL]` CSV/PO import/export in Editor

---

## File tree

```text
Packages/NarrativeFramework/Content/Locale/ILocaleService.cs
Packages/NarrativeFramework/Content/Locale/LocaleService.cs
Packages/NarrativeFramework/Content/Locale/LocaleRegistry.cs
Packages/NarrativeFramework/Content/Locale/SubstitutionEngine.cs
Packages/NarrativeFramework/Content/Locale/LocaleDiagnostics.cs
Packages/NarrativeFramework/Content/Definitions/LocaleTableDefinition.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/LocaleCoverageStage.cs
Packages/NarrativeFramework/Tests/EditMode/Locale/LocaleResolveTests.cs
Packages/NarrativeFramework/Tests/EditMode/Locale/LocaleFallbackTests.cs
Packages/NarrativeFramework/Tests/EditMode/Locale/LocaleSubstitutionTests.cs
Samples~/FrameworkTestPack/Locale/locale_table_en.json
Samples~/FrameworkTestPack/Locale/locale_table_fr.json
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Resolve_ExistingKey_ReturnsText` | Exact match |
| `Resolve_MissingLocale_UsesFallback` | Fallback string |
| `Resolve_Substitution_ReplacesTokens` | `{name}` → value |
| `HasKey_ReturnsFalse_WhenAbsent` | |
| `Pipeline_MissingTranslation_Fails` | Error severity |
| `DuplicateKeySameLocale_FailsBuild` | Pipeline Error |

**Exit criteria:** Locale fallback test in Phase 12 suite.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) S-03, S-04, PR-06 (locale settings Phase 13).

---

## Authoring conventions

Locale keys follow dialogue/content ID naming from the glossary:

| Pattern | Example | Used by |
|---|---|---|
| `{nodeId}_text` | `kim_intro_01_text` | Dialogue node body |
| `{nodeId}_choice_{n}_text` | `kim_intro_choice_01_text` | Choice labels |
| `{beliefId}_title` | `belief_volumetric_title` | Belief cabinet |
| `{itemId}_name` | `item_trenchcoat_name` | Inventory UI Phase 13 |

Authors never embed player-specific values in keys — use `{token}` substitution at resolve time. Pipeline rejects keys containing spaces or uppercase.

### Multi-table merge rules

When multiple `LocaleTableDefinition` assets target the same `localeId`:

1. Build merges in manifest declaration order.
2. Duplicate `(localeId, key)` → **Error** in MVP (no silent override).
3. `[FULL]` explicit table priority field for DLC overlays.

### Debug vs shipping behavior

| Build | Missing key behavior |
|---|---|
| Edit Mode tests | Assert `HasKey` — no silent fallback in test locale |
| Development player | Bracket placeholder + `Debug.LogWarning` |
| Shipping `[FULL]` | Fallback chain only; optional Sentry report |

---

## Related documents

- [content-pipeline.md](content-pipeline.md) — LocaleCoverageStage
- [content-store.md](content-store.md) — table definitions
- [story-dialogue.md](story-dialogue.md) — primary consumer
