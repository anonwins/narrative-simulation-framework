# 39. Story: Outcome — NSF Specification

> **Framework:** Thread/relationship/ideology/conduct outcome synthesis, epilogue generation.
> **Content pack:** Ending branches, epilogue text, outcome weights.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The Outcome determines **how the entire narrative resolves and is summarized for the player**.

It answers:

```text id="e1a2b3"
What ultimately happened?
Who did the player become?
What changed in the world?
What remained unresolved?
```

In An NSF-like RPG, the ending is not a single cutscene.

It is:

```text id="c4d5e6"
A synthesis of all prior narrative systems
```

---

# Core Principle

Endings are not authored as single scripts.

They are:

```text id="f7g8h9"
Aggregated outcomes from many systems over time
```

Including:

```text id="i1j2k3"
Rule Engine state
Emotion
Pacing history
Info Flow history
Relationship systems
Story beat resolution states
Conduct / identity systems
```

---

# High-Level Structure

The system produces:

```text id="l4m5n6"
ENDING = FUNCTION(all_game_state)
```

Result:

```text id="o7p8q9"
Final narrative report
Epilogue scenes
Character fate summaries
World state evolution
Identity classification
```

---

# Core Outcome Categories

The ending system aggregates:

```text id="r1s2t3"
Thread outcomes
Relationship outcomes
Ideological outcomes
Economic outcomes
Personal identity outcomes
World state outcomes
```

---

# Thread outcome evaluation

## Purpose

Determines resolution of main narrative cases.

Example:

```text id="u4v5w6"
Was the incident solved?
Was the truth discovered?
Was the wrong ThreadSubject accused?
```

---

## Structure

```text id="x7y8z9"
CaseOutcome
{
    solved: true/false
    accuracy: partial/full/incorrect
    evidence_quality
    final_accusation
}
```

---

## Possible Results

```text id="a1b2c3"
Correct resolution
Incorrect conviction
Unresolved thread
Partial truth discovered
```

---

# Relationship Outcomes

Tracks emotional and social evolution.

Example:

```text id="d4e5f6"
Companion relationship:
    trust high → loyal partnership ending
    trust low → professional distance ending
```

---

## Structure

```text id="g7h8i9"
RelationshipOutcome
{
    actor_id
    trust_level
    final_state
    last_interaction_quality
}
```

---

## Outcome Types

```text id="j1k2l3"
ally
neutral
estranged
betrayed
lost
romantic (optional system extension)
```

---

# Ideological outcomes

Tracks systemic world consequences.

Example:

```text id="m4n5o6"
faction_guild stronger or weaker
Corporation influence expanded
State authority stabilized or collapsed
```

---

## Structure

```text id="p7q8r9"
PoliticalOutcome
{
    faction_id
    influence_delta
    control_state
}
```

---

# Conduct / Identity Outcome System

Unique narrative-simulation feature.

Determines **what kind of person the player became**.

Example conduct profiles:

```text id="s1t2u3"
Lawful outcome
Chaotic failure outcome
Ideological Radical
Compromised Authority Figure
```

---

## Structure

```text id="v4w5x6"
IdentityProfile
{
    traits
    dominant_emotions
    decision_patterns
    moral_alignment_vector
}
```

---

## Derived From:

```text id="y7z8a9"
choices
faculty usage
emotional state
rule engine flags
Belief
```

---

# Ending Generation System

Endings are **procedurally assembled narrative reports**.

Not prewritten scripts.

---

## Generation Flow

```text id="b1c2d3"
Collect all systems state
    ↓
Score outcome categories
    ↓
Select narrative modules
    ↓
Assemble epilogue scenes
    ↓
Render final sequence
```

---

# Epilogue Generation

Epilogues describe aftermath per domain.

Example:

```text id="e4f5g6"
Kim's future
City political shift
Player reputation
Thread legacy
```

---

## Structure

```text id="h7i8j9"
EpilogueSegment
{
    subject
    narrative_summary
    outcome_variants
}
```

---

## Example Segments

```text id="k1l2m3"
actor_companion → career trajectory
region_primary → political stability
The thread → public interpretation
The player → identity summary
```

---

