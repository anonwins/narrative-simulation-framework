# NSF Package Foundation

**Phase 0 architecture** — empty compilable package, contract surface, test harness.

- Spec: [runtime-kernel.md](../runtime-kernel.md)
- Roadmap: [Phase 0](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md)
- Shared types: [data-model.md](data-model.md)

---

## Design rationale

### Why a dedicated Unity package

NSF targets **multiple games** (NoirSample, SciFiSample, future content packs). A UPM package under `Packages/NarrativeFramework/` keeps framework code isolated from game assets, enables semver, and prevents genre logic leaking into core modules.

### Why asmdefs are decided upfront

Circular references between Story, Presentation, and Cognition destroyed maintainability in monolithic narrative projects. Locked dependency arrows (roadmap matrix) are **non-negotiable**: violating them breaks Edit Mode testing and sample reusability.

### Why Phase 0 is behavior-free

Implementing logic before all contracts exist causes rename churn. Phase 0 proves **composition works**: every `I*Service` compiles, registers, and resolves. Tests only assert assembly load and registry wiring.

---

## Implementation summary

| MVP (Phase 0) | Full (later phases) |
|---|---|
| Folder layout + asmdefs | Full service implementations |
| All interfaces from [contracts.md](contracts.md) | Kernel tick with real phase handlers |
| `Null*` stub for every `I*Service` | Replace stubs phase by phase |
| `NarrativeServiceRegistry` | Optional DI container adapter |
| `NsfVersion`, `NsfModuleIds` | CI semver gates |
| 1 smoke test | 200+ tests by Phase 14 |

---

## Assembly and namespace

### Layout

```text
Packages/NarrativeFramework/
  package.json
  Runtime/
    NarrativeFramework.Runtime.asmdef
    NsfVersion.cs
    NsfModuleIds.cs
    Kernel/
    Registry/
  Content/
    NarrativeFramework.Content.asmdef
  Rules/
    NarrativeFramework.Rules.asmdef
  Simulation/
    NarrativeFramework.Simulation.asmdef
  Cognition/
    NarrativeFramework.Cognition.asmdef
  Social/
    NarrativeFramework.Social.asmdef
  Story/
    NarrativeFramework.Story.asmdef
  Ledger/
    NarrativeFramework.Ledger.asmdef
  Presentation/
    NarrativeFramework.Presentation.asmdef
  Debug/
    NarrativeFramework.Debug.asmdef          (Editor-only references)
  Editor/
    NarrativeFramework.Editor.asmdef
  Tests/
    EditMode/
      NarrativeFramework.Tests.asmdef
Samples~/
  FrameworkTestPack/                         (Phase 2 — data only)
  NoirSample/                                (Phase 15)
  SciFiSample/                               (Phase 16)
```

### Reference matrix (allowed dependencies)

| Assembly | References |
|---|---|
| Runtime | *(none — root)* |
| Content | Runtime |
| Rules | Runtime, Content |
| Simulation | Runtime, Content |
| Cognition | Runtime, Content, Rules |
| Social | Runtime, Simulation |
| Story | Runtime, Content, Rules, Cognition |
| Ledger | Runtime, Story, Simulation, Cognition |
| Presentation | all runtime modules *(bridges only)* |
| Debug | all runtime + Editor |
| Editor | all runtime |
| Tests | all runtime |

**Forbidden:** Cognition, Simulation, Story, or Ledger → Presentation.

### Namespace convention

`NarrativeFramework.{Module}.{Area}` — e.g. `NarrativeFramework.Runtime.Kernel`, `NarrativeFramework.Simulation.Events`.

---

## Public API

See [contracts.md](contracts.md). Phase 0 declares:

- `ISimulationKernel`, `INarrativeServiceRegistry`
- All 32 `I*Service` interfaces
- `IEventBus`, handler interfaces
- Stub concrete types: `NullSimulationKernel`, `NullFactService`, … (one per service)

---

## Internal implementation

### NarrativeServiceRegistry

**Why typed registry:** Unity lacks a standard DI container in player builds. A minimal dictionary-backed registry is test-friendly and explicit.

```csharp
namespace NarrativeFramework.Runtime.Registry
{
    sealed class NarrativeServiceRegistry : INarrativeServiceRegistry
    {
        readonly Dictionary<Type, object> _services = new();

        public void Register<T>(T service) where T : class
            => _services[typeof(T)] = service ?? throw new ArgumentNullException(nameof(service));

        public T GetRequired<T>() where T : class
            => TryGet<T>(out var s) ? s : throw new InvalidOperationException($"Service not registered: {typeof(T).Name}");

        public bool TryGet<T>(out T service) where T : class
        {
            if (_services.TryGetValue(typeof(T), out var obj)) { service = (T)obj; return true; }
            service = null; return false;
        }
    }
}
```

