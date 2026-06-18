# 24. Sim: Persistence — NSF Specification

> **Framework:** Persistent choices, dialogue/Actor memory, world-state mutation, long-term consequences.
> **Content pack:** Save-relevant flag content, epilogue triggers.
> Terminology: [Glossary](terminology-glossary.md)

This system is the **backbone of narrative persistence**.

Almost every major NSF system ultimately feeds into it.

Without this system:

```text
Dialogue = temporary
Choices = temporary
Actors = temporary
World = static
```

With this system:

```text
Dialogue
→ Memory

Memory
→ Consequence

Consequence
→ World Change

World Change
→ Future Content
```

The player gradually creates a unique version of the world.

---

# Core Principle

The game must remember.

Not just:

```text
Story beat completed
```

but:

```text
How it was completed
Who saw it
Who remembers it
Who was affected
What changed
```

The system should continuously answer:

```text
What is true now that was not true before?
```

---

# Core Architecture

```text
PersistenceService
 ├─ GlobalFlags
 ├─ WorldState
 ├─ CharacterMemory
 ├─ DialogueMemory
 ├─ EventHistory
 ├─ RelationshipChanges
 ├─ StateMutations
 ├─ ConsequenceRules
 └─ PersistenceLayer
```

---

# Core Model

Every meaningful player action should be capable of becoming persistent.

```csharp
class Consequence
{
    string Id;

    string SourceEvent;

    ConsequenceType Type;

    bool Persistent;

    DateTime Timestamp;
}
```

---

# State Layers

The world should be modeled as layers.

```text
Global State
Faction State
Location State
Actor State
Relationship State
Story beat state
Dialogue State
```

A single event may affect multiple layers.

---

# Global State

Represents world-level truths.

Examples:

```text
Mercenary Crisis Resolved
Strike Escalated
actor_leader_faction_guild Exposed
Ideology rally occurred
```

Global state affects many systems simultaneously.

---

# World-State Mutation

The world should physically and narratively change.

Examples:

```text
Door unlocked
Body removed
Actor relocated
Area opened
Building damaged
Object destroyed
```

The player should revisit places and notice change.

---

# Mutation Object

```csharp
class WorldMutation
{
    string TargetId;

    MutationType Type;

    object Value;
}
```

Example:

```text
Door
→ Opened

Actor
→ Moved

Container
→ Emptied
```

---

# Persistent Choices

Every major decision should generate state.

Examples:

```text
Accepted Bribe
Refused Bribe
Supported faction_guild
Helped Witness
Betrayed Informant
```

These should remain true forever unless explicitly reversed.

---

# Choice Recording

Recommended format:

```json
{
  "choiceId": "accepted_bribe",
  "timestamp": "day3_14:20"
}
```

The engine should maintain a complete history.

---

# Dialogue Memory

Dialogue must persist.

Example:

```text
Player insulted Actor.
```

Hours later:

```text
Actor remembers insult.
```

This should not require hardcoded scripting.

---

# Dialogue Memory Object

```csharp
class DialogueMemory
{
    string CharacterId;

    string MemoryId;

    int Strength;
}
```

Not all dialogue deserves memory.

Only meaningful interactions.

---

# Memory Categories

Examples:

```text
Promise
Threat
Lie
Insult
Confession
Ideological statement
Important Revelation
```

These become future narrative hooks.

---

# Actor Memory System

Each Actor should maintain memory.

```csharp
class ActorMemory
{
    List<MemoryEntry> Memories;
}
```

Actor memory is one of the biggest sources of narrative continuity.

---

# Memory Entry

```csharp
class MemoryEntry
{
    string EventId;

    float Importance;

    DateTime OccurredAt;
}
```

Importance determines persistence.

---

# Memory Decay

Not all memories should last forever.

Possible categories:

```text
Trivial
Temporary
Important
Permanent
```

Example:

```text
Minor greeting
→ fades

Public humiliation
→ permanent
```

---

# Relationship Consequences

Relationship changes should persist.

Examples:

```text
Trust increased
Respect decreased
Suspicion increased
```

Future dialogue should query these values.

---

# Delayed Consequences

A crucial NSF mechanic.

Many consequences should occur later.

Example:

```text
Player lies.
```

Nothing happens immediately.

Later:

```text
Actor discovers truth.
```

Consequence triggers.

---

# Consequence Scheduling

Support delayed execution.

