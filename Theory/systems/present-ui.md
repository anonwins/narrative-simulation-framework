# 35. Present: UI — NSF Specification

> **Framework:** Dialogue, belief, chronicle, roll UI, tooltips, faculty popups, interruption displays.
> **Content pack:** UI skin labels, typography, art style, layout theming.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The UI is the **narrative-facing visual layer** that renders all gameplay and story information to the player.

It answers:

```text id="u1a2b3"
How is narrative information shown?
How is it interrupted?
How is it emphasized?
How is it framed emotionally?
```

In An NSF-like RPG, UI is not decoration.

It is:

```text id="c4d5e6"
A narrative system in itself
```

---

# Core Principle

Gameplay systems produce data.

UI transforms it into:

```text id="f7g8h9"
Readable experience
Emotional framing
Player attention control
Narrative pacing
```

Without this system:

```text id="i1j2k3"
Dialogue becomes unreadable
Faculty checks feel flat
Beliefs lose meaning
Events lack impact
Narrative timing breaks
```

---

# High-Level Architecture

The UI system sits on top of all narrative systems:

```text id="l4m5n6"
Narrative Systems
    ↓
UI Event Layer
    ↓
Presentation Controllers
    ↓
UI Widgets
    ↓
Rendered Interface
```

---

# Core UI Categories

The system includes:

```text id="o7p8q9"
Dialogue UI
Belief UI
Chronicle UI
Roll UI
Tooltips
Faculty Popups
Interrupt Systems
World Notifications
Inventory UI (narrative-aware)
```

---

# UI Event bus

All UI is driven by events.

Example:

```text id="r1s2t3"
UIEvent:
    ShowDialogue
    ShowBelief
    TriggerRoll
    UpdateChronicle
    ShowTooltip
```

UI does NOT poll game state directly.

---

# Dialogue UI System

## Purpose

Renders conversations with:

```text id="u4v5w6"
Character identity
Emotional tone
Branching choices
Timing control
Animation sync
```

---

## Structure

```text id="x7y8z9"
DialogueUI
{
    speaker_name
    portrait
    text
    choices
    voice_line
    animation_state
}
```

---

## Features

Supports:

```text id="a1b2c3"
Typing effects
Timed delivery
Interrupts
Voice playback sync
Emotional styling
```

---

## Choice Presentation

Example:

```text id="d4e5f6"
[1] Agree
[2] Question
[3] Stay silent
```

Choices can be:

```text id="g7h8i9"
Locked
Hidden
Timed
Faculty-gated
```

---

# Belief UI

## Purpose

Visualizes internal monologue progression.

Unlike dialogue, beliefs are:

```text id="j1k2l3"
Internal systems made visible
```

---

## Structure

```text id="m4n5o6"
BeliefUI
{
    title
    description
    status
    progress
    effects_preview
}
```

---

## States

```text id="p7q8r9"
Locked
Available
In-progress
Completed
Equipped
```

---

## Presentation Style

Belief UI should support:

```text id="s1t2u3"
Fragmented text
Stylized typography
Progress animations
Cognitive distortion effects
```

---

# Chronicle UI

## Purpose

Tracks narrative progression:

```text id="v4w5x6"
Quests
Tasks
Objectives
Clues
Information threads
```

---

## Structure

```text id="y7z8a9"
ChronicleEntry
{
    title
    description
    objectives
    status
}
```

---

## Dynamic Updates

Chronicle must support:

```text id="b1c2d3"
Real-time updates
Partial completion
Hidden objectives
Revealed information
```

---

# Roll UI

## Purpose

Represents probabilistic narrative resolution.

Example:

```text id="e4f5g6"
[Logic 6] - SUCCESS
```

---

## Structure

```text id="h7i8j9"
RollUI
{
    faculty_id
    required_value
    player_value
    modifiers
    result
}
```

---

## Presentation Layers

Faculty checks include:

```text id="k1l2m3"
Pre-check tension
Roll animation
Outcome reveal
Narrative consequence display
```

---

# Tooltips System

## Purpose

Provides contextual information without breaking flow.

Example:

```text id="n4o5p6"
Hover → "actor_companion: Companion role title"
```

---

## Structure

```text id="q7r8s9"
Tooltip
{
    title
    description
    related_entities
}
```

---

## Behavior

Tooltips should:

```text id="t1u2v3"
Appear on hover or focus
Fade in smoothly
Avoid blocking key UI
Update dynamically
```

---

# Faculty Popups

## Purpose

Immediate feedback for faculty changes.

Example:

```text id="w4x5y6"
+1 Drama
-2 Authority
```

