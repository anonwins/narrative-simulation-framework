# 36. Present: Audio — NSF Specification

> **Framework:** Voice barks, faculty voice triggers, ambient narrative audio, companion commentary hooks.
> **Content pack:** Audio assets, voice profiles, bark content.
> Terminology: [Glossary](terminology-glossary.md)

## Purpose

The Audio controls how narrative information is **expressed through sound rather than text or visuals**.

It answers:

```text id="a1b2c3"
What is heard?
When is it heard?
Why is it triggered?
How does it reinforce narrative meaning?
```

In An NSF-like RPG, audio is not background decoration.

It is:

```text id="d4e5f6"
A parallel narrative channel
```

---

# Core Principle

Audio is a **first-class narrative output**, not a post-processing layer.

The system transforms:

```text id="g7h8i9"
State changes → emotional sound events → player perception
```

Without it:

```text id="j1k2l3"
World feels static
Dialogue loses weight
Faculties feel mechanical
Atmosphere collapses
Companions feel lifeless
```

---

# High-Level Architecture

The system sits between narrative logic and sound playback:

```text id="m4n5o6"
Narrative Systems
    ↓
Audio Event Layer
    ↓
Audio Director
    ↓
Audio Engine (Unity / FMOD / Wwise)
    ↓
Sound Output
```

---

# Core Audio Categories

The system includes:

```text id="p7q8r9"
Dialogue Voice Acting
Voice Barks
Faculty Audio Triggers
Ambient Narrative Audio
Companion Commentary
UI Audio Feedback
World Event Audio
Interrupt Audio Stingers
```

---

# Audio Event bus

All audio is triggered via events, not direct playback.

Example:

```text id="s1t2u3"
AudioEvent: PlayVoiceLine
AudioEvent: PlayFacultyInsightSound
AudioEvent: TriggerAmbientMoodShift
```

---

# Dialogue Voice Acting Hooks

## Purpose

Synchronizes spoken dialogue with narrative nodes.

Example:

```text id="v4w5x6"
DialogueNode:
    text_key: kim_intro_01_text
    voice_id: kim_intro_01_vo
```

---

## Structure

```text id="y7z8a9"
VoiceLine
{
    id
    speaker
    audio_clip
    subtitle_key
    timing_data
}
```

---

## Synchronization

Must support:

```text id="b1c2d3"
Lip sync triggers
Text highlighting
Timed subtitle display
Emotional modulation
```

---

# Voice Barks

## Purpose

Short reactive lines triggered by world state.

Example:

```text id="e4f5g6"
Actor notices crime → "Hey!"
Companion reacts → "Be careful."
Guard patrols → "All clear."
```

---

## Structure

```text id="h7i8j9"
VoiceBark
{
    id
    speaker
    trigger_conditions
    priority
    cooldown
}
```

---

## Behavior

Barks must:

```text id="k1l2m3"
Be interruptible
Respect cooldowns
Avoid repetition spam
React to local context
```

---

# Faculty Voice Triggers

## Purpose

Faculties produce audio feedback when activated or relevant.

Example:

```text id="n4o5p6"
Logic faculty: "Something doesn't add up..."
Electrochemistry: nervous laughter
faculty_intuition: whispering intuition
```

---

## Structure

```text id="q7r8s9"
FacultyAudioTrigger
{
    faculty_id
    trigger_type
    audio_clip
    intensity_mapping
}
```

---

## Trigger Types

```text id="t1u2v3"
OnCheckSuccess
OnCheckFailure
PassiveInsight
ThresholdReached
```

---

# Ambient Narrative Audio

## Purpose

Audio that communicates world state without dialogue.

Example:

```text id="w4x5y6"
Rain intensity
Crowd density
Tension level
Ideological unrest
Danger proximity
```

---

## Structure

```text id="z7a8b9"
AmbientLayer
{
    id
    base_loop
    variations
    intensity_curve
}
```

---

## Dynamic Blending

Supports:

```text id="c1d2e3"
Layer crossfading
Intensity modulation
Event-based shifts
Location-based transitions
```

---

# Companion Commentary System

## Purpose

Companions react dynamically to gameplay.

Example:

```text id="f4g5h6"
Kim: "That was reckless."
Kim: "Interesting approach."
Kim: silence
```

---

## Structure

```text id="i7j8k9"
CompanionAudio
{
    companion_id
    trigger_event
    voice_line_id
    priority
}
```

---

## Trigger Sources

