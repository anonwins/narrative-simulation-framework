# NSF Decisions Log

**Single source of truth** for product and architecture choices.

- Specs and architecture docs link here — do not re-open settled forks inline.
- Module architecture **Deferred decisions** sections point here; all locked.
- New decisions append a dated row; superseded rows move to § Superseded.

Last updated: 2026-06-19 (composable library model — NSF-H09, game-extensions)

---

## Product completeness principle (locked)

**NSF must not ship behavior below what [`systems/`](../systems/) specs describe.**

| Label | Meaning |
|---|---|
| **`systems/` specs** | Minimum product capability — what NSF promises |
| **Roadmap phases** | **Implementation order**, not permission to omit spec behavior |
| **`[FULL]` in architecture** | **Phasing note only** — must be implemented by the phase that owns that module unless explicitly **Out of scope** in this log |

Temporary “simplified MVP” implementations that contradict `systems/` are **not allowed**. Phasing means *which phase writes the code*, not *which features exist*.

---

## How to use this log

| Status | Meaning |
|---|---|
| **Locked** | Implement as written |
| **Locked (expert)** | User delegated; rationale recorded |
| **Open** | **None for NSF framework** |

When module architecture “Deferred decisions” conflicts with this log, **this log wins**.

---

## Platform & host

| ID | Decision | Status | Notes |
|---|---|---|---|
| P-01 | **No multiplayer** in NSF v1 | Locked | Single-player, main-thread sim |
| P-02 | **No modding API** in NSF v1 | Locked | Content packs + sibling sample repos |
| P-03 | **Addressables from Phase 2** | Locked | Label `nsf_content_{packId}` |
| P-04 | Composition root | Locked | `NsfSession` in `Runtime/Host/` + sample `NsfGameHost` |
| P-05 | Tick driver | Locked | `EventDriven` player; `Manual` Edit Mode tests |
| P-06 | Sample dimensions | Locked | Noir **3D**, SciFi **2D** |
| P-07 | Sample repos | Locked | Sibling repos `NoirSample/`, `SciFiSample/` |
| P-08 | CI scope | Locked | Edit Mode tests through Phase 17 |
| P-09 | Save model | Locked | Autosave + continue |
| P-10 | Persistence JSON | Locked | Newtonsoft; canonical = no whitespace |
| P-11 | Sample UI | Locked | Noir **uGUI**; SciFi **UITK** |
| P-12 | Root asmdef aggregator | Locked (expert) | **No** — per-module asmdefs only (UPM best practice; explicit dependency graph) |

---

## Events & kernel

| ID | Decision | Status | Notes |
|---|---|---|---|
| EVT-01 | Event bus shape | Locked | **Typed `SimEventBase` hierarchy from Phase 1** — strongly typed payloads per event class; `[SimEventType("id")]` attribute for script/logging; **no stringly dict-primary API** |
| EVT-02 | Re-entrancy | Locked | Throw if `Publish` during `Flush` |
| EVT-03 | Deferred queue | Locked | **Phase 1** — sync immediate + deferred queue per `systems/sim-event.md`; kernel flushes deferred batch in **Events** phase |
| EVT-04 | Cross-thread | Locked | Main thread only (P-01) |
| EVT-05 | Event ordering | Locked | Priority-ordered handlers; deterministic order: action → sim → narrative → chronicle → presentation → save flush |
| EVT-06 | Typed migration | Locked | Phase 17 adds optional Roslyn analyzers / codegen — **not** a second event system |

---

## Audio, voice & TTS

| ID | Decision | Status | Notes |
|---|---|---|---|
| A-01 | TTS out of the box | Locked | `ITextToSpeechProvider` + `IAudioNarrativeService` in Presentation |
| A-02 | Story vs Presentation | Locked | Story = text/keys; Presentation = TTS, clips, queue, spatial resolve |
| A-03 | Spatial audio | Locked | Abstract anchors in core; samples wire 3D/2D |
| A-04 | Voice interrupt | Locked | Finish current, then play next (queue) |
| A-05 | Multi-channel voice | Locked (batch) | Queue policy; stack rules per `story-voice` spec Phase 11 |
| A-06 | Audio middleware | Locked | Unity native + TTS; FMOD optional plugin Phase 17+ if needed |
| A-07 | Default TTS voices (NSF package) | Locked | **`ITextToSpeechProvider`** ships with OS-backed synth; bundled **`VoiceProfileDefinition`** assets `voice_neutral_en_a`, `voice_neutral_en_b`; content **`VoiceDefinition.VoiceProfileId`** selects profile |

