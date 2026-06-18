# 34. Content: Locale — NSF Specification

> **Framework:** String tables, variable substitution, gender handling, voice asset linking.
> **Content pack:** Translated strings, locale-specific formatting.
> Terminology: [Glossary](terminology-glossary.md)

## Purpose

The Locale manages all **language-dependent narrative content** in a structured, scalable way.

It ensures that:

```text id="l1a2b3"
Every piece of text is translatable
Every dialogue line can adapt per language
Every UI and narrative string is consistent
Every variable can be inserted safely
Every voice line maps correctly
```

Without it:

```text id="c4d5e6"
Dialogue becomes hardcoded in one language
AI-generated text becomes untranslatable
Variable insertion breaks sentences
Gendered languages break grammar
Voice acting mismatches text
Content duplication explodes
```

In An NSF-like RPG, localization is not optional—it is core infrastructure.

---

# Core Principle

Never store raw text in gameplay systems.

Instead:

```text id="f7g8h9"
Use localization keys everywhere
```

Example:

```text id="i1j2k3"
kim_intro_01_text
```

Maps to:

```text id="l4m5n6"
Different language strings per locale
```

---

# High-Level Structure

The system is built around:

```text id="o7p8q9"
String Tables
Variable Substitution
Grammar Rules
Gender Handling
Formatting System
Voice Mapping
Fallback Logic
```

---

# String Tables

All text lives in language-specific tables.

Example structure:

```text id="r1s2t3"
LocalizationTable
{
    language: "en",
    entries: {
        kim_intro_01_text: "Good morning.",
        kim_intro_02_text: "We should get moving."
    }
}
```

French:

```text id="u4v5w6"
kim_intro_01_text: "Bonjour, détective."
```

German:

```text id="x7y8z9"
kim_intro_01_text: "Guten Morgen, Detektiv."
```

---

# Localization Key System

Keys must be:

```text id="a1b2c3"
Unique
Stable
Non-semantic
System-generated or validated
```

Example:

```text id="d4e5f6"
dialogue_kim_intro_01_line_03
```

Never use raw text as identifiers.

---

# Dialogue Localization

Dialogue nodes reference keys instead of text.

Example:

```text id="g7h8i9"
DialogueNode
{
    id: kim_intro_01,
    text_key: kim_intro_01_text
}
```

This allows:

```text id="j1k2l3"
Multiple languages
Dynamic updates
Reuse across scenes
```

---

# Variable Substitution

Narrative text often includes dynamic values.

Example:

```text id="m4n5o6"
"You have {money} réal."
```

At runtime:

```text id="p7q8r9"
"You have 12 réal."
```

---

# Substitution Rules

Supported tokens:

```text id="s1t2u3"
{player_name}
{money}
{day}
{location}
{actor_name}
{quest_name}
```

Example:

```text id="v4w5x6"
"Good morning, {player_name}."
```

---

# Context-Aware Substitution

Variables may depend on game state:

```text id="y7z8a9"
IF player is arrested:
    "You are in custody."
ELSE:
    "You are free."
```

---

# Gender Handling

Critical for languages like:

```text id="b1c2d3"
French
German
Russian
Spanish
```

System supports:

```text id="e4f5g6"
Gender-aware grammar variants
```

Example:

```text id="h7i8j9"
"You are {gender:role_male / role_female}"
```

---

# Gendered Sentence Example

English:

```text id="k1l2m3"
"You look tired."
```

French:

```text id="n4o5p6"
"Vous avez l'air {fatigue_male/fatigue_female}."
```

---

# Formatting Tokens

Formatting must be standardized.

Supported:

```text id="q7r8s9"
{bold}
{italic}
{color:red}
{newline}
{indent}
```

Example:

```text id="t1u2v3"
"{bold}Warning:{/bold} Restricted area."
```

---

# Nested Formatting

Example:

```text id="w4x5y6"
"{color:red}{bold}Danger{/bold}{/color}"
```

---

# Pluralization System

Languages require plural rules.

Example:

```text id="z7a8b9"
"{count} item(s)"
```

Better:

```text id="c1d2e3"
one: "{count} item"
other: "{count} items"
```

---

# Complex Plural Rules

