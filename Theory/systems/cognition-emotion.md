# 37. Cognition: Emotion — NSF Specification

> **Framework:** Mood, stress, confidence, shame; influence on narration, checks, belief acquisition.
> **Content pack:** Emotional reaction content, threshold definitions.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The Emotion tracks the **internal affective condition of characters (and optionally the player)** over time.

It answers:

```text id="e1a2b3"
How does this character feel right now?
How does that affect what they say, think, or do?
How does emotion change future outcomes?
```

In An NSF-like RPG, emotion is not flavor text.

It is:

```text id="c4d5e6"
A modifier layer on top of narrative logic
```

---

# Core Principle

Emotions are **state variables that influence all narrative systems**.

They are not dialogue decorations.

They affect:

```text id="f7g8h9"
Dialogue tone
Faculty check outcomes
Belief unlocks
Rule engine conditions
Audio behavior
UI presentation
Actor reactions
```

---

# High-Level Model

Each entity has an emotional profile:

```text id="i1j2k3"
CharacterEmotionState
```

Example:

```text id="l4m5n6"
Kim:
    stress = 0.4
    confidence = 0.8
    irritation = 0.2
    trust = 0.6
```

---

# Core Emotional Dimensions

The system tracks structured emotional axes:

```text id="o7p8q9"
Stress
Confidence
Shame
Excitement
Fear
Anger
Sadness
Trust
Curiosity
Fatigue
```

Each value typically ranges:

```text id="r1s2t3"
0.0 → 1.0
```

or:

```text id="u4v5w6"
-1.0 → +1.0
```

---

# Emotion Structure

```text id="x7y8z9"
EmotionState
{
    stress
    confidence
    shame
    excitement
    fear
    anger
    sadness
    trust
    fatigue
}
```

---

# Emotional Triggers

Emotions change through events:

```text id="a1b2c3"
Dialogue outcomes
Faculty check success/failure
Information discovery
Violence
Time passing
Social interaction
Rule engine events
Audio/visual interrupts
```

---

# Example Emotive Updates

### Failure event:

```text id="d4e5f6"
FAIL roll:
    stress +0.2
    confidence -0.3
    shame +0.4
```

---

### Success event:

```text id="g7h8i9"
SUCCESS:
    confidence +0.2
    stress -0.1
    excitement +0.3
```

---

# Emotional Influence on Dialogue

Dialogue changes based on emotional state:

```text id="j1k2l3"
IF stress > 0.7:
    "He speaks quickly, avoiding eye contact."
```

---

Example:

```text id="m4n5o6"
IF shame > 0.6:
    unlock apologetic dialogue options
```

---

# Emotional Dialogue Modifiers

Same line, different tone:

```text id="p7q8r9"
Base:
    "I don't know."

High confidence:
    "I don't know."

High stress:
    "I-I don't know..."
```

---

# Emotional Roll Modifiers

Emotions influence probability:

```text id="s1t2u3"
stress ↑ → logic penalty
confidence ↑ → authority bonus
shame ↑ → social faculty penalty
excitement ↑ → perception bonus
```

---

Example:

```text id="v4w5x6"
Logic Check:
    base = 6
    stress = 0.8 → -1 modifier
```

---

# Emotional Belief Interaction

Emotions unlock beliefs:

```text id="y7z8a9"
IF shame > 0.6:
    unlock belief: "self_loathing_loop"
```

---

Beliefs also modify emotions:

```text id="b1c2d3"
Belief equipped:
    +confidence
    -stress
```

---

# Emotional Feedback Loop

Core system behavior:

```text id="e4f5g6"
Event → Emotion change → Belief change → Dialogue change → New event conditions
```

This creates emergent narrative behavior.

---

# Emotional Memory

Emotions can persist as memory traces:

```text id="h7i8j9"
"Remembered humiliation"
"Recent success"
"Traumatic encounter"
```

These act as long-term modifiers.

---

