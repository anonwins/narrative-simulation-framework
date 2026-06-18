# 27. Story: Fail Forward — NSF Specification

> **Framework:** Failure-forward design, failure branches, narrative recovery, alt solutions.
> **Content pack:** Failure dialogue, comedy, recovery content per check/interaction.
> Terminology: [Glossary](../terminology-glossary.md)

This is arguably the most important system in the entire game.

Most RPGs treat failure as:

```text id="fc1"
Failure
→ Progress blocked
```

NSF treats failure as:

```text id="fc2"
Failure
→ New content
```

The result is that players stop fearing failure and start becoming curious about it.

This system is one of the primary reasons NSF feels fundamentally different from traditional RPGs.

---

# Core Principle

Failure must be content.

Never:

```text id="fc3"
Failure = nothing happens
```

Never:

```text id="fc4"
Failure = try again until success
```

Instead:

```text id="fc5"
Failure
↓
Story
↓
Reaction
↓
Consequence
↓
New Possibilities
```

The player should often think:

```text id="fc6"
I want to see what happens if I fail.
```

---

# Core Architecture

```text id="fc7"
FailForward
 ├─ FailureBranches
 ├─ FailureEvents
 ├─ RecoveryPaths
 ├─ ComedyGenerator
 ├─ ConsequenceRules
 ├─ AlternativeSolutions
 ├─ FailureMemory
 └─ FailureRewards
```

---

# Core Model

Every meaningful check should support failure content.

```csharp id="fc8"
class FailureOutcome
{
    string Id;

    List<Consequence> Consequences;

    List<ContentNode> FollowUpContent;

    bool ProgressesStory;
}
```

Failure is not merely a boolean.

It is a narrative package.

---

# Failure Categories

Not all failures are equal.

Recommended categories:

```text id="fc9"
Embarrassment
Misunderstanding
Social Damage
Physical Disaster
Lost Opportunity
Wrong Conclusion
Moral Failure
Comedic Collapse
```

Different categories create different emotional tones.

---

# Failure-Forward Design

The most important design rule.

Bad:

```text id="fc10"
Fail check
↓
Nothing
```

Good:

```text id="fc11"
Fail check
↓
Different route appears
```

The investigation should continue.

Only the path changes.

---

# Branch-Based Failure

Every major failure should branch.

Example:

```text id="fc12"
Success
→ Gain trust

Failure
→ Gain suspicion
→ New conversation
→ New obstacle
```

Both outcomes create content.

---

# Narrative Recovery

Failure should rarely end progression.

Instead:

```text id="fc13"
Failure
↓
Complication
↓
Recovery Opportunity
```

The player is recovering from mistakes, not restarting.

---

# Alternative Solutions

Every important obstacle should ideally support:

```text id="fc14"
Primary Solution
Secondary Solution
Failure Solution
```

Example:

```text id="fc15"
Persuade witness

OR

Bribe witness

OR

Witness dislikes you and talks later
```

The story remains alive.

---

# Failure as Characterization

Failures should reveal personality.

Example:

```text id="fc16"
Authority failure
```

might expose:

* insecurity
* arrogance
* desperation

Failure becomes roleplaying.

---

# Failure as Narrative Generation

A failed check should often generate more narrative than a successful one.

Example:

```text id="fc17"
Success:
Door opens.
```

```text id="fc18"
Failure:
Door breaks.
Actor notices.
Argument begins.
New clue revealed.
```

Failure creates stories.

---

# Comedy Generation

One of NSF's defining systems.

Many failures should be funny.

Examples:

```text id="fc19"
Overconfidence
Awkwardness
Self-destruction
Social collapse
Absurd misunderstanding
```

Comedy reduces frustration.

---

# Comedy Model

Failure comedy often follows:

```text id="fc20"
Attempt
↓
Confidence
↓
Unexpected Disaster
↓
Reaction
```

The larger the confidence gap, the funnier the outcome.

---

# Catastrophic Success vs Catastrophic Failure

Support both.

Examples:

```text id="fc21"
Failure:
Fall off chair.
```

```text id="fc22"
Success:
Get information.
```

Sometimes:

```text id="fc23"
Success:
Get information.

But now everyone hates you.
```

Interesting outcomes matter more than pass/fail.

---

# Social Failure

One of the richest failure categories.

Examples:

```text id="fc24"
Offended Actor
Lost credibility
Embarrassed self
Created suspicion
Damaged relationship
```

Social failure should echo later.

---

# Physical Failure

Examples:

```text id="fc25"
Fall
Injury
Broken object
Lost item
Public humiliation
```

These create memorable moments.

---

# Investigative Failure

Critical for inquiry-based games.

Examples:

```text id="fc26"
Wrong ThreadSubject
Missed clue
Incorrect theory
False accusation
```

The investigation should continue despite mistakes.

---

