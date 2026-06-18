# 26. Rules: Gate — NSF Specification

> **Framework:** Faculty, belief, item, time, relationship, political, conduct gates.
> **Content pack:** Gate condition assignments, gated content definitions.
> Terminology: [Glossary](../terminology-glossary.md)

This system controls **when content becomes available**.

It is one of the most important systems in NSF because it creates:

* replayability
* build diversity
* narrative variation
* discovery depth
* player identity

Without content gating:

```text id="cg1"
Every player sees everything.
```

With content gating:

```text id="cg2"
Different characters experience different realities.
```

This system determines:

```text id="cg3"
Who can see this?
Who can access this?
Who can understand this?
Who can unlock this?
```

---

# Core Principle

Content should not be gated primarily by level.

Instead it should be gated by:

```text id="cg4"
Faculties
Beliefs\nItems
Relationships
Politics
Knowledge
Time
World State
```

The goal is:

```text id="cg5"
Different character
→ Different experience
```

not:

```text id="cg6"
Higher level
→ More content
```

---

# Core Architecture

```text id="cg7"
GateService
 ├─ GateDefinitions
 ├─ GateEvaluator
 ├─ FacultyGates
 ├─ BeliefGates
 ├─ ItemGates
 ├─ TimeGates
 ├─ RelationshipGates
 ├─ PoliticalGates
 ├─ FactGates
 └─ WorldStateGates
```

---

# Core Model

Every piece of content should be able to define access rules.

```csharp id="cg8"
class ContentGate
{
    string Id;

    List<Requirement> Requirements;
}
```

The engine evaluates requirements before exposing content.

---

# What Can Be Gated?

Virtually everything.

Examples:

```text id="cg9"
Dialogue Nodes
Story branches
Actor Reactions
Locations
Interactions
Beliefs\nChecks
Narration
Evidence
Endings
```

The system should be universal.

---

# Gate Evaluation

Simple flow:

```text id="cg10"
Content
↓
Requirements
↓
Evaluate
↓
Visible / Hidden
```

The content still exists.

The player simply may not see it.

---

# Faculty Gates

One of the most common forms.

Example:

```json id="cg11"
{
  "faculty": "Logic",
  "minimum": 6
}
```

Unlocks:

```text id="cg12"
Additional observation
Additional dialogue
Additional deduction
```

---

# Passive Faculty Gates

Many NSF-style gates are passive.

Example:

```text id="cg13"
Player enters room
↓
Perception ≥ 7
↓
New description appears
```

No explicit check occurs.

---

# Hard Faculty Gates

Some content should require a threshold.

Example:

```text id="cg14"
Rhetoric ≥ 8
```

Otherwise the option never appears.

---

# Soft Faculty Gates

Alternative approach:

```text id="cg15"
Low faculty
→ weaker content

High faculty
→ richer content
```

This often feels better than hard exclusion.

---

# Belief gates

Beliefs are major content filters.

Example:

```json id="cg16"
{
  "thought": "belief_doctrine_a"
}
```

Unlocks:

* political observations
* ideological dialogue
* faction reactions
* alternative interpretations

---

# Belief-Based Worldviews

Critical design principle.

Beliefs should not merely provide bonuses.

They should unlock:

```text id="cg17"
Different ways of understanding reality.
```

This dramatically improves replayability.

---

# Item Gates

Items can unlock content.

Example:

```json id="cg18"
{
  "item": "Master Key"
}
```

or:

```json id="cg19"
{
  "equipped": "Police Uniform"
}
```

Items can function as social, not merely mechanical, keys.

---

# Tool Gates

Examples:

```text id="cg20"
Flashlight
Lockpick
Camera
Evidence Kit
```

Tools unlock interactions.

---

# Fact gates

One of NSF's most important hidden systems.

Example:

```json id="cg21"
{
  "factKnown": "victim_had_affair"
}
```

New content appears because the player knows something.

Not because they possess something.

---

# Fact-Based Gating

Examples:

```text id="cg22"
Learn secret
↓
New accusation dialogue appears
```

```text id="cg23"
Learn location
↓
New route becomes available
```

Facts themselves becomes progression.

---

# Time Gates

Content availability should depend on time.

Example:

```json id="cg24"
{
  "timeAfter": "18:00"
}
```

Possible uses:

```text id="cg25"
Actor Schedule
Store Hours
Night Events
Weather Events
Meetings
```

---

# Date Gates

Longer-term gating.

Example:

```text id="cg26"
Day 3 unlocks bridge.
```

or:

```text id="cg27"
Event only available after day 5.
```

---

# Relationship Gates

Content depends on social state.

Example:

```json id="cg28"
{
  "relationship": "Kim",
  "trust": 5
}
```

Unlocks:

```text id="cg29"
Personal conversation
Sensitive information
Unique scene
```

---

# Relationship Thresholds

Examples:

```text id="cg30"
Trust
Respect
Affinity
Suspicion
Fear
```

Different content should depend on different dimensions.

---

# Ideology gates

A defining NSF mechanic.

Example:

```json id="cg31"
{
  "politicalAlignment": "Communist"
}
```

or:

```json id="cg32"
{
  "communismScore": 10
}
```

---

# Ideological Content

Politics should unlock:

```text id="cg33"
Observations
Interpretations
Dialogue
Beliefs\nFaction Responses
```

The world should feel different.

---

# Conduct Gates

Conducts are another identity gate.

Examples:

```text id="cg34"
conduct_bold
conduct_humble
conduct_reckless
conduct_negligent
```

Content can react accordingly.

---

# Reputation Gates

Content may depend on reputation.

Examples:

```json id="cg35"
{
  "factionRep": {
    "faction_guild": 6
  }
}
```

This creates faction-specific experiences.

---

# World State Gates

Content depends on world conditions.

Examples:

```text id="cg36"
Door Open
Actor Alive
Strike Active
Investigation Progressed
```

World-state gates are among the most common.

---

# Location Gates

Certain interactions should only appear in specific locations.

Example:

```json id="cg37"
{
  "location": "Harbor"
}
```

This helps contextualize content.

---

# Event Gates

Events can unlock content.

Example:

```text id="cg38"
Mercenary Tribunal Occurred
↓
New dialogue becomes available
```

Events create narrative progression.

---

# Composite Gates

Most interesting content should use multiple requirements.

Example:

```json id="cg39"
{
  "faculty": "Empathy",
  "minimum": 6,
  "relationship": "Kim",
  "trust": 4,
  "timeAfter": "20:00"
}
```

Complex gates create nuanced experiences.

---

# Hidden Gates

Many gates should be invisible.

The player should not see:

```text id="cg40"
Requires Logic 7
```

for every interaction.

Sometimes the content simply appears.

This preserves immersion.

---

# Visible Gates

Some gates should be shown.

Examples:

```text id="cg41"
[Logic] option
[Authority] option
Requires Key
```

Visibility helps planning.

---

# Gate Failure

Failure should not always block progress.

Example:

```text id="cg42"
Content A unavailable
↓
Content B available
```

Alternative paths are critical.

---

# Redundancy Design

Important principle:

```text id="cg43"
Critical content
=
Multiple gates
```

Never lock major story progression behind one narrow requirement.

---

# Narrative Layer Gating

Even descriptions should be gated.

Example:

```text id="cg44"
Low Encyclopedia:
A statue.

High Encyclopedia:
A monument tied to a failed revolution.
```

Same object.

Different experience.

---

# Discovery Layer Gating

Objects can reveal information in stages.

```text id="cg45"
Visible
↓
Inspectable
↓
Interpretable
↓
Meaningful
```

Each stage can have different gates.

---

# Gate Priority

Recommended evaluation order:

```text id="cg46"
World State
Time
Location
Knowledge
Relationship
Beliefs\nPolitics
Faculties
Items
```

This keeps evaluation deterministic.

---

# Dynamic Re-Evaluation

The engine should constantly reevaluate gates.

Example:

```text id="cg47"
Belief acquired
↓
New dialogue appears immediately
```

No reload should be required.

---

# Content Variants

Prefer variant content over binary visibility.

Bad:

```text id="cg48"
See content
or
Do not see content
```

Better:

```text id="cg49"
Normal version

Enhanced version

Ideology version

Faculty version
```

This maximizes content reuse.

---

# Event Integration

Emit:

```text id="cg50"
GateUnlocked
GateLocked
ContentRevealed
RequirementSatisfied
RequirementLost
```

These events can trigger UI updates.

---

# Save Data Requirements

Serialize:

```text id="cg51"
Satisfied Gates
Unlocked Content
Known Facts
Ideology state
Relationship State
Belief State
```

Gating results must persist.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IGateService
{
    bool IsAllowed(string gateId, GateContext context);
    GateDenialReason GetDenialReason(string gateId, GateContext context);
}
```

## Domain model

```csharp
class GateRule { string Id; Condition[] Conditions; string DenialMessageKey; }
class GateContext { string ActorId; string LocationId; }
enum GateDenialReason { None, FacultyTooLow, FlagMissing, FactMissing, ConductThreshold }
```


# Relationship to Other Systems

The Gate sits above nearly every narrative system.

```text id="cg53"
Faculties
Beliefs\nItems
Politics
Relationships
Knowledge
Time
World State
      ↓
Gate
      ↓
Player Experience
```

It determines which version of the game the player actually sees.

---

# Recommended Internal Flow

```text id="cg54"
Player State Changes
          ↓
Gate Evaluation
          ↓
Content Unlocks
          ↓
New Discoveries
          ↓
New Choices
```

This loop constantly creates fresh opportunities.

---

# Most Important Insight

The Gate is the primary source of replayability.

It ensures that different characters encounter different worlds.

The engine should continuously answer:

```text id="cg55"
Why can this player see this?
Why can another player not?
What aspect of the character unlocked this content?
```

If implemented correctly, every build feels like a different investigation.

If implemented incorrectly, all players see nearly the same content and character identity becomes largely cosmetic.
