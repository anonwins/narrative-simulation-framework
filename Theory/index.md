# NSF System Catalog

Catalog of all **Narrative Simulation Framework (NSF)** systems.

- [Framework overview](README.md)
- [Terminology glossary](terminology-glossary.md) (SSOT)
- [Development roadmap](development-roadmap.md) (implementation phases)
- [Decisions log](decisions-log.md) (locked architecture choices)
- [Feature list](feature-list.md) (what NSF’s narrative stack enables)
- [Game extensions](architecture/game-extensions.md) (adding mechanics NSF does not ship)
- [API reference](api-reference.md) (rough as-if-built API docs)
- [Simulation kernel meta-guide](runtime-kernel.md)

- [Architecture index](architecture/index.md) (implementation blueprints)

**Doc layers:** [Glossary](terminology-glossary.md) (SSOT) → [systems/](systems/) (specs) → [architecture/](architecture/) (implementation plans). Platform & Unity host: [architecture/unity-host.md](architecture/unity-host.md).

**40 logical systems · 37 spec files in [`systems/`](systems/)** (systems 1–2, 3–4 share a file each), plus [runtime-kernel.md](runtime-kernel.md). Each system spec has a matching [architecture/](architecture/) blueprint (see [architecture/index.md](architecture/index.md)).

---

## Systems by module

### Runtime

| # | System | File |
|---|---|---|
| 40 | Kernel | [runtime-kernel.md](runtime-kernel.md) |

### Cognition

| # | System | File |
|---|---|---|
| 1–2 | Faculty & Character | [cognition-faculty.md](systems/cognition-faculty.md) |
| 7 | Belief | [cognition-belief.md](systems/cognition-belief.md) |
| 15 | Conduct | [cognition-conduct.md](systems/cognition-conduct.md) |
| 20 | Roll | [cognition-roll.md](systems/cognition-roll.md) |
| 37 | Emotion | [cognition-emotion.md](systems/cognition-emotion.md) |

### Social

| # | System | File |
|---|---|---|
| 14 | Relationship | [social-relationship.md](systems/social-relationship.md) |
| 16 | Ideology | [social-ideology.md](systems/social-ideology.md) |
| 17 | Companion | [social-companion.md](systems/social-companion.md) |
| 22 | Faction | [social-faction.md](systems/social-faction.md) |

### Simulation

| # | System | File |
|---|---|---|
| 10 | Actor | [sim-actor.md](systems/sim-actor.md) |
| 11 | Time | [sim-time.md](systems/sim-time.md) |
| 12 | Location | [sim-location.md](systems/sim-location.md) |
| 13 | Event | [sim-event.md](systems/sim-event.md) |
| 21 | Economy | [sim-economy.md](systems/sim-economy.md) |
| 24 | Persistence | [sim-persistence.md](systems/sim-persistence.md) |
| 28 | Fact | [sim-fact.md](systems/sim-fact.md) |
| 29 | Info Flow | [sim-info-flow.md](systems/sim-info-flow.md) |

### Story

| # | System | File |
|---|---|---|
| 6 | Dialogue | [story-dialogue.md](systems/story-dialogue.md) |
| 8 | Story State | [story-state.md](systems/story-state.md) |
| 19 | Voice | [story-voice.md](systems/story-voice.md) |
| 27 | Fail Forward | [story-fail-forward.md](systems/story-fail-forward.md) |
| 38 | Pacing | [story-pacing.md](systems/story-pacing.md) |
| 39 | Outcome | [story-outcome.md](systems/story-outcome.md) |

### Ledger

| # | System | File |
|---|---|---|
| 5 | Chronicle | [ledger-chronicle.md](systems/ledger-chronicle.md) |
| 23 | Thread | [ledger-thread.md](systems/ledger-thread.md) |

### Rules

| # | System | File |
|---|---|---|
| 26 | Gate | [rules-gate.md](systems/rules-gate.md) |
| 30 | Rule Engine | [rules-engine.md](systems/rules-engine.md) |

### Content

| # | System | File |
|---|---|---|
| 3–4 | Inventory & Equipment | [content-inventory.md](systems/content-inventory.md) |
| 31 | Content Store | [content-store.md](systems/content-store.md) |
| 32 | Script | [content-script.md](systems/content-script.md) |
| 33 | Pipeline | [content-pipeline.md](systems/content-pipeline.md) |
| 34 | Locale | [content-locale.md](systems/content-locale.md) |

### Presentation

