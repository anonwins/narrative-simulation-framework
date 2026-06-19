# NSF Contracts Catalog

**Architecture SSOT for C# public APIs.** Spec [Service contract](../systems/) sections mirror this file. Glossary [§ Contracts catalog](../terminology-glossary.md) is the authoritative name index.

---

## Why a single contracts document

Without one catalog, 37 specs drift into incompatible interface shapes. Agents implementing Phase 0 stubs need **one** place to copy signatures. Architecture system docs link here instead of redefining methods.

**Rules:**

- `I*Service` = public mutation/query surface registered in `INarrativeServiceRegistry`
- `*Registry` = internal module lookup — not registered globally
- `*Engine` / `ScriptCompiler` = concrete orchestrators; may implement an interface or be resolved explicitly
- Core modules never reference Presentation types

---

## Runtime

### SimulationTickPhase

**Why:** The kernel must run modules in causal order. Six phases collapse nine conceptual loop layers into testable, ordered execution (see [runtime-kernel.md](runtime-kernel.md)).

```csharp
namespace NarrativeFramework.Runtime
{
    enum SimulationTickPhase
    {
        Facts,
        Interpretation,
        Social,
        Events,
        Gating,
        Content
    }
}
```

### ISimulationKernel

**Why:** NSF is a simulation runtime, not a bag of services. The kernel enforces loop order and gives tests a single entry point for tick integration.

```csharp
interface ISimulationKernel
{
    SimulationTickPhase CurrentPhase { get; }
    void Initialize(INarrativeServiceRegistry registry);
    void Tick(SimulationTickPhase? singlePhase = null);
}
```

### INarrativeServiceRegistry

**Why:** Unity games need explicit composition. A typed registry avoids string-based service locators and keeps Edit Mode tests deterministic (register mocks per test).

```csharp
interface INarrativeServiceRegistry
{
    void Register<T>(T service) where T : class;
    T GetRequired<T>() where T : class;
    bool TryGet<T>(out T service) where T : class;
}
```

---

## Simulation

### IEventBus

**Why:** Modules must not call each other directly across asmdef boundaries. Events decouple producers (dialogue, sim) from consumers (chronicle projection, debug trace) and enable causal tracing.

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

Typed hierarchy from Phase 1; deferred batch flushed in **Events** tick phase ([decisions-log.md](decisions-log.md) EVT-01–EVT-03).

### IFactService

**Why:** Facts are the atomic truth layer — everything else interprets or reacts. Central registration prevents contradictory world state.

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

### IInfoFlowService

**Why:** “Who knows what” is distinct from facts themselves. Info flow powers investigation, dialogue gating, and fail-forward branches without overloading `IFactService`.

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

### IPersistenceService

**Why:** Save/load must be versioned and aggregate all `*State` DTOs without each service inventing its own JSON format.

```csharp
interface IPersistenceService
{
    int CurrentSchemaVersion { get; }
    SaveSnapshot Capture();
    void Restore(SaveSnapshot snapshot);
}
```

---

## Cognition

### IFacultyService

**Why:** Faculties are interpretive agents, not flat stats. The service exposes values, pools, and group queries for rolls and passive filters.

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

### IRollService

**Why:** Rolls are dramatic world-mutators, not binary pass/fail. Modes (Active, Passive, Repeatable, Gated) encode recovery and fail-forward semantics.

```csharp
interface IRollService
{
    RollResult Resolve(FacultyRoll roll);
    bool CanAttempt(RollMode mode, string rollId, RollContext context);
    void RecoverRepeatable(string rollId);
}
```

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
```

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

### IIdeologyService

```csharp
interface IIdeologyService
{
    float GetAxisValue(string axisId);
    void ShiftAxis(string axisId, float delta);
    string GetDominantLabel(string axisId);
}
```

---

## Story

### IDialogueService

```csharp
interface IDialogueService
{
    void StartConversation(string dialogueGraphId, string speakerActorId);
    DialogueNode GetCurrentNode();
    bool TrySelectChoice(string choiceId, out DialogueAdvanceResult result);
    void EndConversation();
}
```

### IStoryStateService

**Why:** Story flags and beats drive progression; they are not facts (simulation truth) and not chronicle entries (UI projection).

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

### IVoiceService

```csharp
interface IVoiceService
{
    void Speak(VoiceChannel channel, string textKey, string actorOrFacultyId);
    void Stop(VoiceChannel channel);
}
```

### IPacingService

**Why:** Unchecked feedback loops collapse narrative pacing. Pacing is a first-class constraint alongside rules.

```csharp
interface IPacingService
{
    bool CanFireContent(string contentId);
    void RegisterContentWindow(string contentId, PacingWindow window);
    void TickPacing();
}
```

### IOutcomeService

```csharp
interface IOutcomeService
{
    OutcomeSynthesis Synthesize(OutcomeContext context);
    string GetEndingId(OutcomeContext context);
}
```

### Fail-forward (pattern — no interface)

**Why:** Fail-forward is how NSF avoids hard stops. It coordinates `IRollService`, `IDialogueService`, `IStoryStateService`, and `IPacingService`. See [story-fail-forward.md](story-fail-forward.md).

---

## Ledger

### IChronicleService

**Why:** Chronicle is a **read-only projection** for the player journal. It never mutates authoritative thread or story state.

```csharp
interface IChronicleService
{
    IReadOnlyList<ChronicleSection> GetSections();
    IReadOnlyList<ChronicleEntry> GetEntries(string sectionId);
    void RefreshProjection();
}
```

### IThreadService + ThreadEngine

**Why:** Thread owns inquiry logic (evidence, theories, subjects). `ThreadEngine` evaluates resolution; `IThreadService` is the facade registered in the kernel.

```csharp
interface IThreadService
{
    void AddEvidence(string threadId, ThreadEvidence evidence);
    void AddTheory(string threadId, ThreadTheory theory);
    bool TryResolveSubject(string threadId, string subjectId, out ThreadResolution resolution);
}

