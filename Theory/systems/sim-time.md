# 11. Sim: Time — NSF Specification

> **Framework:** In-game clock, time advancement, time-gated events, Actor availability, day progression.
> **Content pack:** Schedule definitions, time-trigger content, day/night presentation.
> Terminology: [Glossary](../terminology-glossary.md)

The time system is not just a clock.

In a NSF-style game, time is a **narrative resource**.

It controls:

* when people are available
* when conversations advance
* when Internal beliefs mature
* when locations change
* when the story phase shifts
* when the player is forced to commit

A traditional RPG clock says:

```text id="x3m1ab"
How long until night?
```

A NSF-style clock says:

```text id="p9q7ls"
What has changed in the world while I was thinking, talking, or failing?
```

---

# Core Principle

Time should be a **simulation + narrative driver**, not just a visual UI element.

```text id="q1v8zn"
Action →
Time Passes →
World Updates →
New Availability / New Events / New Consequences
```

That loop is fundamental.

---

# Core Architecture

```text id="t4n6sp"
TimeService
 ├─ CurrentDay
 ├─ CurrentTime
 ├─ TimeScale
 ├─ TimeEvents
 ├─ TimeGatedTriggers
 ├─ Schedules
 ├─ Durations
 └─ WorldPhaseClock
```

---

# Time Model

The engine should support at least:

```text id="a8m2kc"
Day
Hour
Minute
```

You can simplify internally, but the system must present time clearly to the player.

Recommended structure:

```csharp id="d7q3mx"
class GameTime
{
    int Day;
    int Hour;
    int Minute;
}
```

---

# Time Is Discrete

NSF time is usually better modeled as **discrete jumps**, not continuous simulation.

Examples:

* talking advances time
* reading advances time
* traveling advances time
* waiting advances time

That makes pacing controllable.

---

# Time Advancement Sources

Time should advance from:

```text id="v2s8hn"
Dialogue
Movement
Inspection
Belief internalization
Reading
Waiting
Using services
Sleeping
Cutscenes
Story beat transitions
```

Not every interaction needs to consume time, but many should.

---

# Action Cost System

Every player action may have a time cost.

```csharp id="m8p5du"
class ActionCost
{
    int Minutes;
    bool AdvancesWorldState;
}
```

Examples:

* inspect object = 1 minute
* talk topic = 2–10 minutes
* reading a book = 30+ minutes
* sleeping = several hours

The exact values are design-tunable.

---

# Time as Gating Mechanism

Time should gate content.

Examples:

* Actor only appears in the morning
* store closes at night
* specific story beat only progresses after noon
* a body can be examined only before sunrise
* certain beliefs or events trigger after midnight

This makes the world feel scheduled and alive.

---

# Actor Schedule Integration

Time and Actors are tightly linked.

```text id="z5h1vc"
Time →
Actor Schedule →
Actor Location / State →
Dialogue Availability
```

This system should be deterministic and data-driven.

---

# Location Schedule Integration

Different locations may change behavior across the day.

Examples:

* streets are busier in the day
* bars are active at night
* offices are empty after work
* certain doors unlock only during business hours

The world should feel like it has a rhythm.

---

# Time Slice System

For implementation, useful time slices are:

```text id="l2k9pm"
Morning
Afternoon
Evening
Night
```

These can be mapped onto hour ranges internally.

This is often easier than minute-perfect simulation.

---

# Time Events

The time system should emit events when thresholds are crossed.

Examples:

```text id="c6r1qs"
MiddayReached
EveningStarted
MidnightReached
NewDayStarted
```

These events can trigger:

* Actor movement
* chronicle updates
* story beat transitions
* ambient dialogue
* belief changes

---

# Time-Triggered Narrative Events

Some story events should happen at exact times.

Examples:

* a meeting begins at 22:00
* a phone call arrives at 14:00
* a character leaves at midnight
* a clue becomes available after dawn

These should be defined as scheduled narrative events.

---

