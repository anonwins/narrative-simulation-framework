# 30. Rules: Rule Engine — NSF Specification

> **Framework:** Condition evaluation, boolean logic, IF/THEN rules, state queries, action execution.
> **Content pack:** Rule definitions, condition thresholds, action payloads.
> Terminology: [Glossary](../terminology-glossary.md)

## Purpose

The Rule Engine is the decision-making layer that determines:

```text
What can happen?
When can it happen?
Why can it happen?
What should happen next?
```

Almost every narrative system depends on it.

Including:

```text
Dialogue
Quests
Events
Faculty checks
Reactions
Barks
Companion behavior
Scene unlocking
Ending logic
World simulation
```

Without a rule engine, content becomes hardcoded.

With a rule engine, content becomes data-driven.

---

# Core Principle

The rule engine evaluates conditions and executes outcomes.

Basic structure:

```text
IF
Conditions

THEN
Actions
```

Example:

```text
IF

metric_trust_companion > 4
AND
Day >= 3
AND
BodyExamined

THEN

UnlockScene("KimConfession")
```

The engine decides whether the scene exists.

Not the scene itself.

---

# Responsibilities

The Rule Engine should:

```text
Evaluate conditions
Resolve state queries
Apply boolean logic
Execute actions
Trigger events
Unlock content
Prevent invalid content
```

It does not:

```text
Store facts
Store dialogue
Store quests
Store relationships
```

It queries those systems.

---

# Architecture

The engine sits above all narrative systems.

```text
World State
Dialogue State
Story beat state
Relationship State
Information State
Time State
Faculty State

        ↓

Rule Engine

        ↓

Actions
```

Think of it as the brain connecting every subsystem.

---

# Rule Structure

Every rule contains:

```text
Rule
{
    id
    conditions
    actions
}
```

Example:

```text
Rule
{
    id: unlock_kim_scene

    conditions:
    [
        metric_trust_companion > 4,
        Day >= 3,
        BodyExamined == true
    ]

    actions:
    [
        UnlockScene(KimConfession)
    ]
}
```

---

# Conditions

Conditions evaluate world state.

Examples:

```text
QuestCompleted
QuestFailed
FlagSet
RelationshipValue
TimeOfDay
Location
FacultyLevel
InformationKnown
TraitOwned
EventOccurred
```

Example:

```text
Player.Level >= 10
```

Example:

```text
StoryBeat.thread_main.Completed
```

Example:

```text
Faction.Revolutionaries.Reputation > 50
```

---

# State Queries

Conditions should never directly access raw data.

Instead:

```text
Query()
```

Example:

```text
HasFact("BodyExamined")
```

```text
Relationship("Kim") > 4
```

```text
CurrentDay() >= 3
```

The rule engine becomes independent from implementation details.

---

# Boolean Logic

Rules require logic composition.

Support:

```text
AND
OR
NOT
```

Example:

```text
BodyExamined
AND
metric_trust_companion > 4
```

---

Example:

```text
BodyExamined
OR
AutopsyReportFound
```

---

Example:

```text
NOT
Arrested
```

---

# Nested Logic

Complex rules require grouping.

Example:

```text
(
    metric_trust_companion > 4
    OR
    KimRespect > 5
)

AND

BodyExamined

AND

NOT
CaseSolved
```

Evaluation should support arbitrary nesting.

---

# Comparison Operators

Minimum operators:

```text
==
!=
>
<
>=
<=
```

Example:

```text
Money >= 50
```

```text
Morale < 2
```

```text
CurrentDay == 5
```

---

# Truth Conditions

Most narrative conditions are simple truth checks.

Example:

```text
BodyExamined == true
```

Equivalent:

```text
BodyExamined
```

---

# Dialogue Conditions

Dialogue options use rules constantly.

Example:

```text
Option:
"Tell actor_companion the truth."
```

Condition:

```text
metric_trust_companion >= 3
```

If false:

```text
Option hidden
```

If true:

```text
Option shown
```

---

# Dialogue Branch Rules

Example:

```text
IF

Authority >= 5

THEN

ShowAuthorityOption
```

Player sees a unique branch.

---

# Story beat Conditions

Story beat progression depends on rules.

Example:

```text
IF

BodyExamined
AND
WitnessInterviewed
AND
EvidenceCollected

THEN

AdvanceQuest()
```

Story state (`IStoryStateService`) becomes declarative.

---

# Event Conditions

World events should use rules.

Example:

```text
IF

Day >= 4
AND
StrikeSupport > 60

THEN

StartStrikeEvent
```

---

# Scene Unlocking

