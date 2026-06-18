# 8. Story: Story State — NSF Specification

> **Framework:** Story flags, story progression, branching, consequence propagation, global state.
> **Content pack:** Story beat definitions, flag names, story beat content.
> Terminology: [Glossary](terminology-glossary.md)

This is the system that actually makes the story move.

A traditional RPG often treats progression as:

```text id="q1s8n1"
Objective lists
```

A NSF-style game needs **story beats** to be:

```text id="m7d2xk"
Stateful narrative processes
```

The important difference is that the game is not tracking “what tasks are left.”

It is tracking:

* what the world knows
* what the player knows
* what has been discovered
* what has been ruled out
* what has permanently changed
* what narrative phase the story is in

---

# Core Principle

A story beat is not a checklist.

It is a machine that changes state in response to events.

```text id="u8r3pa"
Event →
Condition →
State Change →
New Content
```

That is the core loop.

---

# Core Architecture

```text id="a9k2me"
StoryStateService
 ├─ GlobalState
 ├─ StoryBeatStates
 ├─ Flags
 ├─ Variables
 ├─ Triggers
 ├─ Conditions
 ├─ Transitions
 └─ Consequences
```

Thread state lives in `ThreadEngine` / `IThreadService` — story state references it via flags and variables, not by owning thread objects.

The machine should be data-driven, not hardcoded.

---

# State Is the Source of Truth

The system should not ask:

```text id="h2p8vw"
Has story beat X been accepted?
```

It should ask:

```text id="r4c6dy"
What state is the narrative in?
```

That means the story state service needs to store more than just “active” or “complete.”

It must track:

* discovered
* offered
* accepted
* in progress
* partially solved
* solved
* failed
* abandoned
* resolved differently
* locked
* hidden
* invalidated

---

# Story beat

A story beat should be represented as a stateful entity.

```csharp id="n9f3qd"
class StoryBeat
{
    string Id;
    string Title;
    StoryBeatState State;

    List<StoryBeatNode> Nodes;
    List<StoryBeatVariable> Variables;
    List<StoryBeatTrigger> Triggers;
    List<StoryBeatConsequence> Consequences;
}
```

The story beat is not a string of text.

It is a finite state machine.

---

# Story beat state machine

A story beat should move through explicit states.

```text id="c7v2jh"
Hidden → Revealed → Active → Advanced → Suspended → Resolved
```

Other states may include:

```text id="z6p1re"
Failed
Abandoned
Resolved Differently
Collapsed
Invalidated
```

A NSF-style game benefits from allowing unresolved and ambiguous outcomes.

Not everything ends neatly.

---

# Narrative Variables

A state machine needs variables, not just flags.

Examples:

```text id="p3w8sn"
victim_identified = true
witness_count = 3
union_trust = 2
gun_recovered = false
```

Variables let the system represent partial progress and changing knowledge.

Flags alone are too crude.

---

# Global Narrative State

Some facts are not local to one story beat.

They affect the whole world.

Examples:

```text id="d1q4la"
Companion trust level
Ideology
Current time of day
Debt status
Major faction tension
World phase
```

These should live in global state, not story-beat-local state.

---

# Chronicle section versus story beat

A NSF-style engine should distinguish between:

```text id="y8m5ct"
Thread
```

and

```text id="f2h7av"
Story beat
```

A thread is a large investigation container.

A story beat is a local or supporting narrative process.

Example:

```text id="b6q2zr"
Thread: Resolve the primary incident
```

inside it:

* identify victim
* interview witnesses
* inspect body
* reconstruct bullet trajectory
* determine motive

Each of those is a story beat state process.

---

# Trigger System

Narrative changes should be event-driven.

```text id="g4v8xq"
WorldEvent → Trigger → Transition
```

Example events:

* item acquired
* Actor spoken to
* location entered
* check passed
* check failed
* time advanced
* Belief internalized
* clue discovered

---

# Trigger Definition

```csharp id="q1n8bt"
class QuestTrigger
{
    string EventType;
    Condition[] Conditions;
    Transition TargetTransition;
}
```

Example:

```json id="s7k4lf"
{
  "eventType": "ACTOR_SPOKEN_TO",
  "npcId": "kim",
  "conditions": [
    { "variable": "victim_identified", "equals": true }
  ]
}
```

This allows the narrative to react precisely.

---

# Conditions

Transitions should be guarded by conditions.

Condition types should include:

```text id="j5r8dp"
Story beat state
Variable value
Item possession
Faculty threshold
Belief possession
Dialogue choice
Location
Time
Faction state
Relationship value
Ideology state
Previous event
```

The system should support combinations.

---

# Transition System

A transition should move the narrative from one state to another.

```csharp id="x4d2kv"
class QuestTransition
{
    Condition[] Conditions;
    string FromState;
    string ToState;
    Effect[] Effects;
}
```

Transitions can update:

* story beat state
* global flags
* chronicle entries
* dialogue options
* Actor memory
* world objects

---

# Persistence

A narrative state machine is only interesting if transitions have consequences.