# Outcome Weighting System

Not all decisions matter equally.

System assigns weights:

```text id="n4o5p6"
major decisions → high weight
minor dialogue → low weight
repeated behaviors → cumulative weight
```

---

# Narrative Aggregation Model

The ending is computed from:

```text id="q7r8s9"
Weighted sum of:
    thread outcomes
    emotional history
    relationship states
    faction influence
    identity markers
```

---

# Temporal Compression

The system compresses time:

```text id="t1u2v3"
Months/years → summarized in narrative form
```

Example:

```text id="v4w5x6"
"Over the following years, the city..."
```

---

# Branching Ending System

Endings are not single outcomes.

They are:

```text id="y7z8a9"
clusters of outcome segments
```

Example:

```text id="b1c2d3"
Ending A: stable resolution
Ending B: institutional collapse
Ending C: personal downfall
Ending D: ideological transformation
```

---

# Dynamic Ending Composition

System selects:

```text id="e4f5g6"
Which Actors appear
Which factions are referenced
Which mysteries are resolved
Which themes are emphasized
```

---

# Narrative Consistency Check

Before final rendering:

System validates:

```text id="h7i8j9"
No contradictory outcomes
No missing thread resolution
No broken character arcs
```

---

# Emotional Closure System

Endings must resolve emotional arcs:

Example:

```text id="k1l2m3"
stress resolved or unresolved
shame acknowledged or suppressed
confidence restored or broken
```

---

# Information Resolution System

From Info Flow:

```text id="n4o5p6"
Which rumors became truth
Which secrets remained hidden
Which leaks shaped world perception
```

---

# Relationship Closure System

Ensures each major Actor has closure:

```text id="q7r8s9"
final interaction state
last emotional exchange
long-term relationship projection
```

---

# Ideological world state collapse

Aggregates faction simulation:

```text id="t1u2v3"
who gained power
who lost influence
what systems destabilized
```

---

# Coping with Incomplete Content

If systems are unfinished:

```text id="v4w5x6"
fallback epilogue fragments
generic outcome templates
neutral summaries
```

---

# Ending Tone System

Tone depends on global state:

```text id="y7z8a9"
hopeful
tragic
ambiguous
absurd
bureaucratic
melancholic
```

---

# Multi-Layer Ending Structure

Endings are composed of layers:

```text id="b1c2d3"
Layer 1 → thread resolution
Layer 2 → personal identity
Layer 3 → relationships
Layer 4 → political world
Layer 5 → thematic summary
```

---

# Replayability System

Endings should vary significantly:

```text id="e4f5g6"
different identity profiles
different thread outcomes
different faction control states
```

---

# Debugging Ending System

Must show:

```text id="h7i8j9"
all contributing variables
weighted scores
selected outcome branches
discarded alternatives
```

---

# Failure Modes

Avoid:

```text id="k1l2m3"
flat generic ending
unresolved main thread
contradictory world states
emotionally incoherent summaries
```

---

# Performance Model

Must handle:

```text id="n4o5p6"
full-state aggregation
multi-system evaluation
branch selection
text generation
```

---


# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IOutcomeService
{
    OutcomeSynthesis Synthesize(OutcomeContext context);
    string GetEndingId(OutcomeContext context);
}
```

## Domain model

```csharp
class OutcomeContext { IReadOnlyStoryStateView Story; IReadOnlyConductView Conduct; IReadOnlyThreadView Threads; }
class OutcomeSynthesis { string EndingId; string SummaryKey; }
```

# Minimum Viable System

For a large-scale narrative RPG:

Implement:

```text id="q7r8s9"
Thread outcome evaluation
Relationship outcome aggregation
Ideology state summary
Identity/cotype system
Epilogue generator
Weighted scoring system
Narrative aggregation pipeline
```

---

# Final Concept

The Outcome is the **final compression of the entire simulation into meaning**.

It ensures that:

```text id="t1u2v3"
Every system contributes to closure
Player actions accumulate into identity
World changes reflect systemic behavior
Narrative feels consequential, not episodic
```

In a large-scale narrative RPG, the ending is not a cutscene.

It is **the final report generated by the entire narrative machine running over the player’s history**.
