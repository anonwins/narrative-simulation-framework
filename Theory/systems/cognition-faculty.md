# 1–2. Cognition: Faculty & Character — NSF Specification

> **Framework:** Attribute groups, interpretive faculties, roll inputs, passive/active faculty behavior, character progression interfaces.
> **Content pack:** Faculty names, attribute labels, signature faculty, conduct presets, voice profiles.
> Terminology: [Glossary](../terminology-glossary.md)

This is not a content list of faculties. This is the **underlying architecture** required to build an NSF character engine.

The single most important thing to understand:

> In NSF, faculties are not stats.
>
> Faculties are autonomous interpreters of reality.

The engine should treat faculties simultaneously as:

1. Numerical values.
2. Dialogue actors.
3. Passive perception systems.
4. Narrative filters.
5. Roll resolution inputs.

Most RPGs only implement #1 and #5.

NSF's uniqueness comes from #2, #3, and #4.

---

# Core Character Architecture

```text
PlayerCharacter
 ├─ Attributes
 ├─ Faculties
 ├─ Beliefs
 ├─ Equipment
 ├─ TemporaryEffects
 ├─ Vitality
 ├─ Morale
 ├─ PoliticalIdentity
 ├─ PsychologicalState
 └─ Chronicle
```

Faculties sit at the center of everything.

---

# Attribute Layer

Top-level attributes:

```text
Intellect
Psyche
Physique
Motorics
```

Each attribute contains:

```text
6 faculties
```

Architecture:

```text
Attribute
 ├─ baseValue
 ├─ modifiedValue
 └─ faculties[]
```

---

# Faculty Layer

Each faculty has:

```text
Faculty
 ├─ baseValue
 ├─ modifiedValue
 ├─ voiceProfile
 ├─ passiveThresholds
 ├─ activeCheckModifier
 ├─ personalityTags
 └─ dialogueRules
```

The faculty is NOT merely:

```text
Logic = 8
```

Instead:

```text
Logic
 value = 8

 personality:
     analytical
     skeptical
     deductive

 voice:
     "This is inconsistent."
```

---

# The 24 Faculty Objects

The engine should not hardcode behavior.

Represent faculties as data.

Example:

```json
{
  "id": "logic",
  "attribute": "intellect",
  "voiceProfile": "logic_voice",
  "personalityTags": [
    "deduction",
    "analysis",
    "consistency"
  ]
}
```

The actual content lives elsewhere.

---

# Faculties Are Internal Actors

This is arguably the most important system.

Treat every faculty as:

```text
Internal Character
```

Architecture:

```text
Speaker
 ├─ Actor
 ├─ Narrator
 ├─ Faculty
 ├─ Belief
 └─ Item
```

Faculty dialogue uses the same infrastructure as Actor dialogue.

Example:

```text
LOGIC:
This doesn't add up.
```

The engine should not distinguish heavily between:

```text
actor_companion speaking
```

and

```text
Logic speaking
```

Both are dialogue sources.

---

# Faculty Voice Trigger System

Faculties do not speak constantly.

They speak when:

```text
Observation occurs
Contradiction appears
Relevant stimulus exists
Check succeeds
Check fails
Belief triggers
```

Architecture:

```text
Trigger
 ├─ context
 ├─ speaker
 ├─ condition
 └─ line
```

Example:

```json
{
  "speaker": "logic",
  "condition": {
    "faculty": "logic",
    "minimum": 5
  }
}
```

---

# Passive Roll

Passive checks are the secret engine of NSF.

Most RPGs:

```text
Player clicks object
```

NSF:

```text
Player walks near object
↓
Faculties inspect object
↓
Relevant faculty speaks
```

Architecture:

```text
World Object
 └─ PassiveChecks[]
```

Example:

```json
{
  "faculty": "perception",
  "difficulty": 12,
  "success":
      "notice_blood"
}
```

---

# Passive Roll

Whenever player enters context:

```text
For each passive check:

    FacultyValue + modifiers

    >= difficulty

        Trigger
```

No dice roll.

Pure threshold system.

---

# Active Roll

Traditional visible check.

Example:

```text
AUTHORITY

Convince actor_intimidating

42%
```

Architecture:

```text
ActiveCheck
 ├─ faculty
 ├─ difficulty
 ├─ modifiers
 ├─ retryPolicy
 └─ outcomes
```

---

# Check Formula

Simplified NSF formula:

```text
2D6
+
Faculty Value
+
Modifiers
>= Difficulty
```

The exact balancing can vary.

The important part:

```text
Randomness exists
Faculty matters
Modifiers matter
```

---

# Repeatable check Architecture

Retryable.

```text
WhiteCheck
```

Properties:

```json
{
  "retryable": true
}
```

Failure:

```text
Check closes
```

Can reopen if:

```text
Faculty increased
Relevant event occurred
Relevant belief unlocked
```

---

# Gated check Architecture

Permanent.

```json
{
  "retryable": false
}
```

Failure remains canon.

No retries.

---

# Failure-Forward Design

Critical requirement.

Traditional RPG:

```text
Failure
=
Content removed
```

NSF:

```text
Failure
=
Alternative content
```

Architecture:

```text
Check
 ├─ successNode
 └─ failureNode
```

Every check should contain both.

Never:

```text
Success → content

Failure → nothing
```

---

# Faculty Influence on Reality

This is what makes NSF unique.

