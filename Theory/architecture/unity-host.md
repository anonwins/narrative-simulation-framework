# Unity Host Integration & Platform Boundaries

**How NSF lives inside a Unity process** — bootstrap, tick scheduling, engine bridges, and what NSF deliberately does *not* own.

- Package layout: [foundation.md](foundation.md)
- Contracts: [contracts.md](contracts.md)
- Full wiring: [integration.md](integration.md)
- Samples: [samples.md](samples.md)
- Roadmap: [Phase 0–14](../development-roadmap.md)

---

## Design rationale

### Why NSF is not a “Unity game template”

NSF is a **narrative simulation library**. Unity supplies rendering, physics, input devices, and scene files. NSF supplies facts, dialogue, rolls, chronicle logic, and gating. Mixing those concerns in one framework makes every game look like the same genre *and* makes Edit Mode testing impossible without loading scenes.

Games choose **2D vs 3D**, art style, camera, and control scheme. NSF exposes **abstract presentation contracts** (`IExplorationService`, `IInteractionService`, headless presenters) that any visual style can implement.

### Why one host document was missing

Module architecture docs assumed a registry and kernel exist. They did not specify **who constructs the registry**, **when `Tick()` runs**, or **how Unity lifecycle maps to narrative state**. This document closes that gap so Phase 0 code has a single SSOT for host integration.

---

## Three layers — responsibility matrix

| Concern | Owner | NSF involvement |
|---|---|---|
| **2D / 3D / orthographic / perspective** | Game + Unity | None — samples pick independently |
| Rendering pipeline (URP/BiRP/HDRP) | Game | None — URP optional in Noir/SciFi samples |
| Cameras, lighting, post-processing | Game | None |
| Character meshes, animation, lip-sync | Game | None — voice *text* via `IVoiceService`; audio playback via `IAudioNarrativeService` bridge |
| Physics, navmesh, collision | Game | None — interaction uses abstract `IWorldObject` IDs |
| Input (keyboard, gamepad, touch) | Game → Presentation | Presentation translates input into `IInteractionService` / dialogue choices |
| Scene files (`.unity`) | Game + Presentation | Simulation uses `location_*` graph; Unity scene path is Presentation metadata only |
| UI layout & skin (Canvas, UITK) | Game + Presentation | NSF defines view models + presenter interfaces |
| Narrative simulation | **NSF** | Core modules |
| Content data (dialogue, beliefs, threads) | Content pack | Loaded via `IContentStore` |
| Save format & schema | **NSF** (`IPersistenceService`) | Game chooses save slot UI / cloud upload |

**Rule:** If it appears on screen or in a `.unity` scene as a GameObject, it is **not** a Cognition/Simulation/Story/Ledger responsibility unless it is a Presentation bridge explicitly documented in [present-*.md](present-ui.md).

### 2D vs 3D — explicit decision

**NSF core does not choose 2D or 3D — samples do.**

- **NSF core:** dimension-agnostic. No references to `Camera`, `SpriteRenderer`, `CharacterController`, or NavMesh in Simulation/Story/Cognition/Ledger asmdefs.
- **NSF Presentation (Phase 13+):** bridges only — e.g. `IExplorationService.Enter(locationId)`; the sample implements enter with 2D scene load, 3D scene load, or pure UI map.
- **Samples (decided):** **NoirSample = 3D**, **SciFiSample = 2D**. Together they cover all NSF modules — see [samples.md](samples.md) § Feature coverage matrix.

```text
NSF:     "Player entered location_warehouse"
Noir:    Load WarehouseScene3D.unity — walkable crime scene
SciFi:   Show deck_map_2D hotspot graph — click corridor B
```

---

## Internal tech stack (NSF package)

