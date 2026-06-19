# Sim Time Service — Implementation Architecture

- Spec: [systems/sim-time.md](../systems/sim-time.md)
- Roadmap: [Phase 6](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why time is a narrative resource, not a UI clock

NSF pacing depends on **discrete time jumps** after meaningful actions (dialogue, travel, reading, waiting). Continuous real-time simulation fights authorial control and makes investigation windows unpredictable. `ITimeService` owns the authoritative in-game clock; Presentation displays `Now` but never advances it.

### Why discrete advancement

Each `Advance(GameTimeDelta)` represents an intentional story beat cost. Dialogue nodes, travel edges, and belief internalization declare minute/hour costs in content. The engine applies them atomically so gates and schedules see a consistent post-action time.

### Why scheduled events are first-class

Story beats ("meeting at 22:00", "rent due at day end") need durable timers independent of frame rate. `Schedule` + `IsScheduledDue` integrate with kernel **Events** phase so due callbacks fire once per tick in deterministic order.

### Why time slices map to hours

Authors think in Morning/Afternoon/Evening/Night; implementation maps hour ranges from content. MVP exposes `GameTime` directly; helper `TimeSliceResolver` derives slice IDs for schedule and gate rules without minute-perfect simulation.

---

## Implementation summary

| MVP (Phase 6) | Full |
|---|---|
| `Now`, `Advance`, overflow normalization | World phase clock separate from daily clock `[FULL]` |
| `Schedule` / `IsScheduledDue` | Recurring schedules (rent, interest) `[FULL]` |
| `TimeAdvanced` event | Time-slice boundary events `[FULL]` |
| Day/hour/minute carry on advance | Consequence decay windows `[FULL]` |
| Persistence of clock + pending schedules | Action cost registry from content `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Simulation`
- **Namespace:** `NarrativeFramework.Simulation.Time`
- **References:** Runtime, Content (schedule definitions)
- **Tick phase:** **Events** (due schedule evaluation) · **Facts** (read-only `Now` for timestamps)

Registered as `ITimeService` in registry. All services read `Now` for event timestamps; only designated callers invoke `Advance`.

---

## Public API

See [contracts.md](contracts.md) — `ITimeService`.

| Member | Contract |
|---|---|
| `GameTime Now { get; }` | Current simulation clock |
| `void Advance(GameTimeDelta delta)` | Additive jump with normalization |
| `void Schedule(string eventId, GameTime at)` | One-shot future trigger |
| `bool IsScheduledDue(string eventId)` | True once when `Now >= scheduledAt`; clears on read |

Internal helpers (not in contracts): `GetTimeSlice()`, `MinutesUntil(GameTime target)`.

---

## Internal implementation

### TimeService

```csharp
sealed class TimeService : ITimeService, IStatefulService<TimeServiceState>
{
    GameTime _now = new() { Day = 1, Hour = 8, Minute = 0 };
    readonly Dictionary<string, ScheduledEvent> _scheduled = new();
    readonly IEventBus _events;

    public GameTime Now => _now;

    public void Advance(GameTimeDelta delta)
    {
        var prior = _now;
        _now = GameTimeMath.Add(_now, delta);
        _events.Publish(new SimEvent {
            Type = EventTypes.TimeAdvanced,
            Timestamp = _now,
            Payload = {
                ["priorDay"] = prior.Day.ToString(),
                ["priorHour"] = prior.Hour.ToString(),
                ["deltaMinutes"] = GameTimeMath.ToTotalMinutes(delta).ToString()
            }
        });
        EmitBoundaryEventsIfCrossed(prior, _now);
    }

    public void Schedule(string eventId, GameTime at)
    {
        ContentIdValidator.IsValid(eventId, "time_evt_");
        _scheduled[eventId] = new ScheduledEvent { Id = eventId, TriggerAt = at };
    }

    public bool IsScheduledDue(string eventId)
    {
        if (!_scheduled.TryGetValue(eventId, out var evt)) return false;
        if (GameTimeMath.Compare(_now, evt.TriggerAt) < 0) return false;
        _scheduled.Remove(eventId);
        return true;
    }
}
```

### GameTimeMath (internal static)

```csharp
static class GameTimeMath
{
    public static GameTime Add(GameTime t, GameTimeDelta d)
    {
        var total = ToTotalMinutes(t) + ToTotalMinutes(d);
        return FromTotalMinutes(total);
    }

    public static int Compare(GameTime a, GameTime b)
        => ToTotalMinutes(a).CompareTo(ToTotalMinutes(b));

    static int ToTotalMinutes(GameTime t) => (t.Day * 24 + t.Hour) * 60 + t.Minute;
    static GameTime FromTotalMinutes(int m) { /* carry day/hour/minute */ }
}
```

### ScheduleRegistry (internal) `[FULL]`

Holds actor schedule definitions keyed by `schedule_*`. MVP: content JSON loaded into registry; tick hook invoked from kernel Social phase when actor schedules land in Phase 6+.

### TimeEventHandler

Subscribes to `TimeAdvanced`. Evaluates actor schedule definitions, calls `ILocationService.Travel` and `IActorService.SetState` for due entries. MVP: stub with manual test schedules.

---

## Definition assets

```csharp
class ScheduleDefinition : ContentDefinition
{
    public string ActorId;
    public List<ScheduleEntry> Entries;
}

class ScheduleEntry
{
    public int StartHour;
    public int EndHour;
    public string LocationId;
    public string AvailabilityOverride;   // optional ActorAvailability name
}

class ActionCostDefinition : ContentDefinition   // [FULL]
{
    public string ActionKey;
    public int Minutes;
    public bool AdvancesWorldState;
}
```

Time slice ranges authored in content pack config:

```text
Morning   → hour 6–11
Afternoon → hour 12–17
Evening   → hour 18–21
Night     → hour 22–5
```

---

## Runtime state

```csharp
class TimeServiceState
{
    public GameTime Now;
    public List<ScheduledEvent> PendingSchedules;
}

class ScheduledEvent
{
    public string Id;
    public GameTime TriggerAt;
}
```

All scheduled one-shots persist across save/load. `[FULL]` recurring obligations (rent) stored as separate ledger in economy service with references to `time_evt_*`.

---

## Core algorithms

### Advance with normalization

1. Convert `Now` + delta to total minutes
2. Carry into day/hour/minute (24h days, no leap semantics)
3. Publish `TimeAdvanced` with prior and delta payload
4. Check crossed boundaries (midnight → `NewDayStarted`, slice changes → `TimeSliceChanged` `[FULL]`)

### Schedule due evaluation (kernel Events phase)

```text
foreach pending schedule where Now >= TriggerAt:
    mark due (IsScheduledDue or batch processor)
    publish TimeEventFired { eventId }
```

Kernel calls `ITimeService` schedule checks after `IEventBus.Flush` so handlers see consistent time.

### Who may Advance

| Caller | Typical delta |
|---|---|
| IDialogueService | per-node `ActionCost` |
| ILocationService | travel edge cost |
| IBeliefService | internalization minutes `[Phase 8]` |
| Script actions | explicit wait/sleep |
| IPacingService | never — reads only |

Unauthorized Advance in tests throws via debug guard `[FULL]`.

### Integration with gates

`IGateService` rules reference `GameTime` windows and `time_slice_*` IDs. Evaluation during **Gating** phase reads `Now` without mutating.

---

## Event contracts + tick phase

| Event | When | Typical subscribers |
|---|---|---|
| `TimeAdvanced` | After every `Advance` | Schedules, economy rent tick, pacing |
| `NewDayStarted` | Day increment | Economy obligations, chronicle |
| `TimeSliceChanged` | Hour range boundary `[FULL]` | Ambient, actor availability |
| `TimeEventFired` | Scheduled ID due | Story scripts, fail-forward |

**Kernel phase:** Events — process due schedules after event flush. **Read-only:** Facts, Gating (use `Now` snapshot at tick start).

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IEventBus | Time notifications |
| ContentIdValidator | Scheduled event IDs |

| Used by | Reason |
|---|---|
| IFactService | `RegisteredAt` timestamps |
| IActorService | Schedule-driven availability |
| ILocationService | Travel time cost |
| IDialogueService | Conversation time cost |
| IEconomyService | Rent/debt due times `[Phase 10]` |
| IInfoFlowService | Propagation timestamps |
| IPersistenceService | Clock + schedules |
| SimEvent | Default timestamp source |

**Forbidden:** TimeService → Presentation (HUD reads via presenter/view model in Phase 13+).

---

## MVP scope (Phase 6)

- [ ] `ITimeService` + `TimeService` + `GameTimeMath`
- [ ] `Advance` with day rollover
- [ ] `Schedule` / `IsScheduledDue` one-shot semantics
- [ ] `TimeAdvanced` event on every advance
- [ ] `TimeServiceState` capture/restore
- [ ] Tests: add delta, rollover, schedule due once, event payload

---

## Full scope

- `[FULL]` `TimeSliceResolver` + boundary events
- `[FULL]` World phase clock (`world_phase_*`) orthogonal to daily time
- `[FULL]` Action cost registry from content
- `[FULL]` Wait/sleep script ops with side-effect hooks
- `[FULL]` Opportunity decay tied to time windows

---

## File tree

```text
Packages/NarrativeFramework/Simulation/Time/ITimeService.cs
Packages/NarrativeFramework/Simulation/Time/TimeService.cs
Packages/NarrativeFramework/Simulation/Time/GameTime.cs
Packages/NarrativeFramework/Simulation/Time/GameTimeDelta.cs
Packages/NarrativeFramework/Simulation/Time/GameTimeMath.cs
Packages/NarrativeFramework/Simulation/Time/ScheduledEvent.cs
Packages/NarrativeFramework/Simulation/Time/ScheduleDefinition.cs
Packages/NarrativeFramework/Simulation/Time/TimeServiceState.cs
Packages/NarrativeFramework/Simulation/Time/TimeEventHandler.cs
Packages/NarrativeFramework/Tests/EditMode/Time/TimeAdvanceTests.cs
Packages/NarrativeFramework/Tests/EditMode/Time/TimeScheduleTests.cs
Packages/NarrativeFramework/Tests/EditMode/Time/TimePersistenceTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Advance_AddsMinutes` | Hour/minute arithmetic |
| `Advance_CarriesDay` | 24h rollover |
| `Schedule_DueOnce` | Second `IsScheduledDue` false |
| `Schedule_NotDueBeforeTime` | Compare boundary |
| `Advance_PublishesTimeAdvanced` | Event bus delivery |
| `CaptureRestore_PreservesNowAndSchedules` | Stateful roundtrip |
| `Kernel_EventsPhase_ProcessesDueSchedules` | Integration with kernel |

Contributes to Phase 6 ≥18 tests combined with actor/location.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-08 (no real-time clock; pause via story/pacing).

---

## Bootstrap and registry wiring

Phase 6 composition registers `TimeService` before `FactService` and `LocationService` so constructors can capture `ITimeService` for timestamps and travel costs. Kernel **Events** phase invokes internal `TimeService.ProcessDueSchedules()` after `IEventBus.Flush`:

```csharp
// SimulationKernel.ExecutePhase — Events branch excerpt
case SimulationTickPhase.Events:
    _registry.GetRequired<IEventBus>().Flush(snapshot);
    _registry.GetRequired<ITimeService>().ProcessDueSchedules();
    break;
```

`ProcessDueSchedules` is internal — not on `ITimeService` contract. Edit Mode tests call it via `InternalsVisibleTo` or kernel full tick.

New game bootstrap sets `Now` from content pack `GameStartTime` definition (default Day 1, 08:00). Restore from save replaces `_now` entirely — no merge with session clock.

---

## Related documents

- [sim-actor.md](sim-actor.md), [sim-location.md](sim-location.md)
- [sim-economy.md](sim-economy.md), [runtime-kernel.md](runtime-kernel.md)
- [story-dialogue.md](../systems/story-dialogue.md) (action costs)
