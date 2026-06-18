# 20. Cognition: Roll — NSF Specification

> **Framework:** 2d6 mechanics, repeatable/gated checks, crits, modifiers, check UI.
> **Content pack:** DC values, faculty assignments, check failure/success copy.
> Terminology: [Glossary](../terminology-glossary.md)

This system determines **whether a roll succeeds, fails, or opens a new branch of play**.

In a NSF-style game, checks are not just gates.

They are:

* uncertainty generators
* narrative forks
* character-expression devices
* failure-content triggers
* probability-based revelation points

A good check system does not merely ask:

```text id="cr1"
Can the player pass?
```

It asks:

```text id="cr2"
What happens when the player tries?
```

---

# Core Principle

Checks should be:

```text id="cr3"
Visible
Contextual
Modifiable
Replayable where appropriate
Failure-forward
Narratively meaningful
```

The system must support both:

* **active checks** the player can see and choose
* **passive checks** the engine resolves automatically

---

# Core Architecture

```text id="cr4"
RollService
 ├─ ActiveChecks
 ├─ PassiveChecks
 ├─ WhiteChecks
 ├─ RedChecks
 ├─ DifficultyModel
 ├─ ModifierStack
 ├─ RollResolver
 ├─ OutcomeGraph
 └─ RetryRules
```

---

# Core Model

A check should be a structured object.

```csharp id="cr5"
class FacultyRoll
{
    string Id;
    string FacultyId;
    int Difficulty;
    CheckType Type;
    bool Retryable;

    Outcome SuccessOutcome;
    Outcome FailureOutcome;
}
```

---

# Check Types

At minimum, the engine should support:

```text id="cr6"
Active Check
Passive Check
Repeatable check
Gated check
Hidden Check
Forced Check
Conditional Check
```

---

# Active Checks

These are checks the player sees and chooses.

Example:

```text id="cr7"
[AUTHORITY] Convince The inquiry subject to cooperate.
```

Active checks should show:

* faculty used
* estimated chance
* whether failure is final or retryable
* what state changes are at stake

---

# Passive Checks

These happen automatically.

Example:

```text id="cr8"
Perception notices a hidden clue.
```

No button press is needed.

Passive checks are one of NSF's most important discovery systems.

---

# Repeatable checks

Repeatable checks are retryable.

Typical behavior:

* fail once
* become locked
* can reopen later if conditions change

Reopening conditions may include:

* faculty increase
* new belief
* new item
* new clue
* story progression

---

# Gated checks

Gated checks are not retryable.

Failure is permanent for that attempt.

This is important because failure becomes part of the story.

The player does not get to “undo” every mistake.

---

# Check Formula

The engine should support a transparent probability calculation.

A practical implementation can be:

```text id="cr9"
Final Score
=
Faculty
+
Equipment
+
Beliefs\n+
Temporary Effects
+
Context Modifiers
+
Roll
```

Then compare against difficulty.

---

# Dice / Randomness

NSF-style checks often use randomness to create uncertainty.

The engine should support:

```text id="cr10"
2d6 or equivalent roll
```

The exact dice model can vary, but the key properties are:

* faculty matters more than randomness
* randomness still matters
* modifiers can shift the odds
* failure remains narratively useful

---

# Modifier Stack

Checks should never depend on only one value.

Recommended modifier sources:

```text id="cr11"
Base Faculty
Clothing
Belief
Consumables
Current Mood / Damage State
Story beat state
Dialogue Context
Location Context
Relationship Context
Temporary Buffs
```

The system should calculate these in a deterministic order.

---

# Deterministic Resolution Order

Recommended order:

```text id="cr12"
1. Base faculty
2. Attribute / faculty caps
3. Equipment
4. Beliefs\n5. Consumables / temporary effects
6. World context modifiers
7. Roll
8. Success/failure threshold
9. Outcome selection
```

This keeps the engine predictable and debuggable.

---

# Difficulty Model

Difficulty should be data-driven.

Example levels:

```text id="cr13"
Trivial
Easy
Medium
Hard
Formidable
Impossible
```

Or a numeric scale.

The important part is that difficulty can vary by:

* object
* Actor
* scene
* story beat stage
* time
* player history

---

# Outcome Graph

A check should not just produce success/fail text.

It should route into a graph.

```text id="cr14"
Check
 ├─ Success → node A
 └─ Failure → node B
```

