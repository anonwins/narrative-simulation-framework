# NSF API Reference (Rough Draft)

**Status:** Rough reference — written as if NSF v1.0 is shipped. Signatures and types mirror [architecture/contracts.md](architecture/contracts.md) and [terminology-glossary.md](terminology-glossary.md). When this doc conflicts with those SSOT files, **the SSOT wins**.

**Package:** `com.narrativesimulation.framework` · **Unity:** 6000.4.10f1 · **License:** MIT

NSF is a composable **narrative simulation library** for Unity. This document covers the public surface of `Packages/NarrativeFramework/`. Games add their own modules via [architecture/game-extensions.md](architecture/game-extensions.md).

---

## Table of contents

1. [Overview](#overview)
2. [Package layout](#package-layout)
3. [Bootstrap & host](#bootstrap--host)
4. [Simulation loop](#simulation-loop)
5. [Runtime](#runtime)
6. [Simulation](#simulation)
7. [Cognition](#cognition)
8. [Social](#social)
9. [Story](#story)
10. [Ledger](#ledger)
11. [Rules](#rules)
12. [Content](#content)
13. [Presentation](#presentation)
14. [Domain types](#domain-types)
15. [Events](#events)
16. [Content definitions](#content-definitions)
17. [Persistence](#persistence)
18. [Content IDs](#content-ids)
19. [Errors & conventions](#errors--conventions)

---

## Overview

### What NSF provides

| Layer | Responsibility |
|---|---|
| **Runtime** | Service registry, simulation kernel, tick scheduling |
| **Simulation** | Facts, actors, time, locations, events, economy, saves |
| **Cognition** | Faculties, rolls, beliefs, conduct, emotion |
| **Social** | Relationships, factions, companions, ideology |
| **Story** | Dialogue, voice, story state, pacing, outcomes |
| **Ledger** | Chronicle (player journal projection), thread (inquiry logic) |
| **Rules** | Rule engine, gates |
| **Content** | Content store, pipeline, script compiler, locale, inventory |
| **Presentation** | UI shell, exploration, interaction, discovery, audio bridges |

### Contract inventory

| Kind | Count | Examples |
|---|---|---|
| `I*Service` | 32 | `IFactService`, `IDialogueService`, … |
| Kernel / registry | 2 | `ISimulationKernel`, `INarrativeServiceRegistry` |
| Event bus | 1 | `IEventBus` |
| Orchestrators | 3 | `RuleEngine`, `ThreadEngine`, `ScriptCompiler` |
| Host | 2 | `NsfSession`, `NsfGameHost` (sample) |
| Patterns | 1 | Fail-forward (no interface) |

### Naming rules

| Suffix | Meaning |
|---|---|
| `I*Service` | Public mutation/query API — registered in `INarrativeServiceRegistry` |
| `*Definition` | Authored content asset (ScriptableObject / JSON) |
| `*State` | Serializable runtime snapshot |
| `*Registry` | Internal ID lookup — **not** publicly registered |
| `*Engine` | Orchestrator (`RuleEngine`, `ThreadEngine`) |

---

## Package layout

```text
Packages/NarrativeFramework/
  Runtime/           NarrativeFramework.Runtime
    Host/            NsfSession, NsfTickDriver, NsfBootstrap
    Kernel/          SimulationKernel
    Registry/        NarrativeServiceRegistry
  Simulation/        NarrativeFramework.Simulation
  Cognition/         NarrativeFramework.Cognition
  Social/            NarrativeFramework.Social
  Story/             NarrativeFramework.Story
  Ledger/            NarrativeFramework.Ledger
  Rules/             NarrativeFramework.Rules
  Content/           NarrativeFramework.Content
  Presentation/      NarrativeFramework.Presentation
  Debug/             NarrativeFramework.Debug (Editor)
  Editor/            NarrativeFramework.Editor
  Tests/             NarrativeFramework.Tests (Edit Mode)
```

**Asmdef rule:** Cognition, Simulation, Story, Ledger, Rules, Social, Content **must not** reference Presentation.

---

## Bootstrap & host

### NsfSession

Pure C# composition root. No `UnityEngine` dependency.

```csharp
namespace NarrativeFramework.Runtime.Host
{
    sealed class NsfSession : IDisposable
    {
        public INarrativeServiceRegistry Registry { get; }
        public ISimulationKernel Kernel { get; }
        public ContentPackHandle Content { get; }

        public static NsfSession Create(NsfSessionConfig config);
        public void Tick(SimulationTickPhase? singlePhase = null);
        public void Dispose();
    }
}
```

### NsfSessionConfig

```csharp
class NsfSessionConfig
{
    public ContentPackManifest Manifest;
    public NsfSessionMode Mode;              // HeadlessTest | EditorPlay | PlayerBuild
    public bool RegisterPresentationServices;
    public bool RegisterDebugSink;
    public NsfModuleSet EnabledModules;      // default: All
    public bool EnableFileLog;               // optional rolling file log
}

enum NsfSessionMode { HeadlessTest, EditorPlay, PlayerBuild }
```

### NsfTickDriver

```csharp
enum TickPolicy { Manual, EventDriven, FixedInterval, EveryFrame }

class NsfTickDriver
{
    public NsfTickDriver(ISimulationKernel kernel, TickPolicy policy);
    public void NotifyPlayerAction();   // queues full kernel tick
    public void Update(float deltaTime);
}
```

**Default:** `EventDriven` — tick runs after player actions (dialogue choice, interaction, time advance), not every frame.

### Typical bootstrap (game)

```csharp
var config = new NsfSessionConfig {
    Manifest = ContentPackManifest.Load("MyGame"),
    Mode = NsfSessionMode.PlayerBuild,
    RegisterPresentationServices = true
};
var session = NsfSession.Create(config);

// Optional: game-owned modules
session.Registry.Register<ICombatService>(new CombatService(session.Registry));
session.Registry.Register<IStatefulService>(session.Registry.GetRequired<ICombatService>());

session.Kernel.Initialize(session.Registry);
```

### NsfGameHost (sample / game MonoBehaviour)

Lives in **game assembly**, not in NSF package. Owns one `NsfSession`, wires `NsfTickDriver` and presentation bridges.

---

## Simulation loop

### Phases

```csharp
enum SimulationTickPhase
{
    Facts,          // IFactService
    Interpretation, // IDialogueService, IVoiceService, cognition
    Social,         // IRelationshipService, IFactionService, IIdeologyService
    Events,         // IEventBus flush, IActorService, ITimeService, ILocationService
    Gating,         // IGateService, IRuleEngine, IPacingService
    Content         // script effects, new facts, content unlock
}
```

### Causal chain

```text
FACTS → INTERPRETATION → SOCIAL → EVENTS → GATING → CONTENT → (new facts) ↺
```

### ISimulationKernel

```csharp
interface ISimulationKernel
{
    SimulationTickPhase CurrentPhase { get; }
    void Initialize(INarrativeServiceRegistry registry);
    void Tick(SimulationTickPhase? singlePhase = null);
}
```

- `Tick()` — runs all six phases in order.
- `Tick(SimulationTickPhase.Facts)` — runs a single phase (Edit Mode tests).

---

## Runtime

### INarrativeServiceRegistry

```csharp
interface INarrativeServiceRegistry
{
    void Register<T>(T service) where T : class;
    T GetRequired<T>() where T : class;
    bool TryGet<T>(out T service) where T : class;
}
```

Use `TryGet` when a module is optional (subset bootstrap). Game-owned services register the same way as NSF services.

### IStatefulService

Services with save-relevant state implement:

```csharp
interface IStatefulService<TState> where TState : class, new()
{
    TState CaptureState();
    void RestoreState(TState state);
}

// Non-generic adapter for persistence discovery:
interface IStatefulService
{
    string StateKey { get; }
    object CaptureState();
    void RestoreState(object state);
}
```

`IPersistenceService` discovers all `IStatefulService` registrations at capture time.

---

## Simulation

### IEventBus

```csharp
interface IEventBus
{
    void Publish(SimEventBase e);
    void PublishImmediate(SimEventBase e, SimulationSnapshot snapshot);
    void Subscribe<T>(IEventHandler<T> handler) where T : SimEventBase;
    void Unsubscribe<T>(IEventHandler<T> handler) where T : SimEventBase;
    void Flush(SimulationSnapshot snapshot);
}

interface IEventHandler<in T> where T : SimEventBase
{
    void Handle(T e, SimulationSnapshot state);
}
```

- `Publish` — deferred; flushed during **Events** phase.
- `PublishImmediate` — sync within current handler (use sparingly).
- Re-entrancy: throws if `Publish` during `Flush`.

### IFactService

Atomic simulation truth.

```csharp
interface IFactService
{
    void Register(FactRecord fact);
    bool TryGet(string factId, out FactRecord fact);
    void Revoke(string factId);
    bool IsActive(string factId);
    IReadOnlyList<FactRecord> GetAll();
}
```

**Fact vs story flag:** Facts are world truth. Story flags are narrative progression switches (`IStoryStateService`).

### IActorService

```csharp
interface IActorService
{
    ActorState GetState(string actorId);
    void SetState(string actorId, ActorState state);
    bool KnowsFact(string actorId, string factId);
    void RememberEvent(string actorId, SimEventBase e);
    string GetLocation(string actorId);
}
```

### ITimeService

Discrete narrative clock — not real-time.

```csharp
interface ITimeService
{
    GameTime Now { get; }
    void Advance(GameTimeDelta delta);
    void Schedule(string eventId, GameTime at);
    bool IsScheduledDue(string eventId);
}
```

### ILocationService

```csharp
interface ILocationService
{
    bool CanTravel(string fromLocationId, string toLocationId);
    void Travel(string actorId, string toLocationId);
    IReadOnlyList<string> GetConnectedLocations(string locationId);
    string GetActorLocation(string actorId);
}
```

One hop per `Travel` call. Graph edges bidirectional by default.

### IInfoFlowService

Who learned what, when, from whom.

```csharp
interface IInfoFlowService
{
    void RecordPropagation(string factId, string fromActorId, string toActorId, GameTime at);
    bool DidActorLearn(string actorId, string factId);
    IReadOnlyList<InfoPropagation> GetHistory(string factId);
}
```

### IEconomyService

```csharp
interface IEconomyService
{
    int GetCurrency(string currencyId);
    void AddCurrency(string currencyId, int amount);
    bool TrySpend(string currencyId, int amount);
    bool TryPurchase(string vendorId, string itemId);
}
```

Default currency ID: `currency_cash`. Additional currencies via content-defined IDs.

### IPersistenceService

```csharp
interface IPersistenceService
{
    int CurrentSchemaVersion { get; }
    SaveSnapshot Capture();
    void Restore(SaveSnapshot snapshot);
}
```

- Schema version: `1` (v1.0).
- Payload: canonical JSON (no whitespace), SHA256 hash in `SaveSnapshot.StateHash`.
- Autosave model: autosave + continue (decisions P-09).

---

## Cognition

### IFacultyService

```csharp
interface IFacultyService
{
    int GetValue(string facultyId);
    int GetModifiedValue(string facultyId);
    void ApplyModifier(string facultyId, int delta, ModifierSource source);
    Vitality GetVitality();
    Morale GetMorale();
    IReadOnlyList<string> GetFacultyIdsInGroup(string groupId);
}
```

Passive checks use **SUM** of modified values for faculties in a group (FAC-01).

### IRollService

2d6 + modifier vs difficulty.

```csharp
interface IRollService
{
    RollResult Resolve(FacultyRoll roll);
    bool CanAttempt(RollMode mode, string rollId, RollContext context);
    void RecoverRepeatable(string rollId);
}
```

**Roll modes:**

| Mode | Behavior |
|---|---|
| `Active` | Standard check; no lockout |
| `Passive` | Automatic notice; no player roll UI |
| `Repeatable` | Can retry until success; locks on failure until recovered |
| `Gated` | Blocked until gate/fact conditions met |

**Modifier stack order:** base faculty → equipment → emotion → belief → conduct → script → dialogue context.

### IBeliefService

```csharp
interface IBeliefService
{
    BeliefPhase GetPhase(string beliefId);
    void Discover(string beliefId);
    void BeginAssimilating(string beliefId);
    void Resolve(string beliefId);
    void Forget(string beliefId);
    IReadOnlyList<string> GetActiveBeliefIds();
}

enum BeliefPhase { Undiscovered, Discovered, Assimilating, Resolved, Forgotten }
```

No hard slot cap on active beliefs (C-05).

### IConductService

```csharp
interface IConductService
{
    int GetScore(string conductId);
    void RecordAction(string conductId, int delta);
    string GetDominantConductProfile();
    IReadOnlyDictionary<string, int> GetAllScores();
}
```

Conduct IDs: `conduct_*` (content-defined). Distinct from public **Standing** (faction reputation).

### IEmotionService

```csharp
interface IEmotionService
{
    EmotionState GetState(string emotionId);
    void ApplyModifier(string emotionId, float delta);
    float GetRollModifier(string facultyId);
    void TickEmotionDecay();
}
```

Emotion IDs: `emotion_*`. Affects rolls via `GetRollModifier`.

---

## Social

### IRelationshipService

```csharp
interface IRelationshipService
{
    RelationshipState GetRelationship(string actorId);
    void ModifyMetric(string actorId, string metricId, float delta);
    float GetMetric(string actorId, string metricId);
}
```

Metric IDs: `metric_*`. Float API; content may display as labels or integers.

### IFactionService

```csharp
interface IFactionService
{
    int GetStanding(string factionId);
    void ModifyStanding(string factionId, int delta);
    FactionStance GetStance(string factionId);
    IReadOnlyList<string> GetMemberActorIds(string factionId);
}
```

Standing decays time-based when `ITimeService` is active.

### ICompanionService

```csharp
interface ICompanionService
{
    string GetCompanionActorId();
    float GetTrustMetric(string metricId);
    void ModifyTrust(string metricId, float delta);
    bool IsAvailableForDialogue();
}
```

One active companion (C-02). Side-channel dialogue allowed during primary conversation (ST-02).

### IIdeologyService

```csharp
interface IIdeologyService
{
    float GetAxisValue(string axisId);
    void ShiftAxis(string axisId, float delta);
    string GetDominantLabel(string axisId);
}
```

Axis values clamp at minimum. Time-based decay supported.

---

## Story

### IDialogueService

Directed graph — not a tree.

```csharp
interface IDialogueService
{
    void StartConversation(string dialogueGraphId, string speakerActorId);
    DialogueNode GetCurrentNode();
    bool TrySelectChoice(string choiceId, out DialogueAdvanceResult result);
    void EndConversation();
}
```

- Choice visibility evaluated via `IRuleEngine` during **Gating**.
- Choice effects executed via `IRuleEngine.ExecuteActions` during **Content**.
- Revisit policy per node: `DialogueVisitPolicy` (ST-01).

### IStoryStateService

```csharp
interface IStoryStateService
{
    bool GetFlag(string flagId);
    void SetFlag(string flagId, bool value);
    T GetVariable<T>(string key);
    void SetVariable<T>(string key, T value);
    void AdvanceBeat(string beatId);
    StoryBeatState GetBeatState(string beatId);
}
```

Flag IDs: `flag_*`. Beat IDs: `beat_*` or content-store IDs.

### IVoiceService

```csharp
interface IVoiceService
{
    void Speak(VoiceChannel channel, string textKey, string actorOrFacultyId);
    void Stop(VoiceChannel channel);
}

enum VoiceChannel { Narrator, Player, NPC, Faculty, Environmental }
```

Text keys resolve through `ILocaleService`. Audio playback via `IAudioNarrativeService` / `ITextToSpeechProvider`.

### IPacingService

```csharp
interface IPacingService
{
    bool CanFireContent(string contentId);
    void RegisterContentWindow(string contentId, PacingWindow window);
    void TickPacing();
}
```

Default policy: `Mixed` (ST-04).

### IOutcomeService

```csharp
interface IOutcomeService
{
    OutcomeSynthesis Synthesize(OutcomeContext context);
    string GetEndingId(OutcomeContext context);
}
```

Evaluates conduct, ideology, thread resolutions, beat completion for ending synthesis.

### Fail-forward (pattern)

No `IFailForwardService`. Coordinates:

- `IRollService` — failure branch IDs
- `IDialogueService` — failure dialogue nodes
- `IStoryStateService` — advance on failure paths
- `IPacingService` — allow retry windows

**Principle:** failure produces content; no hard game over.

---

## Ledger

### IChronicleService

**Read-only projection** — never mutates thread or story state.

```csharp
interface IChronicleService
{
    IReadOnlyList<ChronicleSection> GetSections();
    IReadOnlyList<ChronicleEntry> GetEntries(string sectionId);
    void RefreshProjection();
}
```

Entry types: `Lead`, `Task`, `Clue` (`ChronicleEntryType`). Section IDs: `section_*`.

Rebuilds from authoritative state on load (SIM-03).

### IThreadService

Authoritative inquiry logic.

```csharp
interface IThreadService
{
    void AddEvidence(string threadId, ThreadEvidence evidence);
    void AddTheory(string threadId, ThreadTheory theory);
    bool TryResolveSubject(string threadId, string subjectId, out ThreadResolution resolution);
    IReadOnlyList<ThreadEvidence> GetEvidence(string threadId);  // query helper
}
```

### ThreadEngine

```csharp
class ThreadEngine
{
    ThreadState Evaluate(string threadId, IReadOnlyFactView facts);
}
```

Concrete orchestrator — not registered as `I*Service`. Used internally by `IThreadService`.

---

## Rules

### IRuleEngine

```csharp
interface IRuleEngine
{
    bool Evaluate(Rule rule, RuleContext context);
    void ExecuteActions(IReadOnlyList<Action> actions, RuleContext context);
}

class RuleEngine : IRuleEngine { /* injects INarrativeServiceRegistry */ }
```

### Conditions (built-in)

| `ConditionKind` | Evaluates |
|---|---|
| `All` | All children true |
| `Any` | Any child true |
| `Not` | Negates child |
| `FactActive` | `IFactService.IsActive(id)` |
| `StoryFlag` | `IStoryStateService.GetFlag(id)` |
| `BeatStage` | Beat stage comparison |
| `ConductScore` | `IConductService.GetScore(id)` |
| `ThreadResolved` | Thread subject resolution |

Custom condition plugins: register via game/content assemblies (ST-05).

### Actions (built-in)

| `ActionKind` | Effect |
|---|---|
| `SetStoryFlag` | `IStoryStateService.SetFlag` |
| `AdvanceBeat` | `IStoryStateService.AdvanceBeat` |
| `RegisterFact` | `IFactService.Register` |
| `AddThreadEvidence` | `IThreadService.AddEvidence` |
| `ModifyRelationship` | `[FULL]` |
| `ModifyCurrency` | `[FULL]` |
| `ApplyEmotion` | `[FULL]` |

### IGateService

```csharp
interface IGateService
{
    bool IsAllowed(string gateId, GateContext context);
    GateDenialReason GetDenialReason(string gateId, GateContext context);
}
```

Gate denial includes optional `hint` field for player-facing copy (ST-06).

---

## Content

### IContentStore

```csharp
interface IContentStore
{
    T GetDefinition<T>(string id) where T : ContentDefinition;
    bool TryGetDefinition<T>(string id, out T definition) where T : ContentDefinition;
    IReadOnlyList<string> GetAllIds<T>() where T : ContentDefinition;
}
```

- `GetDefinition` throws `ContentNotFoundException` if missing.
- `TryGetDefinition` for optional lookups.
- Internal `ContentRegistry` — not public.

### IContentPipeline

```csharp
interface IContentPipeline
{
    PipelineReport Validate(ContentPackManifest manifest);
    PipelineReport Build(ContentPackManifest manifest);
    bool TryFix(ContentPackManifest manifest, out PipelineReport report);
}
```

- Validation: hard fail (CP-01).
- `TryFix`: never auto-deletes without explicit flag (CP-02).
- Cross-pack IDs: `{packId}/{contentId}` namespace (SCR-02).

### ScriptCompiler

```csharp
class ScriptCompiler
{
    ScriptCompileResult Compile(string source, string filePath);
    ScriptCompileResult CompileFile(string nsfFilePath);
}
```

`.nsf` DSL — compile-only, shared AST with rules (SCR-01). Not in service registry.

**MVP grammar keywords:** `NODE`, `SPEAKER`, `TEXT`, `CHOICE`, `IF`, `ROLL`, `SET_FLAG`, `REGISTER_FACT`, `END`.

### ILocaleService

```csharp
interface ILocaleService
{
    string Resolve(string key, string localeId, IReadOnlyDictionary<string, string> substitutions);
    string GetFallbackLocale();
    bool HasKey(string key, string localeId);
}
```

Missing key → English fallback (S-04).

### IInventoryService

Narrative items — not combat stat sticks.

```csharp
interface IInventoryService
{
    bool HasItem(string itemId);
    void AddItem(string itemId, int quantity);
    void RemoveItem(string itemId, int quantity);
    void Equip(string itemId, string slotId);
    void Unequip(string slotId);
    IReadOnlyList<ItemModifier> GetEquippedModifiers();
}
```

Equipment modifiers feed `IFacultyService` / `IRollService` via `GetEquippedModifiers()`.

---

## Presentation

Headless-testable bridges. Core modules never reference Canvas/UITK types.

### IUIShell

```csharp
interface IUIShell
{
    void BindDialoguePresenter(IDialoguePresenter presenter);
    void BindChroniclePresenter(IChroniclePresenter presenter);
    void BindBeliefView(IBeliefView beliefView);
    void PushScreen(string screenId);
    void PopScreen();
    float TextScaleMultiplier { get; set; }  // 0.875–1.5, default 1.0 (NSF-H01)
}

interface IDialoguePresenter { void Show(DialogueViewModel model); }
interface IChroniclePresenter { void Show(ChronicleView model); }
interface IBeliefView { void Show(BeliefViewModel model); }
```

### IInteractionService

```csharp
interface IInteractionService
{
    IReadOnlyList<InteractionOption> GetAvailableInteractions(string worldObjectId);
    void ExecuteInteraction(string worldObjectId, string interactionId);
}
```

World object IDs: content-defined (`object_*`). Effects may register facts, set flags, start dialogue, trigger rolls.

### IDiscoveryService

```csharp
interface IDiscoveryService
{
    bool IsDiscovered(string discoverableId);
    void MarkDiscovered(string discoverableId, DiscoverySource source);
    IReadOnlyList<string> GetDiscoveredIds();
}

enum DiscoveryState { Hidden, Hinted, Visible, Discovered }
```

### IExplorationService

```csharp
interface IExplorationService
{
    bool CanEnter(string locationId);
    void Enter(string locationId);
    void Exit(string locationId);
    string GetCurrentLocationId();
}
```

`Enter` calls `ILocationService.Travel` then loads Unity scene (game bridge).

### IAudioNarrativeService

```csharp
interface IAudioNarrativeService
{
    void PlayVoiceLine(string voiceLineId, string localeId);
    void PlayAmbient(string ambientId, AudioAnchor anchor = null);
    void StopChannel(string channelId);
}
```

Voice interrupt policy: finish current, then play next (queue) (A-04).

### ITextToSpeechProvider

```csharp
interface ITextToSpeechProvider
{
    void Speak(string text, VoiceProfileDefinition profile, AudioPlayRequest request);
    void Stop(string channelId);
    bool IsSpeaking(string channelId);
}
```

Bundled profiles: `voice_neutral_en_a`, `voice_neutral_en_b` (A-07).

---

## Domain types

### Shared primitives

```csharp
struct GameTime { public int Day; public int Hour; public int Minute; }
struct GameTimeDelta { public int Days; public int Hours; public int Minutes; }
enum ModifierSource { Equipment, Emotion, Conduct, Script, Debug }
```

### Facts

```csharp
class FactRecord
{
    public string Id;           // fact_*
    public string SubjectId;    // actor_* or location_*
    public string Predicate;    // content verb key
    public string Value;
    public GameTime RegisteredAt;
    public bool IsRevoked;
}
```

### Events

```csharp
abstract class SimEventBase
{
    public string Id;
    public GameTime Timestamp;
    public int Priority;        // lower = earlier (EVT-05)
    public abstract string EventType { get; }
}
```

### Rolls

```csharp
class FacultyRoll
{
    public string RollId;
    public string FacultyId;
    public int Difficulty;
    public RollMode Mode;
    public string SuccessBranchId;
    public string FailureBranchId;
}

class RollResult
{
    public bool Success;
    public int RolledValue;     // 2d6 sum
    public int TargetValue;     // difficulty
    public string OutcomeBranchId;
}

enum RollMode { Active, Passive, Repeatable, Gated }
```

### Dialogue

```csharp
class DialogueNode
{
    public string Id;
    public string SpeakerActorId;
    public string TextKey;
    public IReadOnlyList<DialogueChoice> Choices;
    public Rule VisibilityRule;
}

class DialogueChoice
{
    public string Id;
    public string LabelKey;
    public string TargetNodeId;
    public Rule VisibilityRule;
    public IReadOnlyList<Action> Actions;
    public bool EndsConversation;
}

class DialogueAdvanceResult
{
    public string NewNodeId;
    public bool Ended;
}
```

### Story beats

```csharp
class StoryBeatState
{
    public string BeatId;
    public BeatStage Stage;     // NotStarted | InProgress | Complete
}

enum BeatStage { NotStarted, InProgress, Complete }
```

### Thread

```csharp
class ThreadEvidence
{
    public string Id;
    public string DescriptionKey;
    public string SourceFactId;
}

class ThreadTheory
{
    public string Id;
    public string HypothesisKey;
    public IReadOnlyList<string> SupportingEvidenceIds;
}

class ThreadResolution
{
    public ThreadResolutionKind Kind;  // Confirmed | Refuted | Inconclusive
    public string SubjectId;
}

enum ThreadResolutionKind { Confirmed, Refuted, Inconclusive }
```

### Chronicle

```csharp
class ChronicleSection
{
    public string Id;           // section_*
    public string TitleKey;
    public int SortOrder;
}

class ChronicleEntry
{
    public string Id;
    public ChronicleEntryType Type;  // Lead | Task | Clue
    public string TitleKey;
    public string BodyKey;
    public bool IsComplete;
}

enum ChronicleEntryType { Lead, Task, Clue }
```

### Rules

```csharp
class Rule
{
    public Condition RootCondition;
}

class Condition
{
    public ConditionKind Kind;
    public string TargetId;
    public IReadOnlyList<Condition> Children;
    public int IntOperand;      // for comparisons
    public bool BoolOperand;
}

enum ConditionKind { All, Any, Not, FactActive, StoryFlag, BeatStage, ConductScore, ThreadResolved }

class Action
{
    public ActionKind Kind;
    public string TargetId;
    public bool BoolValue;
    public FactRecord FactPayload;
    public string ThreadId;
    public ThreadEvidence Evidence;
}

enum ActionKind { SetStoryFlag, AdvanceBeat, RegisterFact, AddThreadEvidence }
```

### Cognition pools

```csharp
struct Vitality { public int Current; public int Max; }
struct Morale { public int Current; public int Max; }

class EmotionState
{
    public string EmotionId;
    public float Value;         // typically 0–1
}
```

### Social

```csharp
class RelationshipState
{
    public string ActorId;
    public Dictionary<string, float> Metrics;
}

enum FactionStance { Hostile, Wary, Neutral, Friendly, Allied }
```

### Presentation view models

```csharp
class DialogueViewModel
{
    public string SpeakerName;
    public string Text;
    public IReadOnlyList<DialogueChoiceViewModel> Choices;
}

class ChronicleView
{
    public IReadOnlyList<ChronicleSectionViewModel> Sections;
}

class BeliefViewModel
{
    public IReadOnlyList<BeliefSlotViewModel> Slots;
}

class InteractionOption
{
    public string InteractionId;
    public string Label;
    public bool IsEnabled;
    public string DisabledReasonKey;
}
```

### Simulation snapshot (read-only)

```csharp
class SimulationSnapshot
{
    public IReadOnlyFactView Facts { get; }
    public IReadOnlyStoryStateView Story { get; }
}
```

Built at tick start; immutable for event handler duration.

---

## Events

Typed hierarchy from Phase 1 (EVT-01). Handler order (EVT-05): action → sim → narrative → chronicle → presentation → save flush.

| Event type | Typical publisher | Typical subscribers |
|---|---|---|
| `FactRegistered` | `IFactService` | `IInfoFlowService`, chronicle, debug |
| `FactRevoked` | `IFactService` | Rules, thread |
| `RollResolved` | `IRollService` | Fail-forward, chronicle |
| `DialogueStarted` | `IDialogueService` | Audio, UI |
| `DialogueChoiceSelected` | `IDialogueService` | `IConductService`, `IRelationshipService` |
| `DialogueEnded` | `IDialogueService` | Pacing |
| `StoryFlagChanged` | `IStoryStateService` | Gates, chronicle |
| `StoryBeatAdvanced` | `IStoryStateService` | Voice, pacing |
| `RelationshipChanged` | `IRelationshipService` | Chronicle, dialogue |
| `FactionStandingChanged` | `IFactionService` | Gates, outcomes |
| `ActorMoved` | `ILocationService` | `IActorService`, exploration |
| `LocationEntered` | `ILocationService` | Discovery, companion |
| `TimeAdvanced` | `ITimeService` | Schedules, pacing, conduct decay |
| `ThreadEvidenceAdded` | `IThreadService` | Chronicle projection |
| `ThreadSubjectResolved` | `IThreadService` | Outcomes, chronicle |
| `InteractionExecuted` | `IInteractionService` | Audio, UI refresh |
| `InteractionFailed` | `IInteractionService` | Fail-forward hooks |
| `CurrencyChanged` | `IEconomyService` | UI, chronicle |
| `ItemPurchased` | `IEconomyService` | Conduct, relationship |

Game assemblies may publish custom `SimEventBase` subclasses; subscribe via `IEventBus` (see game-extensions).

---

## Content definitions

All inherit `ContentDefinition`:

```csharp
abstract class ContentDefinition : ScriptableObject
{
    public string Id { get; }
    public string SchemaVersion { get; }
    public virtual void Validate(IValidationContext ctx);
}
```

### Common definition types

| Type | ID prefix | Key fields |
|---|---|---|
| `FacultyDefinition` | `faculty_` | `GroupId`, `StartingValue`, `MinValue`, `MaxValue`, `VoiceProfileId` |
| `FacultyGroupDefinition` | `group_` | Member faculty refs |
| `BeliefDefinition` | `belief_` | Phases, modifiers, text keys |
| `ActorDefinition` | `actor_` | Faction, schedule, dialogue refs |
| `FactionDefinition` | `faction_` | Agenda, standing thresholds |
| `LocationDefinition` | `location_` | Connected edges, scene path (presentation) |
| `StoryFlagDefinition` | `flag_` | Default value |
| `StoryBeatDefinition` | `beat_` | Stage transitions, narration key |
| `DialogueGraphDefinition` | `dialogue_*` | Nodes, entry node |
| `GateRuleDefinition` | `gate_` | Rule tree, denial hint |
| `ThreadDefinition` | `thread_` | Subjects, required facts |
| `ItemDefinition` | `item_*` | Slot, faculty modifiers |
| `ChronicleSectionMapping` | `section_*` | Projection rules |
| `WorldObjectDefinition` | `object_*` | Interaction nodes |
| `VoiceLineDefinition` | `voice_*` | Text key, `VoiceProfileId` |
| `VoiceProfileDefinition` | `voice_neutral_*` | TTS parameters |

Definitions are **not saved** — reload from content pack on boot. Mutable state lives in `*State` DTOs.

---

## Persistence

### SaveSnapshot

```csharp
class SaveSnapshot
{
    public int SchemaVersion;
    public string StateHash;    // SHA256 canonical JSON
    public byte[] Payload;      // UTF-8 SaveGameRoot
}

class SaveGameRoot
{
    public List<ServiceStateEnvelope> Services;
    public GameTime SavedAt;
}

class ServiceStateEnvelope
{
    public string ServiceType;  // stable StateKey
    public string JsonState;
}
```

### Stateful services (save participants)

| Service | StateKey envelope |
|---|---|
| `FactService` | `NarrativeFramework.Simulation.Facts.FactService` |
| `ActorService` | `NarrativeFramework.Simulation.Actors.ActorService` |
| `TimeService` | `NarrativeFramework.Simulation.Time.TimeService` |
| `LocationService` | `NarrativeFramework.Simulation.Locations.LocationService` |
| `InfoFlowService` | `NarrativeFramework.Simulation.InfoFlow.InfoFlowService` |
| `EconomyService` | `NarrativeFramework.Simulation.Economy.EconomyService` |
| `FacultyService` | `NarrativeFramework.Cognition.Faculty.FacultyService` |
| `RollService` | `NarrativeFramework.Cognition.Rolls.RollService` |
| `BeliefService` | `NarrativeFramework.Cognition.Beliefs.BeliefService` |
| `ConductService` | `NarrativeFramework.Cognition.Conduct.ConductService` |
| `EmotionService` | `NarrativeFramework.Cognition.Emotion.EmotionService` |
| `RelationshipService` | `NarrativeFramework.Social.Relationships.RelationshipService` |
| `FactionService` | `NarrativeFramework.Social.Factions.FactionService` |
| `IdeologyService` | `NarrativeFramework.Social.Ideology.IdeologyService` |
| `DialogueService` | `NarrativeFramework.Story.Dialogue.DialogueService` |
| `StoryStateService` | `NarrativeFramework.Story.State.StoryStateService` |
| `VoiceService` | `NarrativeFramework.Story.Voice.VoiceService` |
| `ThreadService` | `NarrativeFramework.Ledger.Thread.ThreadService` |
| `InventoryService` | `NarrativeFramework.Simulation.Inventory.InventoryService` |

`IChronicleService` — **not** persisted; rebuilt via `RefreshProjection()` on load.

Game-owned `IStatefulService` implementations use their own `StateKey` (e.g. `MyGame.Combat.CombatService`).

### Migration

```csharp
interface ISaveMigration
{
    int FromVersion { get; }
    int ToVersion { get; }
    SaveSnapshot Migrate(SaveSnapshot input);
}
```

---

## Content IDs

Format: `{prefix}_{descriptor}` (snake_case). No setting nouns in IDs.

| Prefix | Examples |
|---|---|
| `faculty_` | `faculty_intuition`, `faculty_logic` |
| `group_` | `group_psyche` |
| `belief_` | `belief_doctrine_a` |
| `conduct_` | `conduct_humble`, `conduct_bold` |
| `emotion_` | `emotion_stress` |
| `actor_` | `actor_companion`, `actor_witness` |
| `faction_` | `faction_guild`, `faction_authority` |
| `metric_` | `metric_trust_companion` |
| `fact_` | `fact_incident_time` |
| `flag_` | `flag_thread_resolved` |
| `beat_` | `beat_intro` |
| `thread_` | `thread_main` |
| `section_` | `section_primary` |
| `location_` | `location_warehouse` |
| `gate_` | `gate_clearance_level_2` |
| `dialogue_` | `dialogue_interrogation_a` |
| `object_` | `object_desk` |
| `item_` | `item_keycard` |

Cross-pack namespace: `{packId}/{contentId}` (SCR-02).

Optional per-pack: `actor_player` in manifest (SIM-02).

---

## Errors & conventions

### Common exceptions

| Type | When |
|---|---|
| `ContentNotFoundException` | `IContentStore.GetDefinition` — ID missing post-pipeline |
| `ServiceNotRegisteredException` | `GetRequired<T>` — service not in registry |
| `InvalidOperationException` | Duplicate fact (dev), event re-entrancy, invalid equip slot |
| `ArgumentException` | Invalid content ID prefix at runtime |

### Dev vs release behavior

| Situation | Dev | Release |
|---|---|---|
| Duplicate fact register | Throw | Upsert + warning |
| Missing faculty in passive check | Throw | Log + treat as 0 (FAC-01) |
| Missing locale key | `[key]` placeholder | Fallback English (S-04) |

### Threading

Main thread only (P-01, EVT-04). No background simulation queue in v1.

### Out of NSF package (v1)

| Topic | Where it lives |
|---|---|
| Multiplayer | Not supported |
| Modding API | Content packs only (P-02) |
| Combat / genre loops | Game-owned modules — [game-extensions.md](architecture/game-extensions.md) |
| 2D vs 3D presentation | Game + sample bridges |
| Store / cloud UI | Game repo |

---

## Related documents

| Doc | Role |
|---|---|
| [terminology-glossary.md](terminology-glossary.md) | SSOT — terms, contract names |
| [architecture/contracts.md](architecture/contracts.md) | SSOT — C# signatures |
| [architecture/data-model.md](architecture/data-model.md) | Shared types, persistence |
| [architecture/game-extensions.md](architecture/game-extensions.md) | Game-owned modules |
| [architecture/unity-host.md](architecture/unity-host.md) | Unity lifecycle, tick driver |
| [runtime-kernel.md](runtime-kernel.md) | Simulation loop spec |
| [systems/](systems/) | Per-module behavior specs |

---

*This is a rough API reference for planning and implementation. For behavior semantics, integration patterns, and test plans, see the linked architecture and system specs.*
