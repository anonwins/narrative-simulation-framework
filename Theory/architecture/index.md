# NSF Architecture Index

**Implementation blueprints** for the Narrative Simulation Framework — written *before* C# code, derived from Phase A specs.

- **Behavior & intent:** [systems/](../systems/) specs
- **Terms & API names:** [terminology-glossary.md](../terminology-glossary.md) (SSOT)
- **Build order:** [development-roadmap.md](../development-roadmap.md)
- **Loop model:** [runtime-kernel.md](../runtime-kernel.md) (spec)

Start here → [contracts.md](contracts.md) · [foundation.md](foundation.md) · [unity-host.md](unity-host.md) · [data-model.md](data-model.md)

---

## Why this folder exists

Specs describe *what* NSF must do and *why* it matters narratively. Architecture docs describe *how* we implement it in Unity/C# without ambiguity:

- Exact module boundaries and forbidden dependencies
- Concrete types, file paths, and algorithms
- Event and persistence contracts shared across modules
- MVP vs full scope aligned to roadmap phases
- Test plans that prove correctness without human playtesting until Phase 18

**Principle:** One decision, one home. Signatures live in [contracts.md](contracts.md). Shared data shapes live in [data-model.md](data-model.md). Package layout lives in [foundation.md](foundation.md). Unity host lifecycle lives in [unity-host.md](unity-host.md). System docs explain module-specific *why* and *how*, and link upward — they do not redefine SSOT.

---

## SSOT hierarchy

