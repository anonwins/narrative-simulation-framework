# 28. Sim: Fact — NSF Specification

> **Framework:** Facts, truth states, knowledge ownership, rumor vs known, fact-based gating.
> **Content pack:** Fact definitions, who-knows-what assignments, rumor content.
> Terminology: [Glossary](../terminology-glossary.md)

This system is the **informational reality model** of the game.

Most RPGs track:

```text
Story beat state
Inventory
Dialogue Flags
```

A NSF-style game must additionally track:

```text
Who knows what?
Who believes what?
What is actually true?
Who thinks it is true?
```

This distinction is crucial.

Without it:

```text
Facts = global flags
```

With it:

```text
Facts = distributed information network
```

The player is not simply progressing.

The player is learning.

---

# Core Principle

The game world should distinguish:

```text
Reality
Knowledge
Belief
Rumor
```

These are not the same thing.

Example:

```text
Victim was incident_occurred.
```

May be:

```text
Actually true.
```

while:

```text
Actor believes it was suicide.
```

and:

```text
Faction spreads rumor of political assassination.
```

All three can coexist.

---

# Core Architecture

```text
FactService
 ├─ Facts
 ├─ Fact holders
 ├─ Beliefs
 ├─ Rumors
 ├─ Information Sources
 ├─ Fact Propagation
 ├─ Fact Discovery
 ├─ Fact queries
 └─ Truth Layer
```

---

# Core Model

```csharp
class Fact
{
    string Id;

    string Description;

    TruthState Truth;

    FactVisibility Visibility;
}
```

A Fact exists independently of who knows it.

---

# Core Design Rule

Facts belong to the world.

Facts belong to entities.

Example:

```text
Fact:
Victim had affair.
```

World contains fact.

Player may not know it.

Actor may know it.

Faction may know it.

The fact exists regardless.

---

# Truth Layer

Every fact should have truth state.

```csharp
enum TruthState
{
    True,
    False,
    Unknown,
    Disputed
}
```

Examples:

```text
Victim was incident_occurred.
→ True

faction_guild ordered incident.
→ Unknown

Witness testimony.
→ Disputed
```

---

# Objective Truth

The engine should support objective reality.

Example:

```text
Actual killer = Character X
```

Even if:

```text
Nobody knows.
```

This enables investigations.

---

# Fact holders

Facts can belong to:

```text
Player
Actor
Faction
Location
World
Companion
```

Every holder maintains separate knowledge.

---

# Fact profile

```csharp
class KnowledgeProfile
{
    HashSet<string> KnownFacts;
}
```

Every major entity should possess one.

---

# Known vs Unknown

A critical distinction.

Example:

```text
Fact exists.
```

does NOT imply:

```text
Player knows fact.
```

The game should separate:

```text
World Fact
```

from:

```text
Player Knowledge
```

---

# Discovery

Facts become known through discovery.

Sources:

```text
Dialogue
Observation
Documents
Evidence
Faculties
Companions
Events
```

---

# Fact Acquisition

Basic flow:

```text
Fact Exists
↓
Player Encounters Source
↓
Fact Learned
↓
Fact updated
```

Facts are progression.

---

# Fact States

For player knowledge:

```csharp
enum KnowledgeState
{
    Unknown,
    Heard,
    Known,
    Confirmed
}
```

Example:

```text
Rumor heard
→ Heard

Evidence found
→ Confirmed
```

---

# Heard vs Known

Very important.

Example:

```text
"People say the union did it."
```

Player has:

```text
Heard
```

not:

```text
Known
```

This distinction supports uncertainty.

---

# Fact Confirmation

A fact may evolve.

```text
Unknown
↓
Rumored
↓
Suspected
↓
Confirmed
```

Many inquiry mechanics depend on this.

---

# Belief

Facts are not belief.

Example:

```text
Actor knows victim had affair.
```

But believes:

```text
Affair caused incident.
```

Beliefs interpret facts.

---

# Belief Model

```csharp
class Belief
{
    string Id;

    float Confidence;

    List<string> SupportingFacts;
}
```

Useful for advanced simulation.

---

# Rumor System

Rumors are unverified information.

```csharp
class Rumor
{
    string Id;

    string Statement;

    string Source;
}
```

Rumors may be:

```text
True
False
Partially True
Manipulated
```

---

# Rumor Lifecycle

```text
Rumor
↓
Investigation
↓
Confirmation
OR
Disproof
```

Rumors generate leads.

---

# Information Sources

Every fact should have origins.

Examples:

```text
Witness
Document
Evidence
Observation
Actor
Faction
Companion
```

The source affects credibility.

---

# Source Reliability

Track reliability.

```csharp
enum Reliability
{
    Low,
    Medium,
    High
}
```

Examples:

```text
Drunk witness
→ Low

Forensic report
→ High
```

---

# Fact Ownership

Important concept.

