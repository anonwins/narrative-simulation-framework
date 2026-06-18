# 5. Ledger: Chronicle — NSF Specification

> **Framework:** Sections, tasks, leads, clues, narrative state visualization—not a quest log.
> **Content pack:** Chronicle entries, section titles, political/personal thread content, UI labels.
> Terminology: [Glossary](../terminology-glossary.md)

The Chronicle in *NSF* is **not a quest log**.

A conventional RPG chronicle answers:

```text
What should the player do next?
```

A NSF-style chronicle answers:

```text
What is currently happening in the player character's life?
```

The chronicle is simultaneously:

* Task tracker
* Investigation board
* Character diary
* Political tracker
* Belief history
* Narrative state visualization

If you implement it as a standard MMORPG quest log, the game will not achieve NSF design goals.

---

# Core Architecture

```text
Chronicle
 ├─ Sections
 ├─ Tasks
 ├─ People
 ├─ Locations
 ├─ Clues
 ├─ PoliticalHistory
 ├─ PersonalHistory
 ├─ Statistics
 └─ CompletedEntries
```

The chronicle acts as a read-only projection of game state.

It should not be the source of truth.

The source of truth lives in:

```text
Story State
Dialogue
Belief
World State
```

The chronicle merely visualizes it.

---

# Fundamental Design Rule

Do NOT model narrative as objective chains:

```text
Story beat (wrong shape)
 ├─ Objective 1
 ├─ Objective 2
 └─ Objective 3
```

Model them as:

```text
Investigation
 ├─ Leads
 ├─ Clues
 ├─ Questions
 ├─ ThreadSubjects
 ├─ Theories
 └─ Resolutions
```

This distinction is critical.

NSF feels like inquiry work because information is structured as knowledge rather than objectives.

---

# Chronicle Entry Types

The engine should support:

```text
Main section
Side section
Personal Goal
Thread
Character Thread
Ideology thread
Temporary Lead
Completed Lead
```

Not every chronicle entry should be a checklist objective.

---

# Task System

Tasks are the closest thing to traditional quests.

Architecture:

```text
Task
 ├─ id
 ├─ title
 ├─ description
 ├─ state
 ├─ visibility
 ├─ triggers
 └─ notes
```

State:

```text
Hidden
Active
Completed
Failed
Archived
```

---

# Multi-State Tasks

A task should never be:

```text
Incomplete
Complete
```

Instead:

```text
Task State Machine
```

Example:

```text
Find the Victim's Identity

Active
↓
Discovered Lead
↓
Found Evidence
↓
Identified Victim
↓
Section updated
```

Chronicle reflects current stage.

---

# Soft Objectives

Many NSF tasks intentionally avoid direct instructions.

Bad:

```text
Go to warehouse.
```

Good:

```text
There may be something worth investigating
near the harbor.
```

The chronicle often records suspicions.

Not orders.

Architecture:

```json
{
  "objectiveType": "hint"
}
```

versus

```json
{
  "objectiveType": "explicit"
}
```

---

# Thread

The central thread should be represented as:

```text
Thread
 ├─ Victim
 ├─ ThreadSubjects
 ├─ Witnesses
 ├─ Evidence
 ├─ Theories
 ├─ Open Questions
 └─ Conclusions
```

Not:

```text
Story beat chain
```

This enables inquiry gameplay.

---

# Clue System

Every clue should be an object.

```text
Clue
 ├─ id
 ├─ title
 ├─ description
 ├─ source
 ├─ confidence
 └─ tags
```

Example:

```json
{
  "title": "Boot Prints",
  "source": "crime_scene",
  "confidence": 0.7
}
```

This allows later systems to reason about evidence.

---

# Evidence References

Chronicle entries should reference evidence.

Example:

```text
Victim may have been moved.

Evidence:
- Drag marks
- Tire tracks
```

Architecture:

```text
Task
 └─ RelatedEvidence[]
```

---

# Lead System

A lead is not evidence.

Lead:

```text
Someone worth talking to.
```

Evidence:

```text
Something known.
```

Architecture:

```text
Lead
 ├─ target
 ├─ status
 └─ source
```

Example:

```text
Talk to the union boss.
```

---

# Open Questions System

One of the most important missing features in most inquiry RPGs.

Chronicle should track:

```text
Unanswered Questions
```

Example:

```text
Who killed the victim?
```

```text
Why was the body moved?
```

```text
Where did the shot come from?
```

Architecture:

```text
Question
 ├─ text
 ├─ answered
 ├─ answer
 └─ supportingEvidence[]
```

This creates a genuine investigation structure.

---

# Character Chronicle Entries

NSF frequently tracks personal matters.

Examples:

```text
Find a place to sleep.
Pay your debt.
Recover your gun.
Recover your badge.
```

These are not investigations.

They are life problems.

Architecture:

```text
PersonalThread
 ├─ title
 ├─ state
 └─ updates[]
```

