# Narrative Simulation Framework (NSF)

Design specifications for a **reusable narrative simulation layer** for Unity. NSF targets investigation-heavy, dialogue-first, failure-forward RPGs—the genre often called "Disco-like"—without coupling docs to any single commercial game or IP.

**Games = Framework + Content Pack.** Each game adds content, configuration, and thin game scripts—not new core systems.

Start here → [System Catalog](index.md) · [Terminology Glossary](terminology-glossary.md)

---

## Three layers

```text
Layer 1 — Unity (base engine)
  Rendering, physics, animation, audio playback, input

Layer 2 — NSF (this framework)
  World simulation, actor cognition, dialogue logic, emotional systems,
  rule engine, pacing, content system

Layer 3 — Games (content packs)
  inquiry noir · sci-fi investigation · political thriller · …
  Each = NSF + scripts + data + assets
```

NSF is a **simulation runtime that generates games from content**. Unity handles engine concerns; NSF handles narrative simulation; games supply setting, characters, and authored narrative.

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

---

## Documentation map

| Doc | Purpose |
|---|---|
| [development-roadmap.md](development-roadmap.md) | Implementation roadmap: zero → production (agent phases + human gates) |
| [index.md](index.md) | Catalog of all 40 NSF systems with module tags and file links |
| [terminology-glossary.md](terminology-glossary.md) | Canonical terms, API names, content ID prefixes (SSOT) |
| [runtime-kernel.md](runtime-kernel.md) | Meta-guide: how all modules interact in the simulation loop |
| [appendix-detective-noir-mapping.md](appendix-detective-noir-mapping.md) | Reference content pack: NSF API term → detective-noir instance (not framework requirements) |
| [`systems/`](systems/) | Per-system specifications (37 files: cognition, sim, story, etc.) |
| [`architecture/`](architecture/) | Pre-implementation architecture plans (design before code) |

Individual system specs in [`systems/`](systems/) follow a common shape: **Purpose → Core Principle → Architecture → Integration → Minimum Viable System (where present) → Final Concept**. See [development-roadmap.md](development-roadmap.md) for MVP scope rules when a spec omits that section.

---

## Content pack boundary

| Framework owns | Content pack owns |
|---|---|
| Simulation mechanics and interfaces | Character and faction names |
| Data shapes (beliefs, conduct, facts, story flags) | Faculty group and faculty definitions |
| Rule evaluation, gating, pacing logic | Conduct IDs and ideology axis labels |
| Dialogue/belief/chronicle *systems* | Dialogue/belief/chronicle *content* |
| Roll resolution mechanics | DC values, faculty assignments |
| UI system architecture | UI skin labels (e.g. what to call the belief screen) |

---

## Building a game on NSF

```text
GameProject/
    Packages/NarrativeFramework/    ← NSF package
    Assets/ContentPack/             ← your game's data + assets
        Actors, factions, dialogue graphs, beliefs, threads
    Assets/GameScripts/             ← thin game-specific hooks only
```

No rewriting core systems per game. Extend via content packs and configuration.
