# Game Extensions — Composing NSF + Game-Owned Systems

**How games add mechanics NSF does not ship** — without forking the package or bypassing the narrative simulation.

- Host & layers: [unity-host.md](unity-host.md)
- Contracts: [contracts.md](contracts.md)
- Persistence: [sim-persistence.md](sim-persistence.md)
- Rules plugins: [rules-engine.md](rules-engine.md) · [decisions-log.md](../decisions-log.md) ST-05
- Overview: [README.md](../README.md)

---

## Design rationale

### Why NSF is a library, not a game ceiling

NSF ships a **battle-tested narrative simulation stack** — dialogue, facts, rolls, chronicle, pacing, and the closed loop that connects them. That is its product.

It is **not** the whole game. Games routinely need combat, crafting, vehicles, stealth loops, or genre-specific progression NSF does not define. Those belong in **game-owned modules** registered alongside NSF services, not as undocumented Unity-side silos.

**NSF should not read as "your game can only do what we built."** It should read as: *use our narrative brain; add your own systems through documented hooks; when those systems matter to the story, feed outcomes back in.*

### Why integration still matters

Freedom without contracts produces orphan mechanics — fights that do not persist, chronicle gaps, gates that never see combat outcomes. Extension contracts exist so game features can be **optional peers** of NSF, not parallel universes.

The default integrated NSF stack remains the recommended path for investigation-heavy, dialogue-first games. Extension docs describe how to **add** without **replacing**.

---

## Three layers (revised)

```text
Layer 1 — Unity
  Rendering, physics, animation, audio playback, input, scenes

Layer 2 — NSF (optional modules, shared infrastructure)
  Registry · kernel · events · persistence · rules · content pipeline
  + narrative modules you choose (dialogue, facts, thread, economy, …)

Layer 3 — Game
  Content (definitions, dialogue, art) + game-owned modules (combat, crafting, …)
  + Presentation bridges (2D/3D, UI skin)
```

A game may use **all** NSF modules (samples), **a subset** (dialogue + facts only), or **full stack + game modules** (action-RPG with investigation).

---

## Three ways to add a game feature

| Approach | When to use | Integration |
|---|---|---|
| **Content + configuration** | Feature is authored narrative (new dialogue, items, beliefs, gates) | Native — pipeline + `IContentStore` |
| **Adapter** | Feature has its own loop but **outcomes** should affect the story (combat, chase, hacking minigame) | Game code runs loop → adapter writes facts, flags, rolls, vitality, events |
| **Peer module** | Feature needs authoritative runtime state, saves, and optional tick participation (sustained combat system, crafting) | Game registers `I*Service` + `IStatefulService` in shared registry |

**Decision rule:**

```text
Must the narrative sim remember this?
  → Yes: surface outcomes via facts, flags, events, or rolls (adapter minimum)

Must the game own authoritative loop state across saves?
  → Yes: peer module + IStatefulService

Will multiple games reuse this mechanic?
  → Consider promoting to an NSF module (new spec + architecture doc)
```

---

## Extension contracts

### 1. Game-owned composition root

`NsfSession` is a **convenience preset** that wires the full narrative stack. Games may instead:

- Call `NsfSession.Create(config)` with a module subset `[FULL]` — see `NsfSessionConfig.EnabledModules`
- Build `INarrativeServiceRegistry` manually and register only needed NSF services
- Register game services in the **same registry** before `ISimulationKernel.Initialize`

```csharp
// Game bootstrap (sample or commercial project)
var registry = NsfBootstrap.CreateRegistry(manifest, options);

registry.Register<ICombatService>(new CombatService(registry));
registry.Register<IStatefulService>(registry.GetRequired<ICombatService>());

var kernel = registry.GetRequired<ISimulationKernel>();
kernel.Initialize(registry);
```

**Rule:** Game modules live in **game assemblies** (`MyGame.Runtime.asmdef`), not in `Packages/NarrativeFramework/`. NSF core must not reference game assemblies.

### 2. Registry — optional vs required

NSF services that a game omits must be consumed via `TryGet`, not `GetRequired`, in game and adapter code.

Games register custom services with `Register<T>`. Other game code and adapters resolve them the same way as NSF services.

### 3. Persistence — `IStatefulService`

`IPersistenceService` discovers every `IStatefulService` in the registry and stores each envelope in the versioned save blob.