Many scenes exist but remain locked.

Example:

```text
IF

Relationship(Kim) >= 5

THEN

UnlockScene("KimRooftopTalk")
```

---

# Relationship Rules

Companion interactions often depend on relationship values.

Example:

```text
IF

Trust(Kim) >= 7

THEN

UnlockPersonalDialogue
```

---

# Information Conditions

Rules should integrate with the Info Flow.

Example:

```text
IF

Knows("player_stole_item")
```

Guard reacts.

---

Example:

```text
IF

NOT Knows("player_stole_item")
```

Guard behaves normally.

---

# Time Conditions

Time is one of the most common narrative gates.

Example:

```text
Day >= 3
```

Example:

```text
Hour >= 22
```

Example:

```text
Weekend == true
```

---

# Location Conditions

Example:

```text
PlayerLocation == Harbor
```

Example:

```text
InsideBuilding == true
```

---

# Faculty Conditions

NSF-style checks depend heavily on rules.

Example:

```text
Logic >= 6
```

---

Example:

```text
Empathy >= 4
```

---

Example:

```text
Drama >= 5
AND
metric_trust_companion > 3
```

---

# Rule Priorities

Multiple rules may become valid simultaneously.

Priority resolves conflicts.

Example:

```text
Priority 100
CompanionDeathScene
```

```text
Priority 20
CompanionGreeting
```

Higher priority executes first.

---

# Rule Categories

Organize rules by domain.

Example:

```text
DialogueRules
QuestRules
CompanionRules
EventRules
WorldRules
ReactionRules
EndingRules
```

This improves maintainability.

---

# One-Time Rules

Some rules should execute only once.

Example:

```text
IF

BodyExamined

THEN

PlayCutscene
```

After execution:

```text
Consumed = true
```

Never fires again.

---

# Repeatable Rules

Other rules evaluate continuously.

Example:

```text
IF

PlayerInRestrictedArea
```

Then:

```text
GuardWarning()
```

Can trigger repeatedly.

---

# Rule Cooldowns

Some rules should not spam.

Example:

```text
BeggarComment
```

Cooldown:

```text
10 minutes
```

Prevents constant repetition.

---

# Action Execution

Conditions produce actions.

Examples:

```text
SetFlag()
```

```text
UnlockDialogue()
```

```text
StartQuest()
```

```text
AdvanceQuest()
```

```text
SpawnEvent()
```

```text
ModifyRelationship()
```

```text
RevealInformation()
```

```text
UnlockScene()
```

---

# Action Chains

One rule may trigger multiple actions.

Example:

```text
IF

PlayerConfesses

THEN

ModifyTrust(-2)
RevealInformation()
AdvanceQuest()
UnlockDialogue()
```

---

# Rule Dependencies

Rules can depend on other rules.

Example:

```text
Rule A
```

Unlocks:

```text
Rule B
```

Which unlocks:

```text
Rule C
```

Creates narrative progression.

---

# Event-Driven Evaluation

Avoid checking every rule every frame.

Instead:

```text
OnQuestAdvance
OnDialogueEnd
OnRelationshipChanged
OnDayChanged
OnLocationEntered
```

Evaluate relevant rules only.

This is dramatically faster.

---

# Debugging Support

Every rule evaluation should be inspectable.

Example:

```text
Rule:
unlock_kim_scene

FAILED

Reason:
BodyExamined = false
```

Narrative debugging becomes possible.

Without this, content bugs become nightmares.

---

# Rule Trace

Example:

```text
KimConfessionScene

Blocked By:

metric_trust_companion > 4
Current Value: 2
```

Designers immediately know why content is unavailable.

---

# Save Data

Store:

```text
RuleState
{
    rule_id
    consumed
    last_trigger_time
}
```

Do not save evaluation results.

Recalculate them.

---

# Performance Model

For large RPGs:

```text
Thousands of rules
```

Use:

```text
Event-driven evaluation
Indexed queries
Rule categories
Cached state lookups
```

Never brute-force every rule every frame.

---

# Minimum Viable System

For An NSF-Elysium-inspired RPG:

Implement:

```text
Condition Trees
Boolean Logic
State Queries
Comparison Operators
Rule Priorities
One-Time Rules
Repeatable Rules
Action Execution
Event-Driven Evaluation
Rule Debugging
```

This becomes the central brain of the narrative architecture. Dialogue, quests, companion interactions, information propagation, world events, discoveries, secrets, and endings all become data-driven rules. Every major narrative outcome is ultimately the result of the Rule Engine evaluating world state and deciding what becomes possible next.