---

## Structure

```text id="z7a8b9"
FacultyPopup
{
    faculty_id
    delta
    reason
}
```

---

## Presentation

Should include:

```text id="c1d2e3"
Animation burst
Sound cue
Color coding
Position anchoring
```

---

# Interrupt System

## Purpose

Critical for narrative-simulation pacing.

Interrupts break flow to highlight events:

```text id="f4g5h6"
Internal belief
Faculty check
Flash insight
Sudden realization
External event
```

---

## Types

```text id="i7j8k9"
Soft interrupt
Hard interrupt
Forced pause
Context overlay
```

---

## Example

```text id="l1m2n3"
Dialogue → INTERRUPT → Belief appears → Return to dialogue
```

---

# World Notifications

## Purpose

Communicates systemic events.

Example:

```text id="o4p5q6"
"Time passes..."
"New clue discovered"
"Reputation changed"
```

---

## Structure

```text id="r7s8t9"
Notification
{
    message
    type
    duration
    importance
}
```

---

# UI Layering System

UI must support priority layers:

```text id="u1v2w3"
Dialogue layer
Interrupt layer
Tooltip layer
Notification layer
HUD layer
Debug layer
```

Higher layers override lower ones.

---

# Presentation Controllers

Controllers translate game state → UI events.

Example:

```text id="x4y5z6"
StoryStateService → ChronicleUIController
DialogueService → DialogueUIController
RuleEngine → InterruptController
```

---

# Animation & Timing System

UI is not static.

It includes:

```text id="a7b8c9"
Fade in/out
Slide transitions
Typing effects
Pulse emphasis
Camera-like UI movement
```

---

# Emotional Framing

UI should reinforce tone:

```text id="d1e2f3"
Stress → red flicker
Insight → bright glow
Memory → blurred overlay
Confusion → jitter
```

---

# Input Integration

UI must respond to:

```text id="g4h5i6"
Keyboard
Mouse
Controller
Touch (optional)
```

Supports:

```text id="j7k8l9"
Choice selection
Skipping dialogue
Inspecting tooltips
Navigating chronicles
```

---

# Data Flow

UI never owns logic.

Flow:

```text id="m1n2o3"
Game State
    ↓
Event bus
    ↓
UI Controller
    ↓
UI Widget
    ↓
Render
```

---

# Performance Model

Must support:

```text id="p4q5r6"
Frequent dialogue updates
Real-time interrupts
Large chronicle entries
Faculty spam events
```

Optimizations:

```text id="s7t8u9"
Pooling UI elements
Event batching
Lazy rendering
Text caching
```

---

# Debug UI Mode

Critical for development:

```text id="v1w2x3"
Show all active UI layers
Display event logs
Highlight blocked choices
Show roll math
Trace interrupt causes
```

---

# Accessibility Layer

Must include:

```text id="y4z5a6"
Scalable text
Colorblind-safe modes
Subtitle support
Reduced motion mode
UI contrast settings
```

---

# Integration Points

UI system connects to:

```text id="b7c8d9"
Rule Engine
Scripting Language
Content Database
Locale
Info Flow
```

---


# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IUIShell
{
    void BindDialoguePresenter(IDialoguePresenter presenter);
    void BindChroniclePresenter(IChroniclePresenter presenter);
    void BindBeliefView(IBeliefView beliefView);
    void PushScreen(string screenId);
    void PopScreen();
}

interface IDialoguePresenter { void Show(DialogueViewModel model); }
interface IChroniclePresenter { void Show(ChronicleView model); }
interface IBeliefView { void Show(BeliefViewModel model); }
```

## Domain model

```csharp
class DialogueViewModel { string Speaker; string Text; ChoiceViewModel[] Choices; }
class ChoiceViewModel { string Id; string Label; bool Enabled; }
class BeliefViewModel { string BeliefId; string Title; BeliefPhase Phase; }
```

# Minimum Viable System

For An NSF-like RPG:

Implement:

```text id="e1f2g3"
Dialogue UI
Chronicle UI
Belief UI
Roll UI
Tooltips
Interrupt System
Notification System
UI Event Layer
Layering system
Localization support
```

---

# Final Concept

The UI is not a wrapper around gameplay.

It is the **player’s entire narrative experience interface**.

It ensures:

```text id="h4i5j6"
Information is readable
Emotions are communicated
Choices feel meaningful
Systems feel alive
Narrative timing is controlled
```

In a large-scale narrative RPG, UI is effectively a **second narrative engine running in parallel with gameplay systems**, shaping how every moment is perceived, interpreted, and emotionally experienced.
