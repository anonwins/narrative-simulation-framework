# 19. Story: Voice — NSF Specification

> **Framework:** Narrator, internal monologue, environmental narration, perspective management.
> **Content pack:** Voice profiles, narration text, faculty voice content.
> Terminology: [Glossary](../terminology-glossary.md)

This system controls **how the game speaks to the player**.

It is distinct from the faculty system.

Faculties are internal agents with opinions.

The narrative voice system is the **language layer** that shapes the entire experience:

* narrator voice
* internal monologue framing
* environmental description
* perspective shifts
* tonal modulation
* emotional texture

If the dialogue system is what characters say, the narrative voice system is **how the world is told**.

---

# Core Principle

A NSF-style game does not use one neutral narrator.

It uses a voice architecture.

```text id="nv1"
NarrativeVoice
 ├─ Narrator
 ├─ InternalMonologue
 ├─ EnvironmentalVoice
 ├─ PerspectiveRules
 ├─ ToneRules
 └─ ContextFilters
```

The player should feel that the world is being interpreted, not merely described.

---

# Core Model

Use multiple narrative sources.

```csharp id="nv2"
class NarrativeVoiceContext
{
    string SceneId;
    string SpeakerId;
    string Perspective;
    string Tone;
    List<string> ActiveModifiers;
}
```

The same event may be described differently depending on context.

---

# Voice Layers

Recommended layers:

```text id="nv3"
1. Base Narrator
2. Faculty Commentary
3. Internal Monologue
4. Environmental Narration
5. Companion Commentary
6. Dialogue Text
```

These layers should be able to stack without collapsing into noise.

---

# Narrator

The narrator is the main descriptive voice.

Responsibilities:

* describe scenes
* frame actions
* present transitions
* interpret atmosphere
* support irony, seriousness, or surrealism

The narrator should not be flat or purely objective.

It can be:

* dry
* poetic
* ironic
* uncanny
* clinical
* self-aware

---

# Narrator Is Not Omniscient Neutrality

Important design rule.

The narrator should sometimes:

* speculate
* judge
* exaggerate
* withhold
* reflect the player character’s instability

That creates NSF-style texture.

---

# Internal Monologue

Internal monologue is the player character’s own verbalized inner monologue stream.

It should support:

* fragmented reactions
* self-observation
* doubt
* impulse
* embarrassment
* memory
* emotion
* surreal association

This is distinct from faculty commentary.

Faculties speak as parts of the mind.

Internal monologue is the **whole person thinking**.

---

# Environmental Narration

Locations and objects should be described as if they have a presence.

Examples:

* a room feels tired
* a street feels watched
* a machine seems angry
* a hallway feels abandoned but alert

This can be literal, poetic, or symbolic.

It is a major part of NSF's narrative atmosphere.

---

# Perspective Management

The engine should support perspective shifts.

Possible perspectives:

```text id="nv4"
First-person internal
Third-person narrative
Faculty-filtered
Object-centered
Companion-centered
Momentary omniscient
```

The game may switch perspective briefly for effect, but not randomly.

---

# Perspective Rules

Perspective should depend on:

* scene
* speaker
* faculty trigger
* belief state
* narrative event
* companion presence
* location importance

Example:

```text id="nv5"
faculty_environment may momentarily widen perspective to the city.
```

---

# Tone System

Tone should be explicit metadata.

Examples:

```text id="nv6"
Dry
Poetic
Menacing
Absurd
Tender
Clinical
Tragic
Grotesque
Hopeful
```

Tone can change per line, per scene, or per state.

---

# Tone Switching

The narrator should be able to switch tone in response to:

* emotional state
* faculty level
* political identity
* disaster
* humor
* danger
* revelation

This is one of the most important tools for narrative-simulation writing.

---

# Voice Source Model

Every line of narrative text should have a source.

```text id="nv7"
Narrator
Faculty
Belief
Environment
Companion
System
```

This helps the engine route content correctly.

---

# Narrative Text Object

