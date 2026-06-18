# 10. Sim: Actor — NSF Specification

> **Framework:** Actor memory, schedules, faction membership, dialogue state, reputation effects.
> **Content pack:** Actor definitions, names, personalities, faction assignments.
> Terminology: [Glossary](terminology-glossary.md)

The Actor system is more than character data.

In a NSF-style game, Actors are:

* conversation partners
* memory holders
* world-state actors
* faction representatives
* relationship targets
* narrative triggers
* behavioral mirrors for the player character

A standard RPG Actor system usually answers:

```text id="q7m4tp"
Who can I talk to?
```

A NSF-style Actor system must also answer:

```text id="p2d8ks"
What does this person know, remember, want, fear, and change over time?
```

---

# Core Architecture

```text id="n8f3wl"
ActorService
 ├─ ActorDefinitions
 ├─ ActorInstances
 ├─ DialogueState
 ├─ Memory
 ├─ Relationships
 ├─ FactionLinks
 ├─ Schedules
 ├─ Reactions
 └─ WorldHooks
```

---

# Core Principle

Actors are not static story-beat dispensers.

They are stateful agents.

```text id="u6c1aq"
Actor =
Identity
+
Memory
+
Relationship to player
+
Faction position
+
Current mood
+
Current location
+
Current knowledge
```

---

# Actor Object Model

Every Actor should be represented as a structured, data-driven entity.

```csharp id="j3n9te"
class Actor
{
    string Id;
    string Name;
    ActorState State;
    RelationshipState Relationship;
    MemoryBank Memory;
    Faction Affiliation;
    Schedule Schedule;
    DialogueProfile DialogueProfile;
    BehaviorProfile BehaviorProfile;
}
```

---

# Actor Instances Versus Definitions

Separate:

```text id="r4v8ja"
ActorDefinition
```

from

```text id="m1q6td"
ActorInstance
```

Definition contains identity, base dialogue, faction, portrait, and defaults.

Instance contains current mood, knowledge, trust, location, injuries, flags, and conversation history.

This is important for persistence.

---

# Actor State

Actors need explicit state.

Possible values:

```text id="h2w5kf"
Idle
Talking
Observing
Moving
Working
Sleeping
Alerted
Hostile
Withholding
Unavailable
```

The state changes what the Actor can say and do.

---

# Memory System

One of the most important features.

Actors should remember:

* what the player said
* what the player did
* what the player wore
* what the player believes
* whether the player lied
* whether the player failed a check
* whether the player helped or insulted them

```text id="a9k2rr"
ActorMemory
 ├─ Facts
 ├─ Events
 ├─ ConversationHistory
 ├─ TrustMarkers
 ├─ Betrayals
 └─ Promises
```

---

# Memory As Narrative State

Actor memory should affect future content.

Example:

* if the player insulted an Actor, future dialogue changes
* if the player revealed a fact, later conversations reference it
* if the player failed a check in front of them, they may lose respect
* if the player repeatedly acts kindly, they may open up

Memory should be persistent and consequential.

---

# Relationship

Each Actor should maintain a relationship state with the player.

```csharp id="g6p1xn"
class RelationshipState
{
    int Trust;
    int Respect;
    int Fear;
    int Familiarity;
    int Suspicion;
}
```

These values do not have to be visible to the player, but the game should use them heavily.

---

# Relationship Is Not Friendship

A NSF-style relationship model should support mixed states.

An Actor may simultaneously:

* trust the player professionally
* dislike them personally
* fear their instability
* respect their competence
* pity them
* be politically opposed to them

The engine should support contradictions.

---

# Relationship Reactions

Relationship values should drive:

* dialogue tone
* available topics
* willingness to cooperate
* body language
* chance to share clues
* future help or refusal
* how harshly they judge failures

Example:

```text id="p8j3vc"
High Respect → more direct answers
High Suspicion → guarded dialogue
High Fear → cautious cooperation
```

---

# Dialogue Profile

Actors need a dialogue style definition.

```csharp id="b7x4mz"
class DialogueProfile
{
    string VoiceStyle;
    string VocabularyStyle;
    string Temperament;
    List<string> TopicBiases;
}
```

Examples:

* formal and restrained
* sarcastic and defensive
* warm and practical
* bureaucratic and evasive
* ideological and argumentative

---

# Behavioral Profile

Actor behavior should not be limited to dialogue.

```csharp id="k5q9dv"
class BehaviorProfile
{
    bool Roams;
    bool Works;
    bool Sleeps;
    bool ReactsToPlayer;
    bool ParticipatesInEvents;
}
```

This supports more believable world presence.

---

# Schedule System

Actors should have time-based routines.

```text id="t3n8wa"
Morning → at work
Afternoon → patrol / shop / socialize
Evening → rest / move location
Night → sleep / unavailable
```

Scheduling is essential for a living world.

---

# Location Binding

Actors should be tied to locations or regions.

```text id="v1m6qc"
Home
Workplace
Patrol Area
Hangout Spot
Temporary Location
```

They may move based on schedule or narrative state.

---

# Availability Rules

