# 16. Social: Ideology — NSF Specification

> **Framework:** Ideology profile axes, accumulation from dialogue/actions/beliefs, political gating.
> **Content pack:** Ideology axis names, thresholds, ideology story-beat content.
> Terminology: [Glossary](../terminology-glossary.md)

This system tracks **ideological development**, not morality.

In a traditional RPG, alignment often means:

```text id="pol1"
Good / Evil
Lawful / Chaotic
```

In a NSF-style game, political alignment means:

```text id="pol2"
What economic, social, and historical story does the player believe?
```

The system is about repeated statements, beliefs, and actions that gradually define player character’s political identity.

---

# Core Principle

The player is not picking a political class.

They are accumulating political positions through behavior.

```text id="pol3"
Ideological choice
→ ideological score change
→ belief unlock
→ dialogue access
→ Actor reaction
→ identity formation
```

---

# Core Architecture

```text id="pol4"
IdeologyService
 ├─ IdeologyScores
 ├─ PoliticalEvents
 ├─ BeliefLinks
 ├─ DialogueHooks
 ├─ ActorReactions
 ├─ QuestHooks
 ├─ Milestones
 └─ History
```

---

# Core Model

Track separate ideological lanes.

```csharp id="pol5"
class PoliticalProfile
{
    int Communist;
    int Fascist;
    int Moralist;
    int Ultraliberal;
}
```

These are not necessarily mutually exclusive in implementation, though the writing may treat strong combinations as contradictory or unstable.

---

# Ideology Is Emergent

A player should not be asked:

```text id="pol6"
Choose your politics.
```

Instead, the game observes behavior:

* what the player says
* what they endorse
* what beliefs they internalize
* how they solve problems
* what they mock or defend

That pattern creates the political identity.

---

# Ideology event sources

Ideology score should be updated by:

```text id="pol7"
Dialogue choices
Belief internalization
Story beat resolutions
Actor interactions
Examination reactions
Book reading
Dialogue interruptions
World actions
```

The system should be deeply woven into play, not isolated to special “political” scenes.

---

# Ideology categories

At minimum:

```text id="pol8"
Communist
Fascist
Moralist
Ultraliberal
```

Each should represent a cluster of beliefs, assumptions, rhetoric, and behaviors rather than just a label.

---

# Communist

Core themes:

* labor
* class conflict
* collective ownership
* anti-capitalism
* historical materialism

Typical scoring events:

```text id="pol9"
Defend workers
Criticize capital
Invoke class struggle
Support collective action
```

---

# Fascist

Core themes:

* hierarchy
* nationalism
* nostalgia
* purity
* order through dominance

Typical scoring events:

```text id="pol10"
Praise strongman rule
Invoke ethnic or national identity
Demand purity or discipline
Romanticize authoritarian order
```

---

# Moralist

Core themes:

* moderation
* institutions
* compromise
* stability
* technocratic governance

Typical scoring events:

```text id="pol11"
Defend bureaucracy
Avoid extremes
Trust institutions
Favor orderly reform
```

---

# Ultraliberal

Core themes:

* markets
* self-optimization
* entrepreneurship
* personal branding
* individual gain

Typical scoring events:

```text id="pol12"
Monetize situations
Brag about success
Support market logic
Treat identity as branding
```

---

# Ideology score

Each ideology should have a score.

Example:

```json id="pol13"
{
  "communist": 12,
  "fascist": 3,
  "moralist": 7,
  "ultraliberal": 9
}
```

The system should support:

* cumulative scoring
* weighted scoring
* threshold recognition
* milestone unlocks

---

# Weighted Events

Not all political statements are equal.

Example:

```text id="pol14"
Minor comment on labor:
+1 Communist

Full ideological speech:
+3 Communist
```

Weighting should depend on context and intensity.

---

# Threshold Recognition

Ideological identity becomes visible after thresholds.

Example:

```text id="pol15"
5 points:
- mild recognition

15 points:
- strong ideological drift

30 points:
- political identity lock-in
```

Thresholds should unlock reactions and content.

---

# Dialogue Integration

Dialogue options should be tagged by ideology.