# World Phase Clock

Separate from the daily clock.

This controls large story progression.

Examples:

```text id="u1p7kc"
Arrival
Investigation
Escalation
Revelation
Aftermath
```

The same day/time can feel different depending on world phase.

---

# Time and Internalization

Belief internalization should consume time.

This is crucial.

```text id="k8n4rq"
Assimilating belief →
Time passes →
Character changes
```

This turns psychological progression into a real tradeoff.

---

# Time and Reading

Reading should meaningfully advance time.

That means books are not just lore items.

They are time investments.

This creates the NSF-style pressure of choosing between:

* reading
* investigating
* talking
* sleeping
* waiting

---

# Time and Recovery

Health and morale recovery may depend on time.

Examples:

* sleep restores health/morale
* certain consumables provide immediate relief
* resting at specific places may cost time

The game should make time feel valuable.

---

# Time and Access Windows

Some interactions should only exist in certain time windows.

Examples:

* witness available only before shift change
* worker gone after work
* guard asleep at night
* vendor unavailable on holidays or after hours

This creates natural planning.

---

# Time and Consequence Decay

Some things can change or disappear with time.

Examples:

* a witness leaves
* evidence is moved
* weather changes
* an opportunity is missed
* a conversation topic expires

The system should support both permanent and temporary opportunities.

---

# Time and Failure Pressure

Time should make failure meaningful.

Examples:

* if you wait too long, the lead cools
* if you spend all day talking, you miss an event
* if you sleep too early, you waste a window

This creates tension without combat timers.

---

# Time Skip Operations

The engine should support explicit time jumps.

Examples:

* wait until morning
* sleep until evening
* fast-forward during a scene transition

These operations need clear side effects.

---

# Wait System

Waiting should be a first-class action.

```csharp id="w7m3hf"
class WaitAction
{
    int MinutesToAdvance;
    string Reason;
}
```

Waiting is often necessary for scheduling and internalization, but it should not be free of narrative consequence.

---

# Day Progression Rules

The day should advance according to:

* accumulated minutes
* thresholds reached
* sleep events
* scripted transitions

The engine should never silently skip time without telling the player.

---

# UI Requirements

The player should always know:

* current day
* current time
* approximate time cost of actions
* whether an action advances time
* whether time-sensitive events are pending

Time UI must be clear, because many decisions depend on it.

---

# Time and Chronicle Integration

Chronicle entries often need time context.

Examples:

* “Tonight.”
* “Before noon.”
* “After the meeting.”
* “By morning.”

The chronicle should be able to store time-related constraints and reminders.

---

# Time and Dialogue Integration

Some dialogue choices should advance time; some should not.

Examples:

* quick remark = no meaningful time cost
* long interrogation = significant cost
* lengthy monologue = may advance time

This helps dialogue feel like part of the simulation.

---

# Time and Save Data

Serialize:

```text id="n2v5pa"
Current day/time
Pending time events
Actor schedules
World phase
Timed story beat states
Timed dialogue flags
Timed cooldowns
```

Everything time-dependent must survive reload correctly.

---

# Minimal Engine Interfaces

> Service names from [Glossary](../terminology-glossary.md). Domain types are internal.

## Service contract

```csharp
interface ITimeService
{
    GameTime Now { get; }
    void Advance(GameTimeDelta delta);
    void Schedule(string eventId, GameTime at);
    bool IsScheduledDue(string eventId);
}
```

## Domain model

```csharp
struct GameTime { int Day; int Hour; int Minute; }
struct GameTimeDelta { int Days; int Hours; int Minutes; }
interface ITimeEvent { string Id; GameTime TriggerAt; }
```


# Most Important Insight

The time system is not a clock.

It is a **pressure mechanism** for narrative choice.

It makes the player decide:

```text id="b9r4tz"
What do I do now?
What do I miss if I do this?
What changes while I am elsewhere?
```

That is what makes time in a NSF-style game feel alive rather than decorative.
