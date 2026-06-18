# 13. Sim: Event — NSF Specification

> **Framework:** Trigger events, world events, chronicle/dialogue/story-beat update dispatch.
> **Content pack:** Event definitions, trigger content, narrative payloads.
> Terminology: [Glossary](../terminology-glossary.md)

The Event bus is the core engine infrastructure that makes every other system communicate.

In a NSF-style game, almost nothing should update itself directly.

Instead:

```text id="c9f2le"
Something happens →
An event is emitted →
Systems subscribe →
State changes propagate →
New content appears
```

That architecture is what keeps faculties, dialogue, story beats, chronicle updates, beliefs, locations, Actors, and time synchronized.

---

# Core Principle

Do not let systems reach into each other arbitrarily.

Bad:

```text id="n3v8sa"
Dialogue directly edits chronicle
Story state directly edits Actor memory
Inventory directly edits world scene
```

Good:

```text id="u4b9rd"
Action →
Event →
Subscribers react
```

This prevents brittle code and makes the narrative simulation easier to extend.

---

# Core Architecture

```text id="p7x1mv"
EventBus
 ├─ EventBus
 ├─ EventQueue
 ├─ EventTypes
 ├─ Subscribers
 ├─ Filters
 ├─ Priorities
 ├─ Conditions
 ├─ Dispatchers
 └─ History
```

---

# Event Object Model

Every event should be a structured object.

```csharp id="j8q4tr"
class SimEvent
{
    string Id;
    string Type;
    string SourceId;
    string TargetId;
    GameTime Timestamp;
    Dictionary<string, object> Payload;
}
```

The event is not just a string label.

It must carry context.

---

# Event Types

The engine should support at least these categories:

```text id="v2d8ka"
TriggerEvents
WorldEvents
DialogueEvents
QuestEvents
ChronicleEvents
InventoryEvents
BeliefEvents
FacultyEvents
ActorEvents
TimeEvents
UIEvents
```

These are categories, not necessarily separate buses.

---

# Trigger Events

Trigger events are the smallest meaningful gameplay signals.

Examples:

* enter area
* inspect object
* fail check
* acquire item
* learn fact
* choose dialogue option
* equip clothing
* internalize belief

They are the raw material of all narrative logic.

---

# World Events

World events change the simulation.

Examples:

* door opened
* scene altered
* Actor moved
* corpse removed
* light switched on
* area unlocked
* weather changed

These events often affect multiple systems at once.

---

# Dialogue Events

Dialogue events should cover:

* line spoken
* topic unlocked
* choice selected
* faculty interruption fired
* check passed
* check failed
* conversation ended

These are extremely important because dialogue is core gameplay.

---

# Story events

Story events drive narrative state changes.

Examples:

* story beat started
* clue added
* lead resolved
* Inquiry subject cleared
* objective failed
* thread phase advanced
* story beat completed

These should feed the narrative state machine.

---

# Chronicle Events

The chronicle should not be edited manually.

Instead it should subscribe to events like:

* clue discovered
* task updated
* important fact learned
* Belief internalized
* story beat phase changed

Then the chronicle transforms those events into readable entries.

---

# Inventory Events

Inventory should emit events such as:

* item acquired
* item equipped
* item unequipped
* item consumed
* item used on object
* item sold
* item removed

Other systems can subscribe to those events.

---

# Belief Events

Belief behavior should be event-driven.

Examples:

* belief unlocked
* belief started internalizing
* belief completed
* belief forgotten
* belief bonus activated
* belief bonus removed

This allows Beliefs to influence the world continuously.

---

# Faculty Events

Faculties should listen and react to events.

Examples:

* passive check opportunity
* internal voice trigger
* faculty gain
* faculty cap changed
* faculty boosted by clothing
* faculty interrupted dialogue

The faculty system becomes reactive rather than static.

---

# Actor Events

Actors should emit and receive events.

Examples:

* met Actor
* Actor remembered fact
* Actor heard insult
* Actor changed mood
* Actor moved location
* Actor became available
* Actor relationship changed

This makes Actors feel alive and persistent.

---

# Time Events

Time should generate events when thresholds are crossed.

Examples:

* minute passed
* hour changed
* morning started
* night started
* day ended
* scheduled event triggered

This lets schedules and timed narrative content work cleanly.

---

# Event Bus

The Event Bus is the central router.

```csharp id="w1r9pd"
interface IEventBus
{
    void Publish(SimEvent e);
    void Subscribe(string eventType, IEventHandler handler);
}
```

The bus should support:

* multiple listeners
* filtered subscriptions
* priority ordering
* queued dispatch
* synchronous or deferred handling

---

# Synchronous Versus Deferred Events

Some events should happen immediately.

Examples:

* updating a faculty bonus
* opening a new dialogue option
* changing a scene object state

Some should be deferred.

Examples:

* chronicle updates
* long narrative chains
* heavy UI refreshes

The engine should support both.

---

# Event Queue

