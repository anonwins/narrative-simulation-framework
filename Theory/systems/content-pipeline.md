# 33. Content: Pipeline — NSF Specification

> **Framework:** Author workflow, validation, dependency checking, broken reference detection.
> **Content pack:** Content authored through the pipeline; localization handoff.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The Content Pipeline is the **production system that turns authored (or AI-generated) narrative data into safe, playable, validated game content**.

It answers:

```text id="p1a2b3"
Is this content valid?
Is anything missing?
Will it break the game?
Does it depend on missing systems?
Is it properly localized?
Can it be shipped safely?
```

This is not gameplay logic.

It is **content manufacturing infrastructure**.

Without it:

```text id="c4d5e6"
AI-generated dialogue breaks references
Quests fail silently
Localization is incomplete
Scenes reference missing nodes
Builds become unstable
Narrative systems degrade over time
```

---

# Core Principle

The pipeline is a **continuous validation and transformation system**:

```text id="f7g8h9"
RAW CONTENT
    ↓
VALIDATION
    ↓
DEPENDENCY RESOLUTION
    ↓
LOCALIZATION CHECK
    ↓
TEST SIMULATION
    ↓
BUILD-READY CONTENT
```

Nothing enters the game unless it passes this flow.

---

# Pipeline vs Runtime Systems

Important separation:

```text id="i1j2k3"
Rule Engine → runtime decisions
Content Database → storage
Scripting Language → authoring
Pipeline → safety + verification
```

The pipeline does NOT affect gameplay.

It ensures gameplay won’t break.

---

# High-Level Stages

## 1. Authoring Input

Content comes from:

```text id="l4m5n6"
Humans
AI agents
Tools (editors)
Import scripts
External localization systems
```

Format:

```text id="o7p8q9"
Unvalidated narrative assets
```

---

## 2. Schema Validation

Every asset must match a schema.

Example:

```text id="r1s2t3"
DialogueNode schema check
Story beat schema check
Actor schema check
```

Checks:

```text id="u4v5w6"
Missing fields
Wrong types
Invalid IDs
Malformed structure
```

Fail fast.

---

## 3. Dependency Checking

Ensures all references exist.

Example:

```text id="x7y8z9"
Dialogue references Actor actor_companion
```

Pipeline checks:

```text id="a1b2c3"
Does actor_companion exist?
Does referenced story beat exist?
Does scene exist?
Does asset exist?
```

Detects:

```text id="d4e5f6"
Broken references
Circular dependencies
Missing content nodes
```

---

## 4. Graph Validation

Narrative content forms graphs.

Pipeline validates:

```text id="g7h8i9"
Dialogue flow connectivity
Story beat progression paths
Scene entry/exit points
Dead-end nodes
Orphan nodes
```

Example error:

```text id="j1k2l3"
Node kim_intro_03 has no outgoing transitions
```

---

## 5. Localization Pipeline

Ensures all text is localizable.

Checks:

```text id="m4n5o6"
Missing localization keys
Unused keys
Duplicate keys
Language coverage completeness
```

Example:

```text id="p7q8r9"
ERROR: missing FR translation for kim_intro_01_text
```

Optional step:

```text id="s1t2u3"
Auto-fill fallback language
```

---

## 6. Asset Validation

Checks linked assets:

```text id="v4w5x6"
Missing textures
Broken audio references
Invalid animation clips
Unassigned portraits
```

Example:

```text id="y7z8a9"
Actor kim references missing portrait_asset
```

---

## 7. Rule Engine Compatibility Check

Ensures scripts and content match runtime rules.

Checks:

```text id="b1c2d3"
Undefined flags
Invalid faculty names
Unknown variables
Unsupported operators
```

Example:

```text id="e4f5g6"
ERROR: Faculty "DramaX" does not exist
```

---

## 8. Simulation Testing

The pipeline simulates execution paths.

Example:

```text id="h7i8j9"
Simulate:
Day 1 → Day 10 progression
```

Checks:

```text id="k1l2m3"
Can player soft-lock story beat?
Are required nodes reachable?
Do all scenes resolve?
```

---

## 9. Narrative Playthrough Tests

Automated bots simulate gameplay:

```text id="n4o5p6"
Random choices
Faculty checks
Dialogue traversal
Story beat completion attempts
```

Detects:

```text id="q7r8s9"
Broken progression
Impossible states
Dead ends
Infinite loops
```

---

## 10. Build Packaging

Only validated content is packaged:

```text id="t1u2v3"
Compiled narrative database
Localized bundles
Asset bundles
Script graphs
```

---

# Author Workflow

The pipeline supports iterative creation:

```text id="w4x5y6"
Create content
Run validation
Fix errors
Re-run pipeline
Approve build
Ship
```

---

# AI Agent Integration

Critical for AI-generated content.

Pipeline must:

```text id="z7a8b9"
Reject unsafe content
Auto-correct minor issues
Flag ambiguity
Request regeneration
Ensure schema compliance
```

---

Example AI failure:

```text id="c1d2e3"
Generated dialogue references actor_unknown_character
```

Pipeline response:

```text id="f4g5h6"
ERROR: missing Actor reference
ACTION: regenerate with valid Actor list
```

---

# Automated Fixing Layer

Optional but powerful:

Fixes:

```text id="i7j8k9"
Missing localization fallback
Default asset assignment
Simple reference repair
Auto-insertion of missing dialogue links
```

But NEVER:

```text id="l1m2n3"
Change narrative meaning
Rewrite story intent
Override authored logic
```

---

# Dependency Graph System

All content forms a graph:

```text id="o4p5q6"
Actor → Dialogue → Story State → Scene → Event
```

Pipeline builds:

```text id="r7s8t9"
Full dependency graph
Reverse dependency index
Impact analysis tree
```

Example:

```text id="u1v2w3"
If actor_companion breaks → 42 dialogues affected
```

---

# Broken Reference Detection

Detects:

```text id="x4y5z6"
Missing nodes
Deleted assets
Renamed IDs
Circular references
Invalid transitions
```

Example:

```text id="a7b8c9"
ERROR: Dialogue node kim_intro_02 → missing kim_intro_03
```

---

# Regression Testing

Ensures updates don’t break old content.

Checks:

```text id="d1e2f3"
Old quests still completable
Dialogue paths still valid
Save compatibility intact
```

---

# Content Version Control

Tracks:

```text id="g4h5i6"
Version history
Author changes
AI generation lineage
Patch impact tracking
```

---

# Incremental Builds

Only reprocess changed content:

```text id="j7k8l9"
Changed Actor → revalidate related dialogues only
```

Avoid full rebuilds.

---

# Performance Considerations

Pipeline must scale to:

```text id="m1n2o3"
10,000+ dialogue nodes
1,000+ quests
500+ Actors
Multiple languages
AI-generated expansions
```

Optimization strategies:

```text id="p4q5r6"
Parallel validation
Graph caching
Incremental dependency tracking
Batch localization checks
```

---

# Output Artifacts

Pipeline produces:

```text id="s7t8u9"
Validated Narrative Database
Localized bundles
Dependency reports
Test simulation logs
Build manifests
Error reports
```

---

# Failure Handling

Pipeline must be strict:

```text id="v1w2x3"
Critical error → block build
Non-critical error → warn
Missing localization → fallback or fail
Broken reference → always fail
```

---

# Debug Reports

Every failure should include:

```text id="y4z5a6"
Source file
Exact node ID
Reason for failure
Dependency chain
Suggested fix
```

---

# Tooling Requirements

Essential tools:

```text id="b7c8d9"
Visual graph inspector
Dependency viewer
Localization editor
Test simulation runner
AI content validator
Reference checker
Build dashboard
```

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IContentPipeline
{
    PipelineReport Validate(ContentPackManifest manifest);
    PipelineReport Build(ContentPackManifest manifest);
    bool TryFix(ContentPackManifest manifest, out PipelineReport report);
}
```

## Domain model

```csharp
class ContentPackManifest { string PackId; string SchemaVersion; string[] AssetPaths; }
class PipelineReport { bool Success; PipelineIssue[] Issues; }
class PipelineIssue { string Severity; string Message; string AssetPath; }
```

# Minimum Viable System

For a large-scale narrative RPG:

Implement:

```text id="e1f2g3"
Schema validation
Dependency checking
Localization verification
Graph validation
Broken reference detection
Simulation testing
Build packaging
Error reporting
Incremental processing
```

---


# Final Concept

The Content Pipeline is the **industrial safety system of narrative design**.

It ensures that:

```text id="h4i5j6"
AI-generated content does not break the game
Human-authored content is consistent
Localization is complete
Dependencies are intact
Narrative graphs are valid
Builds are stable
```

It is the final gate between:

```text id="k7l8m9"
"content exists"
AND
"content can be safely shipped"
```
