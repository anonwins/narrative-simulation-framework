# 15. Cognition: Conduct — NSF Specification

> **Framework:** Conduct scoring, conduct emergence, Actor reactions, standing tracking.
> **Content pack:** Conduct IDs (e.g. conduct_humble), scoring rules, recognition thresholds.
> Terminology: [Glossary](../terminology-glossary.md)

The Conduct & Standing tracks **who the player character becomes through behavior**.

This is not a morality system.

This is not an alignment system.

This is not a faction reputation system.

Instead it answers:

```text id="cop1"
What kind of person does the world think you are?
```

and more importantly:

```text id="cop2"
What kind of person are you repeatedly choosing to be?
```

This system is one of NSF's most important hidden narrative mechanics.

---

# Core Principle

The player is not selecting a personality.

The player is **revealing** one through actions.

Bad:

```text id="cop3"
Choose:
[ ] conduct_bold
[ ] conduct_humble
```

Good:

```text id="cop4"
Repeated behavior
→ behavioral scoring
→ conduct emergence
→ Actor reactions
→ new dialogue
```

The player gradually discovers what behavioral pattern they have revealed.

---

# Core Architecture

```text id="cop5"
ConductService
 ├─ ConductEvents
 ├─ ConductScores
 ├─ StandingRules
 ├─ Thresholds
 ├─ ActorReactions
 ├─ DialogueHooks
 ├─ ChronicleHooks
 └─ SelfIdentityTracking
```

---

# Core Model

Use behavioral accumulation.

```csharp id="cop6"
class ConductProfile
{
    Dictionary<string, int> ConductScores;
    // e.g. ["conduct_humble"] = 12, ["conduct_bold"] = 10
}
```

These scores are not mutually exclusive.

A player may score highly in several.

---

# Conduct Events

Conducts are generated from behavior.

Not from story beat outcomes alone.

Not from stats.

Not from class selection.

Example:

```text id="cop7"
Player apologizes repeatedly.
```

Generates:

```text id="cop8"
conduct_humble +1
```

Repeated over time:

```text id="cop9"
conduct_humble +10
```

Eventually:

```text id="cop10"
Conduct recognized
```

---

# Conduct Profiles Are Emergent

The system should not force exclusivity.

Possible state:

```text id="cop11"
conduct_humble = 12
conduct_bold = 10
conduct_negligent = 8
```

This is valid.

The player is messy.

NSF-style identity systems embrace contradiction.

---

# Conduct Categories

At minimum:

```text id="cop12"
conduct_humble
conduct_bold
conduct_reckless
conduct_negligent
conduct_neutral
```

Additional conduct profiles are **content-defined**. The framework stores scores by string ID; meaning and behavior categories live in the content pack.

---

# Defining Conduct Profiles (Content Pack)

Each conduct profile entry in content data defines:

```json
{
  "id": "conduct_humble",
  "displayName": "...",
  "behaviorTags": ["apologizing", "self_blame", "seeking_forgiveness"],
  "scoringRules": [
    { "event": "PLAYER_APOLOGIZED", "value": 1 },
    { "event": "PLAYER_ACCEPTED_BLAME", "value": 1 }
  ]
}
```

Framework docs use abstract IDs (`conduct_humble`, `conduct_bold`, …) as **placeholders only**. A sci-fi content pack might map `conduct_bold` to "Glory-seeking captain"; a political thriller might use different labels with the same mechanics.

One worked example (placeholder IDs):

```text
Player action: repeated deferential dialogue choices
→ conduct_humble +1 per qualifying event

Player action: boastful or attention-seeking choices
→ conduct_bold +1 per qualifying event
```

Do not hardcode setting-specific conduct narratives in framework code.

---

# Conduct Scoring System

Every relevant action should emit a behavioral event.

Example:

```json id="cop23"
{
  "event": "PLAYER_APOLOGIZED",
  "conduct": "conduct_humble",
  "value": 1
}
```

Or:

```json id="cop24"
{
  "event": "PLAYER_BRAGGED",
  "conduct": "conduct_bold",
  "value": 1
}
```

The system listens and updates scores.

---

# Event Integration

Conduct tracking should subscribe to:

```text id="cop25"
Dialogue choices
Story beat resolutions
Item use
World interactions
Belief selections
Actor reactions
```

Nearly every system can contribute.

---

# Weighted Actions

Not all actions should be equal.

Example:

```text id="cop26"
Minor deferential action
+1 conduct_humble

Major emotional confession
+3 conduct_humble
```

Or:

```text id="cop27"
Small boast
+1 conduct_bold

Public grandstanding
+4 conduct_bold
```

Weighting creates stronger narrative identity.

---

# Threshold Recognition

Scores should trigger milestones.

Example:

```text id="cop28"
5 points
→ mild recognition

15 points
→ strong recognition

30 points
→ defining trait
```

Thresholds unlock content.

---

# Recognition Flags

Example:

```csharp id="cop29"
class ConductRecognition
{
    HashSet<string> RecognizedConducts;
}
```

These flags are derived from score thresholds.

---

# Actor Awareness

Actors should react to established Conducts.

Example:

```text id="cop30"
"You apologize a lot."
```

or:

```text id="cop31"
"There they go again—same pattern."
```

Recognition makes the system visible through fiction.

---

# Dialogue Hooks

Dialogue conditions should support:

```json id="cop32"
{
  "requiresConduct": "conduct_humble"
}
```

Or:

```json id="cop33"
{
  "requiresScore": {
    "conduct_bold": 15
  }
}
```

This unlocks conduct-specific content.

---

# Internal Narrator Hooks

Faculties should react.

Examples:

```text id="cop34"
faculty_willpower:
You do not need to defer again.

faculty_impulse:
Lean into the bold pattern.

faculty_logic:
This behavior pattern is becoming statistically significant.
```

Conducts should influence internal dialogue.

---

# Self-Identity System

Conducts are effectively narrative self-identities.

The game should track:

```text id="cop35"
Observed Behaviors
→ Emerging conduct profiles
→ Self Labels
```

The player gradually receives feedback about who they are becoming.

---

# Multiple Simultaneous Identities

Support coexistence.

Example:

```text id="cop36"
conduct_bold = 18
conduct_reckless = 14
conduct_humble = 12
```

This should generate unique reactions.

The player can be:

```text id="cop37"
A doomed celebrity disaster.
```

The system should embrace combinations.

---

# Contradictory Conduct

Contradiction is acceptable.

Example:

```text id="cop38"
conduct_neutral = 15
conduct_bold = 16
```

This may seem odd but reflects real player behavior.

Do not artificially prevent it.

---

# Conduct History

Track major contributing moments.

```text id="cop39"
ConductHistory
 ├─ Event
 ├─ Score Change
 ├─ Timestamp
 └─ Source
```

Useful for analytics and narrative callbacks.

---

# Standing layer

Separate conduct profiles from external reputation.

Example:

```text id="cop40"
Internal Identity:
conduct_reckless

Standing:
Competent public standing
```

These are different systems.

Do not merge them.

---

# Standing

Optional secondary layer.

Tracks what others believe about the player character.

Examples:

```text id="cop41"
Unstable
Brilliant
Reliable
Corrupt
Heroic
Pathetic
```

This is distinct from Conducts.

---

# Conduct and Belief Integration

Beliefs can interact with conduct profiles.

Examples:

```text id="cop42"
belief unlocked
→ increases scoring rate

Belief completed
→ enables new Conduct dialogue
```

The systems should communicate.

---

# Conduct and story state integration

Certain story beat resolutions may contribute.

Example:

```text id="cop43"
Solve problem professionally
+ conduct_neutral

Create unnecessary spectacle
+ conduct_bold
```

Only when behavior is clearly relevant.

---

# Decay System

Usually scores should not decay.

Reason:

```text id="cop44"
Conducts represent cumulative identity.
```

The player is building a history.

Recent behavior can matter separately if desired.

---

# Save Data Requirements

Serialize:

```text id="cop45"
Conduct scores
Recognition Flags
Behavior History
Milestones Reached
Actor Recognition Flags
```

---

# Minimal Engine Interfaces

```csharp id="cop46"
interface IConductService
{
    int GetScore(string conductId);

    void AddScore(string conductId, int amount);

    bool HasConduct(string conductId);
}
```

---

# Recommended Internal Flow

```text id="cop47"
Player Action
      ↓
Behavior Event
      ↓
Conduct Score Update
      ↓
Threshold Check
      ↓
Recognition Unlock
      ↓
Dialogue/Actor Reaction
```

Simple, scalable, and entirely event-driven.

---

# Most Important Insight

The Conduct is not judging the player.

It is **observing patterns**.

The engine should continuously ask:

```text id="cop48"
What behaviors does the player repeat?
What identity emerges from those repetitions?
How does the world respond?
```

If implemented correctly, the player eventually realizes:

```text id="cop49"
I didn't choose to reveal this pattern.

I revealed this pattern because of how I played.
```

That realization is the entire purpose of the system.
