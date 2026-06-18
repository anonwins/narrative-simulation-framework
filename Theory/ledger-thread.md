# 23. Ledger: Thread — NSF Specification

> **Framework:** Evidence reasoning, thread subject tracking, theory generation, thread resolution.
> **Content pack:** Inquiry content, subject definitions, evidence items, resolution branches.
> Terminology: [Glossary](terminology-glossary.md)

This system models the **investigation itself**.

It is separate from:

* Chronicle
* Story State
* Dialogue
* Discovery System

Those systems support the investigation.

The Thread **is the investigation**.

---

# Core Principle

The player is not completing quests.

The player is building an explanation of reality.

The engine should constantly answer:

```text
What happened?
Who did it?
Why did it happen?
What evidence supports that?
What evidence contradicts that?
```

A NSF-style inquiry system is fundamentally:

```text
Facts
→ Hypotheses
→ Theories
→ Conclusions
```

Not:

```text
Story beat marker (content trigger; distinct from thread progress)
→ Objective Complete
```

---

# Core Architecture

```text
ThreadEngine
 ├─ Threads
 ├─ Evidence
 ├─ ThreadSubjects
 ├─ Facts
 ├─ Hypotheses
 ├─ Contradictions
 ├─ TheoryGraph
 ├─ InvestigationState
 └─ ResolutionEngine
```

---

# Core Model

```csharp
class Thread
{
    string Id;

    List<Evidence> Evidence;

    List<ThreadSubject> ThreadSubjects;

    List<Fact> Facts;

    List<Theory> Theories;

    ThreadState State;
}
```

The Thread object becomes the central investigative container.

---

# Evidence System

Evidence is the foundation.

Evidence should be stored separately from chronicle entries.

```csharp
class Evidence
{
    string Id;

    string Description;

    EvidenceType Type;

    Reliability Reliability;

    bool Verified;
}
```

---

# Evidence Categories

Examples:

```text
Physical Evidence
Witness Testimony
Documentary Evidence
Forensic Evidence
Circumstantial Evidence
Behavioral Evidence
Environmental Evidence
```

The game should distinguish them.

---

# Evidence Is Not Truth

Critical design rule.

Evidence only indicates.

It does not automatically prove.

Example:

```text
Bloody knife found.
```

Does not necessarily mean:

```text
Knife owner is killer.
```

The engine should support uncertainty.

---

# Fact System

Facts are stronger than evidence.

Facts represent information the game considers established.

Example:

```text
Victim died at location X.
```

Fact.

---

# Fact Object

```csharp
class Fact
{
    string Id;

    bool Confirmed;

    List<string> SupportingEvidence;
}
```

Facts should be built from evidence.

---

# Inquiry Subject System

An inquiry subject is a candidate explanation.

```csharp
class ThreadSubject
{
    string CharacterId;

    int SuspicionScore;

    List<string> Motives;

    List<string> Opportunities;

    List<string> Contradictions;
}
```

---

# Inquiry Subjects Are Dynamic

The inquiry subject list should evolve.

Examples:

```text
New inquiry subject added
Inquiry subject cleared
Inquiry subject elevated
Inquiry subject hidden
```

Investigation is a moving target.

---

# Suspicion Score

Track suspicion separately from guilt.

Example:

```text
Suspicion = 10
```

means:

```text
Worth investigating.
```

Not:

```text
Definitely guilty.
```

This distinction is essential.

---

# Motive Tracking

ThreadSubjects may accumulate motives.

Examples:

```text
Money
Revenge
Politics
Fear
Jealousy
Self-defense
```

These become evidence inputs.

---

# Opportunity Tracking

Track whether The inquiry subject could plausibly act.

Examples:

```text
Present at scene
Had access
Had weapon
Had knowledge
```

Without opportunity, suspicion weakens.

---

# Means Tracking

Optional but recommended.

Track capability.

Examples:

```text
Physical strength
Technical knowledge
Access to weapon
Special expertise
```

---

# Theory System

One of the most important components.

The player should be able to form theories.

A theory is:

```text
Interpretation of evidence.
```

Not evidence itself.

---

# Theory Object

```csharp
class Theory
{
    string Id;

    string Statement;

    List<string> SupportingFacts;

    List<string> ContradictingFacts;

    TheoryState State;
}
```

---

# Theory States

Suggested states:

```text
Speculative
Plausible
Supported
Strongly Supported
Disproven
Resolved
```

This allows investigations to evolve.

---

# Multiple Theories

The system must support competing theories.

Example:

```text
Theory A:
faction_guild killed victim.

Theory B:
Mercenaries killed victim.

Theory C:
Unknown third party killed victim.
```

All may coexist temporarily.

---

# Contradiction System

Critical for inquiry gameplay.

