# 6. Story: Dialogue — NSF Specification

> **Framework:** Dialogue trees, faculty interruptions, conditional branches, failure branches, dialogue-as-gameplay hub.
> **Content pack:** Dialogue text, Actor voices, faculty interjection content, humor/tone.
> Terminology: [Glossary](terminology-glossary.md)

This is the single most important system in a NSF-style game.

Most RPGs treat dialogue as:

```text
Conversation System
```

NSF treats dialogue as:

```text
Primary Gameplay System
```

Everything flows through dialogue:

* Investigation
* Character progression
* Faculty expression
* Politics
* Relationships
* Humor
* Failure
* Narrative branching

If the dialogue system is wrong, the game will not feel like NSF regardless of how accurate the faculties or inventory are.

---

# Core Principle

A traditional RPG:

```text
World
 ↓
Combat
 ↓
Dialogue
```

A NSF-style RPG:

```text
World
 ↓
Dialogue
 ↓
Everything Else
```

Dialogue is the main gameplay loop.

---

# Core Architecture

```text
DialogueService
 ├─ Conversations
 ├─ Nodes
 ├─ Choices
 ├─ Conditions
 ├─ FacultyInterruptions
 ├─ Checks
 ├─ Reactions
 ├─ WorldEffects
 └─ MemoryUpdates
```

---

# Dialogue Is A Directed Graph

Not a tree.

Bad:

```text
A
├─ B
├─ C
└─ D
```

Good:

```text
A → B → E
 \→ C → E
 \→ D → F
```

Nodes should reconnect frequently.

Otherwise content explodes exponentially.

---

# Core Dialogue Node

```csharp
class DialogueNode
{
    string Id;

    Speaker Speaker;

    string Text;

    List<Choice> Choices;

    List<Condition> Conditions;

    List<Effect> Effects;
}
```

Every spoken line is a node.

---

# Speaker Types

Critical for NSF.

```text
Actor
Narrator
Faculty
Belief
Item
Companion
Player
```

Most RPGs only support:

```text
Actor
Player
```

NSF heavily relies on the others.

---

# Example Conversation

```text
actor_intimidating:
You are not of the superior race.
```

Interruptions:

```text
RHETORIC:
This is nonsense.

AUTHORITY:
Challenge him.

faculty_instinct:
Punch him.
```

Player then chooses.

This structure is fundamental.

---

# Dialogue Layer Stack

A conversation consists of multiple layers.

```text
Actor Speech
↓
Faculty Reactions
↓
Belief Reactions
↓
Companion Reactions
↓
Player Choices
```

Not:

```text
Actor
↓
Player
↓
Actor
```

---

# Faculty Interruptions

One of NSF's defining mechanics.

Faculties continuously comment.

Architecture:

```csharp
class FacultyInterruption
{
    Faculty Source;

    Trigger Trigger;

    string Text;

    Priority Priority;
}
```

---

# Trigger Types

```text
Observation
Contradiction
Threat
Emotion
Lie
Clue
Opportunity
Failure
Success
```

Example:

Actor lies.

```text
DRAMA:
He's lying.
```

---

# Interruption Conditions

Example:

```json
{
  "faculty": "drama",
  "minimum": 6
}
```

Only appears if faculty qualifies.

Different builds see different dialogue.

---

# Internal Conversation System

A major NSF feature.

The player frequently talks to:

```text
Faculties
Beliefs
Objects
Themselves
```

Architecture:

```text
Conversation
 ├─ External
 └─ Internal
```

Internal conversations use the same engine.

---

# Dialogue Visibility Rules

Choices can be:

```text
Visible
Hidden
Locked
Discovered
```

Example:

```json
{
  "requiresFaculty": {
    "authority": 8
  }
}
```

Without Authority 8:

```text
Choice hidden
```

or

```text
[Authority] Locked
```

depending on design.

---

# Dialogue Conditions

Every node can have conditions.

```text
Faculty Threshold
Belief
Story beat state
Item
Relationship
Ideology
Time
Location
Previous Choice
```

Example:

```json
{
  "requiresBelief":
  "mazovian_socioeconomics"
}
```

---

# Dialogue Effects

Every dialogue choice can modify state.

Example:

```text
Gain XP
Lose Morale
Unlock Belief
Start story beat
Advance story beat
Change Relationship
Gain Item
Lose Item
Change Ideology
```

Architecture:

```csharp
class DialogueEffect
{
    EffectType Type;

    object Value;
}
```

---

# The Roll

Dialogue checks are embedded directly into dialogue.

Example:

```text
[Authority - Medium 42%]

Force him to cooperate.
```

Checks are choices.

Not separate gameplay.

---

# Check Node Structure

```csharp
class RollChoice
{
    Faculty Faculty;

    int Difficulty;

    Outcome Success;

    Outcome Failure;
}
```

Both outcomes contain content.

Always.

---

# Failure Forward Requirement

Never:

```text
Success:
Content

Failure:
Nothing
```

Always:

```text
Success:
Content A

Failure:
Content B
```

Failure generates story.

---

# Failure As Content

Traditional RPG:

```text
Failed persuasion.
```

NSF:

```text
Failed persuasion.

Actor laughs.

Companion reacts.

New situation created.
```

Failure must move the narrative.

---

# Critical Success

Special success state.

```text
Natural 12
```

Can produce:

```text
Extra information
Additional rewards
Alternative outcomes
```

---

# Critical Failure

Special failure state.

```text
Natural 2
```

Can produce:

```text
Humiliation
Comedy
Unexpected consequences
```

Critical failures are often more memorable than successes.

---

# Choice Types

The engine should support:

```text
Question
Statement
Lie
Threat
Joke
Observation
Internal belief
Roll
Action
Leave
```

Not just generic responses.

---

# Tone Metadata

Each choice should carry tone.

```json
{
  "tone": "aggressive"
}
```

Possible values:

```text
Aggressive
Empathetic
Funny
Professional
Political
Cowardly
Confident
Absurd
```

Used for personality tracking.

---

# Conduct Integration

Every choice contributes to personality.

Example:

```text
Apologize excessively
```

Adds:

```text
conduct_humble +1
```

Architecture:

```json
{
  "conductEffects":
  {
      "conduct_humble": 1
  }
}
```

---

# Ideology Integration

Choices also contribute ideology.

Example:

```text
Workers should own production.
```

Adds:

```text
Communist +1
```

Architecture:

```json
{
  "politics":
  {
      "communist": 1
  }
}
```

---

# Relationship Effects

Choices affect Actor relationships.

Example:

```json
{
  "relationship":
  {
      "kim": 2
  }
}
```

Relationship changes should happen constantly.

Not only at major events.

---

# Actor Memory System

Actors must remember conversations.

Architecture:

```text
Actor
 ├─ KnownFacts
 ├─ Relationship
 ├─ ConversationFlags
 └─ History
```

Example:

```text
Player insulted Actor
```

Stored permanently.

Future dialogue changes.

---

# Conversation History

Every major choice should be recorded.

```text
ConversationHistory
 ├─ node_id
 ├─ choice_id
 └─ timestamp
```

Used for:

```text
Callbacks
References
Consequences
```

---

# Dialogue Re-entry

Conversations are rarely one-time.

Need support for:

```text
Previously discussed topics
New topics
Exhausted topics
```

Architecture:

```text
Topic
 ├─ Available
 ├─ Discussed
 └─ Closed
```

---

# Topic-Based Conversations

NSF often structures dialogue around topics.

Example:

```text
Ask about victim
Ask about union
Ask about harbor
Leave
```

Topics remain available until exhausted.

---

# Fact unlock

Dialogue is knowledge-driven.

New information unlocks:

```text
New topics
New choices
New theories
```

Architecture:

```text
Fact record
 └─ Unlocks
```

Example:

```text
Learn victim's name
↓
Unlock 12 new dialogue options
```

---

# Faculty-Reactive Writing

The same node can produce different reactions.

Example:

Actor speaks.

Low Empathy:

```text
No reaction.
```

High Empathy:

```text
EMPATHY:
She's terrified.
```

Same conversation.

Different experience.

---

# Multi-Faculty Reactions

Multiple faculties may react simultaneously.

Example:

```text
LOGIC:
This timeline doesn't work.

DRAMA:
He's lying.

faculty_instinct:
Danger.
```

Architecture:

```text
Node
 └─ Reactions[]
```

---

# Priority Resolution

Too many reactions become noise.

Need scoring.

```text
Priority =
Relevance
+
Faculty Value
+
Narrative Importance
```

Display highest-scoring reactions.

---

# Companion Commentary

Companions use same interruption system.

Example:

```text
KIM:
I wouldn't do that.
```

Appears before choice.

Architecture identical to faculties.

---

# Belief Integration

Beliefs may:

```text
Unlock dialogue
Modify dialogue
Create dialogue
```

Example:

```text
belief unlocked:

New conversation branch appears.
```

---

# Object Conversations

NSF often allows conversations with:

```text
Necktie
Mailbox
Corpse
Mirror
Ancient objects
```

Implementation:

Treat objects as speakers.

```text
Speaker Type:
Object
```

No special-case logic needed.

---

# Dialogue Presentation Layer

Every line should clearly show:

```text
Speaker
Portrait/Icon
Text
Tags
```

Example:

```text
AUTHORITY

Don't back down.
```

Speaker identity matters.

---

# Save Data Requirements

Store:

```text
Conversation History
Visited Nodes
Chosen Options
Relationship Changes
Unlocked Topics
Known Facts
Triggered Reactions
```

Everything must persist.

---

# Minimal Engine Interfaces

```csharp
interface ISpeaker
{
    string Name;
    SpeakerType Type;
}

interface IDialogueNode
{
    string Text;

    List<IChoice> Choices;
}

interface IChoice
{
    List<ICondition> Conditions;

    List<IEffect> Effects;
}

interface IConversation
{
    List<IDialogueNode> Nodes;
}
```

---

# Most Important Insight

Most RPG dialogue systems are:

```text
Dialogue → Story
```

NSF's system is:

```text
Dialogue = Story
Dialogue = Investigation
Dialogue = Character Building
Dialogue = Faculty Expression
Dialogue = Politics
Dialogue = Progression
```

The correct architectural model is:

```text
Dialogue Engine
      ↑
      │
Every Other System
```

rather than:

```text
Story State
      ↑
 Dialogue
```

If your engine treats dialogue as the central hub through which faculties, beliefs, relationships, ideology, story beats, and discoveries all flow, you're very close to NSF design goals. If dialogue is merely a UI for Actor conversations, you've missed NSF's core design.
