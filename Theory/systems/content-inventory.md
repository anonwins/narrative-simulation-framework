# 3–4. Content: Inventory & Equipment — NSF Specification

> **Framework:** Item architecture, equipment slots, modifiers, tool capabilities, identity tags, evidence item type.
> **Content pack:** Item definitions, outfit names, dresscode rules, faculty-themed equipment labels.
> Terminology: [Glossary](../terminology-glossary.md)

If you're building a **NSF-style engine**, an exhaustive item list is less useful than a **systems specification**. The AI agent needs to understand the underlying architecture, not the content.

# NSF Inventory & Equipment System — NSF Specification

## Design Philosophy

The inventory system is **not a loot system**.

Its primary purposes are:

1. Modify faculty values.
2. Unlock interactions.
3. Gate content.
4. Express character identity.
5. Support narrative choices.

The system is **not designed around combat power progression**.

There are:

* No item levels.
* No rarity tiers.
* No DPS.
* No armor class.
* No weapon progression.
* No durability.
* No encumbrance.

The player changes clothing because:

* A roll is coming.
* A dialogue challenge is coming.
* They want a roleplay identity.

Not because an item is statistically stronger.

---

# Core Architecture

## Inventory Container

```text
Player
 ├─ Inventory
 │   ├─ Item[]
 │   ├─ Key[]
 │   ├─ QuestItem[]
 │   └─ Currency
 │
 └─ Equipment
      ├─ Head
      ├─ Eyes
      ├─ Neck
      ├─ Torso
      ├─ Shirt
      ├─ Hands
      ├─ Legs
      ├─ Feet
      ├─ LeftHand
      └─ RightHand
```

Inventory capacity:

```text
Unlimited
```

No stack limits required.

No weight calculation required.

---

# Item Definition

Every item should be data-driven.

Example:

```json
{
  "id": "horrific_necktie",
  "name": "Horrific Necktie",
  "type": "clothing",
  "slot": "neck",
  "modifiers": [
    {
      "faculty": "inland_empire",
      "value": 1
    }
  ],
  "tags": [
    "talking_item"
  ]
}
```

---

# Item Categories

Engine should support:

```text
Clothing
Tool
Consumable
QuestItem
Key
Currency
Evidence
Miscellaneous
```

These categories drive behavior.

Not appearance.

---

# Equipment Slots

Required slots:

```text
Head
Eyes
Neck
Torso
Shirt
Hands
Legs
Feet
LeftHand
RightHand
```

Each slot:

```text
0 or 1 equipped item
```

No layering system required.

---

# Faculty Modifier System

This is the most important feature.

Every equipped item contributes modifiers.

Example:

```text
Logic +1
Authority -1
```

Multiple modifiers may exist.

Example:

```text
White Jacket

+1 Suggestion
+1 Drama
-1 Authority
```

Engine formula:

```text
Final Faculty Value
=
Base Faculty
+
Equipment Bonuses
+
Belief Bonuses
+
Temporary Effects
```

---

# Modifier Stack Resolution

Need deterministic order.

```text
Base Faculty
→ Attribute Contribution
→ Equipment
→ Beliefs\n→ Temporary Buffs
→ Environmental Buffs
→ Final Faculty
```

Never hardcode bonuses.

Everything should be data-driven.

---

# Check Preparation Loop

NSF's gameplay loop:

```text
Encounter check
↓
Review probability
↓
Open inventory
↓
Swap clothing
↓
Retry check
```

Engine must support:

```text
Instant equipment swapping
```

No cooldown.

No animation lock.

No time cost.

This behavior is intentional.

---

# Check Preview Integration

Before a check:

```text
Physical Instrument [43%]
```

Player can inspect:

```text
+1 Jacket
+1 Gloves
-1 Boots
```

Engine needs dynamic probability recalculation.

Whenever equipment changes:

```text
Recompute faculty
Recompute chance
Refresh UI
```

Immediately.

---

# Faculty Outfit Emergence

Players naturally create:

```text
Logic Outfit
Authority Outfit
Empathy Outfit
faculty_environment Outfit
Physical Instrument Outfit
```

The engine should not explicitly support outfits.

They emerge automatically from the modifier system.

Optional quality-of-life feature:

```text
Saved Loadouts
```

---

# Tool System

Tools are not clothing.

Tools unlock actions.

Example:

```text
Bolt Cutters
```

Enables:

```text
CanOpenChainLock = true
```

Example:

```text
Flashlight
```

Enables:

```text
CanSeeDarkObjects = true
```

Implementation:

```json
{
  "capabilities": [
    "open_chain_lock"
  ]
}
```

World interactions check capability flags.

Not specific item IDs.

---

# Interaction Gating

Dialogue and world interactions should support:

```text
Requires Item
Requires Capability
Requires Faculty
Requires Belief
```

Example:

```text
Open Door

Requirements:
    Key_Apartment
```

Example:

```text
Cut Fence

Requirements:
    capability = cut_wire
```

---

# Keys System

Keys should be separate from inventory clutter.

Architecture:

```text
Player
 └─ Keyring
      ├─ Apartment Key
      ├─ Harbor Key
      └─ ...
```

Keys persist forever.

No equip step.

---

# Story beat items

Story beat items should support:

```text
Invisible
Visible
Protected
Consumable
```

Protected means:

```text
Cannot Sell
Cannot Drop
```

---

# Evidence System

Important for inquiry gameplay.

Each evidence item should contain:

```json
{
  "evidenceType": "bullet",
  "description": "...",
  "caseTags": [
    "lynching"
  ]
}
```

Allows:

* Dialogue references.
* Chronicle updates.
* Investigation logic.

---

# Currency System

Currency should not be inventory items.

Store separately.

```text
Player.money
```

Supports:

```text
GainMoney()
SpendMoney()
CheckMoney()
```

---

# Consumable Architecture

Consumables should apply temporary modifiers.

Example:

```json
{
  "effect": {
    "attribute": "physique",
    "modifier": 1,
    "duration": 3600
  }
}
```

Effects can:

* Modify attributes.
* Modify faculties.
* Modify caps.

---

# Talking Item System

One of NSF's unique features.

Certain items behave like faculties.

Example:

```text
Horrific Necktie
```

Architecture:

```json
{
  "hasVoice": true,
  "voiceFrequency": 0.35,
  "voicePriority": "medium"
}
```

Dialogue engine should treat item voices exactly like faculty voices.

Possible speaker types:

```text
Faculty
Belief
Item
Narrator
Actor
```

This is critical if recreating NSF's feel.

---

# Clothing Personality System

Items are not merely stat modifiers.

Each item may contain:

```json
{
  "identityTags": [
    "communist",
    "ultraliberal",
    "artistic",
    "authoritarian"
  ]
}
```

Dialogue can react.

Example:

```text
If wearing military gear:
    +1 reaction
```

or

```text
Actor notices expensive clothing.
```

---

# Outfit Recognition System

The game occasionally recognizes combinations.

Engine architecture:

```json
{
  "outfitId": "ceramic_armor_set",
  "requiredItems": [
    "helmet",
    "gauntlets",
    "cuirass"
  ]
}
```

When complete:

```text
Activate Set Tag
```

Not necessarily stat bonuses.

May unlock dialogue.

---

# Inventory UI Requirements

Player must see:

```text
Name
Description
Modifiers
Slot
Tags
Story beat importance
```

For faculty modifiers:

```text
+1 Drama
-1 Authority
```

Must be immediately readable.

---

# Item Examination System

Extremely important.

Every item should support:

```text
Description
Flavor Text
Lore Text
Interaction Text
```

Many NSF items exist primarily for narrative reading.

---

# Serialization Requirements

Save file must store:

```text
Inventory Contents
Equipped Items
Consumable Effects
Money
Keys
Story beat flags
Evidence Collected
```

Everything should be recoverable from data.

---

# Minimal Engine Interfaces

```csharp
interface IItem
{
    string Id;
    string Name;
    ItemType Type;
}

interface IEquipable
{
    Slot Slot;
    Modifier[] Modifiers;
}

interface IConsumable
{
    Effect[] Effects;
}

interface ITool
{
    Capability[] Capabilities;
}

interface IVoiceSource
{
    DialogueLine[] Barks;
}
```

---

# Most Important Implementation Insight

A NSF-style inventory system is **not an RPG equipment system**.

It's better described as:

```text
Narrative State Modifier System
```

Items primarily alter:

* Faculty values
* Dialogue availability
* Character identity
* Investigation capabilities
* Narrative interpretation

The engine should therefore integrate inventory deeply with:

```text
Rolls
Dialogue
Belief
Story state logic
Interactions
Narration System
```

rather than treating it as a separate loot subsystem. That's the architectural choice that makes NSF feel fundamentally different from traditional RPGs.