### Null stub pattern

**Why one stub per interface:** Edit Mode tests swap real implementations incrementally. Stubs satisfy compile and allow `GetRequired` without null checks in consumer skeleton code.

```csharp
sealed class NullFacultyService : IFacultyService
{
    public int GetValue(string facultyId) => 0;
    // ... remaining members: safe defaults, no throw unless contract requires
}
```

`NullSimulationKernel.Tick()` is a no-op; `CurrentPhase` returns `Facts`.

### Bootstrap (Phase 0)

```csharp
static class NsfBootstrap
{
    public static INarrativeServiceRegistry CreatePhase0Registry()
    {
        var registry = new NarrativeServiceRegistry();
        registry.Register<IFacultyService>(new NullFacultyService());
        // ... register all I* stubs
        registry.Register<ISimulationKernel>(new NullSimulationKernel(registry));
        return registry;
    }
}
```

Phase 1 replaces kernel + event + fact stubs with real implementations.

### Version constants

```csharp
public static class NsfVersion
{
    public const string Framework = "0.1.0-phase0";
    public const int SaveSchemaVersion = 1;
}

public static class NsfModuleIds
{
    public const string Runtime = "NarrativeFramework.Runtime";
    // ... one const per asmdef
}
```

---

## Definition assets

Phase 0: none. Phase 2 introduces `ContentDefinition` ScriptableObjects in Content assembly.

---

## Runtime state

Phase 0: no persistent state. `IPersistenceService` stub returns empty snapshot.

---

## Core algorithms

### Phase 0 startup

1. `NsfBootstrap.CreatePhase0Registry()`
2. `kernel.Initialize(registry)` — stores registry reference
3. Tests call `GetRequired<IFacultyService>()` — must not throw

No `Tick()` behavior until Phase 1.

---

## Event contracts

Phase 0: `IEventBus` stub — `Publish` no-op, `Subscribe` stores handlers in memory list (for wiring tests only).

---

## Integration matrix

| Consumer | Phase 0 uses |
|---|---|
| Tests | Full registry |
| Editor | Optional menu to log registered types |
| Samples | Not yet |

---

## MVP scope (Phase 0 deliverables)

- [ ] All folders + asmdefs per layout
- [ ] All interfaces in [contracts.md](contracts.md)
- [ ] `Null*` for every `I*Service` + `NullSimulationKernel`
- [ ] `NsfVersion`, `NsfModuleIds`
- [ ] `run-nsf-edit-mode-tests.ps1` → `NsfAssemblyLoadsTests`
- [ ] Zero `Greywater.*` references

**Exit:** batch compile success; 1/1 test pass.

---

## Full scope

- `[FULL]` Source generators for service registration
- `[FULL]` Roslyn analyzers enforcing asmdef rules
- `[FULL]` Optional Zenject/ VContainer adapter implementing `INarrativeServiceRegistry`

---

## File tree (Phase 0 minimum)

```text
Packages/NarrativeFramework/Runtime/Registry/INarrativeServiceRegistry.cs
Packages/NarrativeFramework/Runtime/Registry/NarrativeServiceRegistry.cs
Packages/NarrativeFramework/Runtime/Kernel/ISimulationKernel.cs
Packages/NarrativeFramework/Runtime/Kernel/NullSimulationKernel.cs
Packages/NarrativeFramework/Runtime/NsfBootstrap.cs
Packages/NarrativeFramework/Runtime/NsfVersion.cs
Packages/NarrativeFramework/{Module}/I*Service.cs        (per module)
Packages/NarrativeFramework/{Module}/Null*Service.cs     (per module)
Packages/NarrativeFramework/Tests/EditMode/NsfAssemblyLoadsTests.cs
run-nsf-edit-mode-tests.ps1                               (repo root)
```

---

## Test plan

| Test | Purpose |
|---|---|
| `NsfAssemblyLoadsTests.AllAssembliesResolve` | Every asmdef loads |
| `NsfAssemblyLoadsTests.Phase0RegistryResolvesAllServices` | Register bootstrap; `GetRequired` for each `I*Service` |
| `NsfAssemblyLoadsTests.KernelInitializeDoesNotThrow` | Smoke |

**Gate:** `.\run-nsf-edit-mode-tests.ps1` exit 0.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) P-03, P-04, P-12, CP-03.

---

## Related documents

- [unity-host.md](unity-host.md) — Unity lifecycle, tick driver, 2D/3D boundaries
- [data-model.md](data-model.md) — DTO conventions (Phase 2+)
- [runtime-kernel.md](runtime-kernel.md) — real tick (Phase 1)
- [editor.md](editor.md) — Editor menus (Phase 2+)