| # | System | File |
|---|---|---|
| 9 | Interaction | [present-interaction.md](systems/present-interaction.md) |
| 18 | Discovery | [present-discovery.md](systems/present-discovery.md) |
| 25 | Exploration | [present-exploration.md](systems/present-exploration.md) |
| 35 | UI | [present-ui.md](systems/present-ui.md) |
| 36 | Audio | [present-audio.md](systems/present-audio.md) |

---

## Full system index

| # | System | Module | File |
|---|---|---|---|
| 1 | Faculty | Cognition | [cognition-faculty.md](systems/cognition-faculty.md) |
| 2 | Character | Cognition | [cognition-faculty.md](systems/cognition-faculty.md) |
| 3 | Inventory | Content | [content-inventory.md](systems/content-inventory.md) |
| 4 | Equipment | Content | [content-inventory.md](systems/content-inventory.md) |
| 5 | Chronicle | Ledger | [ledger-chronicle.md](systems/ledger-chronicle.md) |
| 6 | Dialogue | Story | [story-dialogue.md](systems/story-dialogue.md) |
| 7 | Belief | Cognition | [cognition-belief.md](systems/cognition-belief.md) |
| 8 | Story State | Story | [story-state.md](systems/story-state.md) |
| 9 | Interaction | Present | [present-interaction.md](systems/present-interaction.md) |
| 10 | Actor | Sim | [sim-actor.md](systems/sim-actor.md) |
| 11 | Time | Sim | [sim-time.md](systems/sim-time.md) |
| 12 | Location | Sim | [sim-location.md](systems/sim-location.md) |
| 13 | Event | Sim | [sim-event.md](systems/sim-event.md) |
| 14 | Relationship | Social | [social-relationship.md](systems/social-relationship.md) |
| 15 | Conduct | Cognition | [cognition-conduct.md](systems/cognition-conduct.md) |
| 16 | Ideology | Social | [social-ideology.md](systems/social-ideology.md) |
| 17 | Companion | Social | [social-companion.md](systems/social-companion.md) |
| 18 | Discovery | Present | [present-discovery.md](systems/present-discovery.md) |
| 19 | Voice | Story | [story-voice.md](systems/story-voice.md) |
| 20 | Roll | Cognition | [cognition-roll.md](systems/cognition-roll.md) |
| 21 | Economy | Sim | [sim-economy.md](systems/sim-economy.md) |
| 22 | Faction | Social | [social-faction.md](systems/social-faction.md) |
| 23 | Thread | Ledger | [ledger-thread.md](systems/ledger-thread.md) |
| 24 | Persistence | Sim | [sim-persistence.md](systems/sim-persistence.md) |
| 25 | Exploration | Present | [present-exploration.md](systems/present-exploration.md) |
| 26 | Gate | Rules | [rules-gate.md](systems/rules-gate.md) |
| 27 | Fail Forward | Story | [story-fail-forward.md](systems/story-fail-forward.md) |
| 28 | Fact | Sim | [sim-fact.md](systems/sim-fact.md) |
| 29 | Info Flow | Sim | [sim-info-flow.md](systems/sim-info-flow.md) |
| 30 | Rule Engine | Rules | [rules-engine.md](systems/rules-engine.md) |
| 31 | Content Store | Content | [content-store.md](systems/content-store.md) |
| 32 | Script | Content | [content-script.md](systems/content-script.md) |
| 33 | Pipeline | Content | [content-pipeline.md](systems/content-pipeline.md) |
| 34 | Locale | Content | [content-locale.md](systems/content-locale.md) |
| 35 | UI | Present | [present-ui.md](systems/present-ui.md) |
| 36 | Audio | Present | [present-audio.md](systems/present-audio.md) |
| 37 | Emotion | Cognition | [cognition-emotion.md](systems/cognition-emotion.md) |
| 38 | Pacing | Story | [story-pacing.md](systems/story-pacing.md) |
| 39 | Outcome | Story | [story-outcome.md](systems/story-outcome.md) |
| 40 | Kernel | Runtime | [runtime-kernel.md](runtime-kernel.md) |

---

## Responsibility boundaries

**NSF:** narrative simulation mechanics, `I*Service` contracts, data shapes, rule evaluation, gating, pacing, roll resolution, registry/events/persistence infrastructure, presenter architecture.

**Game:** content (characters, dialogue, beliefs, threads), presentation skin, and **game-owned modules** (combat, crafting, genre-specific loops) integrated via [architecture/game-extensions.md](architecture/game-extensions.md).

See [README.md](README.md) for the three-layer model and Unity package layout.