Examples:

```json id="pol16"
{
  "requiresIdeology": "communist"
}
```

or:

```json id="pol17"
{
  "addsIdeology": {
    "ultraliberal": 1
  }
}
```

This lets political speech become a playable style.

---

# Internal Voice Integration

Faculties should react to politics.

Examples:

```text id="pol18"
Rhetoric:
That is a coherent class analysis.

Volition:
Do you really believe that?

Suggestion:
Say it smoother.

Authority:
Stand behind the position.
```

Ideological identity should affect internal monologue and faculty commentary.

---

# Belief Integration

Beliefs are a major political engine.

Some beliefs should:

* unlock via ideological behavior
* modify ideological scoring
* change dialogue access
* alter worldview
* create ideological bonuses and drawbacks

Example:

```text id="pol19"
Belief internalized →
Communist score gain rate increases
```

---

# Ideology milestones

At high enough scores, the system should trigger milestones.

Examples:

```text id="pol20"
Ideological awakening
First major ideological commitment
Factional confrontation
Self-identification
Ideology story beat unlocked
```

These milestones should feel like character development, not menu progress.

---

# Ideology story beat hooks

The system should unlock ideology-specific content.

Examples:

* communist-themed conversations
* fascist self-confrontation
* moralist institutional debates
* ultraliberal money-making opportunities

Ideology should not be cosmetic.

It should change available story paths.

---

# Actor Reaction Layer

Actors should react differently based on political pattern.

Examples:

* some Actors approve
* some become suspicious
* some mock the player
* some disengage
* some offer ideological debates
* some refuse to cooperate

This should be implemented as relationship and dialogue modifiers, not hardcoded one-off scenes.

---

# Player Self-Image Versus World Label

Track both:

```text id="pol21"
Internal ideological profile
Publicly perceived ideology
```

These may differ.

Example:

* the player thinks they are just being practical
* the world sees a clear political tendency

That tension is very NSF.

---

# Contradiction Support

Players can express contradictory politics.

Example:

```text id="pol22"
High Moralist + High Ultraliberal
```

or

```text id="pol23"
High Communist + High Fascist
```

The engine should not forbid contradictions too early.

Instead, let the writing expose the instability.

---

# Ideology memory

Track the history of ideological statements.

```text id="pol24"
PoliticalHistory
 ├─ first communist statement
 ├─ first moralist statement
 ├─ major ideological argument
 ├─ public embarrassment
 └─ faction-aligned choice
```

This helps with callbacks and long-term consequence.

---

# Reputation Versus Ideology

Do not merge political alignment with general reputation.

Example:

* a character may be respected and politically offensive
* a character may be disliked but politically persuasive
* a character may be seen as incompetent but ideologically committed

Keep these systems separate.

---

# World State Hooks

Ideology should affect:

* available dialogue
* Actor tone
* faction trust
* story beat availability
* Belief unlocks
* narrative flavor
* chronicle phrasing

In other words, politics should tint the entire experience.

---

# Event-Driven Design

Ideology updates should happen through the event system.

Example flow:

```text id="pol25"
Dialogue choice selected
↓
Ideology event emitted
↓
Ideology score updated
↓
Threshold checked
↓
Belief / dialogue / Actor reaction triggered
```

This keeps the system clean and extensible.

---

# Save Data Requirements

Serialize:

```text id="pol26"
Ideology scores
Ideology history
Milestones
Unlocked ideological beliefs
Public political reputation
Triggered political reactions
```

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IIdeologyService
{
    float GetAxisValue(string axisId);
    void ShiftAxis(string axisId, float delta);
    string GetDominantLabel(string axisId);
}
```

## Domain model

```csharp
class IdeologyAxis { string Id; float Value; string LeftLabelKey; string RightLabelKey; }
```


# Most Important Insight

The Ideology is not a morality meter.

It is a **belief-pattern tracker**.

The engine should answer:

```text id="pol28"
What does the player keep saying?
What worldview is forming from those choices?
How does the world respond to that worldview?
```

If the system works, the player will gradually feel that their politics are not selected from a menu.

They are being written by their actions.
