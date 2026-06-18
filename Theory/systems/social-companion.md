# 17. Social: Companion — NSF Specification

> **Framework:** Companion presence, reactions, trust, joint investigation, commentary hooks.
> **Content pack:** Companion character, dialogue, reaction content, relationship arc.
> Terminology: [Glossary](../terminology-glossary.md)

In most RPGs, companions are:

```text
Party members
```

In NSF, the companion system is primarily:

```text
Narrative witness
+
Social mirror
+
Secondary player character
+
Relationship simulator
```

The companion is not there to help win combat.

The companion is there to:

* observe the player
* react to choices
* validate or challenge beliefs
* provide information
* influence tone
* act as an external conscience
* create emotional continuity

For a NSF-style game, the companion system is closer to a permanent co-star than a party system.

---

# Core Principle

The companion should feel like a real person traveling with the player.

The player should constantly ask:

```text
What does my companion think about this?
```

Not:

```text
What abilities does my companion have?
```

---

# Core Architecture

```text
CompanionService
 ├─ CompanionManager
 ├─ RelationshipTracker
 ├─ ApprovalModel
 ├─ ContextCommentary
 ├─ DialogueParticipation
 ├─ CompanionMemory
 ├─ CompanionKnowledge
 ├─ ScheduleIntegration
 └─ NarrativeReactions
```

---

# Core Design Goal

The companion should create the feeling that:

```text
Someone is always watching.
```

Not in a punitive way.

In a social way.

The player develops a relationship with a persistent observer.

---

# Companion Object

```csharp
class Companion
{
    string Id;

    RelationshipState Relationship;

    MemoryBank Memory;

    KnowledgeBase Knowledge;

    CommentaryProfile Commentary;

    CompanionState State;
}
```

---

# Companion State

Typical states:

```text
Following
Present
Separated
Unavailable
Sleeping
Investigating
Waiting
NarrativeLocked
```

The companion is not guaranteed to be present at all times.

---

# Companion Presence

Presence matters.

Many systems should ask:

```csharp
if (CompanionPresent)
```

because:

* dialogue changes
* reactions change
* passive comments become available
* social consequences change

The same action may feel different depending on whether the companion witnesses it.

---

# Narrative Witness Model

One of the most important concepts.

The companion remembers:

```text
What player said
What player did
What player failed
What player believed
```

This creates continuity.

Example:

Player repeatedly behaves recklessly.

Hours later:

```text
Companion references pattern.
```

That feels extremely powerful.

---

# Companion Memory

Companion memory should store:

```text
Major decisions
Embarrassing moments
Ideological statements
Promises
Lies
Successes
Failures
```

Not every detail.

Only socially meaningful events.

---

# Companion Knowledge

Separate memory from knowledge.

```text
Memory
=
What happened

Knowledge
=
What companion knows
```

Example:

The player discovers a clue alone.

Companion knowledge:

```text
Unknown
```

Until informed.

This distinction is important.

---

# Relationship Integration

The companion should use the full relationship system.

Recommended dimensions:

```text
Trust
Respect
Friendship
Confidence
Disappointment
Concern
```

These evolve throughout the game.

---

# Approval Is Not Enough

Avoid a simple:

```text
Liked
Disliked
```

model.

Instead:

Example:

```text
Trust = High
Friendship = High
Respect = Low
```

Possible interpretation:

```text
Likes player personally.
Thinks player is a disaster professionally.
```

This is much richer.

---

# Companion Commentary System

One of the most important outputs.

Companions should comment on:

```text
Locations
Actors
Discoveries
Failures
Ideology statements
Story beat developments
```

The player should feel accompanied.

---

# Commentary Categories

Useful categories:

```text
Observation
Analysis
Humor
Concern
Disapproval
Encouragement
Explanation
Correction
```

Different companions may favor different categories.

---

# Contextual Commentary

Comments should depend on:

```text
Location
Story beat state
Actor Present
Player Behavior
Ideology
Conduct
Relationship
```

Not just trigger once.

---

# First-Time Reactions

Certain events deserve unique commentary.

Examples:

```text
First body examined
First major failure
First political declaration
First embarrassing moment
```

These are memorable anchor moments.

---

# Repeated Behavior Detection

Companions should recognize patterns.

Example:

```text
Player apologizes repeatedly.
```

After enough events:

```text
Companion comments on apology habit.
```

This makes the companion feel observant.