class ThreadEngine
{
    ThreadState Evaluate(string threadId, IReadOnlyFactView facts);
}
```

---

## Rules

### IRuleEngine + RuleEngine

**Why:** Scattered condition checks become unmaintainable. One evaluator serves dialogue, gates, and scripts.

```csharp
interface IRuleEngine
{
    bool Evaluate(Rule rule, RuleContext context);
    void ExecuteActions(IReadOnlyList<Action> actions, RuleContext context);
}
```

### IGateService

```csharp
interface IGateService
{
    bool IsAllowed(string gateId, GateContext context);
    GateDenialReason GetDenialReason(string gateId, GateContext context);
}
```

---

## Content

### IContentStore

**Why:** Games are data-driven. A typed facade over an internal registry keeps content pack boundaries clean.

```csharp
interface IContentStore
{
    T GetDefinition<T>(string id) where T : ContentDefinition;
    bool TryGetDefinition<T>(string id, out T definition) where T : ContentDefinition;
    IReadOnlyList<string> GetAllIds<T>() where T : ContentDefinition;
}
```

Internal: `ContentRegistry` — not public; see [content-store.md](content-store.md).

### IContentPipeline

```csharp
interface IContentPipeline
{
    PipelineReport Validate(ContentPackManifest manifest);
    PipelineReport Build(ContentPackManifest manifest);
    bool TryFix(ContentPackManifest manifest, out PipelineReport report);
}
```

### ScriptCompiler

```csharp
class ScriptCompiler
{
    ScriptCompileResult Compile(string source, string filePath);
    ScriptCompileResult CompileFile(string nsfFilePath);
}
```

### ILocaleService

```csharp
interface ILocaleService
{
    string Resolve(string key, string localeId, IReadOnlyDictionary<string, string> substitutions);
    string GetFallbackLocale();
    bool HasKey(string key, string localeId);
}
```

### IInventoryService

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

---

## Presentation

**Why headless presenters:** Core logic must compile and test in Edit Mode without Canvas. Presenters receive view models; samples wire real UI in Phase 13+.

### IUIShell

```csharp
interface IUIShell
{
    void BindDialoguePresenter(IDialoguePresenter presenter);
    void BindChroniclePresenter(IChroniclePresenter presenter);
    void BindBeliefView(IBeliefView beliefView);
    void PushScreen(string screenId);
    void PopScreen();
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

### IDiscoveryService

```csharp
interface IDiscoveryService
{
    bool IsDiscovered(string discoverableId);
    void MarkDiscovered(string discoverableId, DiscoverySource source);
    IReadOnlyList<string> GetDiscoveredIds();
}
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

### IAudioNarrativeService

```csharp
interface IAudioNarrativeService
{
    void PlayVoiceLine(string voiceLineId, string localeId);
    void PlayAmbient(string ambientId, AudioAnchor anchor = null);
    void StopChannel(string channelId);
}
```

### ITextToSpeechProvider

**Why:** TTS out of the box ([decisions-log.md](../decisions-log.md) A-01). Presentation selects TTS vs pre-recorded clip per `VoiceDefinition`.

```csharp
interface ITextToSpeechProvider
{
    void Speak(string text, VoiceProfileDefinition profile, AudioPlayRequest request);
    void Stop(string channelId);
    bool IsSpeaking(string channelId);
}
```

---

## Shared domain types (summary)

Full schemas: [data-model.md](data-model.md).

| Type | Owner module |
|---|---|
| `SimEventBase`, `SimulationSnapshot` | Simulation / Runtime |
| `FactRecord`, `GameTime` | Simulation |
| `ContentDefinition` | Content |
| `SaveSnapshot`, `ServiceStateEnvelope` | Simulation (persistence) |
| `RollResult`, `FacultyRoll` | Cognition |

---

## Contract inventory (36 entries)

| # | Symbol | Kind |
|---|---|---|
| 1 | `ISimulationKernel` | Kernel |
| 2 | `INarrativeServiceRegistry` | Registry |
| 3 | `IEventBus` | Event bus |
| 4–35 | 32 × `I*Service` | Service |
| — | `RuleEngine`, `ThreadEngine`, `ScriptCompiler` | Concrete orchestrators |
| — | Fail-forward | Pattern |

Phase 0: stub all `I*` interfaces + null implementations. Engines/compiler deferred or no-op until their phases.
