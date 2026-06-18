# 12. Sim: Location — NSF Specification

> **Framework:** Areas, scene transitions, unlockable zones, fast travel, environmental state.
> **Content pack:** Location names, scene art, unlock conditions content.
> Terminology: [Glossary](terminology-glossary.md)

The Location is not a map system.

In most RPGs:

```text
Location = Place where gameplay happens
```

In NSF:

```text
Location = Narrative Container
```

A location is:

* a story space
* a social space
* an investigation space
* a mood space
* a progression gate
* a memory container

The player is not simply moving between maps.

They are moving between narrative contexts.

---

# Core Principle

The world should be structured as:

```text
World
 ├─ Districts
 ├─ Areas
 ├─ Scenes
 ├─ Rooms
 └─ Interaction Points
```

Not:

```text
World
 ├─ Level 1
 ├─ Level 2
 ├─ Level 3
```

The world is spatially coherent.

It should feel like a real place.

---

# Core Architecture

```text
LocationService
 ├─ WorldMap
 ├─ Areas
 ├─ Scenes
 ├─ Rooms
 ├─ Connections
 ├─ AccessRules
 ├─ AmbientState
 ├─ SceneObjects
 └─ SceneEvents
```

---

# Hierarchy

Recommended structure:

```text
World
 └─ District
      └─ Area
           └─ Scene
                └─ Interaction Objects
```

Example:

```text
Martinaise
 ├─ Harbor
 ├─ Whirling-in-Rags
 ├─ Fishing Village
 ├─ Church
 └─ Boardwalk
```

Each is a narrative area.

---

# World Map Layer

The world map is the top-level structure.

```csharp
class WorldMap
{
    List<Area> Areas;
}
```

The map stores:

* connectivity
* travel routes
* accessibility
* unlock conditions

---

# Area Definition

Areas are major navigable regions.

```csharp
class Area
{
    string Id;
    string Name;

    List<Scene> Scenes;

    AreaState State;
}
```

Examples:

```text
Harbor
Village
Church
Apartment Block
Commercial District
```

---

# Scene Definition

A scene is the actual playable space.

```csharp
class Scene
{
    string Id;

    string Name;

    List<WorldObjects> Objects;

    List<Actors> Actors;

    AmbientProfile Ambient;
}
```

A scene is the unit that gets loaded.

---

# Scene Is The Primary Gameplay Container

Everything important exists inside scenes.

```text
Scene
 ├─ Actors
 ├─ Objects
 ├─ Triggers
 ├─ Ambient Audio
 ├─ Dialogue Hooks
 ├─ Clues
 ├─ Interactions
 └─ Visual State
```

The rest of the game mostly interacts through scenes.

---

# Scene State

Every scene should maintain state.

```text
Unvisited
Visited
Explored
Altered
Completed
```

Examples:

Before:

```text
Body hanging
```

After:

```text
Body removed
```

The scene remembers.

---

# Persistent Scene State

Changes to scenes must persist.

Examples:

```text
Door opened
Window broken
Evidence collected
Machine activated
Corpse moved
Graffiti discovered
```

The player should never feel that scenes reset.

---

# Scene Loading Model

Use scene-based streaming.

```text
Enter Scene
↓
Load Scene State
↓
Spawn Actors
↓
Spawn Objects
↓
Apply Narrative State
```

The narrative state is applied after loading.

This is critical.

---

# Narrative Layer

Every scene should contain narrative metadata.

```csharp
class NarrativeProfile
{
    StoryPhase RequiredPhase;
    Faction Influence;
    NarrativeImportance;
}
```

This allows scenes to react to story progression.

---

# World Phase Integration

Scenes may change across story phases.

Example:

Investigation phase:

```text
Harbor open
```

Later:

```text
Harbor tense
```

Later:

```text
Harbor locked down
```

Same scene.

Different state.

---

# Access Control System

Areas should not merely unlock via keys.

Access can depend on:

```text
Story beat progress
Dialogue
Beliefs / Faculties
Relationships
Time
Ideology state
Items
```

This creates natural narrative gating.

---

# Soft Gating

Preferred approach.

Bad:

```text
Invisible wall
```

Good:

```text
Guard refuses entry
```

Or:

```text
Need information first
```

Or:

```text
Need cooperation
```

The world feels more believable.

---

# Hard Gating

Still necessary sometimes.

Examples:

```text
Locked gate
Collapsed bridge
Broken elevator
Destroyed road
```

The engine should support both.

---