| Layer | Choice | Why |
|---|---|---|
| Language | C# (Unity 6000.4.10f1 runtime) | Roadmap lock |
| Packaging | UPM `Packages/NarrativeFramework/` | Multi-game reuse |
| Modularity | asmdefs per module | Enforce Presentation isolation |
| DI / composition | `INarrativeServiceRegistry` | No container dependency in MVP; test-friendly |
| Content authoring | ScriptableObjects + JSON import | Unity-native + pipeline automation |
| Serialization (saves/content) | Newtonsoft JSON | Already in manifest; human-debuggable saves |
| Unit/integration tests | Unity Test Framework **Edit Mode** | No Play Mode required until optional smoke |
| Batch CI | `invoke-unity.ps1` + `run-nsf-edit-mode-tests.ps1` | Agent-verifiable gates |
| UI (Phase 13+) | uGUI for sample skins; presenters abstract | Core tests use headless presenters |
| Script DSL | `.nsf` + `ScriptCompiler` (Phase 12) | Designer-facing logic |
| Threading | **Main thread only** | Unity player loop; no background sim `[FULL]` async queue |
| Networking | **Out of scope** | See § Out of scope |

---

## Host architecture

```text
┌─────────────────────────────────────────────────────────────┐
│  Game / Sample (Layer 3)                                     │
│  MonoBehaviour bootstrap, scenes, 2D/3D art, input routing   │
└──────────────────────────┬──────────────────────────────────┘
                           │ creates & owns
┌──────────────────────────▼──────────────────────────────────┐
│  NsfSession (pure C#) — NarrativeFramework.Runtime.Host      │
│  Registry · Kernel · TickScheduler · ContentPackHandle       │
└──────────────────────────┬──────────────────────────────────┘
                           │ registers
┌──────────────────────────▼──────────────────────────────────┐
│  I*Service implementations (module assemblies)                │
└──────────────────────────┬──────────────────────────────────┘
                           │ optional binds (Phase 13+)
┌──────────────────────────▼──────────────────────────────────┐
│  Presentation bridges (IUIShell, IExplorationService, …)   │
└─────────────────────────────────────────────────────────────┘
```

### NsfSession (canonical composition root)

**Why pure C# session object:** Edit Mode tests construct `NsfSession` without `GameObject`. Samples use a thin `NsfGameHost : MonoBehaviour` that owns one `NsfSession` instance.

```csharp
namespace NarrativeFramework.Runtime.Host
{
    sealed class NsfSession : IDisposable
    {
        public INarrativeServiceRegistry Registry { get; }
        public ISimulationKernel Kernel { get; }
        public ContentPackHandle Content { get; }

        public static NsfSession Create(NsfSessionConfig config);
        public void Tick(SimulationTickPhase? phase = null);
        public void Dispose();
    }

    class NsfSessionConfig
    {
        public ContentPackManifest Manifest;
        public NsfSessionMode Mode;  // HeadlessTest, EditorPlay, PlayerBuild
        public bool RegisterPresentationServices;
        public bool RegisterDebugSink;
        public NsfModuleSet EnabledModules;  // `[FULL]` — default All; subset for games that omit modules
    }

    enum NsfSessionMode { HeadlessTest, EditorPlay, PlayerBuild }
}
```

**Location:** `Packages/NarrativeFramework/Runtime/Host/` — Runtime assembly, **no** `UnityEngine` references in `NsfSession` itself.

### NsfGameHost (sample/game MonoBehaviour)

```csharp
// Lives in sample assembly, NOT in core NSF package
public sealed class NsfGameHost : MonoBehaviour
{
    NsfSession _session;
    NsfTickDriver _tickDriver;

    void Awake()
    {
        _session = NsfSession.Create(BuildConfig());
        _tickDriver = new NsfTickDriver(_session.Kernel, TickPolicy.EventDriven);
        WirePresentation(_session.Registry); // Canvas presenters, exploration bridge
    }

    void Update() => _tickDriver.Update(Time.deltaTime);
    void OnDestroy() => _session.Dispose();
}
```

Games extend bootstrap after `NsfSession.Create`: register game-owned services on `_session.Registry` before the first tick. See [game-extensions.md](game-extensions.md).