---

## Debug & logging

| ID | Decision | Status | Notes |
|---|---|---|---|
| DBG-01 | Editor logging | Locked | **`IDebugTraceSink` → Unity Console** at **Warning + Error** minimum; Info available via `NSF_DEBUG` |
| DBG-02 | Production logging | Locked (expert) | **Optional rolling file log** (`NsfSessionConfig.EnableFileLog`); no in-game debug overlay in player builds |
| DBG-03 | Inspectors | Locked (batch) | Editor read-only service inspectors; **no live mutation** |
| DBG-04 | Causal tooling | Locked (batch) | Kernel step-one-phase + causal trace **Editor only** Phase 14 |

---

## NSF framework standards (locked — not deferred)

Policy and creative-adjacent choices that belong to the **NSF package**, not sample game polish.

| ID | Decision | Notes |
|---|---|---|
| NSF-H01 | **Accessibility hooks** | `IUIShell` + presenters expose **`TextScaleMultiplier`** (0.875–1.5, default 1.0); `UISkinDefinition` includes **HighContrast** palette tokens; colorblind-safe defaults for bundled test skins; **input remapping** = sample/game Presentation, not sim core |
| NSF-H02 | **FrameworkTestPack prose** | Functional template strings only; tests assert **IDs and state**, never literary quality |
| NSF-H03 | **Agent sample narrative** | Sibling repos ship **beat-complete functional** scripts at Phase 15–16 (clear cause/effect, not literary award prose) |
| NSF-H04 | **NSF v1.0 quality gate** | **Automated Edit Mode + integration tests only**; manual playthrough is **sample repo** release checklist, not NSF package blocker |
| NSF-H05 | **NSF distribution** | GitHub UPM package + semver tags; **MIT LICENSE**; CHANGELOG/CONTRIBUTING agent-maintained Phase 17 |
| NSF-H06 | **NSF package art** | **Zero character art in package** — primitives/placeholder materials OK in sample wiring docs; commercial art lives in sample repos |
| NSF-H07 | **Voice casting** | Not an NSF fork — content `VoiceDefinition` + `VoiceProfileId`; TTS profiles sufficient for framework proof (A-07) |
| NSF-H08 | **Localization tooling** | `ILocaleService` + JSON/string tables + pipeline validation; external CAT tool integration optional post-1.0, not a blocker |
| NSF-H09 | **Composable library model** | Games compose NSF modules + game-owned modules; `NsfSession` is a convenience preset, not the only entry point; game features integrate via [game-extensions.md](architecture/game-extensions.md) (registry, `IStatefulService`, events, adapters) — NSF does not cap game mechanics it does not ship |

---

## Game-scoped polish (not NSF deferrals)

These are **sample repo / commercial release** milestones. They do **not** block NSF framework completeness or leave NSF architecture forks open.

| Item | Owner | When |
|---|---|---|
| Literary dialogue rewrite | NoirSample / SciFiSample repos | After NSF 1.0 if desired |
| Commercial character art & audio | Sample repos | Sample store release |
| Manual playthrough on target hardware | Sample repos | Sample QA |
| Store pages (itch/Steam) | Sample or org | Post-1.0 |
| Legal review for commercial sample | Org | Post-1.0 |

---

## Content & script

| ID | Decision | Status | Notes |
|---|---|---|---|
| CP-01 | Pipeline validation | Locked | Hard fail |
| CP-02 | TryFix | Locked | Never auto-delete without explicit flag |
| CP-03 | Addressables | Locked | P-03 |
| SCR-01 | `.nsf` / script DSL | Locked (expert) | **Shared RuleEngine AST** for scripts **and** gates from Phase 12; **compile-only**; **multi-file modules** supported; no runtime interpreter dual path |
| SCR-02 | Cross-pack IDs | Locked (batch) | **`{packId}/{contentId}` namespace**; pipeline hard-fails collision |
| SCR-03 | Content definitions | Locked (batch) | Composition via refs; no deep SO inheritance trees |
| SCR-04 | Runtime duplicate IDs | Locked (batch) | Trust pipeline; debug overwrite logs warning |

---

## Simulation, facts & economy

