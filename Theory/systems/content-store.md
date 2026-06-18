# 31. Content: Content Store — NSF Specification

> **Framework:** Data architecture for dialogue, beliefs, quests, Actors, assets, references.
> **Content pack:** All authored content records stored in the database.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The Content Store is the central repository that stores all authored narrative content.

It answers:

```text
Where does narrative content live?
How is it organized?
How is it referenced?
How is it loaded?
How is it localized?
```

Without this system:

```text
Dialogue becomes scattered.
Story beat data becomes fragmented.
Localization becomes painful.
References break constantly.
AI-generated content has nowhere to exist.
```

The Content Store acts as the single source of truth for all narrative content.

---

# Core Principle

Separate:

```text
Content
```

from

```text
Narrative Logic
```

The Rule Engine decides:

```text
When content is available.
```

The Content Database stores:

```text
What the content actually is.
```

Example:

Rule Engine:

```text
IF metric_trust_companion > 4

THEN

UnlockScene("kim_confession")
```

Content Database:

```text
Scene:
kim_confession
```

Contains:

```text
Dialogue
Choices
Beliefs\nAnimations
Audio
```

---

# High-Level Structure

The database should contain:

```text
Dialogue
Actor Definitions
Beliefs\nQuests
Events
Scenes
Barks
Items
Faculties
Concepts
Localization
Asset References
```

Everything narrative-related lives here.

---

# Content Identity

Every content object must have a unique ID.

Example:

```text
actor_companion
```

```text
dialogue_kim_intro
```

```text
quest_hanged_man
```

```text
belief_volumetric_shit_compressor
```

Never reference content by name.

Always reference IDs.

---

# Content Categories

Recommended top-level structure:

```text
NarrativeDatabase
{
    Actors
    Dialogues
    Beliefs\n    Quests
    Scenes
    Events
    Barks
    Faculties
    Concepts
    Localization
}
```

---

# Dialogue Storage

Dialogue should be stored as data.

Example:

```text
DialogueNode
{
    id
    speaker
    text
    choices
    conditions
    effects
}
```

Example:

```text
DialogueNode
{
    id: kim_intro_01

    speaker: actor_companion

    text:
    "Good morning."

    choices:
    [
        kim_intro_choice_01,
        kim_intro_choice_02
    ]
}
```

Dialogue content should never be hardcoded.

---

# Dialogue Graphs

Dialogue nodes form graphs.

Example:

```text
kim_intro_01
    ↓
Choice A
    ↓
kim_intro_02

Choice B
    ↓
kim_intro_03
```

Database stores the graph.

Dialogue executes it.

---

# Choice Storage

Choices should be separate entities.

Example:

```text
DialogueChoice
{
    id
    text
    destination
    conditions
    effects
}
```

Example:

```text
{
    id: ask_about_body

    text:
    "Tell me about the body."

    destination:
    body_discussion

    conditions:
    [
        body_discovered
    ]
}
```

---

# Belief Storage

Internal beliefs should be first-class content.

Example:

```text
Belief
{
    id
    title
    description
    acquisition_conditions
    effects
}
```

Example:

```text
Belief:
{
    id:
    communist_theory

    title:
    "Mazovian Theory"

    description:
    "A revolutionary framework..."
}
```

Beliefs should not be embedded in dialogue.

---

# Belief Progression

Some beliefs evolve.

Example:

```text
Belief
{
    id

    stages:
    [
        stage_1,
        stage_2,
        stage_3
    ]
}
```

This supports NSF-style Beliefs.

---

# Story beat storage

Story beats are content.

Example:

```text
StoryBeat
{
    id
    title
    description
    stages
    rewards
}
```

---

Example:

```text
StoryBeat:
{
    id:
    thread_main

    stages:
    [
        inspect_body,
        identify_victim,
        find_shooter
    ]
}
```

The Story State manages progress.

The database stores content.

---

# Story beat stage content

Each stage contains:

```text
QuestStage
{
    id
    chronicle_text
    objectives
}
```

Example:

```text
Inspect the body.
```

Stored as content.

---

# Actor Definitions

Actors require structured data.

Example:

```text
Actor
{
    id
    display_name
    portrait
    biography
    faction
    dialogue_sets
}
```

---

Example:

```text
Actor
{
    id:
    actor_companion

    display_name:
    "actor_companion"
}
```

Narrative content references Actor IDs.

---

# Character Metadata

Actor definitions may include:

```text
Occupation
Age
Background
Personality Tags
Relationships
Voice Profile
```

Useful for:

```text
Dialogue
AI generation
Codex systems
```

---

# Scene Storage

Scenes are authored content units.

Example:

```text
Scene
{
    id
    title
    participants
    dialogue_root
}
```

Example:

```text
Scene:
kim_confession
```

Contains all dialogue and event references.

---

# Event Content

Narrative events should exist independently.

Example:

```text
Event
{
    id
    title
    description
}
```

Example:

