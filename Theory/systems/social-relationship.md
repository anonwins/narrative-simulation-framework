# 14. Social: Relationship — NSF Specification

> **Framework:** Trust, affinity, respect, dependency, long-term reactions, companion bonds.
> **Content pack:** Relationship thresholds, Actor-specific reaction content.
> Terminology: [Glossary](../terminology-glossary.md)

The Relationship models how characters feel about each other and how those feelings evolve.

In a typical RPG:

```text
Relationship = Reputation Number
```

In a NSF-style game:

```text
Relationship = Dynamic Social State
```

Characters can simultaneously:

* trust you
* dislike you
* respect you
* fear you
* pity you
* depend on you
* be attracted to you
* be politically opposed to you

The system must support contradiction.

---

# Core Principle

Relationships are not binary.

Bad:

```text
Friendly
Hostile
```

Better:

```text
Trust = 8
Respect = 3
Fear = 1
Suspicion = 7
```

This allows nuanced social interactions.

---

# Core Architecture

```text
RelationshipService
 ├─ RelationshipProfiles
 ├─ RelationshipStates
 ├─ SocialEvents
 ├─ AffinityRules
 ├─ ReputationLayers
 ├─ FactionInfluence
 ├─ MemoryLinks
 └─ RelationshipHistory
```

---

# Scope

The system should support:

```text
Player ↔ Actor
Actor ↔ Actor
Player ↔ Faction
Faction ↔ Faction
```

Even if only Player ↔ Actor is exposed initially.

---

# Relationship Object

```csharp
class Relationship
{
    string SourceId;
    string TargetId;

    int Trust;
    int Respect;
    int Fear;
    int Suspicion;
    int Familiarity;

    RelationshipState State;
}
```

---

# Relationship Dimensions

Recommended dimensions:

## Trust

Measures belief in honesty and reliability.

Examples:

```text
High Trust:
- shares secrets
- accepts explanations

Low Trust:
- doubts statements
- withholds information
```

---

## Respect

Measures perceived competence and capability.

Examples:

```text
High Respect:
- takes player seriously
- follows recommendations

Low Respect:
- dismissive
- patronizing
```

---

## Fear

Measures perceived danger.

Examples:

```text
High Fear:
- avoids conflict
- cooperates cautiously

Low Fear:
- confrontational
- relaxed
```

Fear is not necessarily negative.

---

## Suspicion

Measures uncertainty and distrust.

Examples:

```text
High Suspicion:
- guarded responses
- reduced disclosure

Low Suspicion:
- open communication
```

---

## Familiarity

Measures social exposure.

Examples:

```text
Low Familiarity:
- formal interactions

High Familiarity:
- personal topics
- private jokes
```

---

# Hidden Values

Most relationship values should be hidden.

The player should infer them from:

* dialogue tone
* body language
* available topics
* reactions
* willingness to help

The system is simulation-first, UI-second.

---

# Relationship States

Derived states can be generated from dimensions.

Examples:

```text
Trusted Ally
Professional Partner
Useful Stranger
Annoying Acquaintance
Ideological opponent
Potential Informant
Hostile Witness
```

These are descriptive layers, not replacements for underlying values.

---

# Relationship History

Relationships should remember how they evolved.

```text
RelationshipHistory
 ├─ First Meeting
 ├─ Positive Events
 ├─ Negative Events
 ├─ Major Decisions
 ├─ Betrayals
 └─ Acts of Loyalty
```

History affects future changes.

---

# Social Event Integration

Relationships primarily change through events.

Examples:

```text
Helped Actor
Insulted Actor
Kept Promise
Broke Promise
Solved Problem
Threatened Actor
Protected Actor
Embarrassed Self
```

Each event generates relationship modifications.

---

# Relationship Modifiers

Example:

```json
{
  "event": "HELPED_ACTOR",
  "trust": +2,
  "respect": +1
}
```

Another:

```json
{
  "event": "LIED_AND_CAUGHT",
  "trust": -5,
  "suspicion": +4
}
```

---

# Asymmetric Relationships

Relationships should not automatically mirror.

Example:

```text
Actor trusts player = 9

Player trusts Actor = unknown
```

Or:

```text
Actor respects player = 8

Actor likes player = 2
```

Social reality is asymmetric.

---

# Actor-to-Actor Relationships

Important for narrative richness.

Example:

```text
actor_companion respects actor_merchant
actor_companion distrusts actor_leader_faction_guild
actor_merchant fears political escalation
```

Dialogue can query these relationships.

---

# Relationship Thresholds

Dialogue can require thresholds.

Example:

```json
{
  "requiresTrust": 6
}
```

Or:

```json
{
  "requiresSuspicionBelow": 3
}
```

This controls information flow.

---

# Trust Is Not Access

Important distinction.

An Actor may trust the player but still refuse information because:

* faction loyalty
* legal obligation
* personal shame
* political disagreement

Trust is only one variable.

---

# Faction Influence

Relationships should be influenced by factions.

Example:

```text
Player publicly supports Union

actor_member_faction_guilds:
+Trust

Corporate representatives:
+Suspicion
```

The effect should be gradual, not binary.

---

# Ideological Influence

The political system should affect relationships.

Examples:

```text
Communist statements
Moralist statements
Ultraliberal statements
Fascist statements
```

Actors react based on beliefs.

---

# Memory Integration

Actor memory should feed relationship updates.

Example:

```text
Player insulted Actor three days ago.
```

Even if current conversation is unrelated, memory affects reactions.

---

# Delayed Relationship Effects

Not every consequence should be immediate.

Example:

```text
Player repeatedly lies.
```

Actor may only realize pattern later.

Then:

```text
Trust collapses.
```

Delayed consequences create realism.

---

# Relationship Decay

Optional but useful.

Certain dimensions may slowly drift.

Example:

```text
Familiarity decreases over long absence.
```

Or:

```text
Temporary anger fades.
```

Not all dimensions should decay equally.

---

# Relationship Locks

Major events can permanently alter relationships.

Examples:

```text
Saved Actor's life
Betrayed Actor
Exposed secret
Killed friend
```

These create durable social state.

---

# Relationship-Based Dialogue

Dialogue should query relationship state.

Examples:

```text
High trust:
"I'll tell you something."

Low trust:
"Why are you asking?"
```

This is one of the main outputs of the system.

---

# Relationship-Based Rolls

Relationships can modify checks.

Examples:

```text
High trust:
Lower persuasion difficulty

High suspicion:
Higher persuasion difficulty
```

This creates interaction between social simulation and mechanics.

---

# Companion Relationships

Companions require richer tracking.

Additional dimensions:

```text
Confidence
Approval
Dependence
Disappointment
```

Especially important for Kim-style companions.

---

# Public Versus Private Relationship

An Actor may display one thing and feel another.

Example:

```text
Public:
Friendly

Private:
Suspicious
```

Support:

```csharp
PublicAttitude
PrivateAttitude
```

This allows deception and hidden motives.

---

# Relationship Knowledge

The player should not always know relationship values.

Instead expose:

```text
Observed Trust
Observed Respect
Observed Mood
```

The true state remains internal.

---

# Relationship Queries

The engine should support queries like:

```csharp
GetTrust(source, target)
GetRespect(source, target)
GetRelationshipState(source, target)
```

Dialogue, quests, and Actor AI use these constantly.

---

# Event Triggers

Relationship changes can trigger:

```text
Dialogue Unlock
Dialogue Lock
Story beat update
Actor Relocation
Faction Change
Information Disclosure
Companion Reaction
```

Relationships are narrative drivers.

---

# Save Data Requirements

Serialize:

```text
Relationship Values
Relationship History
Permanent Relationship Flags
Faction Modifiers
Ideological Modifiers
Observed States
```

Relationships must persist exactly.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IRelationshipService
{
    RelationshipState GetRelationship(string actorId);
    void ModifyMetric(string actorId, string metricId, float delta);
    float GetMetric(string actorId, string metricId);
}
```

## Domain model

```csharp
class RelationshipState { float Trust; float Fear; float Respect; float Dependency; }
interface IRelationship { string TargetActorId; RelationshipState State; }
```


# Recommended Internal Formula

Use:

```text
Raw Dimensions
      ↓
Derived Attitude
      ↓
Dialogue Availability
      ↓
Behavior Changes
```

Never store only derived attitudes.

Store the underlying dimensions.

---

# Most Important Insight

A NSF-style relationship system is not about tracking whether somebody likes the player.

It is about modeling:

```text
What does this person think of me?
Why?
What have I done to earn that?
What would change their mind?
```

The system should make every social interaction feel like it leaves a mark.

If it succeeds, Actors become people.

If it fails, Actors become dialogue vending machines.