| ID | Decision | Status | Notes |
|---|---|---|---|
| SIM-01 | Duplicate fact | Locked | Warn + throw dev; upsert + warning release |
| SIM-02 | `actor_player` | Locked | Optional per pack manifest |
| SIM-03 | Chronicle on load | Locked | Rebuild from authoritative state |
| SIM-04 | Scene loading | Locked | `IExplorationService` only |
| SIM-05 | Fact audit trail | Locked | **Full audit in save envelope** (register/revoke timestamps + actor/source) |
| SIM-06 | Fact predicates | Locked (batch) | Validated via content ontology Phase 12 pipeline |
| ECO-01 | Economy scope | Locked | **Full `systems/sim-economy.md` capability** — currency, vendors, pricing, debt/rent pressure, pawn, info market, poverty state, etc.; phased by roadmap, **not simplified** |
| SIM-07 | Location graph | Locked (batch) | Edges bidirectional by default; one hop per `Travel` call |
| SIM-08 | Time service | Locked (batch) | No real-time clock; pause via story/pacing flags |
| SIM-09 | Actor memory | Locked (batch) | Ring buffer by importance per `sim-actor` spec |
| SIM-10 | NPC schedules | Locked (batch) | `ITimeService` triggers schedules |
| SIM-11 | Info-flow forget | Locked (batch) | Integrated with belief forget Phase 8 |

---

## Cognition & social

| ID | Decision | Status | Notes |
|---|---|---|---|
| C-01 | Conduct scoring | Locked | Explicit `RecordAction`; auto event map Phase 11+ per conduct spec |
| FAC-01 | Faculty groups | Locked (expert) | Passive checks use **SUM** of member modified values; **throw** missing faculty in dev; **log + treat as 0** in release |
| FAC-02 | Modifier stacking | Locked (batch) | Add per source; revert on unequip |
| C-02 | Companions | Locked | One active; side-channel allowed |
| C-04 | Factions | Locked | Multi-faction tags; standing decay time-based when time service active |
| C-05 | Beliefs | Locked | No hard slot cap |
| C-06 | Relationship UI | Locked | Engine hidden; Noir labels / SciFi numbers |
| SOC-01 | Relationship metrics | Locked (batch) | Float API; content integers; chapter reset via script |
| SOC-02 | Ideology | Locked (batch) | Clamp at min; time-based decay; Presentation screen Phase 13 |

---

## Story, dialogue & pacing

| ID | Decision | Status | Notes |
|---|---|---|---|
| ST-01 | Dialogue revisit | Locked | `DialogueVisitPolicy` per node |
| ST-02 | Concurrent dialogue | Locked | Primary + companion side-channel |
| ST-03 | Pacing config | Locked | Bootstrap default + per-beat content |
| ST-04 | Pacing default | Locked | `Mixed` policy |
| ST-05 | Rules engine | Locked (batch) | Custom condition plugins Phase 12; `RuleContext` snapshot Phase 14 perf pass |
| ST-06 | Gates | Locked (batch) | Opt-in content refs; denial uses hint field |

---

## Presentation (batch defaults)

| ID | Decision | Notes |
|---|---|---|
| PR-01 | Exploration exit | Default hub; content override per location |
| PR-02 | Discovery scope | Global discoverable registry with per-location visibility |
| PR-03 | Interaction history | Persist in save envelope Phase 10 |
| PR-04 | Roll UI | Separate presenter interface Phase 14 |
| PR-05 | Canvas presenters | Sample repos only; headless in Tests |
| PR-06 | Locale settings | Player locale in Presentation settings Phase 13 |

---

## Samples & showcase

| ID | Decision | Status |
|---|---|---|
| S-01 | Two games, full coverage matrix | Locked |
| S-02 | Relationship UI split | Locked |
| S-03 | Locale en + Noir fr | Locked |
| S-04 | Missing key → English | Locked |
| S-05 | TTS both samples | Locked |

---

## Still open

**None.** All NSF framework decisions are locked in this log.

Game-repo polish → § Game-scoped polish (not architecture forks).

---

## Superseded

| Was | Superseded by |
|---|---|
| Direct SO MVP loading | P-03 |
| `[FULL]` = optional features | Product completeness principle |
| Open O-01 asmdef aggregator | P-12 no root aggregator |
| Revoke-only fact history | SIM-05 audit in save |
| Minimal economy MVP | ECO-01 full sim-economy spec |
| Simple line DSL defer AST | SCR-01 shared AST Phase 12 |
| Stringly sync events (EVT-01 batch 2) | EVT-01 typed + EVT-03 deferred queue Phase 1 |
| O-02 open creative deferral | Split: NSF-H01–H08 locked; game polish § Game-scoped |
| TTS out of core | A-01 |

---

## Related documents

- [architecture/index.md](architecture/index.md)
- [development-roadmap.md](development-roadmap.md)
- [systems/](../systems/) — minimum product capability