Some languages require multiple forms:

```text id="f4g5h6"
singular
few
many
other
```

Example:

```text id="i7j8k9"
Russian plural system support required
```

---

# Voice References

Text must map to audio files.

Example:

```text id="l1m2n3"
DialogueLine
{
    text_key: kim_intro_01_text,
    voice_id: kim_intro_01_vo
}
```

---

# Voice Asset Linking

Voice system:

```text id="o4p5q6"
voice_kim_intro_01_en
voice_kim_intro_01_fr
voice_kim_intro_01_de
```

Each language has separate audio.

---

# Silent vs Voiced Content

System must support:

```text id="r7s8t9"
Fully voiced lines
Partially voiced lines
Text-only lines
Ambient barks
```

---

# Fallback Language System

If translation missing:

```text id="u1v2w3"
Fallback → English
```

Or:

```text id="x4y5z6"
Fallback → base language pack
```

Never crash.

---

# Missing Key Handling

If key is missing:

```text id="a7b8c9"
[missing: kim_intro_01_text]
```

Must be visible in debug mode.

---

# Localization Context Awareness

Some text depends on context:

```text id="d1e2f3"
Actor relationship
Player state
Story beat state
Time of day
```

Example:

```text id="g4h5i6"
"You again."
vs
"Hello."
```

---

# Embedded Dialogue Localization

Bad approach:

```text id="j7k8l9"
Hardcoded strings in scripts
```

Good approach:

```text id="m1n2o3"
Reference localization keys only
```

---

# Runtime Resolution

At runtime:

```text id="p4q5r6"
LocalizationKey → StringTable → Final Text
```

Flow:

```text id="s7t8u9"
GetKey()
→ ResolveLanguage()
→ ApplyVariables()
→ ApplyFormatting()
→ ReturnString()
```

---

# Performance Model

Optimization required for:

```text id="v1w2x3"
Thousands of dialogue lines
Frequent UI updates
Real-time dialogue systems
```

Techniques:

```text id="y4z5a6"
String caching
Precompiled tables
Batch substitution
Lazy loading language packs
```

---

# Editor Requirements

Localization tools must allow:

```text id="b7c8d9"
Side-by-side language editing
Missing key detection
Preview in-game text
Variable simulation
Gender preview
Voice preview
```

---

# Localization Pipeline Integration

Works with:

```text id="e1f2g3"
Content Pipeline
```

Checks:

```text id="h4i5j6"
Missing translations
Unused keys
Broken references
Voice mismatches
Formatting errors
```

---

# AI Content Support

AI-generated content must:

```text id="k7l8m9"
Never output raw text directly into game systems
Always generate localization keys
Always generate base-language entry
Trigger translation pipeline
```

---

Example AI output:

```text id="n1o2p3"
dialogue_kim_ai_0001_text
```

---

# Localization Versioning

Track changes:

```text id="q4r5s6"
Key ID
Change history
Translation updates
Voice updates
```

---

# Regional Variants

Support dialects:

```text id="t7u8v9"
en_US
en_UK
fr_FR
fr_CA
```

---

# UI Text vs Narrative Text

Separate categories:

```text id="w1x2y3"
UI strings
Dialogue strings
Beliefs / system messages
```

Each has different translation priorities.

---

# Debug Tools

Must include:

```text id="z4a5b6"
Missing key viewer
Language switcher
Variable inspector
Fallback tracker
Voice mapping debugger
```

---

# Common Failure Cases

System must prevent:

```text id="c7d8e9"
Hardcoded strings
Missing translations in production
Broken variable injection
Incorrect gender grammar
Audio mismatch
```

---

# Minimum Viable System

For a large-scale narrative RPG:

Implement:

```text id="f1g2h3"
Localization key system
String tables
Variable substitution
Pluralization support
Gender handling
Voice mapping
Fallback system
Debug missing keys
Pipeline integration
```

---

# Final Concept

The Locale ensures that narrative content is:

```text id="i4j5k6"
Language-agnostic at the core
Fully translatable at scale
Safe for AI generation
Consistent across systems
Ready for voiced production
```

It is the layer that makes a large-scale narrative RPG globally shippable without rewriting the narrative system for every language.