---

# Dialogue Participation

Companions should be able to enter dialogue.

Types:

```text
Interruptions
Questions
Clarifications
Corrections
Side Comments
```

The companion becomes an active participant.

---

# Companion Dialogue Hooks

Dialogue nodes should support:

```json
{
  "companionComment": true
}
```

or:

```json
{
  "requiresCompanionPresent": true
}
```

The dialogue engine should allow companion insertion naturally.

---

# Companion-Initiated Dialogue

Companions should sometimes start conversations.

Examples:

```text
After major discovery
After failure
After ideological statement
After dramatic event
```

This helps the relationship feel alive.

---

# Companion Reactions To Conducts

Companions should recognize emerging identities.

Examples:

```text
conduct_humble
conduct_bold
conduct_reckless
conduct_negligent
```

Reaction may range from amusement to concern.

---

# Companion Reactions To Politics

Ideology should influence companion dialogue.

Examples:

```text
Approval
Disagreement
Curiosity
Alarm
Mockery
```

The companion acts as a social reflector.

---

# Companion As Soft Guidance

Companions can provide hints.

Not through UI.

Through conversation.

Example:

```text
"We should probably speak with the witness."
```

This feels natural and immersive.

---

# Companion As Emotional Anchor

The companion helps stabilize the narrative.

They:

* react to chaos
* acknowledge victories
* remember losses
* ground the player emotionally

This is especially important in highly psychological narratives.

---

# Companion Availability

Companions may become unavailable due to:

```text
Time
Story Events
Conflict
Injury
Relationship State
Schedule
```

Presence should not be guaranteed.

---

# Temporary Separation

The engine should support:

```text
Companion leaves
Companion returns
```

without breaking systems.

Dialogue and event hooks must adapt automatically.

---

# Companion-specific fact checks

Companions may provide expertise.

Examples:

```text
Forensics
History
Politics
Geography
Procedure
```

When appropriate they can supply information automatically.

---

# Companion-Assisted Checks

Companions may modify checks.

Examples:

```text
Lower difficulty
Additional clue
Alternative dialogue path
Automatic success
```

This should be used sparingly.

The companion should not replace player agency.

---

# Companion Trust Milestones

Certain thresholds should unlock content.

Examples:

```text
Professional Trust
Personal Trust
Friendship
Mutual Respect
```

These become narrative milestones.

---

# Companion Disappointment

A useful dimension often ignored.

Examples:

```text
Player repeatedly fails
Player lies
Player behaves recklessly
```

Trust may remain.

Disappointment increases.

This creates nuanced relationships.

---

# Companion Loyalty

Separate from friendship.

Example:

```text
Friendship = High
Loyalty = Moderate
```

A companion may like the player but still refuse unethical actions.

---

# Companion Narrative Role

The companion should have:

```text
Public Goal
Private Goal
Beliefs
History
Secrets
Arc
```

They must exist independently of the player.

---

# Companion Event Subscriptions

Companions should listen to:

```text
Dialogue Events
Story events
Ideology events
Conduct Events
Relationship Events
World Events
```

This allows responsive behavior.

---

# Save Data Requirements

Serialize:

```text
Relationship Values
Trust Milestones
Companion Memory
Companion Knowledge
Availability State
Presence State
Triggered Commentary
Seen Events
```

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface ICompanionService
{
    string GetCompanionActorId();
    float GetTrustMetric(string metricId);
    void ModifyTrust(string metricId, float delta);
    bool IsAvailableForDialogue();
}
```

## Domain model

```csharp
class CompanionProfile { string ActorId; string[] MetricIds; }
interface ICompanion { string ActorId; float GetMetric(string metricId); }
```


# Recommended Internal Flow

```text
Player Action
      ↓
Event Emitted
      ↓
Companion Observes
      ↓
Memory Updated
      ↓
Relationship Updated
      ↓
Possible Commentary
```

This loop should happen constantly throughout the game.

---

# Most Important Insight

The companion system is not about giving the player an ally.

It is about giving the player a **persistent human audience**.

The companion should become:

```text
The person who remembers who you were yesterday.
```

They observe:

* your failures
* your beliefs
* your habits
* your growth

Over time, the relationship between player and companion becomes one of the most important narratives in the game.

If implemented correctly, the player will care about the companion's opinion even when there is no mechanical reward for doing so.

That is the hallmark of a successful NSF-style companion system.
