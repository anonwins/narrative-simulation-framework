# 7. Cognition: Belief — NSF Specification

> **Framework:** Belief acquisition, research/internalization timers, slots, stat modifiers, exclusion groups, ideology hooks.
> **Content pack:** Belief definitions, internalization durations, UI skin label for the belief screen.
> Terminology: [Glossary](../terminology-glossary.md)

The Belief is a core NSF progression system.

Most RPGs use:

```text
Perks
Traits
Feats
Talents
Passive Trees
```

NSF replaces all of these with:

```text
Beliefs
```
A Belief is not a perk.
A Belief is:

```text
An idea the player character becomes obsessed with.
```

The Belief is simultaneously:

* Character progression
* Build system
* Ideology system
* Personality system
* Narrative unlock system
* Worldview system

If Faculties answer:

```text
How does the player character perceive reality?
```

Beliefs answer:

```text
What does the player character believe reality is?
```

---

# Core Principle

Traditional RPG:

```text
Level Up
↓
Choose Perk
↓
Gain Bonus
```

NSF:

```text
Experience Event
↓
Acquire Idea
↓
Obsess Over Idea
↓
Internalize Idea
↓
Character Changes
```

Progression is framed as psychological development.

---

# Core Architecture

```text
BeliefService
 ├─ AvailableBeliefs
 ├─ AssimilatingBeliefs
 ├─ ResolvedBeliefs
 ├─ Beliefslots
 └─ BeliefHistory
```

---

# Belief Lifecycle

Every belief passes through states.

```text
Discovered
↓
Available
↓
Assimilating
↓
Resolved
```

Architecture:

```csharp
enum BeliefState
{
    Locked,
    Available,
    Assimilating,
    Resolved
}
```

This state machine is the heart of the system.

---

# Belief Definition

Every Belief is data-driven.

```json
{
  "id": "advanced_race_theory",
  "title": "Advanced Race Theory",

  "internalizationTime": 7200,

  "temporaryEffects": [],

  "finalEffects": []
}
```

Never hardcode behavior.

---

# Belief acquisition

Beliefs are acquired through:

```text
Dialogue
Choices
Repeated Behaviors
Faculty Events
Story events
Discoveries
Ideological statements
```

The player usually does not directly choose them.

The world offers them.

---

# Acquisition Example

Player repeatedly says communist things.

Engine tracks:

```text
Communist Score +1
Communist Score +1
Communist Score +1
Communist Score +1
```

Threshold reached:

```text
Unlock Belief:
belief_doctrine_a
```

Architecture:

```json
{
  "unlockCondition":
  {
      "communistScore": 4
  }
}
```

---

# Belief slots

The cabinet has limited capacity.

Architecture:

```text
Belief

[ ]
[ ]
[ ]
[ ]
```

Each slot contains:

```text
0 or 1 thought
```

This creates meaningful build choices.

---

# Slot Expansion

New slots can be unlocked.

Architecture:

```text
Belief slots
```

becomes

```text
Belief slots + 1
```

Belief capacity is itself progression.

---

# Internalization

The most important mechanic.

Player chooses:

```text
Internalize Belief
```

Belief enters:

```text
Assimilating
```

state.

Not yet active.

---

# Internalization Timer

Every belief has a duration.

Example:

```json
{
  "internalizationTime": 10800
}
```

Measured in game time.

Not real time.

---

# Time Progression

Internalization advances when:

```text
Time passes
```

Examples:

```text
Reading
Talking
Waiting
Traveling
```

No progress if time does not move.

---

# Temporary Effects

Beliefs often produce side effects during internalization.

Example:

```text
Assimilating

-1 Logic
+1 Electrochemistry
```

Architecture:

```json
{
  "temporaryEffects":
  [
    {
      "faculty":"logic",
      "modifier":-1
    }
  ]
}
```

These effects disappear when finished.

---

# Completion Event

When timer completes:

```text
Assimilating
↓
Resolved
```

Trigger:

```text
Popup
Chronicle Entry
Belief Completion Event
```

---

# Permanent Effects

Completed Beliefs grant effects.

Example:

```text
+2 Rhetoric
```

or

```text
Unlock Dialogue
```

or

```text
Gain Money From Books
```

Effects are extremely varied.

---

# Belief Effects Are Not Just Stat Bonuses

Traditional perk:

```text
+5% Damage
```

NSF belief:

```text
Unlock new worldview.
```

Engine must support more than modifiers.

---

# Supported Effect Categories

Beliefs may modify:

```text
Attributes
Faculties
Faculty Caps
Dialogue
Checks
Economy
Relationships
Story state logic
Interactions
Belief slots
Perception Rules
```

Everything.

---

# Modifier Effects

Simple example:

```json
{
  "effects":
  [
    {
      "faculty":"rhetoric",
      "modifier":2
    }
  ]
}
```

---

# Faculty Cap Effects

Example:

```json
{
  "effects":
  [
    {
      "faculty":"authority",
      "capModifier":1
    }
  ]
}
```

A common NSF pattern.

---

# Unlock Effects

Example:

```json
{
  "effects":
  [
    {
      "unlockDialogueTag":"communist"
    }
  ]
}
```

New dialogue appears.

---

# Economy Effects

Example:

```json
{
  "effects":
  [
    {
      "moneyPerBookRead":5
    }
  ]
}
```

Beliefs can create entirely new economic rules.

---

# World Rule Effects

Beliefs can alter simulation rules.

Example:

```json
{
  "effects":
  [
    {
      "containerXPBonus":1
    }
  ]
}
```

This is beyond ordinary perks.

---

# Belief As Build System

In most RPGs:

```text
Build =
Stats
```

In NSF:

```text
Build =
Stats
+
Beliefs
```

Two characters with identical faculties can play completely differently.

---

# Ideology System

One of the cabinet's major functions.

Ideology axes:

```text
Communist
Fascist
Moralist
Ultraliberal
```

Each ideology is represented largely through beliefs.

---

# Ideology progression

Example:

```text
Ideological choice
↓
Ideology score
↓
Ideology belief unlock
↓
Internalization
↓
Ideological identity
```

This feels organic.

---

# Personality Progression

Not all Beliefs are political.

Categories include:

```text
Psychological
Political
Professional
Absurd
Philosophical
Behavioral
Economic
Existential
```

---

# Conduct Integration

Repeated behaviors unlock beliefs.

Example:

```text
Apologize repeatedly
```

Triggers:

```text
conduct_humble
```

belief unlock.

Architecture:

```json
{
  "requiresBehavior":
  {
      "apologyCount":5
  }
}
```

---

# Belief Discovery Sources

Beliefs may come from:

```text
Actors
Books
Faculties
Objects
Locations
Events
Dialogue
```

Everything can inspire ideas.

---

# Belief Suggestion System

Often the game offers:

```text
You have begun considering...
```

Architecture:

```text
Belief Proposal
```

Player may:

```text
Accept
Ignore
```

---

# Mutually Exclusive Beliefs

Recommended feature.

Example:

```text
belief_doctrine_a
```

may conflict with:

```text
belief_doctrine_b
```

Architecture:

```json
{
  "conflicts":
  [
      "ultraliberalism"
  ]
}
```

---

# Belief Tags

Every belief should have tags.

Example:

```json
{
  "tags":
  [
      "political",
      "communist"
  ]
}
```

Useful for:

```text
Dialogue
Achievements
Statistics
Narrative Reactivity
```

---

# Belief-Driven Dialogue

Critical system.

Dialogue may require:

```json
{
  "requiresBelief":
  "jamais_vu"
}
```

This creates highly replayable conversations.

---

# Belief-Driven Checks

Beliefs may modify checks.

Example:

```json
{
  "checkModifier":
  {
      "authority":2
  }
}
```

---

# Belief-Driven Perception

Some beliefs reveal new content.

Example:

```text
Without Belief:
No interaction.
```

```text
With Belief:
New dialogue appears.
```

This is a major NSF pattern.

---

# Belief Events

Beliefs should be event listeners.

Architecture:

```text
Belief
 ├─ OnDialogue
 ├─ OnCheck
 ├─ OnLocationEnter
 ├─ OnItemAcquire
 └─ OnQuestUpdate
```

Beliefs become active systems.

Not passive data.

---

# Belief Voice System

Many Beliefs speak.

Architecture:

```text
Speaker
 ├─ Faculty
 ├─ Belief
 ├─ Actor
 ├─ Narrator
 └─ Item
```

Beliefs should use the dialogue engine.

Not special UI.

---

# Belief Commentary

Example:

```text
VOLITION:
Stay focused.
```

vs

```text
THOUGHT:
Maybe none of this matters.
```

Different sources.

Same infrastructure.

---

# Forgetting Beliefs

NSF allows removing beliefs.

Architecture:

```text
Resolved
↓
Forgotten
```

Requires resource expenditure.

---

# Forgetting Effects

Removes:

```text
Bonuses
Penalties
Unlocks
```

Character changes again.

---

# Chronicle Integration

Belief lifecycle should generate entries.

Example:

```text
Belief discovered
Belief internalizing
Belief completed
Belief forgotten
```

The chronicle tracks psychological development.

---

# Save Data Requirements

Serialize:

```text
Available Beliefs
Assimilating beliefs
Resolved Beliefs
Belief Timers
Belief slots
Belief history
Forgotten Beliefs
```

Everything must persist.

---

# Minimal Engine Interfaces

```csharp
interface IBelief
{
    string Id;

    BeliefState State;

    float Progress;

    List<Effect> TemporaryEffects;

    List<Effect> FinalEffects;
}

interface IBeliefService
{
    List<IBelief> Available;

    List<IBelief> Assimilating;

    List<IBelief> Resolved;
}
```

---

# Relationship To Other Systems

Belief interacts with:

```text
Dialogue
Faculty Service
Character System
Chronicle
Story State
Economy
Ideology
Conduct
```

It is a central hub.

---

# Most Important Insight

The Belief is **not a perk tree**.

The correct mental model is:

```text
Psychological operating layer
```

Faculties determine:

```text
How the player character thinks.
```

Beliefs determine:

```text
What the player character thinks about.
```

A true NSF-style implementation should make the player feel that they are not selecting upgrades.

They are allowing ideas to take root in the player character's mind and permanently reshape how they understand the world.

That distinction is what makes the Belief distinct from traditional perk trees.
