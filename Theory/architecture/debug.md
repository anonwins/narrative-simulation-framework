# Debug Module — Implementation Architecture

- Spec: [runtime-kernel.md](../runtime-kernel.md) (Debugging section)
- Roadmap: [Phase 14](../development-roadmap.md)
- Foundation: [foundation.md](foundation.md) — Debug asmdef
- Integration: [integration.md](integration.md)

---

## Design rationale

### Why Debug is Editor-only and separate from Presentation

Inspectors, causal traces, and roll math overlays are development affordances — not player-facing bridges. Placing debug UI in Presentation would blur the "headless presenter" boundary and risk shipping debug hooks in player builds. `NarrativeFramework.Debug` references all runtime modules **plus** Unity Editor APIs; it never ships in standalone player assemblies when stripped correctly.

### Why service inspector matters for NSF complexity

Forty interconnected services make "printf debugging" unusable. A unified **service inspector** shows live registry contents, tick phase, and last event batch — the first tool an agent uses when `FullLoopIntegrationTests` fails mid-loop.

### Why causal tracing follows kernel phases

Bugs often manifest as "gate blocked wrongly" when the real fault was a fact registered in the wrong phase. Debug trace aligns log lines to `SimulationTickPhase` so developers see Facts before Gating violations immediately.

### Why core must not reference Debug

Runtime assemblies compile for player builds. Debug attaches **observers** via Editor windows and optional `IDebugTraceSink` registration at bootstrap in Editor only — inversion of control, not compile-time dependency from Story/Simulation.

---

## Implementation summary

| MVP (Phase 14) | Full (Phase 17+) |
|---|---|
| Service registry inspector window | Performance per-phase timing |
| Fact viewer (read-only) | Save snapshot diff tool |
| Event trace log | Chronicle projection diff |
| Roll log with seed + modifiers | Visual graph of rule firings `[FULL]` |
| Tick stepper (single phase) | Record/replay tick `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Debug`
- **Platform:** Editor-only (`includePlatforms: ["Editor"]`)
- **Namespace:** `NarrativeFramework.Debug`
- **References:** All runtime assemblies, `UnityEditor`

**Forbidden:** Runtime → Debug (reverse reference).

---

## Public API

Debug exposes **Editor tools**, not player contracts. Optional runtime hook interface (declared in Runtime, implemented in Debug):

```csharp
// Runtime/Debug/IDebugTraceSink.cs — no UnityEditor references
interface IDebugTraceSink
{
    void OnTickPhaseStarted(SimulationTickPhase phase);
    void OnEventPublished(SimEvent evt);
    void OnRollResolved(RollResult result);
    void OnFactChanged(FactRecord fact, FactChangeKind kind);
}

enum FactChangeKind { Registered, Revoked, Updated }
```

`NsfBootstrap.CreateEditorRegistry()` registers `DebugTraceSink` when `UNITY_EDITOR`.

No new glossary `I*Service` — debug is tooling, not simulation.

---

## Internal implementation

### NsfDebugWindow (hub)

```csharp
#if UNITY_EDITOR
class NsfDebugWindow : EditorWindow
{
    [MenuItem("Tools/NSF/Debug Dashboard")]
    static void Open() => GetWindow<NsfDebugWindow>("NSF Debug");

    INarrativeServiceRegistry _registry;
    IDebugTraceSink _trace;
    int _selectedTab;

    void OnGUI()
    {
        _selectedTab = GUILayout.Toolbar(_selectedTab, new[] { "Services", "Facts", "Events", "Rolls", "Tick" });
        switch (_selectedTab)
        {
            case 0: DrawServiceInspector(); break;
            case 1: DrawFactViewer(); break;
            case 2: DrawEventTrace(); break;
            case 3: DrawRollLog(); break;
            case 4: DrawTickStepper(); break;
        }
    }
}
#endif
```

### ServiceInspectorPanel

Read-only reflection over registry:

- Lists registered service types
- Expandable: dump `IFactService` count, active dialogue node ID, current location
- **Never** mutates services except explicit "Debug Apply" gated actions `[FULL]`

### FactViewerPanel

```csharp
sealed class FactViewerPanel
{
    readonly IFactService _facts;

    public void Draw()
    {
        foreach (var fact in _facts.GetAllFacts())
            EditorGUILayout.LabelField(fact.Id, fact.Value?.ToString() ?? "—");
    }
}
```

Read-only in MVP — mutations would bypass rule engine.

### EventTraceBuffer

Ring buffer of last N events:

```csharp
sealed class EventTraceBuffer : IDebugTraceSink
{
    readonly Queue<TraceEntry> _entries = new();
    const int MaxEntries = 500;

    public void OnEventPublished(SimEvent evt)
    {
        Enqueue(new TraceEntry(DateTime.UtcNow, SimulationTickPhase.Events, evt));
    }
}
```

UI: filter by type, tick phase, search fact IDs in payload.

### RollLogPanel

Captures from `IDebugTraceSink.OnRollResolved`:

| Column | Source |
|---|---|
| Seed | `RollResult.Seed` |
| Faculty | `RollResult.FacultyId` |
| DC / total | `RollResult` |
| Modifiers | Equipment, emotion, conduct |
| Outcome | Pass/fail |

Critical for verifying deterministic integration tests.

### TickStepperPanel

Wraps `ISimulationKernel.Tick(singlePhase)`:

```text
[Tick Facts] [Tick Interpretation] ... [Tick Full]
Current phase: Gating
Last trace: FactRegistered(fact_test_clue)
```

Uses same kernel instance as Play Mode / Edit Mode tests.

### DebugOverlayHook `[FULL]`

Optional scene overlay in sample scenes — draws current location, active thread, blocked gates. Lives in Debug, invoked from sample bootstrap `#if UNITY_EDITOR`.

