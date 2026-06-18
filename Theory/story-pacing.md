# 38. Story: Pacing — NSF Specification

> **Framework:** Reveal timing, scene sequencing, cooldowns, content density (when vs can).
> **Content pack:** Pacing rules per story beat, scene order content.
> Terminology: [Glossary](terminology-glossary.md)

## Purpose

The Pacing controls **when narrative content should occur**, not whether it is allowed to occur.

It answers:

```text id="p1a2b3"
When should this scene happen?
When is the player ready for this reveal?
When is repetition becoming fatigue?
When should tension rise or fall?
```

This is different from:

```text id="c4d5e6"
Rule Engine → "Can this happen?"
```

Pacing is about **timing, rhythm, and emotional structure**.

---

# Core Principle

Even valid content can be wrong at the wrong time.

Example:

```text id="f7g8h9"
Companion confession scene
```

May be:

```text id="i1j2k3"
Allowed by rules
Unlocked in database
Ready in scripting system
```

But still:

```text id="l4m5n6"
Too early emotionally
Too close to previous reveal
Placed after high-intensity event
```

So it should NOT trigger yet.

---

# Core Responsibilities

The pacing system manages:

```text id="o7p8q9"
Reveal timing
Mystery unfolding
Emotional rhythm
Scene spacing
Content repetition control
Intensity curves
Narrative breathing room
```

---

# Pacing vs Rule Engine

Clear separation:

```text id="r1s2t3"
Rule Engine → eligibility
Pacing System → timing
```

Example:

```text id="u4v5w6"
Rule Engine:
    metric_trust_companion > 4 → TRUE

Pacing System:
    WAIT 2 more in-game days
```

---

# Pacing State Model

Each narrative thread tracks pacing state:

```text id="x7y8z9"
PacingState
{
    last_scene_time
    intensity_level
    reveal_progress
    cooldowns
    density_score
}
```

---

# Reveal Pacing System

Controls how information is disclosed over time.

Example:

```text id="a1b2c3"
Mystery: "Who killed the victim?"
```

Reveals are staged:

```text id="d4e5f6"
Stage 1 → body discovered
Stage 2 → cause of death
Stage 3 → Inquiry subject introduced
Stage 4 → contradiction revealed
Stage 5 → truth exposed
```

---

# Reveal Curve

Each mystery has a progression curve:

```text id="g7h8i9"
slow burn → buildup → acceleration → climax → resolution
```

System ensures:

```text id="j1k2l3"
No skipping stages
No clustering reveals too tightly
No premature resolution
```

---

# Mystery Pacing

Mysteries must avoid overload:

Rules:

```text id="m4n5o6"
Limit simultaneous open mysteries
Stagger clue delivery
Balance certainty vs ambiguity
Control clue density per time unit
```

---

# Scene Sequencing System

Scenes are ordered dynamically based on pacing constraints.

Example:

```text id="p7q8r9"
Scene A (intense interrogation)
Scene B (quiet reflection)
Scene C (new lead discovery)
```

System ensures:

```text id="s1t2u3"
Contrast between scenes
Avoid back-to-back high intensity
Maintain narrative rhythm
```

---

# Intensity Curve System

Each scene has intensity value:

```text id="v4w5x6"
0.0 → calm
1.0 → extreme tension
```

Example:

```text id="y7z8a9"
interrogation = 0.8
walk_in_city = 0.3
breakthrough = 0.9
```

System enforces:

```text id="b1c2d3"
No sustained high intensity without cooldown
No flat long stretches without variation
```

---

# Cooldown System

Prevents repetition and fatigue.

Example:

```text id="e4f5g6"
Companion emotional scene played → cooldown 3 days
Faculty check fail state → cooldown 1 hour
Major reveal → cooldown until next act
```

---

# Content Density Control

Controls how much narrative content appears per time unit.

Example:

```text id="h7i8j9"
Too many:
    dialogues
    rolls
    reveals
in short time → overload
```

System regulates:

```text id="k1l2m3"
spacing between events
frequency of interactions
burst vs quiet cycles
```

---

# Narrative Fatigue Prevention

Detects when player is overloaded:

Signals:

```text id="n4o5p6"
Too many choices in short time
Repeated emotional spikes
Too many unresolved threads
```

Response:

```text id="q7r8s9"
Delay content
Insert ambient scenes
Trigger downtime interactions
```

---

# Emotional Pacing Integration

Pacing depends on emotional system:

Example:

```text id="t1u2v3"
High stress → slow pacing
Low engagement → increase event frequency
Post-reveal → cooldown phase
```

---

# Information Pacing Integration

From Info Flow:

Rules:

```text id="v4w5x6"
Don’t spread too many rumors at once
Delay faction knowledge updates
Stagger revelations across Actors
```

---

# Dialogue Pacing System

Dialogue must respect rhythm:

Rules:

```text id="y7z8a9"
Avoid long uninterrupted monologues
Insert response opportunities
Space emotional lines
```

---

# Roll Pacing

Faculty checks must be distributed:

Rules:

```text id="b1c2d3"
No consecutive high-stakes checks
Cooldown after failure streaks
Alternate tension levels
```

---

# Narrative Thread Interleaving

Multiple storylines are interwoven:

Example:

```text id="e4f5g6"
Main incident thread
+
Companion storyline
+
Ideological unrest subplot
```

System ensures:

```text id="h7i8j9"
Threads alternate
Not resolved simultaneously
Balanced attention distribution
```

---

# Scene Selection Algorithm

When multiple scenes are eligible:

System chooses based on:

```text id="k1l2m3"
priority
intensity balance
recency
player fatigue
story progression weight
```

---

# Priority vs Pacing Conflict

Sometimes:

```text id="n4o5p6"
Rule Engine says YES
Pacing System says WAIT
```

Resolution:

```text id="q7r8s9"
Pacing overrides timing
Rule Engine remains authoritative for eligibility
```

---

# Narrative Delay Buffers

Delays are intentional design tools:

Example:

```text id="t1u2v3"
Player discovers clue
→ no immediate explanation
→ delay 1–2 scenes
→ reveal context later
```

---

# Rhythm Model

Narrative pacing follows cycles:

```text id="v4w5x6"
Tension → Release → Calm → Build-up → Climax
```

System enforces balance across gameplay.

---

# Global vs Local Pacing

Two levels:

```text id="y7z8a9"
Global pacing → act-level structure
Local pacing → scene-level rhythm
```

---

# Act-Level Control

Acts have pacing goals:

```text id="b1c2d3"
Act 1 → introduction + mystery setup
Act 2 → escalation + fragmentation
Act 3 → convergence + resolution
```

---

# Local Scene Control

Within a scene:

```text id="e4f5g6"
dialogue spacing
interrupt timing
roll placement
```

---

# Anti-Spam Controls

Prevents:

```text id="h7i8j9"
Repeated bark lines
Repeated notifications
Repeated rolls
```

---

# Adaptive Pacing System

Advanced feature:

System adapts based on player behavior:

```text id="k1l2m3"
fast player → increase density
slow player → reduce pressure
explorative player → delay main progression
```

---

# Debugging Pacing

Tools must show:

```text id="n4o5p6"
scene timeline
intensity graph
cooldown timers
blocked scenes
density metrics
```

---

# Failure Modes

Avoid:

```text id="q7r8s9"
Too many reveals too fast
Long empty stretches
Unresolved thread accumulation
Mood inconsistency
```

---

# Performance Model

Must handle:

```text id="t1u2v3"
Continuous evaluation of narrative queues
Multiple concurrent story threads
Real-time adaptation to player actions
```

Optimizations:

```text id="v4w5x6"
event-driven updates
cached pacing state
thread-level aggregation
```

---

# Minimum Viable System

For a large-scale narrative RPG:

Implement:

```text id="y7z8a9"
Scene cooldown system
Intensity tracking
Reveal staging system
Content density control
Thread sequencing
Basic pacing state per narrative arc
Rule Engine integration
```

---

# Final Concept

The Pacing is the **invisible conductor of storytelling rhythm**.

It ensures that:

```text id="b1c2d3"
Reveals land at the right emotional moment
Scenes are spaced for impact, not just availability
Mysteries unfold gradually instead of collapsing
Player attention is guided rather than overwhelmed
Narrative tension rises and falls intentionally
```

In a large-scale narrative RPG, pacing is not content management.

It is **the structure that makes meaning feel intentional instead of accidental**.
