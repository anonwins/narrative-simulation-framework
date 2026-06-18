# 18. Present: Discovery — NSF Specification

> **Framework:** Hidden objects, layered visibility, passive/active discovery, discovery memory.
> **Content pack:** Discoverable definitions, faculty-gated reveal content.
> Terminology: [Glossary](terminology-glossary.md)

This system controls **what the player can notice, when they can notice it, and how the world reveals itself over time**.

In a standard RPG, perception often means:

```text id="pd1"
Chance to spot hidden object
```

In a NSF-style game, perception is broader:

```text id="pd2"
What details become real to the player, and under what conditions?
```

It is not just a detection stat.

It is a **revelation system**.

---

# Core Principle

The world should not expose all of its meaning at once.

Instead:

```text id="pd3"
World State
→ Visibility Rules
→ Faculty Thresholds
→ Narrative Context
→ Discovery Event
→ New Knowledge
```

This makes exploration feel investigative rather than merely spatial.

---

# Core Architecture

```text id="pd4"
DiscoveryService
 ├─ HiddenObjects
 ├─ HiddenInteractions
 ├─ RevealRules
 ├─ VisibilityLayers
 ├─ DiscoveryEvents
 ├─ FacultyGates
 ├─ ContextGates
 └─ DiscoveryMemory
```

---

# Core Model

Every discoverable thing should have visibility state.

```csharp id="pd5"
class DiscoverableEntity
{
    string Id;
    DiscoveryState State;
    List<RevealCondition> RevealConditions;
    List<DiscoveryEffect> Effects;
}
```

Possible states:

```text id="pd6"
Hidden
Hinted
Visible
Discovered
Exhausted
```

---

# Hidden Objects

Hidden objects are not merely offscreen objects.

They are objects that exist in the scene but are not yet exposed to the player.

Examples:

* clue under a rug
* bloodstain on a wall
* hidden compartment
* faint footprint
* concealed note
* small object in shadow

A hidden object should become visible only when conditions are met.

---

# Hidden Interactions

An object may be visible but not fully interactive.

Examples:

* a drawer is visible but cannot yet be opened
* a faint stain is visible but not examinable until noticed
* a machine is present but not understood
* a person is present but not recognized as important

This distinction is crucial.

Visibility and interaction are separate systems.

---

# Environmental Reveals

The system should support information revealed by the environment itself.

Examples:

* the room feels wrong
* a draft suggests a hidden door
* a shadow points to a missing object
* distant sound implies another space
* a smell reveals decay
* architecture reveals history

These are often not hard objects but contextual discoveries.

---

# Layered Information Visibility

The same scene should expose information in layers.

Example:

```text id="pd7"
Layer 1: A room.
Layer 2: A disturbed desk.
Layer 3: A hidden note.
Layer 4: A connection to the thread.
```

The player should be able to return with more faculties, beliefs, or context and see more.

---

# Faculty-gated Revelation

Perception and related faculties should reveal hidden content.

Examples:

* Perception reveals small physical details
* faculty_environment reveals environmental presence or atmosphere
* Visual Calculus reconstructs what happened
* Encyclopedia contextualizes historical details
* faculty_intuition reveals strange emotional or symbolic meaning

This should be data-driven.

```json id="pd8"
{
  "revealFaculty": "perception",
  "difficulty": 11,
  "reveals": ["bloodstain"]
}
```

---

# Context-Gated Revelation

A player should not discover everything just by walking near it.

Some reveals should require context such as:

* a prior dialogue
* a story beat state
* a time of day
* a Belief internalized
* a tool in hand
* a specific location state

This makes discovery feel earned.

---

# Discovery Events

When something is revealed, the system should emit an event.

Examples:

```text id="pd9"
ObjectDiscovered
HiddenInteractionUnlocked
EnvironmentalClueRevealed
PatternRecognized
```

These events feed:

* dialogue
* chronicle
* quests
* Actor reactions
* faculty commentary

---

# Discovery Memory

The game should remember what the player has already seen.

```text id="pd10"
DiscoveryMemory
 ├─ first_seen
 ├─ first_revealed
 ├─ inspected
 ├─ explained
 └─ exhausted
```

This allows the world to stop repeating discovery text once it is already known.

---

# Hinting System

Not every hidden thing should be fully invisible.

The system should support hints such as:

* subtle glints
* unusual lighting
* brief narration
* faculty whispers
* object outlines
* ambient sound cues

This helps guide the player without fully solving the reveal.

---

# Passive Discovery

Some discoveries should happen automatically when conditions are met.

Examples:

* high Perception notices an item in a corner
* faculty_environment picks up a feeling in a room
* Logic notices an inconsistency
* Visual Calculus reconstructs movement

This is different from an explicit click-to-search action.

---

# Active Discovery

The engine should also support active searching.

Examples:

* inspect room
* look under bed
* search container
* examine wall
* scan area

Active searching can reveal things that passive perception misses.

---

# Discovery Thresholds

Each reveal should have a threshold.

Example:

```text id="pd11"
Perception 4 = faint clue
Perception 6 = exact object
Perception 8 = interpretive detail
```

Thresholds should be tunable per object or scene.

---

# Multiple Reveal Paths

The same thing may be discoverable through different systems.

Example:

A hidden passage may be revealed by:

* Perception
* Visual Calculus
* a key clue from dialogue
* a tool interaction
* a belief that changes interpretation

This makes the world feel systemic rather than linear.

---

# Reveal Priority

If multiple discoveries are possible at once, the system should prioritize them.

Suggested priority:

```text id="pd12"
Critical narrative reveal
Faculty-based physical reveal
Contextual reveal
Flavor reveal
Ambient reveal
```

This prevents information overload.

---

# Information Density Control

Scenes should not dump too many reveals at once.

The system should support:

* staged reveals
* delayed reveals
* reveal batching
* reveal suppression after exhaustion

This keeps exploration readable.

---

# Exhaustion State

Once a scene has been fully explored, discoveries should taper off.

States can become:

```text id="pd13"
Fully Explored
Partially Explored
Unexplored
```

This helps the game communicate whether more can still be found.

---

# Redundancy Handling

The system should avoid repeatedly telling the player the same thing.

Example:

* first time: “You notice a scrape on the floor.”
* later: no repeat unless new context makes it relevant

If repeated, it should be for a new reason.

---

# Faculty Commentary Integration

Faculties should comment on discoveries.

Examples:

* Perception: “There’s something here.”
* Logic: “That pattern is not accidental.”
* faculty_environment: “This place remembers.”
* faculty_intuition: “It feels watched.”

This turns discovery into character expression.

---

# Scene Reveal State

Each scene should track what has been revealed.

```csharp id="pd14"
class SceneRevealState
{
    List<string> RevealedObjects;
    List<string> RevealedInteractions;
    List<string> TriggeredHints;
}
```

This is important for persistence and revisits.

---

# Interaction Unlocking

Discovery often unlocks new interactions rather than simply new objects.

Example:

* object already visible
* new interaction becomes available after clue is found
* player can now ask about it, use it, or inspect it more deeply

That means discovery is often about meaning, not just visibility.

---

# Environmental Interpretation Layer

Some discoveries should not be physical at all.

Examples:

* a room feels hostile
* a street seems abandoned
* a location gives a memory
* a place suggests political tension

These are interpretive discoveries and should be treated as real gameplay output.

---

# Discovery and Chronicle Integration

Important reveals should write to the chronicle automatically.

Examples:

* clue found
* hidden route discovered
* Inquiry subject connection noticed
* environmental pattern identified

This helps the player remember what matters.

---

# Discovery and story state integration

Some story beat progress should depend on discovery state.

Example:

* a story beat should not advance until the player has discovered the relevant clue
* a theory should only unlock once enough evidence has been seen
* a conversation branch should open only after a reveal

This makes perception mechanically meaningful.

---

# Discovery and Time

Some hidden elements should only appear at certain times.

Examples:

* morning light reveals a mark
* night reveals a shadow
* rain reveals tracks
* after an event, a location changes

Time should influence what can be discovered.

---

# Discovery and Actors

Actors may react when the player discovers things.

Examples:

* a witness becomes nervous
* An inquiry subject changes behavior
* a companion comments
* an Actor notices the player noticing

This can create subtle social tension.

---

# Save Data Requirements

Serialize:

```text id="pd15"
Revealed objects
Triggered hints
Scene discovery state
Hidden interaction unlocks
Exhausted discoveries
Discovery history
```

---

# Minimal Engine Interfaces

```csharp id="pd16"
interface IDiscoverable
{
    string Id;
    DiscoveryState State;
    bool CanReveal(GameContext context);
    void Reveal(GameContext context);
}

interface IDiscoveryService
{
    void EvaluateScene(Scene scene, GameContext context);
}
```

---

# Most Important Insight

Perception is not only about spotting objects.

It is about **progressive understanding**.

The system should let the player gradually answer:

```text id="pd17"
What is physically here?
What did I miss?
What does it mean?
What changed because I noticed it?
```

If this system is strong, the world feels deep, layered, and investigative.

If it is weak, the game becomes a flat room with clickable props.