# Connectivity Graph

Locations should be connected via graph.

```text
Whirling
   ↓
Square
 ↓   ↓
Harbor Village
```

Architecture:

```csharp
class Connection
{
    Area From;
    Area To;

    AccessCondition[] Conditions;
}
```

---

# Travel System

Travel should consume time.

```text
Move
↓
Advance Time
↓
Update World
```

This ties movement into the Time.

---

# Fast Travel

Should exist only after familiarity.

Requirements may include:

```text
Visited
Discovered
Unlocked
```

Fast travel should still advance time.

---

# Location Discovery

Areas should support discovery.

States:

```text
Hidden
Known
Discovered
Visited
Mapped
```

Discovery itself can be progression.

---

# Ambient Profile

Every scene should define mood.

```csharp
class AmbientProfile
{
    AudioProfile Audio;

    WeatherProfile Weather;

    MoodProfile Mood;
}
```

This is extremely important for NSF-style atmosphere.

---

# Mood Layer

Each location should carry emotional metadata.

Examples:

```text
Lonely
Industrial
Decaying
Hopeful
Oppressive
Dreamlike
Sacred
```

This can drive:

* narration
* faculty reactions
* music
* visual effects

---

# Faculty-Reactive Locations

Locations should interact with faculties.

Examples:

High faculty_environment:

```text
Location speaks.
```

High Encyclopedia:

```text
Historical information appears.
```

High faculty_intuition:

```text
Strange interpretations emerge.
```

The same location should feel different for different builds.

---

# Location Memory

The player should remember places.

The game should help.

Architecture:

```text
LocationMemory
 ├─ First Visit
 ├─ Major Events
 ├─ Discovered Facts
 └─ Relationships Formed
```

Useful for chronicle integration.

---

# Scene Event bus

Scenes should emit events.

Examples:

```text
EnteredScene
ExitedScene
FirstVisit
ReturnedVisit
ObjectDiscovered
ActorEncountered
```

These feed:

* chronicle
* quests
* Beliefs\n* dialogue

---

# Local World State

Each scene should have local state.

Examples:

```text
DoorOpen
GeneratorActive
FireLit
CorpsePresent
```

Stored separately from global state.

---

# Global World State

Some changes affect many scenes.

Examples:

```text
StrikeEscalated
StormArrived
BridgeOpened
MartialLawDeclared
```

These states can alter multiple locations simultaneously.

---

# Environmental Storytelling Layer

Every scene should support passive storytelling.

Examples:

```text
Graffiti
Trash
Furniture
Architecture
Lighting
Objects
```

These are not merely decoration.

They communicate narrative information.

---

# Investigation Layer

Locations should contain:

```text
Evidence
Leads
Witnesses
Secrets
```

A location is often a source of knowledge.

Not just scenery.

---

# Scene-Specific Dialogue

Dialogue availability should depend on location.

Example:

```text
In church:
Special church dialogue.
```

```text
At harbor:
Special harbor dialogue.
```

Location context should matter.

---

# Scene-specific beliefs

Some beliefs only trigger in specific places.

Examples:

```text
Standing by ocean
Inside church
Near corpse
At strike line
```

Location becomes a narrative trigger.

---

# Companion Integration

Companions should react to locations.

Examples:

```text
First visit
Important clue
Dangerous place
Emotional place
```

Location commentary is a major source of characterization.

---

# Save Data Requirements

Serialize:

```text
Visited Areas
Visited Scenes
Discovered Areas
Scene States
Object States
Scene Events
Location Memories
Fast Travel Unlocks
```

Everything must persist.

---

# Minimal Engine Interfaces

```csharp
interface IArea
{
    string Id;
    string Name;
    List<IScene> Scenes;
}

interface IScene
{
    string Id;
    SceneState State;

    List<IWorldObject> Objects;
}

interface ILocationService
{
    IScene CurrentScene;
}
```

---

# Recommended Internal Separation

Keep these systems separate:

```text
Location
    ↓
Scene System
    ↓
Interaction
```

Do NOT merge them.

A location contains objects.

An object is not a location.

This separation prevents massive architectural problems later.

---

# Most Important Insight

Most RPGs treat locations as:

```text
Containers for gameplay
```

NSF treats locations as:

```text
Containers for meaning
```

Each area should answer:

```text
What happened here?
Who lives here?
What does this place believe?
What secrets does it contain?
How does it change over time?
```

If the engine supports those questions, locations will feel like places.

If it only supports movement between maps, they will feel like levels.