Sometimes there should also be:

* critical success
* critical failure
* partial success
* success with cost
* failure with progress

---

# Failure-Forward Design

This is one of the most important requirements.

Never:

```text id="cr15"
Failure = nothing
```

Always:

```text id="cr16"
Failure = different story path
```

Examples:

* the player embarrasses themselves
* the Actor becomes suspicious
* a clue is missed but another route opens
* a companion reacts
* a new check becomes available later

---

# Partial Success

The system should support intermediate outcomes.

Examples:

* player gets the information, but annoys the Actor
* player opens the lock, but damages the object
* player notices the clue, but only partially understands it

This is often more interesting than binary outcomes.

---

# Critical Success

A strong success should be able to:

* reveal extra information
* unlock an alternative path
* provide a reward
* create a better social outcome
* reduce future difficulty

---

# Critical Failure

A strong failure should be able to:

* create humiliation
* produce comedy
* trigger a worse state
* permanently alter the narrative
* spawn a new opportunity from the disaster

Critical failure is often highly memorable in NSF-style design.

---

# Retry Rules

The system should explicitly define retry behavior.

Possible retry rules:

```text id="cr17"
Never retryable
Retry after faculty increase
Retry after new clue
Retry after new belief
Retry after world change
Retry after time passage
```

This matters especially for Repeatable checks.

---

# Check Unlocking

A check may appear only when requirements are met.

Examples:

* after learning a fact
* after hearing a rumor
* after entering a location
* after equipping the right item
* after talking to a specific Actor

Checks should be unlockable and lockable dynamically.

---

# Hidden Checks

Some checks should remain invisible until triggered.

Examples:

* a passive perception reveal
* a contextual intuition check
* a hidden lie detection response

Hidden checks should not confuse the player with excessive UI noise.

---

# Forced Checks

Some checks should trigger automatically because the narrative demands it.

Example:

* the player character lies to the world
* a belief intrudes
* a moment of panic occurs
* a faculty forces an interpretation

These are important for NSF-style voice-driven play.

---

# Check Ownership

Every check should know what system owns it.

Examples:

* dialogue owns a persuasion check
* world interaction owns a lockpicking check
* perception owns a reveal check
* belief system owns an internalization-related check

This keeps the engine modular.

---

# UI Requirements

The player should see:

* faculty used
* chance to succeed
* modifiers
* outcome stakes
* retry status
* whether failure is acceptable or catastrophic

The UI should make risk legible.

---

# Probability Clarity

The player should be able to understand:

```text id="cr18"
Why is this 42%?
```

Not necessarily the exact math, but enough of it to feel fair.

That means the UI must expose modifiers in some form.

---

# Event Integration

Checks should emit events such as:

```text id="cr19"
CheckStarted
CheckSucceeded
CheckFailed
CheckCritSucceeded
CheckCritFailed
CheckUnlocked
CheckRetried
```

These events feed:

* dialogue
* quests
* chronicle
* Actor memory
* companion reactions

---

# Faculty Commentary Integration

Faculties may comment before, during, or after a check.

Examples:

* before: “You can do this.”
* during: “No, wait.”
* after success: “Correct.”
* after failure: “That was not enough.”

The check system should support this naturally.

---

# World State Integration

Checks should depend on world state.

Examples:

* a door is easier to force if damaged
* a witness is harder to persuade if angry
* a clue becomes easier to detect after rain
* a check disappears if the scene changes

The engine must treat checks as embedded in world simulation, not isolated minigames.

---

# Persistence

Serialize:

```text id="cr20"
Check state
Failed checks
Unlocked Repeatable checks
Consumed Gated checks
Roll history
Outcome history
```

This prevents inconsistent behavior after reload.

---

# Minimal Engine Interfaces

```csharp id="cr21"
interface RollDefinition
{
    string Id;
    string FacultyId;
    int Difficulty;
    bool CanRetry(GameState state);
    RollResult Resolve(GameState state);
}
```

---

# Most Important Insight

The check system is not about deciding pass or fail.

It is about deciding:

```text id="cr22"
What does this attempt cost?
What does success reveal?
What does failure create?
What changes in the world because the player tried?
```

If the system does that well, checks become dramatic moments rather than mechanical speed bumps.

If it does that poorly, the game becomes a list of dice gates with text attached.