```text id="l1m2n3"
Dialogue outcomes
Faculty checks
Information discoveries
Player actions
World events
Rule engine triggers
```

---

# Interrupt Audio System

## Purpose

Signals narrative interruptions.

Example:

```text id="o4p5q6"
Insight unlocked → audio sting
Flashback → distortion sound
Faculty check → tension build-up
Revelation → tonal shift
```

---

## Structure

```text id="r7s8t9"
AudioInterrupt
{
    id
    trigger_event
    sound_stinger
    fade_behavior
}
```

---

# UI Audio Feedback

## Purpose

Reinforces UI interactions.

Example:

```text id="u1v2w3"
Hover tooltip → soft click
belief unlocked → chime
Story beat updated → tone shift
Choice selected → confirmation sound
```

---

## Structure

```text id="x4y5z6"
UIAudioEvent
{
    ui_action
    audio_clip
    priority
}
```

---

# World Event Audio

## Purpose

Represents systemic world changes.

Example:

```text id="a7b8c9"
Strike begins → crowd noise increases
Weather shift → storm build-up
Alarm triggered → siren system
```

---

## Structure

```text id="d1e2f3"
WorldAudioEvent
{
    event_id
    spatial_context
    intensity
    duration
}
```

---

# Spatial Audio Layer

Audio must be positioned in world space.

Example:

```text id="g4h5i6"
Actor shouting from alley
Radio playing in room
Crowd noise in distance
```

---

## Structure

```text id="j7k8l9"
SpatialAudioSource
{
    position
    attenuation_curve
    occlusion
}
```

---

# Priority System

Audio conflicts are resolved via priority:

```text id="m1n2o3"
Dialogue voice > Companion bark > Ambient sound
```

Rules:

```text id="p4q5r6"
Higher priority interrupts lower
Equal priority blends
Critical events override all
```

---

# Cooldown System

Prevent repetition spam:

```text id="s7t8u9"
Same bark cannot repeat within X seconds
Faculty audio has cooldown per faculty
Ambient shifts limited per zone
```

---

# Emotional Audio Mapping

Audio reinforces narrative tone:

```text id="v1w2x3"
Tension → low frequency drones
Insight → high shimmer tones
Failure → muted distortion
Success → uplifting cue
```

---

# Audio State Machine

Each scene has an audio state:

```text id="y4z5a6"
CALM
TENSION
ESCALATION
REVELATION
AFTERMATH
```

Transitions are triggered by narrative systems.

---

# Integration Points

Audio system connects to:

```text id="b7c8d9"
Rule Engine
Scripting Language
UI System
Dialogue
Info Flow
World Simulation
```

---

# Data Flow

```text id="e1f2g3"
Game Event
    ↓
Audio Event Dispatcher
    ↓
Audio Director
    ↓
Mixing System
    ↓
Playback Engine
```

---

# Audio Authoring System

Must support:

```text id="h4i5j6"
Event mapping
Voice assignment
Ambient layering
Faculty sound triggers
Companion dialogue linking
```

---

# Debug Tools

Critical tools include:

```text id="k7l8m9"
Audio event log
Trigger source tracing
Voice line playback inspector
Spatial audio visualization
Priority conflict viewer
```

---

# Performance Model

Must handle:

```text id="n1o2p3"
Hundreds of simultaneous audio events
Dynamic ambient layering
Frequent UI triggers
Dialogue + bark overlap
```

Optimizations:

```text id="q4r5s6"
Audio pooling
Event batching
Voice streaming
LOD audio mixing
```

---

# Failure Handling

System must degrade gracefully:

```text id="t7u8v9"
Missing voice → fallback text
Missing audio → silent continue
Overlapping audio → priority resolution
```

---

# Minimum Viable System

For a large-scale narrative RPG:

Implement:

```text id="w1x2y3"
Dialogue voice hooks
Voice bark system
Faculty audio triggers
Ambient layering
Companion commentary audio
UI sound feedback
Interrupt stingers
Audio event system
Priority + cooldown system
Spatial audio support
```

---

# Final Concept

The Audio is a **parallel storytelling layer** that operates alongside text, UI, and simulation.

It ensures:

```text id="z4a5b6"
The world feels alive even when nothing is happening
Faculties feel psychological, not mechanical
Dialogue carries emotional weight
Companions feel present and reactive
Environment communicates mood continuously
```

In a large-scale narrative RPG, audio is not support.

It is **narrative presence made audible**.
