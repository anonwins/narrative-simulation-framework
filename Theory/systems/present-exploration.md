# 25. Present: Exploration — NSF Specification

> **Framework:** Point-and-click navigation, area unlocking, discovery pacing, movement.
> **Content pack:** Map layout, area names, unlock narrative.
> Terminology: [Glossary](../terminology-glossary.md)

This system governs **how the player moves through the world, encounters content, and gradually uncovers information**.

It is not merely movement.

It is the system that controls:

```text
Where can the player go?
What can they see?
What do they notice?
What becomes available next?
```

In a NSF-style game, exploration is not about combat encounters or loot density.

It is about:

```text
Movement
→ Observation
→ Discovery
→ Conversation
→ Understanding
```

The exploration system is therefore tightly connected to:

* Location
* Actor
* Discovery
* Dialogue
* Story State
* Time

---

# Core Principle

The world should reveal itself gradually.

Not:

```text
Entire map available immediately
```

Instead:

```text
Explore
↓
Gain knowledge
↓
Unlock routes
↓
Reveal new opportunities
↓
Expand world understanding
```

Exploration should feel investigative.

---

# Core Architecture

```text
ExplorationService
 ├─ Navigation
 ├─ MovementController
 ├─ AreaUnlocking
 ├─ DiscoveryPacing
 ├─ ExplorationState
 ├─ EnvironmentalInteractions
 ├─ TravelTransitions
 ├─ ExplorationEvents
 └─ ExplorationMemory
```

---

# Core Model

```csharp
class ExplorationState
{
    List<string> VisitedLocations;

    List<string> DiscoveredObjects;

    List<string> UnlockedAreas;

    List<string> TriggeredExplorationEvents;
}
```

---

# Navigation System

Navigation controls movement through the world.

NSF-style navigation is generally:

```text
Point-and-click
```

rather than:

```text
Direct WASD control
```

The system should support either implementation internally.

---

# Point-and-Click Movement

Core loop:

```text
Click Destination
↓
Path Generated
↓
Character Walks
↓
Arrival Event
```

Movement itself is not the gameplay.

What happens during movement is the gameplay.

---

# Movement Controller

Responsibilities:

```text
Pathfinding
Collision
Speed
Animation Selection
Arrival Detection
Interaction Range
```

Movement should be reliable and unobtrusive.

---

# Pathfinding

Support:

```text
Walkable Areas
Dynamic Obstacles
Actor Avoidance
Temporary Blockers
```

The player should rarely fight the controls.

---

# Movement States

Examples:

```text
Idle
Walking
Running
Interacting
Transitioning
Blocked
```

These states drive animation and interaction logic.

---

# Exploration as Discovery

The primary purpose of movement is:

```text
Discover content.
```

Not:

```text
Traverse distance.
```

Every explored area should potentially contain:

* dialogue
* clues
* Actors
* environmental storytelling
* interactions
* new routes

---

# Area Unlocking

Large sections of the world should unlock over time.

Examples:

```text
Bridge repaired
Door opened
Authority granted
Story beat advanced
Time progressed
```

Area unlocking creates progression without leveling.

---

# Area State Model

```csharp
class AreaAccess
{
    string AreaId;

    bool IsUnlocked;

    List<UnlockCondition> Conditions;
}
```

---

# Unlock Conditions

Examples:

```text
Story beat Completion
Time Requirement
Dialogue Outcome
Item Possession
Ideology state
Faction Relationship
```

Unlocking should emerge naturally from play.

---

# Exploration Gating

Gates should feel believable.

Good:

```text
Locked gate
Police restriction
Flooded route
Closed business
```

Bad:

```text
Invisible wall
```

Narrative justification matters.

---

# Discovery Pacing

One of the most important responsibilities.

The system should regulate:

```text
How much content is revealed
How quickly it appears
When discoveries occur
```

Avoid overwhelming the player.

---

# Discovery Density

Every area should have a target density.

Examples:

```text
High Density
Moderate Density
Low Density
```

A small room may contain many interactions.

A street may contain few but important ones.

---

# Exploration Loops

Recommended loop:

```text
Enter Area
↓
Observe
↓
Interact
↓
Learn
↓
Unlock New Lead
↓
Move Elsewhere
```

This drives inquiry-style exploration.

---

# Environmental Interaction Layer

Exploration should constantly expose interactable elements.

Examples:

```text
Objects
Containers
Signs
Machines
Corpses
Documents
Landmarks
```