### IntegrationFailureDiagnostics helper

When `FullLoopIntegrationTests` fail, dumping raw assertion text is insufficient. A shared helper formats registry + trace state:

```csharp
static class IntegrationFailureDiagnostics
{
    public static string BuildReport(INarrativeServiceRegistry registry, IDebugTraceSink sink)
    {
        var sb = new StringBuilder();
        sb.AppendLine("=== NSF Integration Failure Report ===");
        sb.AppendLine($"Phase: {registry.GetRequired<ISimulationKernel>().CurrentPhase}");
        foreach (var fact in registry.GetRequired<IFactService>().GetAllFacts())
            sb.AppendLine($"  FACT {fact.Id}");
        // Append last 20 trace entries from sink buffer
        return sb.ToString();
    }
}
```

Tests call `TestContext.WriteLine(report)` on failure — agents read Unity Test Runner output without opening Editor.

### Presentation trace separation

Debug may **observe** headless presenter last models for integration diagnosis, but must not bind Canvas presenters. A read-only `PresenterCapturePanel` `[FULL]` shows:

| Presenter | Last model summary |
|---|---|
| `HeadlessDialoguePresenter` | Speaker, choice count |
| `HeadlessChroniclePresenter` | Section count |
| `HeadlessBeliefView` | BeliefId, phase |

This panel queries test bundle instances passed by reference in Edit Mode — not production player UI.

---

## Definition assets

None — Debug consumes runtime state only.

---

## Runtime state

| State | Owner | Persisted |
|---|---|---|
| Event trace buffer | `EventTraceBuffer` | No (session) |
| Roll log | `RollLogBuffer` | No |
| Window prefs | EditorPrefs | Local |

---

## Core algorithms

### Attach debug sink at bootstrap

1. `NsfBootstrap.CreateRegistry()` builds production services
2. If Editor: `registry.Register<IDebugTraceSink>(new CompositeDebugSink(...))`
3. Kernel, event bus, roll service check `TryGet<IDebugTraceSink>()` before publish
4. Player build: sink absent, checks no-op

### Trace write path (non-invasive)

```csharp
// In EventBus.Publish — Simulation assembly
public void Publish(SimEvent evt)
{
    _queue.Enqueue(evt);
    _registry.TryGet<IDebugTraceSink>(out var sink);
    sink?.OnEventPublished(evt);
}
```

Simulation references `IDebugTraceSink` **interface in Runtime** — not Debug assembly.

---

## Event contracts

Debug **observes** all `SimEvent` types; does not publish domain events.

Optional Editor-only:

```csharp
class DebugCommandIssuedEvent : SimEvent { public string CommandId; }
```

For replay tooling `[FULL]`.

---

## Integration matrix

| Observes | Via |
|---|---|
| `INarrativeServiceRegistry` | Service inspector |
| `IFactService` | Fact viewer |
| `IEventBus` | Trace sink |
| `IRollService` | Roll log |
| `ISimulationKernel` | Tick stepper |

| Consumed by | Usage |
|---|---|
| Developers / agents | Diagnose integration failures |
| Phase 14 gate | Manual verification alongside automated tests |

**Forbidden:** Debug types in Runtime implementations except `IDebugTraceSink` interface.

---

## MVP scope (Phase 14)

- [ ] `NarrativeFramework.Debug` asmdef Editor-only
- [ ] `IDebugTraceSink` in Runtime + no-op when unregistered
- [ ] `NsfDebugWindow` with 5 tabs
- [ ] Event trace ring buffer (500 entries)
- [ ] Roll log with seed display
- [ ] Tick single-phase stepper
- [ ] Menu: `Tools/NSF/Debug Dashboard`
- [ ] Wired into `FullLoopIntegrationTests` failure diagnostics (log dump helper)

---

## Full scope

- `[FULL]` Rule engine evaluation tree visualizer
- `[FULL]` Save snapshot diff
- `[FULL]` Performance timing per phase (Phase 17 benchmarks)
- `[FULL]` Chronicle projection vs thread source diff

---

## File tree

```text
Packages/NarrativeFramework/Runtime/Debug/IDebugTraceSink.cs
Packages/NarrativeFramework/Debug/NarrativeFramework.Debug.asmdef
Packages/NarrativeFramework/Debug/NsfDebugWindow.cs
Packages/NarrativeFramework/Debug/Panels/ServiceInspectorPanel.cs
Packages/NarrativeFramework/Debug/Panels/FactViewerPanel.cs
Packages/NarrativeFramework/Debug/Panels/EventTracePanel.cs
Packages/NarrativeFramework/Debug/Panels/RollLogPanel.cs
Packages/NarrativeFramework/Debug/Panels/TickStepperPanel.cs
Packages/NarrativeFramework/Debug/Internal/EventTraceBuffer.cs
Packages/NarrativeFramework/Debug/Internal/RollLogBuffer.cs
Packages/NarrativeFramework/Debug/Internal/CompositeDebugSink.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `DebugSink_OnEvent_IncrementsBuffer` | Edit Mode with sink registered |
| `DebugWindow_OpensWithoutPlayMode` | Editor test `[UnityTest]` optional |
| `SinkAbsent_PlayerCodePath_NoThrow` | Publish without sink |
| `RollLog_CapturesSeed` | Deterministic replay |

Debug UI manual verification acceptable; automated coverage on sink + buffers.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) DBG-01–DBG-04.

---

## Related documents

- [runtime-kernel.md](runtime-kernel.md) — tick phases for stepper
- [integration.md](integration.md) — failure diagnosis during full loop
- [editor.md](editor.md) — setup pipeline menu sibling
- [present-ui.md](present-ui.md) — headless vs debug UI separation
