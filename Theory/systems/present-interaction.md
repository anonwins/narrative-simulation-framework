# 9. Present: Interaction — NSF Specification

> **Framework:** Clickables, containers, faculty/tool-gated interactions, context-sensitive actions.
> **Content pack:** Scene object definitions, interaction copy, gating assignments.
> Terminology: [Glossary](../terminology-glossary.md)

This is the system that makes the world feel clickable, readable, and reactive.

In a traditional RPG, world interaction is often:

```text id="j3m8ka"
Press button to loot / open / talk / inspect
```

In a NSF-style game, world interaction is:

```text id="v8tqnc"
A layered conversation between the player, the environment, faculties, tools, and narrative state
```

The player is not just interacting with objects.

They are interrogating the world.

---

# Core Principle

The world should behave like a dense field of authored affordances.

Not every object is a pickup.

Not every object is interactive in the same way.

Not every interaction is visible at first.

The engine must support:

* clickable environment objects
* hidden interactables
* faculty-gated reveals
* tool-gated actions
* context-sensitive reactions
* one-time and repeat interactions
* state-dependent interaction text
* narrative consequences from simple clicks

---

# Core Architecture

```text id="y7k1rf"
InteractionService
 ├─ SceneObjects
 ├─ InteractionNodes
 ├─ VisibilityRules
 ├─ Requirements
 ├─ Effects
 ├─ InteractionHistory
 ├─ Highlighting
 ├─ ToolChecks
 └─ Rolls
```

The system should be data-driven and authored per object, not hardcoded globally.

---

# Fundamental Design Rule

A world object is not just a mesh.

It is a node in the narrative graph.

```text id="p6h4tb"
Object
 ├─ visual representation
 ├─ interaction affordances
 ├─ conditions
 ├─ text
 ├─ effects
 └─ memory
```

This allows an ashtray, corpse, locker, poster, or window to all behave differently.

---

# Interaction Object Model

Every interactable should be represented by a structured object.

```csharp id="h4m9as"
class WorldObject
{
    string Id;
    string DisplayName;
    InteractionProfile Profile;
    List<InteractionNode> Interactions;
    List<StateRule> StateRules;
}
```

The object may exist in the world without being interactable until conditions are met.

---

# Interaction Node

Each interaction should be its own node.

```csharp id="b2y8mk"
class InteractionNode
{
    string Id;
    string Text;
    List<Condition> Requirements;
    List<Effect> Effects;
    InteractionResult SuccessResult;
    InteractionResult FailureResult;
}
```

A single object may have many interaction nodes.

Example:

* inspect
* open
* pry
* take
* smell
* comment on
* use tool on
* faculty observe

---

# Affordance System

The player must be able to infer possible actions.

Affordances should be:

```text id="w0x9cl"
Visible
Hidden
Conditional
Suggested
Forbidden
```

Example:

```text id="m4t2zi"
A locked cabinet may show:
- Open
- Examine
- Pry open
- Knock on it
```

Depending on state, some options may not appear.

---

# Interaction Discovery

Not all interactions are visible immediately.

They can be revealed by:

* higher faculty values
* specific beliefs
* specific tools
* dialogue information
* previous discoveries
* location state
* time of day
* story beat state

This is important for narrative-simulation discovery pacing.

---

# Faculty-gated Interaction Visibility

Faculties should reveal hidden object details.

Examples:

* Perception reveals a tiny clue on a desk
* faculty_environment reveals a feeling tied to a place
* Visual Calculus reveals what happened in a room
* faculty_intuition adds a strange emotional reading
* Encyclopedia contextualizes an artifact

Architecture:

```json id="n8r1qp"
{
  "revealFaculty": "perception",
  "difficulty": 10,
  "revealedInteraction": "inspect_bloodstain"
}
```

---

# Tool-Gated Interaction Visibility

Some actions require tools rather than faculties.

Examples:

* flashlight to inspect darkness
* prybar to force open something
* bolt cutters to cut chain or wire
* key to unlock a door
* plastic bag to collect bottles

Tool checks should be capability-based.

```json id="q1z4sb"
{
  "requiresCapability": "cut_chain"
}
```

---

# Context-Sensitive Interactions

The same object can behave differently based on context.

Example:

```text id="t5n7ha"
Locked door
```

Later:

```text id="h7d2mv"
Open door
```

Or:

```text id="k3p9zx"
Door
→ knock
→ listen
→ kick
→ unlock
```

The engine should not assume one interaction per object.

---

# Interaction States

Objects need state.

Examples:

```text id="r9c6ve"
Untouched
Observed
Inspected
Opened
Taken
Broken
Locked
Destroyed
Exhausted
```

State changes affect available interactions.

---

# One-Time and Repeatable Interactions

Some interactions should happen once only.

Examples:

* discovering a clue
* picking up an item
* reading a note for the first time

Others should repeat.

Examples:

* talking to a corpse again
* inspecting a room again after gaining a new faculty
* commenting on an object in different ways

The interaction system should distinguish these.

---

# Interaction History

The game should remember what the player has already done.

```text id="d8a1qv"
InteractionHistory
 ├─ object_id
 ├─ action_id
 ├─ first_seen_time
 ├─ last_seen_time
 └─ times_used
```

This allows objects to change their lines or options over time.

---

# Feedback Layer

Every interaction should produce feedback.

That includes:

* text
* sound
* animation
* UI highlight
* faculty reaction
* inventory result
* chronicle update
* story beat trigger

Even a small click should feel narratively meaningful when relevant.

---

# Examination System

Inspection is a major interaction type.

Examining an object should support:

* description
* multiple layers of detail
* faculty-dependent commentary
* hidden reveal branches
* follow-up actions

Example:

```text id="z6h2tx"
Examine desk
```

