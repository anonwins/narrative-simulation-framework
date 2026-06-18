# Appendix: Detective Noir Content Pack Mapping

**Reference content pack only.** Maps a detective-noir sample game to NSF API terms. None of these legacy or instance names are framework requirements.

Definitions: [`terminology-glossary.md`](terminology-glossary.md)

---

## NSF API term → detective-noir instance

Framework docs use abstract IDs and API names (left column). A detective-noir content pack might implement them as (right column):

| NSF API term | Detective-noir instance (example) |
|---|---|
| `conduct_humble` | "Sorry Cop" / self-deprecating investigator |
| `conduct_bold` | "Superstar Cop" / boastful investigator |
| `conduct_reckless` | "Apocalypse Cop" / catastrophic thinker |
| `conduct_negligent` | "Hobocop" / disheveled scavenger |
| `conduct_neutral` | "Boring Cop" / by-the-book professional |
| `faction_guild` | Dock workers' union |
| `faction_authority` | City security force / RCM |
| `faction_enterprise` | Industrial corporation |
| `actor_companion` | Partner lieutenant character |
| `metric_trust_companion` | Partner trust metric |
| `faculty_environment` | Ambient/world-feel faculty (Shivers) |
| `faculty_intuition` | Empathic/symbolic faculty (Empathy) |
| `faculty_instinct` | Fight-or-flight faculty (Physical Instrument) |
| `belief_doctrine_a` | Collectivist economic belief |
| `belief_doctrine_b` | Market-liberal belief |
| `incident_primary` | Murder / hanged-man case |
| `thread_main` | Main homicide investigation |
| `section_primary` | Primary chronicle section for homicide |
| `region_primary` | City-state setting |
| Player character | Amnesiac investigator protagonist |

---

## Legacy commercial game → NSF term

| Legacy (reference) | NSF framework term |
|---|---|
| Disco Elysium | Narrative simulation RPG / NSF |
| Thought Cabinet | Belief / `IBeliefService` |
| Thought(s) | Belief(s) |
| Copotype | Conduct |
| White check | Repeatable roll |
| Red check | Gated roll |
| Case (monolithic) | Thread + Chronicle section |
| Journal | Chronicle |

---

## Implementation name bridge (Greywater Wharf)

Greywater code may lag NSF glossary naming. Bridge during migration:

| Greywater code (may lag) | NSF API term |
|---|---|
| `ThoughtCabinet` / `ThoughtResearchService` | `BeliefService` / `IBeliefService` |
| `ThoughtCabinetUI` | `BeliefView` |
| `SkillDefinition` | `FacultyDefinition` |
| `ISkillCheckService` | `IRollService` |
| `IdeologyService` | `IIdeologyService` |
| `PartnerService` | `ICompanionService` |
| `KimTrust` flag | `metric_trust_companion` |
| `.gwdlg` graphs | Content-pack dialogue assets |

---

## When to use this appendix

- Porting content from a detective-noir setting
- Reading older notes with legacy terminology
- Aligning Greywater implementation with NSF docs

Do **not** cite this appendix in framework requirement statements.
