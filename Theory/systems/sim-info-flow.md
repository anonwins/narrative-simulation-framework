# 29. Sim: Info Flow — NSF Specification

> **Framework:** Rumor spread, witnesses, leakage, discovery chains, public vs secret knowledge.
> **Content pack:** Propagation rules content, witness definitions, leak triggers.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The Info Flow tracks the movement of information through the world.

This is **not a fact database**.

Facts answer:

```text
What is true?
```

Information propagation answers:

```text
Who knows it?
When did they learn it?
Who told them?
How reliable is it?
Who else did they tell?
```

This system allows:

```text
Rumors
Witness reports
Leaked information
Conspiracies
Public news
Secret knowledge
Discovery chains
Delayed reactions
False information
```

Without this system, Actors often behave as if knowledge instantly teleports across the world.

Players notice this quickly.

---

# Core Principle

Truth and knowledge are separate.

Example:

```text
The king is dead.
```

Truth:

```text
King = Dead
```

Knowledge:

```text
Guard A knows
Guard B doesn't know
Merchant doesn't know
Rebels know
Queen knows
```

The world reacts according to knowledge, not truth.

---

# Information Record

Every piece of information becomes an information object.

Example:

```text
Information:
{
    id: "king_dead",
    content: "The king is dead"
}
```

The content itself may refer to a fact.

The propagation system only tracks ownership.

---

# Fact ownership

Each entity can know information.

```text
Character
Faction
Organization
Settlement
Government
Media source
```

Example:

```text
Actor:
{
    known_information:
    [
        king_dead,
        tax_increase,
        rebellion_location
    ]
}
```

---

# Acquisition Record

Whenever knowledge is acquired:

```text
KnowledgeEntry
{
    information_id
    source
    timestamp
    confidence
}
```

Example:

```text
{
    information_id: king_dead
    source: witness_guard
    timestamp: day_14
    confidence: 0.95
}
```

Now the game knows:

```text
What was learned
Who learned it
When
From whom
How certain they are
```

---

# Information Sources

Information can originate from:

```text
Witnessing
Conversation
Documents
Observation
Investigation
Rumors
News systems
Interrogation
Magic
Technology
```

Example:

```text
Player sees incident.

Information:
incident_witnessed
```

Source:

```text
direct_witness
```

Confidence:

```text
100%
```

---

# Witness System

Witnesses create information.

Example:

```text
Player steals item.
```

Nearby Actor:

```text
Witnessed theft.
```

Generated:

```text
information:
player_stole_item
```

Ownership:

```text
Witness knows
```

Not:

```text
Entire city knows
```

Yet.

---

# Information Transfer

Facts move through interactions.

Example:

```text
Witness
→ Guard
```

Guard gains:

```text
player_stole_item
```

Fact entry:

```text
source = witness
```

Propagation begins.

---

# Rumor Propagation

Information may spread automatically.

Example:

```text
Tavern patron learns rumor.
```

Daily update:

```text
Chance to tell nearby people.
```

Result:

```text
Patron
→ Merchant

Patron
→ Farmer

Patron
→ Traveler
```

Fact graph expands.

---

# Public Knowledge

Some information eventually becomes public.

Example:

```text
King dies.
```

Initially:

```text
Royal family knows
```

Then:

```text
Guards know
```

Then:

```text
Capital knows
```

Then:

```text
Entire nation knows
```

Propagation occurs over time.

Not instantly.

---

# Secret Knowledge

Some information remains restricted.

Example:

```text
Hidden assassination plot
```

Known by:

```text
Conspirator A
Conspirator B
Conspirator C
```

Nobody else.

Propagation is intentionally limited.

---

# Leakage

Secrets may leak.

Example:

```text
Conspirator drinks in tavern.
```

Chance:

```text
Loose talk
```

Creates:

```text
Rumor version
```

Now outsiders may learn fragments.

---

# Confidence Levels

Facts should include certainty.

Example:

```text
Direct witness:
confidence = 1.0
```

Rumor:

```text
confidence = 0.4
```

Third-hand rumor:

```text
confidence = 0.15
```

Actors can react differently.

```text
Certain
Suspicious
Uncertain
Disbelieving
```

---

# Information Mutation

Information may change while spreading.

Example:

Original:

```text
Bandits attacked caravan.
```

After propagation:

```text
50 bandits attacked caravan.
```

Later:

```text
100 bandits destroyed entire trade route.
```

Rumors evolve.

Not all information remains accurate.

---

# Discovery Chains

Information often unlocks further information.

Example:

```text
Player finds key.
```

Learns:

```text
key_exists
```

Then:

```text
key_opens_vault
```

Then:

```text
vault_contains_documents
```

Then:

```text
documents_implicate_noble
```

Facts create more knowledge.

---

# Information Dependencies

Some information cannot be understood without prior knowledge.

Example:

```text
Secret code phrase
```

Requires:

```text
knows_cipher
```

Otherwise:

```text
Meaning unknown
```

This creates investigative gameplay.

---

# Faction Information Sharing

Organizations may share knowledge internally.

Example:

```text
Guard Captain learns theft.
```

Propagation:

```text
Captain
→ Guard Network
```

After delay:

```text
Entire guard faction knows
```

This is not conversation-based.

It is organizational propagation.

---

# Organizational Delay

Information should travel at realistic speeds.

Example:

```text
Village
→ Capital
```

Travel time:

```text
3 days
```

Reaction occurs after arrival.

Not immediately.

---

# Reaction Lag

Fact and action are separate.

Example:

```text
Faction learns rebellion.
```

Immediately:

```text
Fact updated
```

Later:

```text
Meeting scheduled
```

Later:

```text
Orders issued
```

Later:

```text
Troops deployed
```

Information causes reactions over time.

---

# Public Event Generation

Highly propagated information can become events.

Example:

```text
Dragon attack witnessed.
```

Spreads through:

```text
Witnesses
Merchants
Travelers
Officials
```

Eventually:

```text
Public Event:
Dragon Threat
```

Entire region responds.

---

# Investigation Gameplay

Players can trace information origins.

Example:

```text
Who told you?
```

Actor:

```text
Merchant
```

Merchant:

```text
Traveler
```

Traveler:

```text
Guard
```

Guard:

```text
Witness
```

The full propagation chain can be reconstructed.

---

# Information Graph

Internally:

```text
Witness
    ↓
Guard
    ↓
Merchant
    ↓
Faction
    ↓
Mayor
```

This forms a directed graph.

Useful for:

```text
Investigations
Rumors
Politics
Conspiracies
Story state logic
```

---

# False Information

Not all information must be true.

Example:

```text
Information:
"The player is a spy."
```

Fact status:

```text
False
```

Fact status:

```text
Widely believed
```

The world reacts to belief.

Not necessarily reality.

---

# Save Structure

```text
Information
{
    id
    content
    truth_status
}
```

```text
KnowledgeEntry
{
    information_id
    owner
    source
    timestamp
    confidence
}
```

```text
PropagationEdge
{
    from
    to
    information_id
    timestamp
}
```

---

# Minimum Viable System

For An NSF-Elysium-inspired RPG:

Implement:

```text
Information Objects
Fact ownership
Witness Generation
Conversation Transfer
Faction Sharing
Confidence Levels
Secrets
Rumors
Delayed Reactions
Discovery Chains
```

This alone creates worlds where information behaves like a real resource rather than a magical global variable. Actors can know different things, learn at different times, spread information selectively, conceal secrets, leak rumors, and react only after information reaches them. That dramatically increases the believability of investigations, politics, social dynamics, and narrative consequences.