```csharp
class ScheduledConsequence
{
    string TriggerId;

    DateTime TriggerTime;
}
```

Or:

```text
After story beat
After Day 3
After Meeting Actor
```

---

# Hidden Consequences

Not all consequences should be visible.

Example:

```text
Faction opinion changed.
```

Player receives no notification.

The effect appears later through behavior.

This creates realism.

---

# World History

The game should preserve a history of major events.

```text
EventHistory
 ├─ Day
 ├─ Event
 ├─ Participants
 └─ Consequences
```

This supports narrative callbacks.

---

# Callback System

Past actions should reappear.

Examples:

```text
"You said this yesterday."

"You helped me earlier."

"You promised."
```

Callbacks are one of the strongest consequence signals.

---

# State Querying

Every major system should query consequences.

Examples:

```csharp
HasFlag("accepted_bribe")

HasMemory("insulted_witness")

GetRelationship("kim")
```

The entire game depends on this.

---

# Consequence Graph

Represent consequences as chains.

```text
Choice
↓
Memory
↓
Relationship Change
↓
Dialogue Unlock
↓
Story branch
↓
World Mutation
```

This creates deep reactivity.

---

# Faction Consequences

Choices should affect faction state.

Example:

```text
Help faction_guild
↓
faction_guild Trust Increases
↓
Corporate Suspicion Increases
```

These effects should persist.

---

# Ideological consequences

Ideological identity should create long-term effects.

Examples:

```text
Repeated communist statements

→ future political dialogue
→ faction reactions
→ Belief unlocks
```

---

# Location Consequences

Locations should remember.

Examples:

```text
Object taken
Door broken
Actor removed
Event occurred
```

Returning to a location should not reset it.

---

# Companion Consequences

Companions should remember.

Examples:

```text
Failure
Dishonesty
Heroic act
Ideological position
```

These memories influence relationship evolution.

---

# Story beat consequences

Story beat outcomes should affect future content.

Not simply:

```text
Story beat completed
```

But:

```text
How completed
Who helped
Who suffered
Who benefited
```

These variables matter later.

---

# Narrative Consistency

The system should prevent contradictions.

Bad:

```text
Actor thanks player.

Actor later acts as if never met.
```

Good:

```text
All future content references saved state.
```

Consistency is one of the primary goals.

---

# Event-Sourced Design

Recommended architecture:

```text
Action
↓
Event
↓
State Mutation
↓
Persistence
↓
Future Query
```

This scales well.

---

# Consequence Severity

Not all consequences are equal.

Suggested levels:

```text
Minor
Moderate
Major
Critical
```

Severity influences:

* memory persistence
* callback frequency
* world impact

---

# Save File Structure

Recommended sections:

```text
GlobalFlags
WorldMutations
RelationshipData
ActorMemories
FactionState
PoliticalState
StoryBeatState
ThreadState
DialogueHistory
EventHistory
```

These should be serialized independently.

---

# Persistence Requirements

The game should survive:

```text
Save
Load
Reload
Scene Change
Time Skip
```

without losing consequence state.

---

# Debugging Tools

Critical for development.

Support:

```text
View Active Flags
View Memories
View Consequence Graph
View World Mutations
View Event History
```

Narrative debugging becomes impossible otherwise.

---

# Minimal Engine Interfaces

```csharp
interface IPersistenceService
{
    void Apply(SimEvent e);

    bool HasFlag(string id);

    object GetState(string key);
}
```

---

# Recommended Internal Flow

```text
Player Action
      ↓
Event
      ↓
Consequence Rule
      ↓
State Mutation
      ↓
Persistence
      ↓
Future Content Changes
```

Every major system should participate in this loop.

---

# Relationship to Other Systems

This system sits above almost everything.

```text
Dialogue
      ↓
Actor Memory
      ↓
Relationship
      ↓
Faction
      ↓
Story State
      ↓
World State
      ↓
Persistence
```

It acts as the central persistence layer for narrative simulation.

---

# Most Important Insight

The Persistence is what makes the player feel that choices matter.

It is not about branching endings.

It is about ensuring that the world remembers.

The engine should continuously answer:

```text
What happened?
Who remembers?
What changed?
What remains changed?
```

If implemented correctly, the player feels like they are living inside a persistent reality.

If implemented incorrectly, the game becomes a series of disconnected conversations that forget themselves the moment they end.
