# Narrative Simulation Framework (NSF)

Design specifications for a **reusable narrative simulation library** for Unity. NSF targets investigation-heavy, dialogue-first, failure-forward RPGs—the genre often called "Disco-like"—without coupling docs to any single commercial game or IP.

**NSF is the narrative brain — not the whole game.** Games compose NSF modules they need, add game-owned systems (combat, crafting, genre loops), and connect outcomes back through documented extension contracts. See [architecture/game-extensions.md](architecture/game-extensions.md).

Start here → [System Catalog](index.md) · [Terminology Glossary](terminology-glossary.md) · [Decisions Log](decisions-log.md)

---

## Three layers

```text
Layer 1 — Unity (base engine)
  Rendering (2D/3D — game choice), physics, animation, audio playback, input, scenes

Layer 2 — NSF (narrative simulation library)
  Shared infrastructure: registry, kernel, events, persistence, rules, content pipeline
  Optional narrative modules: dialogue, facts, rolls, chronicle, thread, economy, …
  Dimension-agnostic — see architecture/unity-host.md

Layer 3 — Games
  Content (characters, dialogue, beliefs, threads) + game-owned modules + presentation
  inquiry noir (3D sample) · sci-fi investigation (2D sample) · …
  NoirSample + SciFiSample exercise the full NSF stack — see architecture/samples.md
```

Unity handles engine concerns. NSF handles **narrative simulation** — the closed loop from facts through dialogue to chronicle. Games supply setting, authored narrative, presentation, and any mechanics NSF does not ship.

---

## Unity package layout (target)

Modular package—not a monolithic game project:

```text
/NarrativeFramework
    /Runtime        — kernel, tick loop, simulation coordination
    /Cognition      — faculties, beliefs, conduct, emotion, rolls
    /Social         — relationships, factions, companions, ideology
    /Simulation     — actors, time, locations, events, facts, economy, persistence
    /Story          — dialogue, voice, story state, pacing, outcomes, fail-forward
    /Ledger         — chronicle, thread
    /Rules          — rule engine, gates
    /Content        — store, script DSL, pipeline, locale, inventory
    /Presentation   — UI, audio, exploration, interaction, discovery
    /Debug          — inspectors, visualizers, causal tracing
```

Games add their own assemblies (e.g. `MyGame.Runtime`) for peer modules and adapters — not inside this package.

---

## Documentation map

| Doc | Purpose |
|---|---|
| [api-reference.md](api-reference.md) | Rough API reference (as-if-built) — all public services, types, events |
| [development-roadmap.md](development-roadmap.md) | Implementation roadmap: zero → production (agent phases + human gates) |
| [index.md](index.md) | Catalog of all 40 NSF systems with module tags and file links |
| [terminology-glossary.md](terminology-glossary.md) | Canonical terms, API names, content ID prefixes (SSOT) |
| [decisions-log.md](decisions-log.md) | Locked product/architecture choices (replaces “deferred” forks) |
| [feature-list.md](feature-list.md) | What NSF’s narrative stack enables — player/designer view |
| [runtime-kernel.md](runtime-kernel.md) | Meta-guide: how all modules interact in the simulation loop |
| [architecture/game-extensions.md](architecture/game-extensions.md) | How games add mechanics NSF does not ship (adapters, peer modules) |
| [appendix-detective-noir-mapping.md](appendix-detective-noir-mapping.md) | Reference content pack: NSF API term → detective-noir instance (not framework requirements) |
| [`systems/`](systems/) | Per-system specifications (37 files: cognition, sim, story, etc.) |
| [`architecture/`](architecture/) | Implementation blueprints — start at [architecture/index.md](architecture/index.md); Unity host & platform boundaries: [architecture/unity-host.md](architecture/unity-host.md) |

## Documentation layers

```text
terminology-glossary.md     SSOT — terms, API names, contracts catalog
decisions-log.md            Locked forks — Addressables, samples, TTS, pacing, …
systems/*.md                Per-system behavior specs (Service contract + Domain model)
architecture/*.md           Implementation blueprints (after Phase A; before code)
development-roadmap.md      Phased build order and deliverables
```

Individual system specs in [`systems/`](systems/) follow: **Purpose → Core Principle → Architecture → Integration → Minimal Engine Interfaces (Service contract + Domain model) → Minimum Viable System → Final Concept**.

---

## Responsibility boundaries

| NSF owns | Game owns |
|---|---|
| Narrative simulation mechanics and `I*Service` contracts | Characters, setting, art, presentation skin |
| Data shapes (beliefs, conduct, facts, story flags) | Faculty definitions, conduct IDs, dialogue content |
| Rule evaluation, gating, pacing logic | DC values, ideology axis labels |
| Dialogue/belief/chronicle *systems* | Dialogue/belief/chronicle *content* |
| Roll resolution mechanics | Faculty assignments per game |
| Registry, events, persistence envelope | **Game-owned modules** (combat, crafting, …) |
| UI system architecture (presenters) | UI skin labels; game-specific HUD for non-NSF loops |

**Game-owned mechanics** integrate via [game-extensions.md](architecture/game-extensions.md) — adapters (outcomes → facts/flags) or peer modules (`IStatefulService` in the shared registry). NSF does not block features it does not ship; it defines how those features connect to the narrative sim.

---

## Building a game on NSF

```text
GameProject/
    Packages/NarrativeFramework/    ← NSF package (narrative library)
    ../NoirSample/ or ../SciFiSample/   ← sibling sample repos (Phase 15–16)
    Assets/ContentPack/             ← your game's data + assets
        Actors, factions, dialogue graphs, beliefs, threads
    Assets/GameScripts/             ← game modules, adapters, presentation bridges
        MyGame.Runtime.asmdef       ← references NSF; registers game services
```

Compose the NSF modules you need. Add game systems alongside them. Do not fork NSF per game — extend through content, configuration, adapters, and documented peer modules.