---

## Tick scheduling

**Why event-driven default:** Disco-like games advance simulation on **player actions** and **time skips**, not every Unity frame. Idle `Update()` ticking wastes work and risks pacing bugs.

### TickPolicy

| Policy | When `Kernel.Tick()` runs | Used by |
|---|---|---|
| `Manual` | Caller invokes explicitly | Edit Mode tests |
| `EventDriven` | After queued player actions + optional batch flush | Player build (default) |
| `FixedInterval` | Every N seconds of game time | `[FULL]` ambient sim |
| `EveryFrame` | Each `Update()` | **Discouraged** — debug only |

```csharp
class NsfTickDriver
{
    public NsfTickDriver(ISimulationKernel kernel, TickPolicy policy);
    public void NotifyPlayerAction();  // queues full tick
    public void Update(float deltaTime); // handles FixedInterval only
}
```

**Integration with services:** `IDialogueService.TrySelectChoice`, `IInteractionService.ExecuteInteraction`, and `ITimeService.Advance` call `NotifyPlayerAction()` via a small `INarrativeCommandSink` registered at bootstrap — avoids Presentation calling kernel directly.

---

## Bootstrap variants

| Variant | Entry | Presentation | Debug |
|---|---|---|---|
| Phase 0 smoke | `NsfBootstrap.CreatePhase0Registry()` | None | None |
| Edit Mode unit test | `NsfSession.Create(HeadlessTest)` | Optional headless presenters | Optional `IDebugTraceSink` |
| Integration test | `NsfIntegrationBootstrap.CreateFullLoopRegistry()` | Headless bundle | Trace sink |
| Editor play | `NsfGameHost` + sample scene | Sample Canvas presenters | Debug window |
| Player build | `NsfGameHost` | Sample presenters | Stripped |

**SSOT for full loop wiring:** [integration.md](integration.md). **SSOT for host lifecycle:** this document.

### Content pack mounting

**Locked:** [decisions-log.md](../decisions-log.md) P-03 — **Addressables from Phase 2** for all packs (including FrameworkTestPack). Label pattern `nsf_content_{packId}`.

| Phase | Loading strategy |
|---|---|
| 2+ | Addressables groups per content pack; manifest lists addressable keys |
| Edit Mode tests | May use synchronous test helpers that load labeled groups without full player init |

```csharp
class ContentPackHandle
{
    public IContentStore Store { get; }
    public static ContentPackHandle Load(ContentPackManifest manifest);  // resolves Addressables
}
```

Pipeline **hard-fails** invalid manifests at import (decisions log CP-01); runtime loads validated definitions only.

---

## Unity engine bridge map

Presentation owns all Unity API touchpoints. Core never imports `UnityEngine`.

| Unity subsystem | Presentation service | Bridge responsibility |
|---|---|---|
| SceneManagement | `IExplorationService` | Map `location_*` → scene path from `SceneDefinition`; load/unload |
| Input System / legacy input | `IInteractionService` | Raycast/pointer → `worldObjectId` |
| uGUI / UITK | `IUIShell` + presenters | Noir sample: uGUI; SciFi sample: UITK (decisions log P-11) |
| AudioSource / clips / TTS | `IAudioNarrativeService` + `ITextToSpeechProvider` | Play voice/ambient from content IDs; TTS default in package; samples wire spatial anchors |
| (none in core) | — | Physics, animation, VFX stay in sample scripts |

See [sim-location.md](sim-location.md): `SceneDefinition.UnityScenePath` is **Presentation-only metadata** — `ILocationService` never reads it.

---

## Build profiles

| Profile | Assemblies | Scripting defines |
|---|---|---|
| Player | Runtime modules + Presentation (sample) | `NSF_PLAYER` |
| Editor | + Editor + Debug | `UNITY_EDITOR`, `NSF_DEBUG` |
| Tests | + Tests | `NSF_TEST` |