1. [terminology-glossary.md](../terminology-glossary.md)
2. [decisions-log.md](../decisions-log.md) — locked product/architecture choices
3. [development-roadmap.md](../development-roadmap.md)
3. [runtime-kernel.md](../runtime-kernel.md) (spec)
4. [systems/*.md](../systems/)
5. **architecture/** (this folder — must not contradict 1–5)

---

## Document template

Every architecture file follows:

| Section | Purpose |
|---|---|
| Header | Links to spec, glossary, roadmap phase, contracts |
| Design rationale | **Why** this module exists; problems it solves |
| Implementation summary | MVP vs full in one paragraph |
| Assembly & namespace | asmdef, references, forbidden couplings |
| Public API | Link to [contracts.md](contracts.md); module-specific notes only |
| Internal implementation | Classes, registries, orchestrators |
| Definition assets | ScriptableObject schemas |
| Runtime state | Serializable DTOs; persistence ownership |
| Core algorithms | Step-by-step flows; deterministic behavior |
| Event contracts | Typed `SimEventBase` hierarchy; kernel tick phase |
| Integration matrix | Depends on / consumed by / forbidden |
| MVP scope | Roadmap phase deliverables |
| Full scope | `[FULL]` stretch items |
| File tree | `Packages/NarrativeFramework/...` |
| Test plan | Edit Mode tests, seeds, exit criteria |
| Deferred decisions | **Locked** — [decisions-log.md](../decisions-log.md) |

---

## Planning vocabulary (MVP · `[FULL]` · defer · out of scope)

We **are** still in thorough planning. These labels describe **when** a decision or feature lands in the build — not whether it was thought through.

| Label | Meaning | Planning status |
|---|---|---|
| **MVP scope** | Must ship for the roadmap phase listed in that doc. Fully specified in architecture + specs. | **Decided** — implement in that phase |
| **`[FULL]`** | Same module, **later implementation phase** — **not** a scope cut. Must still match [`systems/`](../systems/) by the owning phase. | **Phased delivery** — see [decisions-log.md](../decisions-log.md) § Product completeness |
| **Deferred decisions** | Resolved in [decisions-log.md](../decisions-log.md). | **Closed — zero NSF deferrals** |
| **Out of scope** ([unity-host.md](unity-host.md)) | Never NSF core (multiplayer, store pages, cloud upload UI). | **Decided exclusion** |

**Product rule:** [`systems/`](../systems/) specs are the capability floor. [`decisions-log.md`](../decisions-log.md) locks **all** NSF forks. Game polish → decisions log § Game-scoped.

---

## Dependency layers (authoring order)

```text
Layer 0  index, contracts, foundation, unity-host, data-model
Layer 1  runtime-kernel, sim-event, sim-fact
Layer 2  content-store, content-pipeline, content-script, content-locale, content-inventory
Layer 3  cognition-faculty, cognition-roll
Layer 4  story-dialogue, story-state, story-fail-forward, rules-engine, rules-gate
Layer 5  sim-time, sim-location, sim-actor
Layer 6  social-*, cognition-belief, cognition-conduct, cognition-emotion
Layer 7  ledger-*, sim-info-flow, sim-economy, sim-persistence, story-voice, story-pacing, story-outcome
Layer 8  present-*, debug, editor, integration, samples
```

---

## Cross-cutting documents

| File | Phase | Status |
|---|---|---|
| [contracts.md](contracts.md) | 0 | done |
| [foundation.md](foundation.md) | 0 | done |
| [data-model.md](data-model.md) | 0–2 | done |
| [runtime-kernel.md](runtime-kernel.md) | 1 | done |
| [unity-host.md](unity-host.md) | 0–14 | done |
| [debug.md](debug.md) | 14 | done |
| [editor.md](editor.md) | 2, 14 | done |
| [integration.md](integration.md) | 14 | done |
| [samples.md](samples.md) | 2, 15–16 | done |

---

## System architecture documents (37)

| File | Module | Phase | Status |
|---|---|---|---|
| [cognition-belief.md](cognition-belief.md) | Cognition | 8 | done |
| [cognition-conduct.md](cognition-conduct.md) | Cognition | 8 | done |
| [cognition-emotion.md](cognition-emotion.md) | Cognition | 8 | done |
| [cognition-faculty.md](cognition-faculty.md) | Cognition | 3 | done |
| [cognition-roll.md](cognition-roll.md) | Cognition | 3 | done |
| [content-inventory.md](content-inventory.md) | Content | 12 | done |
| [content-locale.md](content-locale.md) | Content | 12 | done |
| [content-pipeline.md](content-pipeline.md) | Content | 2 | done |
| [content-script.md](content-script.md) | Content | 12 | done |
| [content-store.md](content-store.md) | Content | 2 | done |
| [ledger-chronicle.md](ledger-chronicle.md) | Ledger | 9 | done |
| [ledger-thread.md](ledger-thread.md) | Ledger | 9 | done |
| [present-audio.md](present-audio.md) | Presentation | 13 | done |
| [present-discovery.md](present-discovery.md) | Presentation | 13 | done |
| [present-exploration.md](present-exploration.md) | Presentation | 13 | done |
| [present-interaction.md](present-interaction.md) | Presentation | 13 | done |
| [present-ui.md](present-ui.md) | Presentation | 13 | done |
| [rules-engine.md](rules-engine.md) | Rules | 5 | done |
| [rules-gate.md](rules-gate.md) | Rules | 5 | done |
| [sim-actor.md](sim-actor.md) | Simulation | 6 | done |
| [sim-economy.md](sim-economy.md) | Simulation | 10 | done |
| [sim-event.md](sim-event.md) | Simulation | 1 | done |
| [sim-fact.md](sim-fact.md) | Simulation | 1 | done |
| [sim-info-flow.md](sim-info-flow.md) | Simulation | 10 | done |
| [sim-location.md](sim-location.md) | Simulation | 6 | done |
| [sim-persistence.md](sim-persistence.md) | Simulation | 10 | done |
| [sim-time.md](sim-time.md) | Simulation | 6 | done |
| [social-companion.md](social-companion.md) | Social | 7 | done |
| [social-faction.md](social-faction.md) | Social | 7 | done |
| [social-ideology.md](social-ideology.md) | Social | 7 | done |
| [social-relationship.md](social-relationship.md) | Social | 7 | done |
| [story-dialogue.md](story-dialogue.md) | Story | 4 | done |
| [story-fail-forward.md](story-fail-forward.md) | Story | 4, 11 | done |
| [story-outcome.md](story-outcome.md) | Story | 11 | done |
| [story-pacing.md](story-pacing.md) | Story | 11 | done |
| [story-state.md](story-state.md) | Story | 4 | done |
| [story-voice.md](story-voice.md) | Story | 11 | done |

---

## Kernel tick phases (reference)

| Phase | Primary services |
|---|---|
| `Facts` | `IFactService` |
| `Interpretation` | `IDialogueService`, `IVoiceService`, cognition |
| `Social` | `IRelationshipService`, `IFactionService`, `IIdeologyService` |
| `Events` | `IEventBus`, `IActorService`, `ITimeService`, `ILocationService` |
| `Gating` | `IGateService`, `IRuleEngine`, `IPacingService` |
| `Content` | `IContentStore`, script effects, new facts |

See [runtime-kernel.md](runtime-kernel.md) for orchestration detail.

---

## Verification checklist (Phase B complete)

- [x] 47 files exist under `architecture/`
- [x] [contracts.md](contracts.md) defines all public APIs
- [x] Every `systems/` file has matching architecture doc
- [x] Kernel tick phase assigned for every event-emitting module
- [x] No asmdef violations documented (core → Presentation)
- [x] Chronicle documented as read-only projection
- [x] Headless presenter pattern in Presentation docs
- [x] Roadmap phases 0–16 covered
- [x] [Theory/README.md](../README.md) and [index.md](../index.md) link here