```csharp id="nv8"
class NarrativeLine
{
    string Source;
    string Text;
    string Tone;
    string ContextTag;
    List<Condition> Conditions;
}
```

The engine should choose the right line based on context.

---

# Faculty Versus Voice

Faculties are speakers.

Narrative voice is the framing layer.

Example:

```text id="nv9"
LOGIC:
That cannot be right.

Narrator:
The room remains silent.
```

The first is an internal character.

The second is the storytelling layer.

Keep them separate.

---

# Environmental Object Voice

Some objects should be described almost as if they speak.

Examples:

* a corpse
* a mirror
* a locked door
* a ruined sign
* a city block

This is not literal dialogue.

It is narrative personification.

---

# Unreliable Narration Support

The engine should support unreliable interpretation.

Possible sources of unreliability:

* intoxication
* fear
* magical thinking
* trauma
* ideology
* obsessive belief
* poor perception

That means the narrator can sometimes be wrong, biased, or incomplete.

---

# Narrative Bias Layers

The description of the same thing may be altered by:

```text id="nv10"
Faculty
Belief
Ideology
Conduct
Companion presence
Health / morale
Time of day
Location mood
```

This allows the world to be perceived differently across playthroughs.

---

# Reinterpretation

The same earlier line may gain new meaning later.

Example:

* first visit: “A plain room.”
* later: “Not plain. Hidden evidence was here.”

The narrative voice should support revision, not just initial description.

---

# Dynamic Description Generation

The engine should support composition.

Example structure:

```text id="nv11"
Base description
+ faculty overlay
+ emotional overlay
+ contextual overlay
+ consequence overlay
```

This is better than writing one monolithic line for every situation.

---

# Context-Specific Narration

Narrative lines should depend on:

* discovered facts
* current scene state
* current relationship state
* current ideology
* current emotional condition
* current time

This makes narration responsive rather than static.

---

# Environmental Narration Triggers

The environment should narrate when:

* entering a new scene
* inspecting a notable object
* discovering a clue
* hearing an important sound
* revisiting a changed location

These are often brief but powerful.

---

# Internal Monologue Triggers

Internal monologue should fire on:

* player actions
* failures
* discoveries
* emotional events
* dialogue outcomes
* memory cues
* odd environmental details

Not every action needs a line.

The system should be selective.

---

# Narrative Silence

Silence is important.

Not every object or event should produce commentary.

The engine should support:

```text id="nv12"
No narration
```

when the moment should remain understated.

That restraint helps stronger moments land.

---

# Voice Priority

If several narrative voices are eligible, the system should choose by priority.

Suggested order:

```text id="nv13"
Critical story narration
Faculty commentary
Companion commentary
Environmental narration
Flavor narration
```

This reduces clutter and ensures important lines are not buried.

---

# Multi-Voice Layering

Sometimes several voices should appear in sequence.

Example:

```text id="nv14"
Narrator: The room is still.

Logic: Too still.

faculty_environment: This place is listening.

Companion: Are you hearing that too?
```

The system should support layered delivery without hardcoding each case.

---

# Serialization Requirements

Serialize:

```text id="nv15"
Narrative line history
Triggered narration flags
Perspective state
Tone modifiers
Unreliable narration state
Voice source history
```

This helps keep repeated descriptions from becoming stale.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IVoiceService
{
    void Speak(VoiceChannel channel, string textKey, string actorOrFacultyId);
    void Stop(VoiceChannel channel);
}
```

## Domain model

```csharp
enum VoiceChannel { Narrator, Environmental, Faculty, Actor }
class VoiceLine { VoiceChannel Channel; string TextKey; string SourceId; }
```


# Most Important Insight

The Voice is what makes the game feel authored, not merely simulated.

It answers:

```text id="nv17"
How does the game speak when no one is speaking?
How does it interpret a room?
How does it color a failure?
How does it make a belief feel alive?
```

If this system is strong, even simple events can feel profound.

If it is weak, the game becomes mechanically correct but emotionally flat.