A consequence may:

```text id="z1b6nm"
Unlock dialogue
Change Actor behavior
Enable new scene
Close a path
Open another clue
Start a new story beat
Fail an old one
Modify reputation
Advance time
```

NSF is built on consequences that persist.

---

# Branching Structure

The system should support branching, but not in a rigid “choose one path forever” way.

Better pattern:

```text id="o9f3rw"
State A
 ├─ leads to B
 ├─ leads to C
 └─ leads to D
```

Then later branches can reconverge or diverge again.

This keeps the story reactive without exploding into unmanageable content.

---

# Partial Progress

Important for inquiry gameplay.

A story beat should support partial states like:

```text id="w8n2ms"
Unknown inquiry subject
Known inquiry subject
Inquiry subject confirmed
Inquiry subject cleared
```

or

```text id="r6g3ta"
Lead found
Lead investigated
Lead disproven
Lead replaced
```

This is much better than binary success/failure.

---

# Knowledge-Dependent Progression

Narrative progression should depend on knowledge, not only actions.

Example:

```text id="h1p8cw"
The player can have the item,
but not understand its significance yet.
```

That means the same state can mean different things depending on what the player knows.

This is essential for NSF-style investigation design.

---

# World Response Layer

When a state changes, the world must respond.

That includes:

* dialogue unlocking
* Actor lines changing
* environmental props changing
* chronicle updates
* map markers appearing or disappearing
* rolls becoming available
* old options closing

The state machine should notify all dependent systems.

---

# Event Bus Integration

This system should be driven by a central event bus.

```text id="m2d8ca"
EventBus
 ├─ DialogueEvents
 ├─ CombatlessActionEvents
 ├─ InvestigationEvents
 ├─ TimeEvents
 ├─ ItemEvents
 └─ QuestEvents
```

Story state logic subscribes to events rather than polling constantly.

That makes it cleaner and easier to maintain.

---

# Narrative Nodes

A story beat should be made of narrative nodes, not just objectives.

```csharp id="k3f7ta"
class NarrativeNode
{
    string Id;
    string Text;
    Condition[] EntryConditions;
    Effect[] OnEnter;
    Transition[] Transitions;
}
```

A node can represent:

* An NSFvered fact
* a new phase
* a failed attempt
* a resolution
* an aftermath state

---

# Story beat states should be visible to the chronicle

The chronicle is a view.

The state machine is the engine.

The chronicle should read from the machine and reflect:

* active leads
* current phase
* failed attempts
* open questions
* resolved conflicts

Never make the chronicle the authoritative state store.

---

# Failure States

Failure should be a first-class narrative state.

Examples:

```text id="e6v9ph"
The witness fled.
The Inquiry subject destroyed evidence evidence.
The deal collapsed.
The lead was wrong.
```

Failure should not delete the story beat.

It should move it into a new state that creates different content.

---

# Reversible and Irreversible Transitions

Some transitions should be reversible.

Examples:

* a clue can be reinterpreted
* a theory can be revised
* An inquiry subject can be cleared

Some should be irreversible.

Examples:

* an Actor dies
* a location is destroyed
* trust is permanently lost
* a check outcome becomes canonical

The system must support both.

---

# World Phase System

Large narrative milestones should change the world phase.

Example phases:

```text id="j8w3ld"
Arrival
Investigation
Escalation
Revelation
Resolution
Aftermath
```

Different phases can alter what quests are available and how Actors behave.

---

# Multi-beat dependency graph

Quests should be able to depend on each other.

```text id="v5r9ab"
Story beat A →
Story beat B →
Story beat C
```

But also:

```text id="x0q8ne"
Story beat A + Story beat D → Story beat E
```

This is important for complex investigations.

---

# Narrative Locking

Some content should be locked until state conditions are met.

Examples:

* An inquiry subject will not speak until trust is built
* a room is inaccessible until a key fact is known
* a political conversation only opens after a certain Belief is internalized

This makes progress feel earned.

---

# Save Data Requirements

Serialize:

```text id="c8t4hr"
Story beat states
Global Flags
Story beat variables
Narrative Nodes Visited
Resolved Branches
Failed Branches
World Phase
Dependency Graph Progress
```

Everything must be restorable exactly.

---

# Minimal Engine Interfaces

```csharp id="p7v1qa"
interface IStoryStateService
{
    void HandleEvent(SimEvent e);
    StoryState GetState(string id);
}

interface IStoryBeat
{
    string Id;
    StoryBeatState State;
}

interface IStoryBeatTrigger
{
    bool Matches(SimEvent e, StoryContext context);
}

interface IStoryBeatTransition
{
    void Apply(StoryContext context);
}
```

---

# Most Important Insight

Story state (`IStoryStateService`) is not about telling the player what to do.

It is about modeling what the story has become.

A NSF-style narrative machine should answer:

```text id="b3k6wc"
What is true now?
What changed?
What is impossible now?
What new possibilities exist?
```

If the engine can do that, it can support NSF-style storytelling.

If it cannot, it will only produce a checklist log with better writing.