```text
event_strike_begins
```

Rule Engine triggers it.

Database stores it.

---

# Bark Storage

Ambient dialogue requires separate storage.

Example:

```text
Bark
{
    id
    text
    conditions
}
```

Example:

```text
"The harbor smells worse every day."
```

These are lightweight dialogue assets.

---

# Faculty Content

Narrative faculties may contain authored text.

Example:

```text
Faculty
{
    id
    display_name
    description
}
```

Example:

```text
Empathy
```

Contains:

```text
Faculty descriptions
UI text
Flavor content
```

---

# Concept Storage

Useful for:

```text
Lore
Politics
History
Locations
Factions
```

Example:

```text
Concept
{
    id
    title
    article_text
}
```

Acts like an encyclopedia or codex.

---

# Locale

Never store localized text directly in content objects.

Store keys.

Example:

```text
DialogueNode
{
    text_key:
    kim_intro_01_text
}
```

Localization:

```text
kim_intro_01_text

English:
"Good morning."

French:
"Bonjour."

German:
"Guten Morgen."
```

---

# Localization Database

Recommended structure:

```text
Localization
{
    language
    entries
}
```

Example:

```text
en
fr
de
es
```

---

# Asset Linking

Content should reference assets.

Not embed them.

Example:

```text
Actor
{
    portrait_asset
    voice_asset
}
```

Example:

```text
portrait_kim
```

Not:

```text
kim_portrait.png
```

Use asset IDs.

---

# Asset Reference Layer

Recommended:

```text
AssetReference
{
    id
    asset_type
    path
}
```

Example:

```text
portrait_kim
```

Maps to:

```text
Assets/UI/Portraits/Kim.png
```

This avoids broken references.

---

# Cross-References

Content must be linkable.

Example:

Dialogue references:

```text
Actor
StoryBeat
Belief
Item
```

Example:

```text
Dialogue:
mentions
belief_communism
```

Player can inspect it later.

---

# Reference Validation

Every reference should be validated.

Example:

```text
Dialogue references actor_companion
```

Validation:

```text
Does actor_companion exist?
```

If not:

```text
Database error
```

Automatic validation is critical.

---

# Content Versioning

Every content object should support:

```text
Version
Modified Date
Author
```

Useful for:

```text
Team workflows
AI-generated content
Rollback
```

---

# Content Tags

Support tagging.

Example:

```text
politics
communism
investigation
romance
harbor
```

Useful for:

```text
Search
Filtering
AI tools
Editor tools
```

---

# Data Format

Recommended:

```text
JSON
YAML
ScriptableObjects
```

For Unity:

```text
ScriptableObjects
```

or

```text
JSON + Import Pipeline
```

are ideal.

---

# Database Loading

Content should load independently.

Example:

```text
LoadDialogueDatabase()
LoadQuestDatabase()
LoadActorDatabase()
```

Not:

```text
LoadEntireGameDatabase()
```

Modularity matters.

---

# Editor Support

Designers should be able to:

```text
Create dialogue
Create quests
Create Beliefs\nCreate Actors
Search content
Validate references
Preview localization
```

Without touching code.

---

# AI Integration

AI-generated content should enter through the database.

Example:

```text
Generate Actor
```

Creates:

```text
Actor entry
Dialogue entries
Localization keys
Asset references
```

Everything becomes structured content.

---

# Save Game Relationship

The database is immutable content.

Save files contain:

```text
Player choices
Story beat progress
Fact state
Relationships
Flags
```

Never duplicate content in saves.

Save references only.

Example:

```text
CurrentDialogue:
kim_intro_03
```

Not:

```text
Full dialogue text
```

---

# Validation Pipeline

On every build:

Check:

```text
Missing IDs
Duplicate IDs
Broken references
Missing localization keys
Missing assets
Invalid story beat links
Invalid dialogue links
Unused content
```

Fail the build if critical issues exist.

---


# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IContentStore
{
    T GetDefinition<T>(string id) where T : ContentDefinition;
    bool TryGetDefinition<T>(string id, out T definition) where T : ContentDefinition;
    IReadOnlyList<string> GetAllIds<T>() where T : ContentDefinition;
}
```

## Domain model

```csharp
class ContentDefinition { string Id; string SchemaVersion; }
class ContentRegistry { T Get<T>(string id); void Register<T>(T definition); }
abstract class ContentDefinitionBase : ContentDefinition { }
```

# Minimum Viable System

For An NSF-Elysium-inspired RPG:

Implement:

```text
Dialogue Database
Choice Database
Belief Database
Story beat store
Actor Database
Scene Database
Localization Database
Asset References
Cross-Reference Validation
Content IDs
```

This becomes the permanent home of all authored narrative content. The Rule Engine decides when content becomes available, but the Content Store stores every dialogue line, story beat, belief, actor, scene, event, localization string, and asset reference in a structured, searchable, validated format that can scale to tens of thousands of narrative entries without becoming unmanageable.
