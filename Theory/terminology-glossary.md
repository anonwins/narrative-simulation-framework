# NSF Terminology Glossary

**Single source of truth** for the Narrative Simulation Framework (NSF).

- Specs link here — do not redefine terms inline.
- Framework examples use **abstract placeholder IDs** (§ Content IDs).
- Setting-specific names belong in content packs or [`appendix-detective-noir-mapping.md`](appendix-detective-noir-mapping.md).

---

## SSOT rules

1. **Define once** in this glossary.
2. **Three layers** for every concept: Module → Prose → API (C#) → Content ID.
3. **Legacy names** only in § Migration map and the appendix.
4. New terms require a glossary entry before use in specs.
5. Deprecate `*System` suffix in new API names; prefer `*Service`, `*Engine`, `*Registry`.

---

## Module taxonomy

| Module | Prefix | Responsibility |
|---|---|---|
| **Runtime** | `runtime-` | Kernel, tick loop, simulation coordination |
| **Cognition** | `systems/cognition-` | Player mind: faculties, beliefs, conduct, emotion, rolls |
| **Social** | `systems/social-` | Relationships, factions, companions, ideology |
| **Simulation** | `systems/sim-` | World model: actors, time, locations, events, facts, economy, persistence |
| **Story** | `systems/story-` | Dialogue, voice, story state/flags, pacing, outcomes, fail-forward |
| **Ledger** | `systems/ledger-` | Chronicle (player record) + Thread (structured inquiry) |
| **Rules** | `systems/rules-` | Rule engine, gates |
| **Content** | `systems/content-` | Store, script DSL, pipeline, locale, inventory |
| **Presentation** | `systems/present-` | UI, audio, exploration, interaction, discovery |

### Module index (spec → primary service)

| Module | Spec | Primary API |
|---|---|---|
| Runtime | `runtime-kernel.md` | `ISimulationKernel` |
| Cognition | `systems/cognition-faculty.md` | `IFacultyService` |
| Cognition | `systems/cognition-belief.md` | `IBeliefService` |
| Cognition | `systems/cognition-conduct.md` | `IConductService` |
| Cognition | `systems/cognition-emotion.md` | `IEmotionService` |
| Cognition | `systems/cognition-roll.md` | `IRollService` |
| Social | `systems/social-relationship.md` | `IRelationshipService` |
| Social | `systems/social-faction.md` | `IFactionService` |
| Social | `systems/social-companion.md` | `ICompanionService` |
| Social | `systems/social-ideology.md` | `IIdeologyService` |
| Simulation | `systems/sim-actor.md` | `IActorService` |
| Simulation | `systems/sim-time.md` | `ITimeService` |
| Simulation | `systems/sim-location.md` | `ILocationService` |
| Simulation | `systems/sim-event.md` | `IEventBus` |
| Simulation | `systems/sim-fact.md` | `IFactService` |
| Simulation | `systems/sim-info-flow.md` | `IInfoFlowService` |
| Simulation | `systems/sim-economy.md` | `IEconomyService` |
| Simulation | `systems/sim-persistence.md` | `IPersistenceService` |
| Story | `systems/story-dialogue.md` | `IDialogueService` |
| Story | `systems/story-voice.md` | `IVoiceService` |
| Story | `systems/story-state.md` | `IStoryStateService` |
| Story | `systems/story-pacing.md` | `IPacingService` |
| Story | `systems/story-outcome.md` | `IOutcomeService` |
| Story | `systems/story-fail-forward.md` | (design pattern) |
| Ledger | `systems/ledger-chronicle.md` | `IChronicleService` |
| Ledger | `systems/ledger-thread.md` | `IThreadService`, `ThreadEngine` |
| Rules | `systems/rules-engine.md` | `IRuleEngine` |
| Rules | `systems/rules-gate.md` | `IGateService` |
| Content | `systems/content-store.md` | `IContentStore` |
| Content | `systems/content-script.md` | `ScriptCompiler` |
| Content | `systems/content-pipeline.md` | `IContentPipeline` |
| Content | `systems/content-locale.md` | `ILocaleService` |
| Content | `systems/content-inventory.md` | `IInventoryService` |
| Presentation | `systems/present-ui.md` | `IUIShell` |
| Presentation | `systems/present-audio.md` | `IAudioNarrativeService` |
| Presentation | `systems/present-exploration.md` | `IExplorationService` |
| Presentation | `systems/present-interaction.md` | `IInteractionService` |
| Presentation | `systems/present-discovery.md` | `IDiscoveryService` |
| Runtime | `runtime-kernel.md` | `INarrativeServiceRegistry` |

---

## Contracts catalog

**Canonical public API** for NSF. Spec `# Minimal Engine Interfaces → Service contract` sections must match these signatures. Domain types belong in spec `Domain model` subsections only.

| Kind | Symbol | Module | Role |
|---|---|---|---|
| Kernel | `ISimulationKernel` | Runtime | Tick loop orchestration |
| Registry | `INarrativeServiceRegistry` | Runtime | Service locator / DI surface |
| Event bus | `IEventBus` | Simulation | Pub/sub narrative events |
| Service | 32 × `I*Service` | per module | State mutation + queries |
| Engine | `RuleEngine` | Rules | Concrete rule evaluator (implements `IRuleEngine`) |
| Engine | `ThreadEngine` | Ledger | Concrete inquiry orchestrator (used by `IThreadService`) |
| Compiler | `ScriptCompiler` | Content | Concrete NSF Script compiler |
| Pattern | Fail-forward | Story | No service — integration pattern only |

### Registry naming

| Symbol | Visibility | Role |
|---|---|---|
| `IContentStore` | Public facade | Content pack access API |
| `ContentRegistry` | Internal | ID → definition lookup inside Content module |
| `FactRegistry` | Internal | ID → active `FactRecord` index inside Simulation |
| `StoryFlagRegistry` | Internal | ID → flag definition index inside Story |

**Rule:** `*Registry` = internal lookup; `I*Store` / `I*Service` = public facade.

### SimulationTickPhase

Implementation tick order (Phase 1 kernel). Maps nine conceptual loop layers to six code phases:

| Phase | Enum value | Conceptual layers | Primary services |
|---|---|---|---|
| 1 | `Facts` | Layer 1: Facts | `IFactService` |
| 2 | `Interpretation` | Layer 2: Interpretation | `IDialogueService`, `IVoiceService`, cognition interpretation |
| 3 | `Social` | Layers 3–4: Relationship + Faction | `IRelationshipService`, `IFactionService`, `IIdeologyService` |
| 4 | `Events` | Layers 5–6: Event + World State | `IEventBus`, `IActorService`, `ITimeService`, `ILocationService` |
| 5 | `Gating` | Layer 7: Constraints | `IGateService`, `IRuleEngine`, `IPacingService` |
| 6 | `Content` | Layers 8–9: New Content + New Facts | `IContentStore`, script effects, new fact emission |

### Runtime infrastructure

```csharp
enum SimulationTickPhase { Facts, Interpretation, Social, Events, Gating, Content }

interface ISimulationKernel
{
    SimulationTickPhase CurrentPhase { get; }
    void Tick(SimulationTickPhase? singlePhase = null);
    void Initialize(INarrativeServiceRegistry registry);
}

interface INarrativeServiceRegistry
{
    T GetRequired<T>() where T : class;
    bool TryGet<T>(out T service) where T : class;
    void Register<T>(T service) where T : class;
}
```

### Presentation presenters (Phase 13)

Headless-testable UI bridges — not referenced by core modules:

| Symbol | Role |
|---|---|
| `IUIShell` | Presentation root; wires presenters |
| `IDialoguePresenter` | Dialogue UI model sink |
| `IChroniclePresenter` | Chronicle UI model sink |
| `IBeliefView` | Belief slot UI model sink |

---

## API suffix rules

| Suffix | Use for |
|---|---|
| `*Service` | Runtime behavior, state mutation |
| `*Definition` | Authored ScriptableObject / asset |
| `*State` | Serializable runtime snapshot |
| `*Registry` | ID → definition lookup |
| `*Store` | Large content repository |
| `*Gate` | Access rule |
| `*Controller` | Presentation bridge |
| `*Engine` | Orchestrators only: `RuleEngine`, `ThreadEngine` |

Interface pattern: `I{Concept}Service` unless ambiguous, then `I{Module}{Concept}Service`.

---

## Content IDs

Format: `{prefix}_{descriptor}` (snake_case). No setting nouns.

| Prefix | Module | Examples |
|---|---|---|
| `faculty_` | Cognition | `faculty_intuition`, `faculty_instinct` |
| `belief_` | Cognition | `belief_doctrine_a` |
| `conduct_` | Cognition | `conduct_humble`, `conduct_bold` |
| `emotion_` | Cognition | `emotion_stress` |
| `actor_` | Simulation | `actor_companion`, `actor_merchant` |
| `faction_` | Social | `faction_guild`, `faction_authority` |
| `metric_` | Social | `metric_trust_companion` |
| `fact_` | Simulation | `fact_incident_time` |
| `flag_` | Story | `flag_thread_resolved` |
| `beat_` | Story | `beat_intro`, `beat_resolution` (optional; may use content-store IDs) |
| `thread_` | Ledger | `thread_main` |
| `section_` | Ledger | `section_primary` |
| `location_` | Simulation | `location_warehouse` |
| `group_` | Cognition | `group_psyche` (faculty group) |

---

## Term dictionary by module

### Framework meta

| Term | Definition |
|---|---|
| **NSF** | Narrative Simulation Framework — reusable narrative simulation layer for Unity. |
| **Content pack** | Game built on NSF: scripts + data + assets + config. |
| **Player character** | Protagonist entity; role is content-defined. |

### Cognition

| Term | API | Notes |
|---|---|---|
| **Faculty** | `Faculty`, `FacultyDefinition`, `IFacultyService` | Autonomous interpretive faculty: value, voice, passive filter, roll input. |
| **Faculty group** | `FacultyGroup` | Top-level stat bucket (default 4×6 schema is content-defined). |
| **Roll** | `FacultyRoll`, `IRollService`, `RollResult` | Faculty roll resolution. |
| **Roll mode** | `RollMode` | Active, Passive, Repeatable, Gated. |
| **Belief** | `Belief`, `BeliefDefinition`, `IBeliefService` | Idea the player assimilates; not a static perk. |
| **Belief phase** | `BeliefPhase` | Discovered → Assimilating → Resolved → Forgotten. |
| **Conduct** | `ConductScore`, `IConductService` | Emergent identity from repeated behavior (`conduct_*` IDs). |
| **Standing** | `StandingScore` | Public reputation (distinct from conduct). |
| **Emotion** | `EmotionState`, `IEmotionService` | Mood/stress/confidence affecting narration and rolls. |
| **Vitality / Morale** | `Vitality`, `Morale` | Wellbeing pools derived from content-assigned faculties. |

### Social

| Term | API | Notes |
|---|---|---|
| **Actor** | `ActorDefinition` | Any autonomous character (see Simulation). |
| **Companion** | `ICompanionService` | Social subsystem for persistent partner actor. |
| **Relationship** | `IRelationshipService` | Trust, affinity, respect, dependency. |
| **Faction** | `IFactionService`, `FactionId` | Organized group with agenda and reputation. |
| **Ideology** | `IIdeologyService`, `IdeologyAxis` | Worldview accumulation on content-defined axes. |

### Simulation

| Term | API | Notes |
|---|---|---|
| **Actor** | `IActorService`, `ActorDefinition` | World character with memory, schedule, dialogue state. |
| **Fact** | `IFactService`, `FactRecord` | Atomic persistent truth unit. |
| **Info flow** | `IInfoFlowService` | Who learned what, when, from whom. |
| **Persistence** | `IPersistenceService` | Save/load and long-term consequence. |
| **Event** | `SimEvent`, `IEventBus` | Triggered world/narrative update. |
| **Time** | `ITimeService` | In-game clock and schedules. |
| **Location** | `ILocationService` | Areas, transitions, fast travel. |

### Story

| Term | API | Notes |
|---|---|---|
| **Story flag** | `StoryFlag`, `StoryFlagRegistry` | Boolean/enum narrative state driver. |
| **Story state** | `IStoryStateService` | Progression orchestrator: flags, variables, beat transitions. |
| **Story beat** | `StoryBeat`, `StoryBeatDefinition`, `StoryBeatState` | Atomic progression unit inside story state (start → advance → complete). Not a chronicle entry. |
| **Pacing** | `IPacingService` | When content may fire. |
| **Voice** | `IVoiceService` | Narrator, monologue, environmental narration. |
| **Outcome** | `IOutcomeService` | Ending synthesis. |
| **Fail forward** | — | Design pattern: failure produces content. |

### Ledger

| Term | API | Notes |
|---|---|---|
| **Chronicle** | `IChronicleService`, `ChronicleView` | Player-facing narrative record (not a quest log). |
| **Chronicle section** | `ChronicleSection` | Chronicle grouping for one storyline. |
| **Thread** | `IThreadService`, `ThreadEngine` | Evidence/theory/subject inquiry logic. |
| **Thread subject** | `ThreadSubject` | Entity or hypothesis under inquiry. |
| **Lead / Task / Clue** | `ChronicleEntryType` | Entry types in chronicle. |
| **Evidence / Theory** | `ThreadEvidence`, `ThreadTheory` | Thread reasoning objects. |

### Rules

| Term | API | Notes |
|---|---|---|
| **Rule engine** | `IRuleEngine`, `Rule`, `Condition`, `Action` | IF/THEN evaluation. |
| **Gate** | `IGateService`, `GateRule` | Content access rule. |

### Content

| Term | API | Notes |
|---|---|---|
| **Content store** | `IContentStore`, `ContentRegistry` | Public facade + internal ID lookup. |
| **Fact registry** | `FactRegistry` | Internal index of active facts (via `IFactService`). |
| **Story flag registry** | `StoryFlagRegistry` | Internal index of flag definitions (via `IStoryStateService`). |
| **Script** | `ScriptNode`, `.nsf` files | NSF Script DSL. |
| **Content pipeline** | `IContentPipeline` | Authoring validation workflow. |
| **Locale** | `ILocaleService` | Localization. |

### Presentation

| Term | API | Notes |
|---|---|---|
| **Faculty interjection** | `FacultyInterjection` | Faculty break-in during dialogue. |
| **Belief view** | `BeliefView` | Belief slot UI (skin label = content). |
| **UI shell** | `IUIShell` | Presentation layer root. |
| **Dialogue presenter** | `IDialoguePresenter` | Headless-testable dialogue UI bridge. |
| **Chronicle presenter** | `IChroniclePresenter` | Headless-testable chronicle UI bridge. |

### Runtime

| Term | API | Notes |
|---|---|---|
| **Simulation kernel** | `ISimulationKernel` | Meta-coordination of all modules in the simulation loop. |
| **Service registry** | `INarrativeServiceRegistry` | Resolves all `I*Service` implementations at runtime. |
| **Simulation tick phase** | `SimulationTickPhase` | Kernel execution order (see § SimulationTickPhase). |

---

## Term relationships

```text
ThreadEngine      — inquiry logic (evidence, theories, subjects); authoritative for investigation
StoryState        — drives progression (flags, story beats, variables)
Chronicle         — read-only projection (sections, leads, tasks, clues)
ChronicleSection  — chronicle grouping for one storyline; not a story beat ID
StoryBeat         — one progression unit in IStoryStateService; not the whole thread
Task / Lead / Clue — chronicle entry types; projected from thread + story state + facts
```

### When to use which term

| You mean… | Use | Not |
|---|---|---|
| Evidence, theories, subjects, inquiry resolution | **Thread** / `ThreadEngine` | Story beat, quest |
| Start / advance / complete a narrative step | **Story beat** / `IStoryStateService` | Thread, task |
| Player journal grouping (“the homicide storyline”) | **Chronicle section** / `section_*` | Thread object, story beat |
| “Talk to X”, “find the key” in the chronicle UI | **Task** / **Lead** / **Clue** | Story beat (unless authoring a beat transition) |
| Player-facing label “Quests” in a detective game | **Content pack UI skin** | Framework API name |
| Atomic world truth | **Fact** / `IFactService` | Story flag, chronicle text |

**Actor vs Companion:** Actor is the simulation entity type; Companion is the social subsystem wrapping a partner `ActorId`.

**Conduct vs Standing:** Conduct = internal behavior scores (`conduct_*`); Standing = public reputation label.

**Fact vs Story flag:** Fact = simulation truth; story flag = narrative progression switch owned by story state.

---

## Migration map (intermediate → NSF API)

| Old | New |
|---|---|
| Skill / ISkill / SkillDefinition | Faculty / IFaculty / FacultyDefinition |
| ISkillCheckService | IRollService |
| Skill check / white-red check | Roll / Repeatable or Gated roll |
| Thought / Thought Cabinet | Belief / BeliefView |
| Copotype / archetype (identity) | Conduct / `conduct_*` |
| Investigation Journal | Chronicle |
| Case (journal grouping) | Chronicle section / `section_*` |
| Case (monolithic investigation) | Thread + chronicle section |
| Quest / quest log (framework) | Split: **Thread** + **Story beat** + **Chronicle** (see § When to use which term) |
| Quest (content pack UI label) | Keep in content pack; map to story beat / section in data |
| InquirySubject | ThreadSubject |
| Investigation Thread | Thread / ThreadEngine |
| Belief Internalization | Belief / IBeliefService |
| Internalizing / Internalized | Assimilating / Resolved |
| Behavioral Archetype | Conduct |
| Narrative flag | Story flag |
| NPC (generic) | Actor |
| Political alignment | Ideology |
| Reputation (public) | Standing |
| Knowledge system | Fact / IFactService |
| Check resolution | Roll |
| Save-state consequence | Persistence |
| Information propagation | Info flow |
| Content gating | Gate |
| Narrative Simulation Core | Simulation kernel |

---

## Legacy renames (commercial game → NSF)

| Legacy | NSF |
|---|---|
| Disco Elysium | NSF / narrative simulation RPG |
| Thought Cabinet | Belief system |
| Thought(s) | Belief(s) |
| Copotype | Conduct |
| White / Red check | Repeatable / Gated roll |
| HP / MOR | Vitality / Morale |

Full instance mappings: [`appendix-detective-noir-mapping.md`](appendix-detective-noir-mapping.md).

---

## Integration section headers

| Legacy | NSF |
|---|---|
| `# Belief Integration` | `# Belief Integration` |
| `# Conduct Integration` | `# Conduct Integration` |
| `# Belief UI` | `# Belief View` |

---

## Authoring rules

- Spec title: `# {N}. {Module}: {Concept} — NSF Specification`
- Framework examples: placeholder IDs only (§ Content IDs)
- **Term usage:** follow § When to use which term — do not use "quest" in framework API/prose except when contrasting with chronicle design or describing content-pack UI skins
- Scope block (systems specs use `../terminology-glossary.md`):

```markdown
> **Framework:** [mechanics, interfaces, data shapes NSF owns]
> **Content pack:** [names, labels, IDs, UI skin the game owns]
> Terminology: [Glossary](../terminology-glossary.md)
```

- Spec tail structure (required):

```markdown
# Minimal Engine Interfaces
> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract
(canonically matches glossary § Contracts catalog)

## Domain model
(internal types only)

# Minimum Viable System
```
