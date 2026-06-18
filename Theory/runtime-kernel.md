# 40. Runtime: Kernel — NSF Specification

> **Framework:** Meta-coordination model—how all NSF systems interact in the closed simulation loop.
> **Content pack:** None (this is pure framework architecture).
> Terminology: [Glossary](terminology-glossary.md)

## Purpose

The Simulation Kernel is the **meta-system that defines how all narrative systems interact as a single living model**.

It is not a gameplay system.

It is not a feature.

It is:

```text id="s1a2b3"
The coordination model for the entire narrative engine
```

It defines how:

```text id="c4d5e6"
facts become dialogue
dialogue becomes relationships
relationships influence factions
factions trigger events
events mutate world state
world state gates content
gated content produces new facts
```

NSF organizes these responsibilities into modules: **Runtime**, **Cognition**, **Social**, **Simulation**, **Story**, **Ledger**, **Rules**, **Content**, and **Presentation**. See [System Catalog](index.md) for the full mapping.

---

# Core Principle

All narrative systems are **not independent systems**.

They are:

```text id="f7g8h9"
interdependent state transformations in a closed simulation loop
```

Nothing exists in isolation.

Everything feeds everything else.

---

# The Simulation Loop

At the highest level, the system runs as:

```text id="i1j2k3"
FACTS
  ↓
INTERPRETATION (Dialogue / Perception)
  ↓
SOCIAL STATE (Relationships)
  ↓
SYSTEM STATE (Factions / Institutions)
  ↓
WORLD EVENTS
  ↓
WORLD STATE UPDATE
  ↓
CONSTRAINTS (Gating / Rules / Pacing)
  ↓
NEW CONTENT UNLOCKED
  ↓
NEW FACTS GENERATED
  ↺
```

This loop runs continuously at different time scales.

---

# Layer 1: Facts Layer

Facts are atomic truth units.

Examples:

```text id="l4m5n6"
"Companion trusts the player"
"The subject was seen at location_warehouse"
"Time of death is 3:14 AM"
```

Facts are:

```text id="o7p8q9"
persistent
referenceable
system-readable
```

---

# Layer 2: Interpretation Layer (Dialogue)

Facts become **expressed meaning** through dialogue.

Example:

```text id="r1s2t3"
Fact: subject was seen at location_warehouse
↓
Dialogue: Actor mentions seeing someone at dock
```

Dialogue is not truth.

It is:

```text id="u4v5w6"
filtered perception of facts
```

---

# Layer 3: Relationship Layer

Interpretation modifies social bonds.

Example:

```text id="x7y8z9"
truth revealed → trust increases
lie detected → trust decreases
conflict → hostility increases
```

Structure:

```text id="a1b2c3"
RelationshipState
{
    trust
    fear
    respect
    dependency
}
```

---

# Layer 4: Faction Layer

Relationships scale into institutions.

Example:

```text id="d4e5f6"
Actor trust in player → faction reputation
individual actions → institutional response
```

Factions maintain:

```text id="g7h8i9"
control
influence
stance toward player
internal stability
```

---

# Layer 5: Event Layer

Factions and relationships generate events.

Example:

```text id="j1k2l3"
trust breakdown → interrogation event
faction instability → riot event
information leak → investigation event
```

Events are:

```text id="m4n5o6"
triggered consequences of accumulated state
```

---

# Layer 6: World State Layer

Events permanently modify simulation state.

Example:

```text id="p7q8r9"
district unsafe
faction weakened
Actor relocated
new patrol patterns
```

World state includes:

```text id="s1t2u3"
geography
politics
economy
social stability
accessibility
```

---

# Layer 7: Gate Layer

World state controls what is possible.

Example:

```text id="v4w5x6"
If district unsafe → some scenes locked
If faction hostile → dialogue branches change
If trust high → new story nodes unlocked
```

This layer enforces:

```text id="y7z8a9"
what content exists for the player at any moment
```

---

# Layer 8: Content Generation Layer

Gating produces new narrative material.

Example:

```text id="b1c2d3"
new dialogue unlocked
new quests appear
new Beliefs triggered
new events scheduled
```

This feeds back into facts.

---

# Core Simulation Properties

The system must maintain:

---

## 1. Causality

```text id="e4f5g6"
No effect without cause
No event without state precedent
```

---

## 2. Consistency

```text id="h7i8j9"
All systems agree on shared truth
No contradictory world states
```

---

## 3. Traceability

```text id="k1l2m3"
Every outcome must be explainable
Every change has origin chain
```

---

## 4. Propagation

```text id="n4o5p6"
Small facts ripple into large consequences
```

---

## 5. Stability

```text id="q7r8s9"
Simulation must avoid feedback collapse
Avoid infinite escalation loops
```

---

# Time Model

Simulation operates across time scales:

```text id="t1u2v3"
instant → dialogue reaction
minutes → emotional shift
hours → event triggers
days → faction movement
weeks → world state change
```

---

# State Graph Model

Everything is a graph:

```text id="v4w5x6"
Nodes → entities / facts / events
Edges → influence / causality / dependency
```

Example:

```text id="y7z8a9"
Fact → Relationship → Faction → Event → World State
```

---

# Feedback Loops

The system is recursive:

```text id="b1c2d3"
World state changes → new facts → new dialogue → new relationships → new world state
```

Key danger:

```text id="e4f5g6"
uncontrolled escalation loops
```

Must be regulated by pacing + rule systems.

---

# System Boundaries

Each subsystem has responsibility:

```text id="h7i8j9"
Rule Engine → validity
Pacing → timing
Emotion → internal state
Audio/UI → expression layer
Simulation Core → coordination
```

---

# Conflict Resolution

When systems disagree:

Example:

```text id="k1l2m3"
Rule Engine: event allowed
Pacing: event delayed
Emotion: character unstable
```

Resolution hierarchy:

```text id="n4o5p6"
Pacing + Emotional constraints can delay
Rule Engine defines legality
Simulation Core arbitrates timing conflicts
```

---

# Emergence Principle

Complex narrative emerges from simple rules:

```text id="q7r8s9"
No scripted global story required
Only local rules + propagation
```

---

# Determinism vs Variability

System supports both:

```text id="t1u2v3"
deterministic core simulation
+ probabilistic emotional/social variation
```

---

# Debugging the Simulation

Must expose:

```text id="v4w5x6"
full causal chain tracing
state snapshots
event history
relationship evolution graphs
faction state timelines
```

---

# Performance Model

Must scale to:

```text id="y7z8a9"
thousands of facts
hundreds of Actors
multiple factions
continuous event simulation
```

Optimization strategies:

```text id="b1c2d3"
event-driven updates
lazy evaluation of distant systems
aggregation of low-priority simulation
```

---

# Minimum Viable Simulation Core

For a large-scale narrative RPG:

Implement:

```text id="e4f5g6"
fact system
relationship system
faction system
event system
world state system
content gating system
causal propagation system
time-based simulation ticks
```

---

# Final Concept

The Simulation Kernel is the **invisible machine beneath everything**.

It ensures:

```text id="h7i8j9"
All narrative systems are unified
All changes have consequences
All content emerges from simulation
The world behaves consistently over time
Player actions propagate meaningfully
```

In a large-scale narrative RPG, this is not a feature.

It is the **reason the world feels like it exists when the player is not looking at it**.
