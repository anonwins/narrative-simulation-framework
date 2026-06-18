# 22. Social: Faction — NSF Specification

> **Framework:** Faction reputation, influence, stance, institutional reactions, internal stability.
> **Content pack:** Faction IDs, agendas, member definitions, political content.
> Terminology: [Glossary](../terminology-glossary.md)

The Faction models **organized groups with shared interests, power structures, and agendas**.

This is not simply:

```text id="fac1"
Faction = Reputation Bar
```

A NSF-style faction system is closer to:

```text id="fac2"
Faction = Living political actor
```

Every major faction should:

* want things
* fear things
* know things
* hide things
* influence people
* react to player behavior

The player is investigating a world shaped by competing groups.

---

# Core Principle

The player does not join factions like a typical RPG.

Instead:

```text id="fac3"
Faction observes player
↓
Faction updates opinion
↓
Faction changes behavior
↓
World reacts
```

The player becomes entangled in factional politics whether they want to or not.

---

# Core Architecture

```text id="fac4"
FactionService
 ├─ Factions
 ├─ Members
 ├─ FactionRelationships
 ├─ FactionReputation
 ├─ FactionKnowledge
 ├─ FactionInfluence
 ├─ FactionEvents
 ├─ PoliticalLinks
 └─ FactionGoals
```

---

# Core Model

```csharp id="fac5"
class Faction
{
    string Id;

    string Name;

    FactionGoals Goals;

    FactionResources Resources;

    FactionInfluence Influence;

    ReputationProfile Reputation;
}
```

A faction is a world entity, not just a dialogue tag.

---

# Major Faction Categories

A NSF-style game should support:

```text id="fac6"
Government
Police
Corporations
Labor Organizations
Criminal Groups
Ideological movements
Local Communities
Religious Organizations
Informal Networks
```

Not all factions need equal size.

---

# Example Faction Structure

A faction should contain:

```text id="fac7"
Leadership
Members
Territory
Goals
Enemies
Allies
Resources
Secrets
```

These properties drive behavior.

---

# Faction Membership

Actors should belong to one or more factions.

```csharp id="fac8"
class FactionMembership
{
    string FactionId;

    MembershipStrength Strength;

    RoleType Role;
}
```

Possible roles:

```text id="fac9"
Leader
Officer
Member
Associate
Informant
Former Member
```

---

# Multi-Faction Membership

Support multiple memberships.

Example:

```text id="fac10"
Police Officer
+
Ideological sympathizer
```

or

```text id="fac11"
actor_member_faction_guild
+
Community Leader
```

This creates believable social complexity.

---

# Faction Goals

Every faction should have explicit goals.

Example:

```text id="fac12"
Protect territory
Control labor
Increase profits
Preserve order
Influence elections
Hide information
```

Faction behavior emerges from goals.

---

# Faction Resources

Track faction capabilities.

Examples:

```text id="fac13"
Money
Ideological influence
Violence Capacity
Information
Personnel
Territory
Public Support
```

These determine what a faction can actually do.

---

# Faction Influence

Influence represents power in the world.

Example:

```text id="fac14"
Local Influence
Regional Influence
Institutional Influence
```

Influence can affect:

* dialogue
* story beat outcomes
* Actor behavior
* event generation

---

# Faction Reputation

Track player standing with factions.

```csharp id="fac15"
class FactionReputation
{
    int Trust;
    int Respect;
    int Suspicion;
    int Hostility;
}
```

Do not use a single reputation number.

---

# Reputation Is Not Friendship

Example:

```text id="fac16"
Trust = Low
Respect = High
```

Meaning:

```text id="fac17"
Faction thinks player is competent
but not aligned.
```

Nuance is important.

---

# Faction Observation

Factions should learn from player actions.

Examples:

```text id="fac18"
Supported faction goal
Undermined faction goal
Helped member
Insulted leader
Leaked information
Solved problem
```

These emit faction reputation events.

---

# Indirect Reputation Changes

Important mechanic.

Helping one faction may affect another.

Example:

```text id="fac19"
Help faction_guild
↓
faction_guild Trust +2

Corporation Suspicion +2
```

The world should feel interconnected.

---

# Faction Relationships

Factions should have opinions of each other.

```csharp id="fac20"
class FactionRelationship
{
    string FactionA;
    string FactionB;

    RelationshipType Type;
}
```

Possible values:

```text id="fac21"
Ally
Partner
Neutral
Rival
Enemy
```

---

# Dynamic Faction Relations

Faction relationships can change.

Examples:

```text id="fac22"
Negotiation
Conflict
Shared Threat
Ideological shift
```

