# 32. Content: Script — NSF Specification

> **Framework:** Dialogue DSL, event scripting, checks, conditions, effects, state mutations.
> **Content pack:** Scripted content files authored in the DSL.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The NSF Script is the **author-facing logic layer** used to define interactive narrative behavior in a structured, readable format.

It bridges:

```text id="q8x7x1"
Content Store
+
Rule Engine
+
Gameplay Systems
```

Without it:

```text id="a1m2k3"
Content becomes hardcoded in code
or
Rules become unmanageable JSON blobs
or
Designers cannot author complex behavior safely
```

With it:

```text id="b9c0d1"
Dialogue, quests, rolls, and consequences become scriptable data
```

---

# Core Principle

The scripting language is not general-purpose programming.

It is:

```text id="c2d3e4"
A constrained narrative DSL
```

Optimized for:

```text id="d5e6f7"
Dialogue flow
Condition checks
State changes
Event triggers
Narrative consequences
```

Not for:

```text id="e8f9g0"
Physics
Rendering
AI simulation
General computation
```

---

# High-Level Structure

Every script is composed of:

```text id="h1i2j3"
NODE
    CONDITIONS
    CHECKS
    TEXT
    CHOICES
    EFFECTS
    TRANSITIONS
```

Example:

```text id="k4l5m6"
NODE: kim_intro

TEXT:
    "Good morning."

CONDITIONS:
    Day >= 2

CHOICES:
    greet_kim
    ignore_kim
```

---

# NODE System

A NODE is the basic unit of narrative flow.

Example:

```text id="n7o8p9"
NODE kim_intro_01
```

Each node contains:

```text id="q1r2s3"
Dialogue
Logic
Branches
Effects
```

Nodes form graphs, not linear scripts.

---

# Dialogue DSL

Dialogue is declared inside nodes.

Example:

```text id="t4u5v6"
NODE kim_intro

TEXT:
    "We should get moving."
```

Speaker is optional:

```text id="w7x8y9"
SPEAKER: Kim
TEXT: "We should get moving."
```

---

# Choice Declaration

Choices define branching paths.

Example:

```text id="z1a2b3"
CHOICE greet_kim
    TEXT: "Good morning."
    GO_TO: kim_reply_positive
```

```text id="c4d5e6"
CHOICE ignore_kim
    TEXT: "..."
    GO_TO: kim_reply_negative
```

---

# Conditions System

Conditions gate content.

Example:

```text id="f7g8h9"
CONDITIONS:
    metric_trust_companion > 3
    Day >= 2
```

Supports boolean logic:

```text id="i1j2k3"
AND / OR / NOT
```

Example:

```text id="l4m5n6"
CONDITIONS:
    (metric_trust_companion > 3 OR Authority > 5)
    AND NOT Arrested
```

---

# Rolls

Faculty checks are first-class DSL constructs.

Example:

```text id="o7p8q9"
CHECK Logic >= 6
```

Branching result:

```text id="r1s2t3"
IF SUCCESS:
    GO_TO logic_success_node

IF FAILURE:
    GO_TO logic_failure_node
```

---

# Dice Resolution (Optional Layer)

For RPG-style checks:

```text id="u4v5w6"
CHECK Empathy >= 5
ROLL 2d6 + Empathy
```

Outcomes:

```text id="x7y8z9"
Critical Success
Success
Failure
Critical Failure
```

Each maps to different nodes or effects.

---

# Effects System

Effects mutate game state.

Example:

```text id="a1b2c3"
EFFECTS:
    SET Flag BodyExamined = true
    ADD Trust(Kim, +1)
    UNLOCK Scene kim_confession
```

---

# State Mutations

Supported mutations:

```text id="d4e5f6"
SET flag
ADD value
SUB value
TOGGLE boolean
APPEND list
REMOVE item
REVEAL information
LOCK content
UNLOCK content
```

Example:

```text id="g7h8i9"
EFFECTS:
    ADD Money +20
    SET QuestStage = 2
```

---

# Event Triggering

Scripts can trigger system events.

Example:

```text id="j1k2l3"
EFFECTS:
    TRIGGER Event "PoliceBackupArrives"
```

This connects to:

```text id="m4n5o6"
Rule Engine
World Simulation
AI Systems
```

---

# Conditional Text

Text can vary based on state.

Example:

```text id="p7q8r9"
TEXT:
    IF metric_trust_companion > 5:
        "I trust you on this."
    ELSE:
        "We need more evidence."
```

---

# Inline Conditions

Short conditions inside dialogue:

```text id="s1t2u3"
TEXT:
    "You {IF Arrested: 'look guilty'} {ELSE: 'look tired'}."
```

---

# Dialogue Flow Control

Control commands:

```text id="v4w5x6"
GO_TO node_id
RETURN
END
JUMP node_id
```

Example:

```text id="y7z8a9"
CHOICE:
    TEXT "Leave"
    GO_TO end_scene
```

---

# Scene Scripting

Scenes are scripted sequences of nodes.

Example:

```text id="b1c2d3"
SCENE kim_rooftop

START kim_intro_01
```

Supports:

```text id="e4f5g6"
Camera control
Animation triggers
Dialogue sequencing
Audio cues
```

---

# Animation Hooks

Example:

```text id="h7i8j9"
EFFECTS:
    PLAY Animation Kim_Nod
    PLAY SFX radio_static
```

---

# Audio Hooks

Example:

```text id="k1l2m3"
EFFECTS:
    PLAY Voice kim_intro_line_01
    PLAY Music tension_theme
```

---

# Variable System

Scripts access variables:

```text id="n4o5p6"
metric_trust_companion
Day
Money
Location
StoryBeatState
Flags
Information
Faculties
```

Example:

```text id="q7r8s9"
IF Money < 10
```

---

# Global vs Local State

Two scopes:

```text id="t1u2v3"
GLOBAL:
    world state

LOCAL:
    scene state
```

Example:

```text id="w4x5y6"
LOCAL: dialogue branch only
GLOBAL: story_beat_progression
```

---

# Script Modularity

Scripts are reusable components.

Example:

```text id="z7a8b9"
CALL check_kim_trust
```

Instead of duplicating logic.

---

# Script Inheritance (Optional)

Advanced:

```text id="c1d2e3"
BASE NODE
    reused structure
```

Derived nodes extend behavior.

---

# Fail-Safe Handling

If conditions fail:

```text id="f4g5h6"
GO_TO fallback_node
```

Never break narrative flow.

---

# Debug Mode

Every script should support tracing:

```text id="i7j8k9"
NODE: kim_intro
FAILED CONDITION: metric_trust_companion > 3
CURRENT VALUE: 1
```

---

# Execution Model

Scripts are interpreted, not compiled.

Flow:

```text id="l1m2n3"
Load Node
Evaluate Conditions
Run Rolls
Present Text
Await Choice
Apply Effects
Transition Node
```

---

# Integration with Rule Engine

The scripting language does NOT replace the Rule Engine.

Instead:

```text id="o4p5q6"
Script = authored narrative logic
Rule Engine = global systemic enforcement
```

Example:

```text id="r7s8t9"
Script:
    unlock dialogue option

Rule Engine:
    verifies if allowed
```

---

# Integration with Content Database

Scripts reference content IDs:

```text id="u1v2w3"
Actor: actor_companion
story_beat: section_primary
Belief: belief_doctrine_a
```

No raw strings.

---

# Error Handling

If a script references invalid content:

```text id="x4y5z6"
ERROR:
Missing node kim_reply_positive
```

System must fail safely or fallback.

---

# Tooling Requirements

A usable scripting system requires:

```text id="a7b8c9"
Syntax validation
Autocomplete
Graph visualization
Debug step-through
State inspector
Reference checker
```

Without tooling, DSLs collapse under complexity.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
class ScriptCompiler
{
    ScriptCompileResult Compile(string source, string filePath);
    ScriptCompileResult CompileFile(string nsfFilePath);
}
```

## Domain model

```csharp
class ScriptNode { string Id; ScriptCondition[] Conditions; ScriptEffect[] Effects; }
class ScriptCompileResult { bool Success; ScriptNode[] Nodes; string[] Errors; }
class ScriptCondition { string Expression; }
class ScriptEffect { string Action; string Target; }
```

# Minimum Viable System

For large-scale narrative RPG:

Implement:

```text id="d1e2f3"
NODE system
Dialogue text system
CHOICE branching
CONDITIONS engine
SKILL CHECK system
EFFECTS system
STATE mutation
FLOW control (GO_TO, END)
VARIABLE access
DEBUG tracing
```

---


# Final Concept

The NSF Script is the **bridge between writers and simulation**.

It allows designers to write:

```text id="g4h5i6"
"If the player embarrasses actor_companion and fails an empathy check, actor_companion becomes distant for 2 days and refuses rooftop conversations."
```

without touching code, without hardcoding logic, and without breaking the Rule Engine.

It is the layer that makes NSF-style narrative complexity *actually buildable*.