An Actor can be unavailable because of:

* time of day
* story beat state
* story phase
* injury
* arrest
* death
* leaving the area
* refusing contact
* being occupied

The system should treat availability as stateful, not binary.

---

# Faction Links

Actors should be linked to factions.

```text id="s8d4hx"
Faction
 ├─ Union
 ├─ faction_authority
 ├─ faction_enterprise
 ├─ Local Residents
 ├─ Church
 └─ Ideology groups
```

Faction affiliation should influence:

* what they know
* what they hide
* how they react
* who they defend
* who they distrust
* which dialogue options they offer

---

# Faction Loyalty Versus Personal Loyalty

These should be separate.

An Actor may:

* support their faction
* privately disagree with leadership
* help the player for personal reasons
* betray the faction if pushed

That complexity creates NSF-style social tension.

---

# Fact model

Actors should have a knowledge bank.

```text id="d9u2ks"
KnownFacts
 ├─ about self
 ├─ about player
 ├─ about world
 ├─ about thread
 ├─ about factions
 └─ about other Actors
```

The dialogue system can query this to unlock or block topics.

---

# What Actors Know Matters More Than What They Say

An Actor should sometimes know something they refuse to say.

That distinction must be modeled.

```text id="e4n7qp"
Known = true
Shared = false
```

This is essential for investigation gameplay.

---

# Information Sharing Rules

Actors should have rules for revealing information:

* trust threshold
* payment requirement
* faction loyalty
* politeness requirement
* time-sensitive disclosure
* faculty-gated interrogation
* story beat state dependency

---

# Mood System

Actors should have changing mood states.

```text id="f1r6sb"
Calm
Guarded
Annoyed
Angry
Afraid
Sad
Suspicious
Hopeful
Exhausted
```

Mood should affect both dialogue and behavior.

---

# Mood Is Temporary, Memory Is Persistent

This distinction is important.

* Mood changes quickly
* Memory lasts
* Relationship changes more slowly
* Faction alignment may be fixed or semi-fixed

---

# Reactivity System

Actors should react to:

* player appearance
* faculties
* equipment
* dialogue tone
* time of day
* nearby events
* violence
* other Actors
* ideological statements

Example:

```text id="k9m4qa"
If player wears police uniform:
    Actor may become more cooperative
```

Or:

```text id="x6p2ls"
If player looks unstable:
    Actor may become more cautious
```

---

# Recognition System

Actors may recognize the player in different ways:

* in this role
* as a drunk
* as a local
* as a threat
* as a celebrity
* as a nuisance
* as a political type

Recognition can be dynamic and cumulative.

---

# Conversation Gatekeeping

Actors should control access to dialogue topics.

Each topic can require:

* a known fact
* a relationship threshold
* a belief
* an item
* a faction state
* a previous conversation outcome
* a roll

This keeps conversations layered and reactive.

---

# Companion Compatibility

Some Actors are companions or quasi-companions.

They need additional systems for:

* following the player
* commenting during dialogue
* reacting to choices
* judging failures
* helping in investigations
* tracking trust over time

Kim-style companion behavior should be a first-class Actor mode.

---

# Actor Reaction Events

Actors should listen for world events.

Examples:

* item acquired
* clue discovered
* check failed
* insult delivered
* threat issued
* Belief internalized
* ideology shift
* time progression
* faction conflict

These events can trigger new dialogue or behavior changes.

---

# Multi-Layered Actor Identity

Each Actor should carry multiple identity layers:

```text id="h0q5mw"
Public Role
Private Goal
Faction Role
Personal Belief
Emotion
Narrative Function
```

This allows the same character to operate as a mechanic, a witness, an ideological opponent, and a human being all at once.

---

# Player-Facing Presentation

The system should support:

* portrait
* name
* role title
* faction tag
* mood indicator
* conversation history
* known facts
* relationship cues

The player should quickly understand who this person is in the world.

---

# Death, Removal, and Irreversibility

Actors need long-term state transitions.

Possible outcomes:

* dies
* leaves
* is arrested
* becomes hostile
* becomes friendly
* becomes inaccessible
* changes faction
* changes role

These changes must persist.

---

# Serialization Requirements

Save:

```text id="c4x1gv"
Actor states
Memory
Relationship values
Faction status
Location
Schedule progress
Dialogue history
Known facts
Mood
Availability
```

---

# Minimal Engine Interfaces

```csharp id="b2k9ja"
interface IActor
{
    string Id;
    string Name;
    ActorState State;
    RelationshipState Relationship;
}

interface IActorMemory
{
    void Remember(SimEvent e);
    bool Knows(string factId);
}

interface IBehaviorProfile
{
    void Update(ActorContext context);
}
```

---

# Most Important Insight

A NSF-style Actor is not a dialogue container.

It is a **stateful social agent**.

The engine should make every Actor feel like a person with:

* memory
* opinions
* loyalties
* moods
* habits
* knowledge gaps
* reactions to the player
* a role in the world beyond conversation

If the system can do that, the world will feel inhabited.

If it cannot, the game will feel like a list of dialogue trees attached to mannequins.