# Delayed Failure Consequences

Failure does not always trigger immediately.

Example:

```text id="fc27"
Lie successfully told.
```

Later:

```text id="fc28"
Lie exposed.
```

The failure arrives hours later.

---

# Failure Memory

Failures should be remembered.

```csharp id="fc29"
class FailureMemory
{
    string FailureId;

    List<string> Witnesses;

    bool Permanent;
}
```

Actors can reference old failures.

---

# Reputation Impact

Failure can affect:

```text id="fc30"
Trust
Respect
Fear
Suspicion
```

This creates meaningful social consequences.

---

# Companion Reactions

Companions should react strongly to failures.

Examples:

```text id="fc31"
Approval
Disapproval
Concern
Amusement
Embarrassment
```

These reactions often become memorable content.

---

# Failure Unlocks

Failures may unlock unique content.

Example:

```text id="fc32"
Fail authority check
↓
Self-doubt dialogue unlocked
```

Players should occasionally gain access to things only failure reveals.

---

# Exclusive Fail Forward

A powerful replayability tool.

Example:

```text id="fc33"
Success route
```

and

```text id="fc34"
Failure route
```

contain entirely different scenes.

---

# Failure Rewards

Failures can provide value.

Examples:

```text id="fc35"
New information
New relationship state
New dialogue
new belief
New theory
```

The player should not feel completely punished.

---

# Faculty-Specific Failure

Different faculties should fail differently.

Example:

```text id="fc36"
Logic failure
→ Wrong conclusion

Drama failure
→ Misread deception

Empathy failure
→ Misread emotion

Authority failure
→ Social humiliation
```

Failure should reflect the faculty involved.

---

# Cascading Failure

One failure may create more opportunities.

```text id="fc37"
Failure
↓
New problem
↓
New check
↓
New scene
```

This creates emergent storytelling.

---

# Failure Severity

Suggested levels:

```text id="fc38"
Minor
Moderate
Major
Catastrophic
```

Not every failure should be devastating.

---

# Failure Timing

Failures can occur:

```text id="fc39"
Immediately
Soon
Delayed
Endgame
```

Different timing creates different narrative effects.

---

# Failure Visibility

Sometimes the player should know they failed.

Example:

```text id="fc40"
Check Failed
```

Sometimes they should not.

Example:

```text id="fc41"
Player believes wrong theory.
```

Only later is the mistake revealed.

---

# Failure and World Mutation

Failures should change the world.

Examples:

```text id="fc42"
Door destroyed
Relationship damaged
Faction angered
Opportunity lost
```

The world remembers.

---

# Failure and Beliefs\n
Failures can unlock beliefs.

Examples:

```text id="fc43"
Humiliation
↓
New Belief Available
```

This converts mistakes into character growth.

---

# Failure and Conduct

Repeated failures can influence identity.

Examples:

```text id="fc44"
conduct_humble
conduct_negligent
conduct_reckless
```

Behavior emerges from mistakes.

---

# Failure and Ideologys

Ideological failures should exist.

Examples:

```text id="fc45"
Public ideological collapse
Failed argument
Faction backlash
```

Politics should carry risk.

---

# Event Integration

Emit:

```text id="fc46"
CheckFailed
FailureBranchEntered
RecoveryStarted
FailureRemembered
AlternativeRouteUnlocked
```

Other systems subscribe to these events.

---

# Save Data Requirements

Serialize:

```text id="fc47"
Failure History
Failure Memories
Failure Consequences
Unlocked Fail Forward
Recovery State
```

Failure state must persist.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
// Fail-forward is a design pattern — no I*Service.
// Integration: IRollService emits failure branches; IDialogueService loads alternate nodes;
// IStoryStateService advances flags on failure paths; IPacingService never hard-stops progression.
```

## Domain model

```csharp
class FailForwardBranch { string RollId; string FailureNodeId; string RecoveryNodeId; }
class FailForwardOutcome { bool HardStop; string AlternateContentId; }
```


# Relationship to Other Systems

Failure sits at the center of NSF's narrative loop.

```text id="fc49"
Roll
        ↓
Fail Forward
        ↓
Dialogue
Relationships
Beliefs\nQuests
World State
```

Many of the game's best moments originate from failure outcomes.

---

# Recommended Internal Flow

```text id="fc50"
Attempt
↓
Failure
↓
Consequence
↓
Reaction
↓
Recovery
↓
New Opportunity
```

This transforms mistakes into stories.

---

# Most Important Insight

Most RPGs use failure to stop the player.

NSF uses failure to entertain the player.

The engine should continuously answer:

```text id="fc51"
What is interesting about failing here?
What story only exists because failure happened?
How does failure reveal character?
How does failure create a better scene than success?
```

If implemented correctly, players stop save-scumming and start embracing mistakes.

That is one of the defining characteristics of NSF's design.