Game peer modules that hold save-relevant state **must** implement `IStatefulService<TState>` (or the non-generic adapter) with a unique `StateKey` (e.g. `MyGame.Combat.CombatService`).

**Adapter-only features** usually do not need their own envelope — they persist **outcomes** through existing NSF state (facts, flags, vitality, conduct).

### 4. Events — `IEventBus`

Game modules publish typed `SimEventBase` subclasses (game assembly) so NSF subscribers can react when wired:

| Game publishes | NSF may react |
|---|---|
| `CombatResolvedEvent` (adapter) | `IFactService` handler registers `fact_*`; chronicle projection |
| Custom game event | Info flow, actor memory, conduct — if game registers handlers |

Subscribe in the **Events** tick phase or via deferred queue per [sim-event.md](sim-event.md). Do not mutate NSF services from unrelated Unity `Update()` without going through the event/tick boundary.

### 5. Rules — condition/action plugins

Per [decisions-log.md](../decisions-log.md) ST-05, games register custom **conditions** and **actions** on the shared `RuleEngine` (Phase 12+). Use for dialogue branches and gates that read game service state:

```text
Condition: CombatService.HasDefeated("actor_rival")
Action:     CombatService.StartEncounter("encounter_alley")
```

Prefer adapters that **write facts** when the outcome is narrative truth — gates and thread logic already understand facts.

### 6. Narrative adapters (default pattern)

Thin game-layer classes that translate game outcomes into NSF primitives:

```text
CombatAdapter
  OnEncounterEnd(result):
    IFactService.Register(fact_enemy_defeated)
    IStoryStateService.SetFlag(flag_alley_fight_done)
    IConductService.RecordAction(conduct_ruthless)   // if applicable
    IEmotionService.ApplyStress(...)                   // if applicable
    IEventBus.Publish(CombatResolvedEvent)
```

Same pattern applies to chases, stealth alerts, crafting milestones, or any mechanic where **the story** must remember what happened.

---

## Optional NSF modules

Games are not required to register every NSF service. Common subsets:

| Game profile | Typical NSF modules | Game adds |
|---|---|---|
| Pure investigation (samples) | Full stack | Content + presentation only |
| Visual novel + light sim | Dialogue, story state, facts, rolls, chronicle | Minimal exploration bridge |
| Action-RPG + investigation | Dialogue, facts, rolls, chronicle, social, inventory | Combat peer module + adapters |
| Custom genre | Pick per feature | Peer modules + adapters |

Omitted modules: use `TryGet` in bootstrap; do not register `Null*` stubs unless a dependency chain requires them.

---

## What not to do

| Anti-pattern | Why it fails |
|---|---|
| Combat/logic only in MonoBehaviours, no registry | Saves, chronicle, gates never see it |
| Forking NSF package per game | Loses UPM reuse and test matrix |
| Bolting game logic into NSF asmdefs | Violates package boundary; blocks other games |
| Game state only in PlayerPrefs / parallel JSON | Split-brain saves; load order bugs |
| Expecting chronicle/thread to auto-update | Requires facts, events, or explicit projection hooks |

---

## Promoting a game module to NSF

When a mechanic proves reusable across games (e.g. shared combat layer):

1. Add `systems/` spec + `architecture/` blueprint
2. Add glossary entries and content ID prefixes if needed
3. Append decision to [decisions-log.md](../decisions-log.md)
4. Move implementation from game assembly to `Packages/NarrativeFramework/`

Do not promote prematurely — let the adapter pattern carry v1 until a second game needs the same module.

---

## MVP scope (documentation + Phase 0+)

- [x] Extension model documented (this file)
- [ ] `NsfSessionConfig.EnabledModules` or documented manual bootstrap presets `[FULL]`
- [ ] Sample `CombatAdapter` stub in one game repo showing fact/flag write `[FULL]`
- [ ] Rule engine game plugin registration API `[FULL]` Phase 12
- [ ] Integration test: game `IStatefulService` round-trips in save envelope `[FULL]`

---

## Related documents

- [unity-host.md](unity-host.md) — platform boundaries, `NsfSession`, tick driver
- [integration.md](integration.md) — full-loop bootstrap reference
- [samples.md](samples.md) — reference games (full NSF stack; no game-owned combat in samples v1)
- [README.md](../README.md) — composable model overview