**Why strip Debug from player:** Inspectors and trace sinks must not ship in production builds. Debug asmdef references `UnityEditor`; excluded from player builds via platform constraints.

Player build must not register `IDebugTraceSink` unless `DEVELOPMENT_BUILD` and explicit opt-in.

---

## Out of scope — deliberately not discussed yet

These are **game or post-v1 NSF package concerns**, not reasons games cannot add their own systems. Game-owned mechanics use [game-extensions.md](game-extensions.md). Documented here to avoid silent assumptions.

| Topic | Status | Notes |
|---|---|---|
| **2D vs 3D** | **Sample assignment** | NoirSample 3D + SciFiSample 2D — see [samples.md](samples.md) |
| Multiplayer / sync | **Out of scope v1** | Locked — decisions log P-01 |
| Modding API | **Out of scope v1** | Locked — content packs only (P-02) |
| Addressables | **Phase 2 required** | Locked — P-03 |
| Cloud saves | Game | NSF defines snapshot bytes only; autosave model P-09 |
| Analytics / telemetry | Game | Optional hooks in sample |
| Accessibility (NSF hooks) | **Locked** NSF-H01 — presenter scale + high-contrast tokens; sample playtest optional Phase 18 |
| Lip-sync / cinematic timeline | Game | `IVoiceService` delivers text keys |
| Procedural narrative generation | Out of scope | AI assists content pipeline, not runtime gen |
| Platform export (Steam, consoles) | Game repo | NSF is package-only |
| Localization workflow | Partial | `ILocaleService`; Noir **fr** + en; missing key → **English fallback** (S-03, S-04) |
| Performance SLAs | Phase 17 | Tick phase timing budgets |
| Visual novel mode vs walkable 3D | Game presentation | Same NSF session |
| Combat / real-time action | **Game-owned** (not NSF v1) | Peer module + adapter into facts/flags/events; promote to NSF module if reusable — [game-extensions.md](game-extensions.md) |
| Legal / ratings / store pages | Phase 18 | Human gate |

When a topic moves from game-owned to NSF core, add a glossary entry + `systems/` spec + architecture doc — do not bolt onto core modules silently. Game-owned features use [game-extensions.md](game-extensions.md).

---

## MVP scope (Phase 0–1)

- [ ] `NsfSession` skeleton in Runtime.Host (no UnityEngine)
- [ ] `NsfBootstrap` delegates to session factory by phase
- [ ] `NsfTickDriver` with `Manual` + `EventDriven`
- [ ] Document tick policy in integration tests
- [ ] `ContentPackHandle.Load` stub (empty store Phase 0)

Phase 13+:

- [ ] Sample `NsfGameHost` in NoirSample
- [ ] Presentation bridges wired per [present-*.md](present-ui.md)

---

## File tree

```text
Packages/NarrativeFramework/Runtime/Host/
  NsfSession.cs
  NsfSessionConfig.cs
  NsfSessionMode.cs
  NsfTickDriver.cs
  TickPolicy.cs
  INarrativeCommandSink.cs
  ContentPackHandle.cs
Samples~/NoirSample/Scripts/
  NsfGameHost.cs              (sample-owned MonoBehaviour)
  NoirPresentationWiring.cs   (3D — sample choice; SciFi uses 2D)
```

---

## Test plan

| Test | Asserts |
|---|---|
| `NsfSession_CreateHeadlessTest_DoesNotRequireUnityEngine` | No scene load |
| `TickDriver_EventDriven_TicksOncePerAction` | Kernel tick count |
| `NsfSession_Dispose_ClearsRegistry` | No leaked singletons |

---

## Related documents

- [foundation.md](foundation.md) — asmdefs and registry
- [integration.md](integration.md) — full loop bootstrap
- [present-exploration.md](present-exploration.md) — scene bridge
- [samples.md](samples.md) — per-sample visual choices

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) P-03–P-04, P-08, P-11–P-12.