# Emotional Decay System

Emotions naturally decay over time:

```text id="k1l2m3"
stress → slowly returns to baseline
confidence → stabilizes after events
anger → cools down
```

Decay rates are tunable per character.

---

# Emotional Personality Profiles

Characters define emotional tendencies:

```text id="n4o5p6"
Kim:
    low anger gain
    moderate stress response
    high confidence stability
```

```text id="q7r8s9"
Player character:
    high volatility
    rapid emotional shifts
```

---

# Emotional Threshold Effects

Certain behaviors trigger at thresholds:

```text id="t1u2v3"
IF stress > 0.9:
    trigger panic dialogue
    reduce faculty effectiveness
```

---

```text id="w4x5y6"
IF confidence < 0.2:
    unlock self-doubt Beliefs\n```

---

# Emotional Influence on Rule Engine

Emotions can be conditions:

```text id="z7a8b9"
IF stress > 0.7 AND Day >= 3:
    unlock breakdown scene
```

---

# Emotional Influence on Information System

Emotion affects perception reliability:

```text id="c1d2e3"
high stress → misinterpretation chance
high confidence → stronger belief in rumors
```

---

# Emotional Influence on Audio System

Emotions drive sound design:

```text id="f4g5h6"
stress → heartbeat audio
fear → low drone
confidence → stable tone
shame → muffled audio filter
```

---

# Emotional Influence on UI System

UI reflects internal state:

```text id="i7j8k9"
high stress → shaking UI
low confidence → dimmed choices
high excitement → bright highlights
```

---

# Emotional Conflict System

Characters can have conflicting emotions:

```text id="l1m2n3"
confidence high
fear also high
```

This creates:

```text id="o4p5q6"
hesitation behavior
contradictory dialogue options
unstable decision-making
```

---

# Group Emotion (Optional)

Groups or factions can share emotional averages:

```text id="r7s8t9"
Crowd:
    anger = 0.8
    fear = 0.3
```

Used for:

```text id="u1v2w3"
Riots
Crowd behavior
Faction tension
Atmospheric shifts
```

---

# Emotion Storage

Stored per entity:

```text id="x4y5z6"
EmotionStateSnapshot
{
    entity_id
    timestamp
    emotion_values
}
```

Optional history tracking enables analytics and replay systems.

---

# Emotional Events

Standardized emotional events:

```text id="a7b8c9"
OnHumiliation
OnSuccess
OnThreat
OnRevelation
OnLoss
OnSupport
```

Each maps to emotion changes.

---

# Debugging Emotional Systems

Tools must show:

```text id="d1e2f3"
current emotion values
emotion history timeline
trigger sources
rule influences
dialogue modifiers
roll modifiers
```

---

# Performance Model

Must support:

```text id="g4h5i6"
Frequent updates per Actor
Real-time dialogue evaluation
Large group emotional tracking
Cross-system influence propagation
```

Optimizations:

```text id="j7k8l9"
Event-based updates
Cached emotion vectors
Decay batching
Selective simulation for off-screen Actors
```

---

# Failure Modes

System must avoid:

```text id="m1n2o3"
Emotion explosion (unstable feedback loops)
Over-amplified penalties
Unreadable dialogue states
Excessive randomness
```

---

# Minimum Viable System

For a large-scale narrative RPG:

Implement:

```text id="p4q5r6"
Emotion state per character
Core emotion axes
Event-driven updates
Dialogue influence hooks
Faculty check modifiers
Belief system integration
Decay system
Rule engine integration
```

---

# Final Concept

The Emotion is the **invisible psychological layer of the narrative engine**.

It ensures that:

```text id="s7t8u9"
Characters react like people, not scripts
Dialogue changes based on internal state
Failures have emotional consequences
Success has psychological weight
The world feels psychologically continuous
```

In a large-scale narrative RPG, emotion is not narration.

It is **stateful simulation of mind** running alongside gameplay systems.
