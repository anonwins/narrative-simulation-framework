# Content Script Compiler — Implementation Architecture

- Spec: [systems/content-script.md](../systems/content-script.md)
- Glossary: [NSF Script](../terminology-glossary.md)
- Roadmap: [Phase 12](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [content-pipeline.md](content-pipeline.md) · [content-store.md](content-store.md)

---

## Design rationale

### Why a DSL instead of only ScriptableObjects

ScriptableObjects excel for structured data; they are poor for branching narrative flow authored by writers. NSF Script is the **bridge between writers and simulation** — dialogue, choices, conditions, roll nodes, and flag effects in one readable file without C# changes.

### Why ScriptCompiler is concrete, not I*Service

Compilation is a build-time transform, not a per-tick runtime service. The glossary lists `ScriptCompiler` alongside `RuleEngine` and `ThreadEngine` as orchestrators resolved explicitly by pipeline and Editor tools — not registered in `INarrativeServiceRegistry`.

### Why MVP subset first

Full NSF Script includes flow control, variables, debug tracing, and complex condition grammar. Phase 12 MVP targets: **dialogue refs, flag sets, roll nodes** — enough for agent-authored sample `.nsf` and pipeline integration without blocking on a complete parser.

### Why compile to definitions, not bytecode

Runtime systems already consume `ContentDefinition` assets and dialogue graph models. The compiler **lowers** script AST into existing definition types (`DialogueGraphDefinition`, flag mutation actions) so Story and Rules modules need no second interpreter.

### Why separate from Rule Engine

Rule Engine evaluates conditions at runtime. Script **authors** conditions as text/expressions that compile into `Rule` / `Condition` structures. Writers never call `IRuleEngine` directly; compiled output does.

---

## Implementation summary

| MVP (Phase 12) | Full |
|---|---|
| Lexer + parser for NODE / CHOICE / ROLL / SET_FLAG | GO_TO, END, VARIABLE access |
| `Compile` / `CompileFile` entry points | DEBUG trace directives |
| Emit `ScriptNode[]` + errors | Source maps for pipeline issues |
| Pipeline `ScriptCompileStage` | IDE language server `[FULL]` |
| Sample `.nsf` in FrameworkTestPack | Full condition expression grammar |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Content` |
| **References** | Runtime, Content definitions, Rules (condition AST types Phase 5+) |
| **Namespace** | `NarrativeFramework.Content.Script` |
| **Tick phase** | None — compile at build; runtime executes compiled graphs via Story services |

---

## Public API

See [contracts.md](contracts.md) — `ScriptCompiler`.

**Module notes:**

- `Compile(source, filePath)` — `filePath` used in error messages and `[FULL]` source maps.
- `CompileFile(nsfFilePath)` — reads UTF-8 text, delegates to `Compile`.
- Returns `ScriptCompileResult` — never throws for syntax errors; check `Success`.

Not registered in service registry. Resolved by `ContentPipeline` and Editor import menu.

---

## Internal implementation

### ScriptCompiler

```csharp
namespace NarrativeFramework.Content.Script
{
    sealed class ScriptCompiler
    {
        public ScriptCompileResult Compile(string source, string filePath)
        {
            var lexer = new NsfLexer(source, filePath);
            var tokens = lexer.Tokenize();
            if (lexer.Errors.Count > 0)
                return ScriptCompileResult.FromErrors(lexer.Errors);

            var parser = new NsfParser(tokens, filePath);
            var ast = parser.ParseDocument();
            if (parser.Errors.Count > 0)
                return ScriptCompileResult.FromErrors(parser.Errors);

            var emitter = new ScriptDefinitionEmitter();
            return emitter.Emit(ast, filePath);
        }

        public ScriptCompileResult CompileFile(string nsfFilePath)
        {
            var source = File.ReadAllText(nsfFilePath);
            return Compile(source, nsfFilePath);
        }
    }
}
```

### Lexer / Parser (MVP grammar)

MVP keywords:

```text
NODE <id>
  SPEAKER <actor_id>
  TEXT <locale_key>
  CHOICE <id> -> <target_node>
  ROLL <roll_id> FACULTY <faculty_id> DC <n> SUCCESS -> <node> FAIL -> <node>
  SET_FLAG <flag_id> <true|false>
END
```

```csharp
sealed class NsfLexer
{
    public List<CompileError> Errors { get; } = new();
    public IReadOnlyList<Token> Tokenize() { /* ... */ }
}

sealed class NsfParser
{
    public List<CompileError> Errors { get; } = new();
    public ScriptDocument ParseDocument() { /* ... */ }
}
```

### ScriptDefinitionEmitter

Lowers AST to:

- `ScriptNode[]` (intermediate, for tests)
- `[FULL]` `DialogueGraphDefinition` + embedded roll refs for pipeline registration

```csharp
sealed class ScriptDefinitionEmitter
{
    public ScriptCompileResult Emit(ScriptDocument doc, string filePath)
    {
        var nodes = new List<ScriptNode>();
        foreach (var nodeAst in doc.Nodes)
            nodes.Add(ConvertNode(nodeAst));
        return new ScriptCompileResult { Success = true, Nodes = nodes.ToArray() };
    }
}
```

### Domain types (spec-aligned)

```csharp
class ScriptNode
{
    public string Id;
    public ScriptCondition[] Conditions;
    public ScriptEffect[] Effects;
}

class ScriptCondition
{
    public string Expression;  // MVP: flag checks; FULL: rule engine expr
}

class ScriptEffect
{
    public string Action;   // SET_FLAG, START_DIALOGUE, ...
    public string Target;
}

class ScriptCompileResult
{
    public bool Success;
    public ScriptNode[] Nodes;
    public string[] Errors;
}
```

---

## Definition assets

Compiler **output** becomes content definitions — not standalone ScriptableObjects for raw `.nsf` text.

| Output | Registered as |
|---|---|
| Dialogue flow | `DialogueGraphDefinition` (Phase 4 consumer) |
| Roll nodes | References `FacultyRoll` definition IDs |
| Flag effects | `StoryFlagDefinition` mutations in graph actions |

Source `.nsf` files live in content pack; pipeline compiles before `ContentRegistry` load.

---

## Runtime state / persistence

No runtime compiler state. Compiled definitions are immutable in store.

`ScriptCompileResult` is ephemeral — pipeline either merges into manifest or fails build.

---

## Core algorithms

### Compile pipeline

1. **Tokenize** — line-oriented MVP; track line/column for errors.
2. **Parse** — build `ScriptDocument` with `ScriptNodeAst` list and choice/roll edges.
3. **Validate refs** — node targets exist; IDs match `ContentIdValidator` prefixes.
4. **Emit** — produce `ScriptNode[]` + optional definition objects.
5. **Hand off** — `ScriptCompileStage` adds emitted definitions to `PipelineContext.LoadedDefinitions`.

### Roll node lowering

```text
ROLL check_intimidate FACULTY faculty_authority DC 12 SUCCESS -> node_win FAIL -> node_lose
```

Emits:

- `ScriptEffect` with action `INVOKE_ROLL`, target `check_intimidate`
- Graph edges stored on dialogue definition for Story service `[integration Phase 4/12]`

### Error reporting

```csharp
class CompileError
{
    public string FilePath;
    public int Line;
    public int Column;
    public string Message;
}
```

Format: `{filePath}({line},{col}): error NSF001: unknown node target 'node_missing'`

---

## Event contracts + kernel tick phase

**No direct kernel participation.** Compiled dialogue executes in **Content** phase via `IDialogueService` when player selects choices.

| Indirect event | Source |
|---|---|
| `DialogueStarted` | Story service after script graph entry |
| `RollResolved` | Cognition after ROLL node `[Phase 3/12 integration]` |
| `FlagChanged` | Story state after SET_FLAG effect |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `ContentIdValidator` | ID checks in emitter |
| `IContentPipeline` | ScriptCompileStage invocation |
| `IRuleEngine` types (Phase 5+) | Condition lowering `[FULL]` |
| `FacultyRoll` / roll definitions | ROLL nodes |

| Used by | Reason |
|---|---|
| `ContentPipeline` | Build-time compile |
| Editor import | Recompile on `.nsf` save |
| Agent-authored samples | Phase 12 exit criteria |
| `IDialogueService` | Consumes compiled graphs |

---

## MVP scope checklist (Phase 12)

- [ ] `ScriptCompiler.Compile` / `CompileFile`
- [ ] MVP grammar: NODE, TEXT key, CHOICE, ROLL, SET_FLAG, END
- [ ] `ScriptCompileResult` with Success + Errors
- [ ] `ScriptCompileStage` in pipeline
- [ ] Sample `.nsf` file compiles without errors
- [ ] Broken sample produces actionable compile errors
- [ ] Compiled output registrable in ContentRegistry (via emitter → definitions)

---

## Full scope

- `[FULL]` CONDITIONS block with full rule expression grammar
- `[FULL]` GO_TO, END, VARIABLE read/write
- `[FULL]` DEBUG trace directives for sim replay
- `[FULL]` Source maps linking runtime errors to `.nsf` line
- `[FULL]` Source generator: `.nsf` → C# constants `[data-model deferred]`

---

## File tree

```text
Packages/NarrativeFramework/Content/Script/ScriptCompiler.cs
Packages/NarrativeFramework/Content/Script/ScriptCompileResult.cs
Packages/NarrativeFramework/Content/Script/ScriptNode.cs
Packages/NarrativeFramework/Content/Script/ScriptCondition.cs
Packages/NarrativeFramework/Content/Script/ScriptEffect.cs
Packages/NarrativeFramework/Content/Script/CompileError.cs
Packages/NarrativeFramework/Content/Script/Lexer/NsfLexer.cs
Packages/NarrativeFramework/Content/Script/Lexer/Token.cs
Packages/NarrativeFramework/Content/Script/Parser/NsfParser.cs
Packages/NarrativeFramework/Content/Script/Parser/ScriptDocument.cs
Packages/NarrativeFramework/Content/Script/Emitter/ScriptDefinitionEmitter.cs
Packages/NarrativeFramework/Content/Pipeline/Stages/ScriptCompileStage.cs
Packages/NarrativeFramework/Tests/EditMode/Script/ScriptCompilerGoodSampleTests.cs
Packages/NarrativeFramework/Tests/EditMode/Script/ScriptCompilerErrorTests.cs
Samples~/FrameworkTestPack/Scripts/sample_beat.nsf
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Compile_ValidSample_Success` | Success true, Nodes.Length > 0 |
| `Compile_MissingTarget_Fails` | Error contains node id + line |
| `CompileFile_ReadsUtf8` | Same as Compile with path |
| `RollNode_EmitsInvokeRollEffect` | Effect action + target |
| `SetFlag_EmitsCorrectEffect` | Target flag_id |
| `Pipeline_WithNsf_IncludesCompiledDefs` | Build Success with script path |

**Exit criteria:** ≥15 Phase 12 tests; agent-authored sample `.nsf` runs through compiler.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SCR-01, SCR-02, SCR-03, SCR-04.

---

## Related documents

- [content-pipeline.md](content-pipeline.md) — ScriptCompileStage
- [content-store.md](content-store.md) — registered definitions
- [story-dialogue.md](story-dialogue.md) — runtime graph execution (Phase 4)
- [cognition-roll.md](cognition-roll.md) — ROLL node resolution