Interactions are exploration rewards.

---

# Interaction Visibility

Not all interactions should be immediately visible.

States:

```text
Hidden
Hinted
Visible
Exhausted
```

The exploration system should cooperate with discovery mechanics.

---

# Environmental Storytelling

Locations should communicate information.

Examples:

```text
Damage
Graffiti
Architecture
Trash
Lighting
Sound
```

The player learns through observation.

---

# Scene Entry Events

Entering an area should trigger evaluation.

Examples:

```text
Location Narration
Actor Availability
Passive Checks
Story beat updates
Time Events
```

This makes arrival meaningful.

---

# Revisiting Areas

Important requirement.

Revisiting should not feel identical.

Areas may change because of:

```text
Time
Story beat progress
Actor Movement
Weather
World Events
```

The player should be rewarded for returning.

---

# Fast Travel

Optional.

If implemented:

```text
Only between known locations
```

Fast travel should not bypass important discovery moments too early.

---

# Travel Transitions

Movement between major locations should support:

```text
Narration
Time Advancement
Event Triggering
Companion Dialogue
```

Travel is an opportunity for content.

---

# Exploration Events

Examples:

```text
LocationEntered
AreaUnlocked
ObjectDiscovered
LandmarkObserved
RouteOpened
```

These events feed other systems.

---

# Actor Discovery

Exploration is one of the primary ways Actors are introduced.

Example:

```text
Explore
↓
Meet Actor
↓
Dialogue Opens
↓
Story beat appears
```

The exploration system drives social content.

---

# Exploration and Time

Time should affect exploration.

Examples:

```text
Shop opens
Actor leaves
Area becomes accessible
Night-only interaction appears
```

Movement and time are tightly linked.

---

# Exploration and Perception

Exploration provides opportunities.

Perception determines what is noticed.

Example:

```text
Player enters room
↓
Perception evaluates
↓
Hidden clue revealed
```

Keep responsibilities separate.

---

# Exploration and Thread

Exploration should generate evidence.

Examples:

```text
Crime Scene
Witness
Physical Clue
Document
```

Many thread discoveries begin through movement.

---

# Discovery Memory

Track:

```text
Visited Locations
Seen Objects
Triggered Descriptions
Area Completion State
```

This prevents repetition.

---

# Exploration Completion

Optional area metrics:

```text
Unvisited
Partially Explored
Fully Explored
```

Useful internally even if hidden from players.

---

# Area States

Each area should support:

```text
Locked
Accessible
Explored
Changed
Critical
```

These states influence content generation.

---

# Exploration Fatigue Prevention

Avoid:

```text
Long empty walks
```

Instead:

```text
Frequent observations
Frequent interactions
Frequent discoveries
```

The player should regularly encounter meaningful content.

---

# World Expansion Pattern

Recommended progression:

```text
Small Hub
↓
Neighborhood
↓
District
↓
Extended District
```

The world should feel larger over time.

---

# Event Integration

Emit:

```text
LocationEntered
LocationExited
AreaUnlocked
LandmarkDiscovered
ExplorationMilestone
```

Many systems subscribe to these events.

---

# Save Data Requirements

Serialize:

```text
Visited Locations
Unlocked Areas
Discovery State
Triggered Exploration Events
Travel History
```

Exploration progress must persist.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IExplorationService
{
    bool CanEnter(string locationId);
    void Enter(string locationId);
    void Exit(string locationId);
    string GetCurrentLocationId();
}
```

## Domain model

```csharp
class ExplorationNode { string LocationId; string[] ConnectedLocationIds; }
```


# Relationship to Other Systems

Exploration is one of the primary content delivery systems.

```text
Exploration
      ↓
Perception
      ↓
Dialogue
      ↓
Chronicle
      ↓
Thread progress
      ↓
World Change
```

Many gameplay loops begin with movement.

---

# Recommended Internal Flow

```text
Move
↓
Observe
↓
Discover
↓
Interact
↓
Learn
↓
Unlock New Lead
↓
Move Again
```

This creates the rhythm of investigative exploration.

---

# Most Important Insight

The Exploration is not about traversing space.

It is about **revealing information at the correct pace**.

The engine should continuously answer:

```text
What can the player reach?
What can the player notice?
What becomes interesting next?
What new understanding has exploration created?
```

If implemented correctly, every walk through the world feels like part of an investigation.

If implemented incorrectly, exploration becomes dead travel between dialogue nodes.