Possible layers:

* generic description
* evidence found
* faculty commentary
* internal reaction
* tool requirement
* follow-up interaction

---

# Environmental Layering

Objects should support layered information.

Example:

```text id="f5p2kc"
Room
 ├─ geometry
 ├─ visible objects
 ├─ hidden objects
 ├─ interactive objects
 └─ narrative objects
```

A room is not just a scene background.

It is a stack of authored data.

---

# Hidden Objects

Some objects are invisible until revealed.

Revealing them may depend on:

* faculty threshold
* item in hand
* time elapsed
* prior dialogue
* narrative phase

Examples:

* hidden bullet hole
* hidden note
* hidden smudge
* hidden passage
* hidden smell source

---

# Interaction Requirements

Every interaction should be able to require:

```text id="c8v1un"
Faculty
Item
Capability
Belief
Story beat state
Actor State
Time
Location
Previous Interaction
```

Example:

```json id="u3p9zs"
{
  "requiresFaculty": { "visual_calculus": 6 },
  "requiresStoryBeatState": "crime_scene_investigated"
}
```

---

# Failure Interactions

Interaction failure should still matter.

Examples:

* can’t open drawer
* can’t identify object
* can’t reach ledge
* can’t calm a witness
* can’t break seal

Failure should often create new narrative state, not dead end.

---

# Retryable Interaction Design

Some interactions can be retried later.

Examples:

* locked door after finding key
* unreadable note after gaining light
* faculty-gated observation after raising faculty

The system should support re-evaluation of previously blocked interactions.

---

# Spatial Interaction Model

World interactions are tied to location and spatial position.

Engine should support:

* hotspot-based clicks
* proximity-based prompts
* line-of-sight checks
* occlusion
* interior/exterior state
* region-based interaction enabling

This is especially important for point-and-click style navigation.

---

# World Object Categories

Useful categories:

```text id="q9m2lx"
Container
Prop
Evidence
Door
Terminal
Sign
Furniture
Body
Poster
Machine
Switch
Person-like Object
```

Each category can share defaults, while still allowing per-object overrides.

---

# Object Reactivity

Objects should react to:

* having been inspected
* having an item used on them
* being discovered via faculty
* changes in story beat phase
* day progression
* dialogue revelations

Example:

```text id="e4k8bz"
Corpse
```

can gain new interactions after:

* a roll
* a tool
* a story beat stage
* a conversation with Kim

---

# State-Dependent Text

The text shown for an object should change with state.

Example:

Before clue discovered:

```text id="u8m5pf"
A worn cabinet.
```

After clue discovered:

```text id="d2x6ra"
A worn cabinet. You noticed the scratches earlier.
```

This creates the feeling that the world remembers.

---

# Interaction and Dialogue Integration

World interaction often leads directly into dialogue.

Examples:

* examine mirror → inner monologue
* inspect corpse → faculty commentary
* use tool on object → reaction line
* click poster → ideology belief
* inspect machine → technical dialogue branch

The system must hand off seamlessly to dialogue.

---

# Interaction and Chronicle Integration

Important discoveries should generate chronicle updates automatically.

Examples:

* clue found
* door opened
* witness point discovered
* hidden passage revealed
* Inquiry subject connection inferred

The interaction system should emit narrative events rather than directly editing chronicle text.

---

# Interaction and Inventory Integration

Objects often yield items or require items.

Examples:

* take item from drawer
* use key on lock
* combine object with inventory tool
* collect evidence item
* store found object

Inventory and world interaction should communicate through capability and item requirements.

---

# Interaction and Faculty Voice Integration

Many world objects should produce internal commentary.

Examples:

* faculty_environment on a street corner
* faculty_intuition on a strange sign
* Logic on a broken machine
* Encyclopedia on a historic plaque

This should be treated as an interaction result, not a separate system.

---

# Temporary World State

Some interactions can change the environment.

Examples:

* door opened
* light switched on
* item removed
* corpse moved
* machine activated
* window broken
* evidence destroyed

These changes should persist in world state.

---

# Interaction Groups

Objects may belong to groups.

Example:

```text id="z1d7qn"
Drawer Set
Door Set
Locker Set
Evidence Cluster
```

A group can share behavior, while individual members still differ.

---

# Interaction Priority

When multiple interactions are valid, the system must choose or display them in a sensible order.

A likely priority:

```text id="n4r8st"
Critical narrative interaction
Faculty-reveal interaction
Tool interaction
Generic inspect interaction
Flavor interaction
```

This prevents clutter.

---

# UI Requirements

The player should see:

* what is interactable
* what is newly revealed
* what is faculty-gated
* what requires a tool
* what is already exhausted
* what changed since last visit

Good UI is essential because this system is dense.

---

# Save Data Requirements

Serialize:

```text id="b2n9rc"
Object states
Interaction history
Hidden reveals
Opened containers
Destroyed objects
Collected evidence
World changes
```

World interactions are persistent facts, not ephemeral events.

---

# Minimal Engine Interfaces

```csharp id="v5r0pm"
interface IWorldObject
{
    string Id;
    string Name;
    ObjectState State;
    List<IInteractionNode> Interactions;
}

interface IInteractionNode
{
    bool CanExecute(GameContext context);
    InteractionResult Execute(GameContext context);
}
```

---

# Most Important Insight

A NSF-style world interaction system is not about “click to use object.”

It is about:

```text id="k8p3vc"
The world as a layered investigative text
```

The player should feel that every room, item, door, poster, corpse, machine, and corner of the environment may contain:

* information
* mood
* memory
* evidence
* jokes
* secrets
* consequences

If the system can support that, the world will achieve NSF design goals.

If it cannot, the game will feel like a standard adventure game with better writing.