---

# Dynamic Update System

Chronicle entries evolve.

Bad:

```text
Static objective-list description.
```

Good:

```text
Description rewritten after discoveries.
```

Example:

```text
Initial:
Find the missing gun.
```

Later:

```text
A witness claims they saw someone
carrying your gun near the fishing village.
```

The same task changes.

---

# Update Log Structure

Every task should maintain history.

```text
Task
 ├─ CurrentDescription
 └─ UpdateHistory[]
```

Example:

```text
Update #1
Body discovered.

Update #2
Victim identified.

Update #3
Possible inquiry subject identified.
```

Players can reconstruct events.

---

# Chronicle Writing Style

Critical requirement.

Chronicle text should be:

```text
Subjective
Character-driven
Narrative
```

Not:

```text
Collect 5 clues.
```

Example:

```text
You still haven't found your gun.
This should concern you more than it does.
```

The chronicle has personality.

---

# Faculty-aware Chronicle Updates

Unique NSF feature.

Chronicle updates may depend on faculties.

Example:

High Logic:

```text
The timeline appears inconsistent.
```

High Empathy:

```text
The witness seemed frightened.
```

Architecture:

```text
ChronicleUpdate
 ├─ text
 └─ conditions
```

---

# Belief Integration

Chronicle should reference beliefs.

Example:

```text
New lead unlocked by thought:
Advanced Race Theory
```

Architecture:

```text
Entry
 └─ RelatedBeliefs[]
```

Beliefs can create tasks.

Tasks can create beliefs.

Bidirectional relationship.

---

# Ideology tracking

NSF quietly tracks ideology.

Architecture:

```text
PoliticalProfile
 ├─ communist
 ├─ moralist
 ├─ ultraliberal
 └─ fascist
```

Chronicle can expose:

```text
Ideology discoveries
Ideology milestones
Ideology-related chronicle entries
```

---

# Character Identity Tracking

Chronicle should track:

```text
Who the player character is becoming.
```

Example:

```text
conduct_reckless
conduct_humble
conduct_bold
conduct_neutral
```

Architecture:

```text
PersonaTracker
 ├─ conduct
 ├─ score
 └─ milestones
```

These emerge from player behavior.

---

# Relationship Tracking

Optional but highly recommended.

```text
Relationship
 ├─ Actor
 ├─ affinity
 ├─ trust
 └─ history
```

Chronicle can summarize:

```text
actor_companion increasingly trusts you.
```

---

# Chronicle board Architecture

For a true NSF-style engine:

Add an inquiry board.

```text
Chronicle board
 ├─ Clues
 ├─ People
 ├─ Locations
 ├─ Questions
 └─ Connections
```

This is not required for NSF accuracy.

But it naturally emerges from the chronicle data.

---

# Automatic Entry Generation

Chronicle entries should be event-driven.

Never:

```text
Story state manually updates chronicle.
```

Instead:

```text
World Event
↓
Chronicle Event
↓
Update Entry
```

Architecture:

```text
EventBus
 └─ ChronicleSubscriber
```

Example:

```text
WitnessInterviewed
```

Automatically updates:

```text
Relevant task
Relevant clue
Relevant inquiry subject
```

---

# Failure Recording

Extremely important.

The chronicle should record failures.

Example:

```text
You failed to convince the witness.
```

Not:

```text
Failure disappears.
```

Failure is story.

---

# Multiple Resolution States

Tasks should support:

```text
Solved
Partially Solved
Abandoned
Failed
Resolved Differently
```

NSF frequently resolves problems in unexpected ways.

---

# Save Data Requirements

Serialize:

```text
Active Tasks
Completed Tasks
Failed Tasks
Clues
Leads
Questions
Evidence
Ideology state
Persona State
Update History
Relationship State
```

Everything should survive reload.

---

# Minimal Engine Interfaces

```csharp
interface IChronicleServiceEntry
{
    string Id;
    string Title;
    EntryState State;
}

interface ITask
{
    List<ChronicleUpdate> Updates;
}

interface IChronicleServiceSection
{
    List<Lead> Leads;
    List<Clue> Clues;
    List<Question> Questions;
}

interface IChronicleService
{
    List<IChronicleEntry> Entries;
}
```

---

# Most Important Insight

A NSF-style chronicle is best understood as:

```text
Chronicle
```

rather than:

```text
Chronicle
```

The chronicle's purpose is not merely to tell the player what to do.

Its purpose is to maintain a living record of:

* Investigations
* Relationships
* Failures
* Discoveries
* Ideological development
* Personal crises
* Character transformation

The player should be able to open the chronicle and answer:

```text
What do I know?
Who is under inquiry?
What have I failed at?
Who am I becoming?
What mysteries remain unsolved?
```

If the chronicle can answer those five questions, it will feel much closer to *NSF* than a traditional RPG quest log ever could.