This allows evolving world states.

---

# Faction Knowledge

Every faction should possess information.

```text id="fac23"
Known Facts
Rumors
Secrets
Evidence
Witnesses
```

Information becomes a faction resource.

---

# Information Ownership

Important concept.

Example:

```text id="fac24"
Fact exists
↓
Faction knows fact
↓
Player may acquire fact
```

Facts should not magically belong to everyone.

---

# Faction Territory

Support territorial control.

Examples:

```text id="fac25"
Neighborhood
Building
Business
District
```

Territory affects:

* Actor presence
* safety
* dialogue
* encounters

---

# Territory Influence

Entering a faction-controlled area may alter:

```text id="fac26"
Available dialogue
Actor behavior
Risk level
Story beat opportunities
```

The world should feel politically mapped.

---

# Faction Reactions

Factions should react to player identity.

Inputs:

```text id="fac27"
Ideology
Conduct
Relationships
Story beat Choices
Standing
```

Different factions care about different things.

---

# Faction Events

Examples:

```text id="fac28"
Strike
Meeting
Violence
Election
Arrest
Negotiation
Investigation
```

Events create world movement.

---

# Faction Event bus

```text id="fac29"
Faction Goal
↓
Faction Action
↓
World Event
↓
Player Reaction
↓
Faction Update
```

The player participates in a larger ecosystem.

---

# Faction Dialogue Integration

Dialogue should support:

```json id="fac30"
{
  "requiresFactionRep": {
    "faction_guild": 5
  }
}
```

Or:

```json id="fac31"
{
  "blockedByFactionHostility": "WildPines"
}
```

Faction state influences conversation availability.

---

# Faction story beat integration

Quests often involve factions indirectly.

Examples:

```text id="fac32"
Protect faction
Expose faction
Negotiate between factions
Investigate faction
Work around faction
```

Factions generate narrative pressure.

---

# Faction Memory

Factions should remember major events.

Examples:

```text id="fac33"
Player helped strike
Player exposed corruption
Player sided with leadership
```

Memory creates continuity.

---

# Public Versus Private Factions

Important for inquiry-based games.

A faction may present:

```text id="fac34"
Public Goal
```

while secretly pursuing:

```text id="fac35"
Private Goal
```

This enables conspiracy and investigation.

---

# Hidden Influence

Not all faction power should be visible.

Examples:

```text id="fac36"
Ideological pressure
Financial leverage
Informants
Corruption
```

The player gradually uncovers hidden structures.

---

# Faction and Companion Reactions

Companions should react to faction involvement.

Examples:

```text id="fac37"
Approval
Concern
Warning
Disagreement
```

Faction politics should feel socially significant.

---

# Faction and Economy

Factions can affect:

```text id="fac38"
Prices
Employment
Loans
Resources
Access
```

Economic systems and factions should interact.

---

# Faction and Locations

Locations should know:

```csharp id="fac39"
LocationOwnerFaction
```

This allows:

* territory control
* environmental storytelling
* security behavior
* faction encounters

---

# Faction and Ideology

Ideological identity should influence faction responses.

Example:

```text id="fac40"
Communist player
↓
Certain factions become friendlier

Others become suspicious
```

This should be gradual, not binary.

---

# Serialization Requirements

Serialize:

```text id="fac41"
Faction Reputation
Faction Relationships
Faction fact state
Faction Event History
Faction Territory State
Faction Goal Progress
```

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface IFactionService
{
    int GetStanding(string factionId);
    void ModifyStanding(string factionId, int delta);
    FactionStance GetStance(string factionId);
    IReadOnlyList<string> GetMemberActorIds(string factionId);
}
```

## Domain model

```csharp
class FactionDefinition { string Id; string NameKey; }
enum FactionStance { Neutral, Friendly, Hostile, Allied }
interface IFaction { string Id; int Standing; FactionStance Stance; }
```


# Recommended Internal Flow

```text id="fac43"
Player Action
      ↓
Faction Observation
      ↓
Reputation Update
      ↓
Goal Evaluation
      ↓
Faction Response
      ↓
World Change
```

This keeps factions reactive without scripting every possibility.

---

# Most Important Insight

The Faction is not about choosing sides.

It is about simulating a world where organized groups have competing interests.

The engine should constantly answer:

```text id="fac44"
Who benefits?
Who loses?
Who notices?
Who reacts?
```

If implemented correctly, factions become invisible engines driving the narrative.

The player stops seeing isolated Actors and starts seeing the political machinery behind them.

That realization is one of the defining experiences of a NSF-style world.