Two players see different worlds.

Example:

Low Perception:

```text
Room
```

High Perception:

```text
Room
Blood stain
Scratch marks
Loose coin
```

Engine architecture:

```text
Perception Layer
```

World objects expose:

```json
{
  "visibilityFaculty": "perception",
  "difficulty": 10
}
```

---

# Narrative Filter System

Every faculty filters interpretation.

Example object:

```text
Broken Chair
```

Logic:

```text
Recently damaged.
```

Conceptualization:

```text
A monument to collapse.
```

faculty_instinct:

```text
Danger.
```

Same object.

Different interpretation.

Architecture:

```text
Object
 └─ Interpretation[]
```

Example:

```json
{
  "faculty": "logic",
  "line": "Recently damaged."
}
```

---

# Faculty Personality System

Every faculty should have:

```text
Persona
Goals
Biases
Emotional tone
```

Example:

Logic

```text
Goal:
Consistency

Bias:
Reason

Tone:
Clinical
```

faculty_instinct

```text
Goal:
Survival

Bias:
Threat

Tone:
Aggressive
```

Authority

```text
Goal:
Dominance

Bias:
Hierarchy

Tone:
Commanding
```

This drives writing generation.

---

# Faculty Conflict System

Faculties frequently disagree.

Example:

```text
EMPATHY:
She's scared.

AUTHORITY:
She's challenging you.
```

Architecture:

```text
DialogueEvent
 └─ MultipleFacultyResponses[]
```

Not:

```text
One faculty owns event.
```

Instead:

```text
Several faculties compete.
```

---

# Faculty Priority System

When multiple faculties qualify:

Need selection rules.

Example:

```text
Priority Score
=
Context Relevance
+
Faculty Value
+
Narrative Importance
```

Highest scores speak first.

---

# Signature Faculty System

Character creation:

```text
Choose one faculty
```

Effects:

```text
Bonus point
Higher growth ceiling
Narrative emphasis
```

Architecture:

```json
{
  "isSignatureFaculty": true
}
```

---

# Character Build Philosophy

No classes.

Instead:

```text
Character =
Faculty Distribution
```

Two players:

```text
Logic 10
Authority 2
```

vs

```text
Logic 2
Authority 10
```

Experience radically different content.

---

# Experience System

Experience grants:

```text
Faculty Points
```

Faculty points increase:

```text
Faculty Rank
```

Not:

```text
Strength
Dexterity
Level-based damage
```

---

# Faculty Cap System

Faculties have caps.

Example:

```text
Cap determined by parent attribute.
```

Architecture:

```text
MaximumFaculty
=
Attribute Value
+
Cap Modifiers
```

This prevents unlimited specialization.

---

# Modifier Pipeline

Final faculty:

```text
Base Faculty
+
Equipment
+
Beliefs\n+
Temporary Effects
+
Story beat Effects
=
Final Faculty
```

Must be recomputed dynamically.

---

# Vitality and Morale Architecture

NSF has two vitality systems.

```text
Vitality
Morale
```

Vitality derives from:

```text
Endurance
```

Morale derives from:

```text
Volition
```

Implementation:

```text
VitalityMax
=
f(Endurance)
```

```text
MoraleMax
=
f(Volition)
```

---

# Mental Damage System

Not all damage is physical.

Example:

```text
Embarrassment
Humiliation
Existential crisis
```

Can damage:

```text
Morale
```

Architecture:

```text
DamageType
 ├─ Physical
 └─ Psychological
```

---

# Belief Integration

Faculties and Beliefs are separate systems.

But beliefs modify faculties.

Example:

```json
{
  "facultyModifiers": {
    "rhetoric": 1,
    "authority": -1
  }
}
```

Beliefs become another modifier source.

---

# Dialogue Gating

Dialogue can require:

```text
Faculty Threshold
Roll
Belief
Attribute
Story beat state
```

Example:

```json
{
  "requiresFaculty": {
    "empathy": 6
  }
}
```

No roll.

Just availability.

---

# Content Generation Rule

Every dialogue node should support:

```text
Faculty Interruptions
```

Example:

```text
Actor speaks

↓
Empathy interjects

↓
Player choices appear
```

This is one of NSF's defining features.

---

# Save Data Requirements

Must serialize:

```text
Attributes
Faculties
Faculty XP
Faculty Caps
Signature Faculty
Vitality
Morale
Temporary Effects
Belief Modifiers
Triggered Passive Checks
```

---

# Minimal Engine Interfaces

```csharp
class FacultyGroupState
{
    int BaseValue;
    int ModifiedValue;
}

class FacultyState
{
    int BaseValue;
    int ModifiedValue;

    VoiceProfile Voice;

    PassiveRule[] PassiveRules;
}

interface RollDefinition
{
    Faculty Faculty;
    int Difficulty;

    Outcome Success;
    Outcome Failure;
}
```

---

# Most Important Insight

A NSF-style faculty system is **not a stat system**.

It is a combination of:

```text
Character System
+
Narrative AI System
+
Discovery
+
Dialogue
+
Progression System
```

The correct mental model is:

```text
Every faculty is a tiny character
living inside the player character's head,
continuously interpreting the world.
```

Once an engine is built around that principle, most of NSF's distinctive mechanics emerge naturally. Without that principle, you'll get a standard RPG with faculty rolls rather than a true NSF-style experience.