A fact should know:

```text
Who currently knows it.
```

Example:

```csharp
Fact
{
    Holders:
    {
        Player,
        Kim,
        actor_leader_faction_guild
    }
}
```

This enables propagation.

---

# Fact Propagation

Facts should spread.

Example:

```text
Player learns fact
↓
Tells Kim
↓
actor_companion knows fact
```

Fact transfer should be explicit.

---

# Propagation Model

```text
Source
↓
Transmission
↓
Recipient
↓
Fact update
```

This creates believable information flow.

---

# Secret Information

Some facts should remain private.

Example:

```text
Actor knows secret.
```

Player does not.

This creates investigative goals.

---

# Public Knowledge

Other facts are effectively global.

Examples:

```text
Strike occurring
Building burned
Election happened
```

Everyone knows.

---

# Fact Visibility

Suggested states:

```text
Private
Restricted
Public
Global
```

Visibility affects propagation rules.

---

# Fact-Based Dialogue

One of the most important uses.

Dialogue should frequently check:

```json
{
  "requiresFact":
  "victim_affair"
}
```

Fact unlocks conversation.

---

# Knowledge-Driven Dialogue

Many NSF conversations are actually:

```text
Do you know enough to ask this?
```

rather than:

```text
Did the story beat complete?
```

This is a critical distinction.

---

# Fact-Based Responses

Example:

```text
Player learns clue.
↓
New accusation appears.
```

The player gained no item.

Only knowledge.

Yet content changed.

---

# Fact and Faculties

Faculties frequently generate facts.

Examples:

```text
Perception
→ Notice clue

Logic
→ Infer fact

Encyclopedia
→ Historical knowledge

Empathy
→ Emotional fact
```

Faculties are major fact producers.

---

# Fact and Thread

The Thread consumes facts.

Example:

```text
Fact
↓
Theory
↓
Conclusion
```

Facts are inputs to investigation.

---

# Fact and Chronicle

The Chronicle records knowledge.

The Fact service owns knowledge.

Important distinction.

```text
Chronicle
=
Presentation

Fact service
=
Truth Database
```

---

# Fact and Gate

Many gates should be knowledge-based.

Example:

```json
{
  "factKnown":
  "union_corruption"
}
```

Facts themselves becomes progression.

---

# Fact and Factions

Factions possess knowledge.

Examples:

```text
faction_guild knows labor secrets.

Corporation knows financial secrets.

Actors know thread facts.
```

Information becomes a faction resource.

---

# Fact and Actor Behavior

Actors should react based on what they know.

Example:

```text
Actor knows player lied.
```

Behavior changes.

Even if player assumes secrecy.

---

# False Knowledge

Support incorrect information.

Example:

```text
Player believes:
faction_guild killed victim.
```

Reality:

```text
Wrong.
```

inquiry gameplay benefits greatly from this.

---

# Information Hierarchy

Recommended structure:

```text
Reality
↓
Facts
↓
Knowledge
↓
Beliefs
↓
Dialogue
↓
Actions
```

This creates coherent reasoning.

---

# Dynamic Fact updates

Facts may change.

Example:

```text
Witness recants statement.
```

Facts should update accordingly.

---

# Fact history

Store:

```text
When learned
How learned
Source
Confidence
```

Useful for investigative content.

---

# Event Integration

Emit:

```text
FactDiscovered
FactConfirmed
FactShared
RumorHeard
BeliefChanged
KnowledgeUpdated
```

Many systems subscribe.

---

# Save Data Requirements

Serialize:

```text
Fact Database
Fact holders
Known Facts
Rumors
Beliefs
Confirmation State
Discovery History
```

Fact state must persist.

---

# Minimal Engine Interfaces

```csharp
interface IFactService
{
    bool KnowsFact(
        Entity entity,
        string factId);

    void LearnFact(
        Entity entity,
        string factId);

    void ShareFact(
        Entity source,
        Entity target,
        string factId);
}
```

---

# Relationship to Other Systems

This system sits beneath much of the narrative architecture.

```text
Fact service
        ↓
Dialogue
Thread
Gate
Chronicle
Relationships
Factions
Events
```

Many narrative systems ultimately operate on information.

---

# Recommended Internal Flow

```text
World Reality
      ↓
Fact Exists
      ↓
Fact Discovered
      ↓
Fact updated
      ↓
Dialogue Unlocked
      ↓
Theory Generated
      ↓
New Discoveries
```

This creates the investigative loop.

---

# Most Important Insight

The Fact service is the system that separates:

```text
What is true
```

from:

```text
What people know
```

The engine should continuously answer:

```text
What happened?
Who knows it?
Who thinks they know it?
Who is wrong?
Who can learn it next?
```

If implemented correctly, the world feels like a network of information moving between people.

If implemented incorrectly, knowledge collapses into story flags and the investigation loses much of its depth.