Evidence can contradict theories.

```text
Theory
↓
New Fact Appears
↓
Theory Strength Changes
```

The engine should evaluate contradictions automatically.

---

# Investigation Graph

Represent investigation as a graph.

```text
Evidence
 ↓
Facts
 ↓
ThreadSubjects
 ↓
Theories
 ↓
Resolution
```

This is more flexible than linear objectives.

---

# Evidence Relationships

Evidence should reference other evidence.

Example:

```text
Bullet
↓
Matches Weapon
↓
Owned By ThreadSubject
```

The network itself becomes valuable information.

---

# Hidden Evidence

Some evidence should remain undiscovered.

The investigation should be incomplete until found.

This creates exploration pressure.

---

# False Leads

Support intentionally misleading information.

Examples:

```text
Mistaken witness
Forged document
Manipulated clue
Coincidental evidence
```

Without false leads, inquiry gameplay becomes trivial.

---

# Reliability System

Evidence should have reliability.

```text
High Reliability
Medium Reliability
Low Reliability
Unknown Reliability
```

Example:

```text
Eyewitness testimony
```

may be less reliable than:

```text
Forensic analysis
```

---

# Evidence Confidence

Store confidence separately.

```csharp
EvidenceConfidence
{
    Confirmed
    Probable
    Uncertain
}
```

This prevents binary reasoning.

---

# Investigation Progress

Track thread progress independently from story beat chain progress.

Example:

```text
Thread progress = 65%
Thread progress = 20%
```

The player may understand the mystery before finishing the story beat chain.

---

# Deduction Triggers

The system should detect important combinations.

Example:

```text
Fact A
+
Fact B
+
Fact C
```

unlocks:

```text
New Theory
```

This makes the player feel like they are reasoning.

---

# Faculty Integration

Faculties should interact heavily.

Examples:

```text
Logic
→ identifies contradictions

Visual Calculus
→ reconstructs events

Empathy
→ evaluates testimony

Drama
→ detects lies

Rhetoric
→ identifies ideological motive

Perception
→ reveals clues
```

The Thread should be a major consumer of faculty output.

---

# Dialogue Integration

Dialogue should query thread and story beat state.

Example:

```json
{
  "requiresEvidence": "weapon_found"
}
```

or:

```json
{
  "requiresTheory": "union_conspiracy"
}
```

Investigation should unlock conversation.

---

# Chronicle Integration

The chronicle records findings.

The Thread stores investigative truth.

Keep them separate.

```text
Chronicle
=
Player-facing notes

Thread
=
Structured investigative data
```

---

# Companion Integration

Companions should:

* discuss theories
* challenge assumptions
* remember contradictions
* suggest leads

The companion becomes a reasoning partner.

---

# Premature Conclusions

The player should be able to form conclusions too early.

Example:

```text
Arrest wrong ThreadSubject.
```

This should remain possible if the game supports it.

The investigation system should tolerate mistakes.

---

# Thread resolution

Resolution should evaluate:

```text
Known Facts
Evidence Strength
Theory Strength
Critical Discoveries
```

Not merely:

```text
Story beat completed
```

---

# Resolution States

Examples:

```text
Solved
Mostly Solved
Partially Solved
Incorrect Conclusion
Unresolved
```

A inquiry-based game benefits from imperfect endings.

---

# Multiple Truth Layers

A strong system supports:

```text
What happened
Who did it
Why they did it
Who benefited
```

These may be discovered separately.

---

# Event Integration

The Thread should emit:

```text
EvidenceFound
FactConfirmed
TheoryUnlocked
TheoryDisproven
SuspectAdded
SuspectCleared
CaseResolved
```

Other systems subscribe to these.

---

# Save Data Requirements

Serialize:

```text
Evidence Database
Fact Database
ThreadSubject State
Theory State
Investigation Graph
Thread progress
Resolution State
```

The entire reasoning structure must persist.

---

# Minimal Engine Interfaces

```csharp
interface IThreadService
{
    void AddEvidence(string evidenceId);

    void AddFact(string factId);

    void AddSuspect(string suspectId);

    void EvaluateTheories();
}
```

---

# Recommended Internal Flow

```text
Discovery
      ↓
Evidence
      ↓
Fact Formation
      ↓
Theory Generation
      ↓
Theory Testing
      ↓
Resolution
```

This should be the core loop of the investigation.

---

# Most Important Insight

The Thread is not a task tracker.

It is a **belief-management system**.

The engine should continuously model:

```text
What does the player know?
What do they think happened?
How confident are they?
What evidence supports them?
What evidence threatens their theory?
```

If implemented correctly, the player feels like a inquiry.

If implemented incorrectly, the player is simply following objectives until a cutscene reveals the answer.