A queue is useful when multiple events happen in a chain.

Example:

```text id="l6b7qn"
Inspect corpse
→ faculty event
→ clue event
→ chronicle event
→ story beat event
→ dialogue unlock event
```

A queue prevents re-entrant chaos.

---

# Event Ordering

Events need deterministic order.

Example order:

```text id="f8m2cv"
1. Source action event
2. Simulation update
3. Narrative updates
4. Chronicle updates
5. UI refresh
6. Save-state flush
```

This makes debugging and replay much easier.

---

# Event Priority

Some handlers should run before others.

Example:

* story beat state updates before chronicle wording
* world state updates before dialogue availability
* faculty recalculation before passive check refresh

Priority prevents stale reads.

---

# Event Filters

Handlers should subscribe only to relevant events.

Example:

```json id="r3n5fa"
{
  "eventType": "ITEM_ACQUIRED",
  "itemTag": "clothing"
}
```

This avoids giant switch statements and unnecessary work.

---

# Event Payloads

Payloads should carry structured data.

Example:

```json id="x7k2md"
{
  "itemId": "horrific_necktie",
  "slot": "neck",
  "source": "hotel_room"
}
```

Do not rely only on event names.

The payload is where the meaning lives.

---

# Idempotency

Events should be safe to process once.

Important for save/load and replay.

Example:

```text id="q1c8rx"
Chronicle update should not duplicate if event is replayed
```

Use event IDs and processed-event tracking.

---

# Event History

The system should keep a history of important events.

```text id="m9v4zt"
EventHistory
 ├─ event_id
 ├─ type
 ├─ timestamp
 ├─ source
 ├─ payload_summary
 └─ processed_by
```

This is useful for debugging, analytics, and narrative recall.

---

# Event Replay

A strong engine can replay events from save state.

That allows:

* debugging
* restoring derived state
* reconstructing narrative conditions
* re-synchronizing systems after load

The event log should be treated as an important simulation artifact.

---

# Event Sourcing Versus Derived State

Recommended pattern:

* keep current state for fast access
* keep event history for reconstruction and debugging

Do not make every query scan the entire event log during normal gameplay.

---

# Conditional Event Generation

Some events should only be emitted if conditions are met.

Example:

```text id="d4y8kp"
If perception threshold passed:
    emit hidden_clue_found
```

This is common for passive checks and faculty commentary.

---

# Event Chaining

One event can produce another.

Example:

```text id="z8v1ac"
Story beat advanced
→ chronicle updated
→ dialogue unlocked
→ Actor reaction changed
```

Chaining should be intentional and controlled.

---

# Preventing Event Loops

The engine must avoid infinite feedback loops.

Example:

```text id="t7n3mh"
Chronicle update reflects story beat update
Story beat update emits chronicle update
```

Use guards such as:

* event source tracking
* per-event processing flags
* cycle detection
* deferred batching

---

# Domain Boundaries

The event system should be the only thing most systems talk to.

Examples:

```text id="p2w6ea"
Dialogue → emits event
Story State → listens
Chronicle → listens
Actor memory → listens
UI → listens
```

This keeps modules separated.

---

# Narrative Events

A NSF-style game needs many narrative-specific event types.

Examples:

```text id="k4h9rp"
FactLearned
TheoryUpdated
SuspicionRaised
TrustChanged
IdeologyShifted
BeliefUnlocked
CheckFailed
CheckSucceeded
MemoryTriggered
```

These are the backbone of reactive storytelling.

---

# UI Event Layer

UI should not infer state by polling if possible.

Instead, use events like:

* inventory changed
* dialogue options refreshed
* chronicle entry changed
* faculty bonus recalculated
* check availability changed

This makes the interface responsive and simpler.

---

# Event Debugging Tools

This system needs strong debugging support.

Useful features:

* event log viewer
* event type filter
* source/target tracing
* payload inspection
* replay last N events
* subscriber inspection

Without this, complex narrative bugs become very hard to diagnose.

---

# Save Data Requirements

Serialize:

```text id="u8m1kw"
Pending Events
Event History
Processed Event IDs
Critical Event Flags
Queued Narrative Changes
```

This helps maintain consistency across reloads.

---

# Minimal Engine Interfaces

```csharp id="c7x2rd"
class SimEvent
{
    string Id;
    string Type;
    GameTime Timestamp;
}

interface IEventHandler
{
    void Handle(SimEvent e, SimulationSnapshot state);
}

interface IEventBus
{
    void Publish(SimEvent e);
    void Subscribe(string eventType, IEventHandler handler);
}
```

---

# Most Important Insight

The Event bus is not just backend plumbing.

It is the nervous system of the entire game.

It connects:

* dialogue
* quests
* chronicle
* faculties
* Beliefs\n* inventory
* Actors
* time
* locations

If this system is clean, the rest of the engine can stay modular and reactive.

If it is weak, everything turns into direct dependencies and special-case code.

In a NSF-style engine, events are how reality changes.
