# Cognition Roll — Implementation Architecture

- Spec: [systems/cognition-roll.md](../systems/cognition-roll.md)
- Glossary: [Roll modes](../terminology-glossary.md) · [Fail-forward](../terminology-glossary.md)
- Roadmap: [Phase 3](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)
- Related: [cognition-faculty.md](cognition-faculty.md) · [story-fail-forward.md](story-fail-forward.md)

---

## Design rationale

### Why rolls are world-mutators, not pass/fail gates

NSF checks answer: what does the attempt cost, what does success reveal, what does failure create? `RollResult` carries `OutcomeBranchId` for fail-forward routing — never a hard stop. Failed rolls emit alternate content via Story services, not empty responses.

### Why four RollModes

| Mode | Behavior |
|---|---|
| **Active** | Player-initiated; 2d6 + faculty + modifiers vs DC |
| **Passive** | Threshold only; no dice; triggers on context entry |
| **Repeatable** | Active with recovery via `RecoverRepeatable` |
| **Gated** | Locked until `CanAttempt` conditions satisfied |

Modes encode recovery and fail-forward semantics without separate services.

### Why deterministic modifier order

Spec resolution order: base faculty → caps → equipment → beliefs → consumables → world context → roll → threshold → outcome. Fixed order keeps Edit Mode tests reproducible and debug traces readable.

### Why CanAttempt is separate from Resolve

Gates, pacing, and repeatable lockout are checked before spending player attention or vitality. Dialogue UI calls `CanAttempt` before showing roll; `Resolve` assumes attempt is legal.

### Why IRollService in Cognition

Rolls interpret faculties — they belong with cognition, not Story. Story consumes `RollResult.OutcomeBranchId` to advance dialogue graphs.

---

## Implementation summary

| MVP (Phase 3) | Full |
|---|---|
| `Resolve(FacultyRoll)` Active + Passive | Critical success/failure bands |
| 2d6 + modified faculty vs DC | Partial success outcomes |
| `RollMode` all four enum values | Custom dice models `[FULL]` |
| `CanAttempt` / `RecoverRepeatable` | Pacing integration (Phase 11) |
| Outcome branch IDs | Vitality cost on attempt `[FULL]` |
| Tests match spec example tables | Location/relationship context mods |

---

## Assembly and namespace

| Item | Value |
|---|---|
| **Assembly** | `NarrativeFramework.Cognition` |
| **References** | Runtime, Content, Rules |
| **Namespace** | `NarrativeFramework.Cognition.Rolls` |
| **Tick phase** | **Interpretation** (passive) · **Content** (active player choice) |

---

## Public API

See [contracts.md](contracts.md) — `IRollService`.

**Module notes:**

- `Resolve(roll)` — returns `RollResult`; publishes `RollResolved` event.
- `CanAttempt(mode, rollId, context)` — checks gated locks, repeatable cooldown, gate rules.
- `RecoverRepeatable(rollId)` — clears lockout after fail-forward branch taken.

---

## Internal implementation

### RollService

```csharp
namespace NarrativeFramework.Cognition.Rolls
{
    sealed class RollService : IRollService, IStatefulService<RollServiceState>
    {
        readonly IFacultyService _faculty;
        readonly IInventoryService _inventory;  // Phase 12; stub empty list Phase 3
        readonly IEmotionService _emotion;      // Phase 8; stub 0 modifier Phase 3
        readonly IRandom _rng;
        readonly RollAttemptRegistry _attempts = new();

        public RollResult Resolve(FacultyRoll roll)
        {
            var modified = _faculty.GetModifiedValue(roll.FacultyId);
            var totalModifier = AggregateModifiers(roll);

            if (roll.Mode == RollMode.Passive)
                return ResolvePassive(roll, modified + totalModifier);

            var rolled = Roll2D6();
            var total = rolled + modified + totalModifier;
            var success = total >= roll.Difficulty;

            var result = new RollResult
            {
                Success = success,
                RolledValue = rolled,
                TargetValue = roll.Difficulty,
                OutcomeBranchId = success ? roll.SuccessBranchId : roll.FailureBranchId
            };

            if (roll.Mode == RollMode.Repeatable && !success)
                _attempts.Lock(roll.RollId);

            PublishResolved(roll, result);
            return result;
        }

        public bool CanAttempt(RollMode mode, string rollId, RollContext context)
        {
            if (mode == RollMode.Gated && !EvaluateGate(context))
                return false;
            if (mode == RollMode.Repeatable && _attempts.IsLocked(rollId))
                return false;
            return true;
        }

        public void RecoverRepeatable(string rollId) => _attempts.Unlock(rollId);
    }
}
```

### Modifier aggregation

```csharp
int AggregateModifiers(FacultyRoll roll)
{
    int sum = 0;
    foreach (var m in _inventory.GetEquippedModifiers())
        if (m.FacultyId == roll.FacultyId) sum += m.Delta;
    sum += (int)_emotion.GetRollModifier(roll.FacultyId);
    // belief, conduct, context — Phase 8+ hooks
    return sum;
}
```

Phase 3 tests: inventory/emotion stubbed to zero.

### Domain types

```csharp
class FacultyRoll
{
    public string RollId;
    public string FacultyId;
    public int Difficulty;
    public RollMode Mode;
    public string SuccessBranchId;
    public string FailureBranchId;
}

class RollResult
{
    public bool Success;
    public int RolledValue;
    public int TargetValue;
    public string OutcomeBranchId;
}

enum RollMode { Active, Passive, Repeatable, Gated }

class RollContext
{
    public string GateId;
    public string LocationId;
    public Dictionary<string, string> Tags;
}
```

### RollServiceState

```csharp
class RollServiceState
{
    public HashSet<string> LockedRepeatableRollIds;
}
```

---

## Definition assets

Roll definitions may live as:

- Embedded in dialogue graph nodes (Phase 4)
- `RollDefinition` content asset `[FULL]` with ID `roll_*`

MVP: `FacultyRoll` struct passed from Story/dialogue at invoke time; fields populated from content.

---

## Runtime state / persistence

Repeatable lockout set persists in `RollServiceState` (Phase 10 save).

Passive checks are stateless per invocation.

---

## Core algorithms

### Active resolution

1. Read `GetModifiedValue(facultyId)`.
2. Sum modifiers in spec order.
3. Roll 2d6 (inclusive 2–12).
4. `total = roll + modified + modifiers`.
5. `success = total >= Difficulty`.
6. Select branch ID; publish event.

### Passive resolution

1. No RNG.
2. `success = (modified + modifiers) >= Difficulty`.
3. `RolledValue = 0` in result; `TargetValue = Difficulty`.

### Repeatable lockout

On failure: lock `rollId` until `RecoverRepeatable` — typically when player completes fail-forward branch.

### Gated mode

`CanAttempt` delegates to `IGateService` / `IRuleEngine` when `RollContext.GateId` set (Phase 5 integration).

### Fail-forward contract

Story layer **must** provide non-empty `FailureBranchId`. Pipeline validates dialogue graphs for failure edges (Phase 4+).

---

## Event contracts + kernel tick phase

| Event | Payload | Phase |
|---|---|---|
| `RollResolved` | rollId, success, outcomeBranchId | Content / Interpretation |
| `RollAttemptBlocked` | rollId, reason `[FULL]` | Gating |

Passive rolls fire during **Interpretation** when exploration context changes. Active rolls fire during **Content** on player choice.

Subscribers: `IDialogueService` (branch), debug trace, `[FULL]` `IChronicleService`.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IFacultyService` | Base/modified values |
| `IInventoryService` | Equipment modifiers (Phase 12) |
| `IEmotionService` | Mood modifiers (Phase 8) |
| `IGateService` | Gated mode (Phase 5) |
| `IRandom` | Testable RNG injection |

| Used by | Reason |
|---|---|
| `IDialogueService` | Skill check nodes |
| `IInteractionService` | World skill checks (Phase 13) |
| Fail-forward pattern | Alternate branches |
| `ScriptCompiler` ROLL nodes | Phase 12 |

---

## MVP scope checklist (Phase 3)

- [ ] `IRollService` full contract
- [ ] Active 2d6 + faculty + DC
- [ ] Passive threshold mode
- [ ] Repeatable lock + recover
- [ ] Gated CanAttempt stub (always true until Phase 5)
- [ ] `RollResult` with OutcomeBranchId
- [ ] `RollResolved` event
- [ ] Tests: spec table examples, passive vs active, repeatable cycle

---

## Full scope

- `[FULL]` Critical bands (natural 12 / 2)
- `[FULL]` Partial success tier
- `[FULL]` Success with cost
- `[FULL]` Vitality/morale spend on attempt
- `[FULL]` RollDefinition content assets
- `[FULL]` Debug replay with fixed seed

---

## File tree

```text
Packages/NarrativeFramework/Cognition/Rolls/IRollService.cs
Packages/NarrativeFramework/Cognition/Rolls/RollService.cs
Packages/NarrativeFramework/Cognition/Rolls/FacultyRoll.cs
Packages/NarrativeFramework/Cognition/Rolls/RollResult.cs
Packages/NarrativeFramework/Cognition/Rolls/RollMode.cs
Packages/NarrativeFramework/Cognition/Rolls/RollContext.cs
Packages/NarrativeFramework/Cognition/Rolls/RollAttemptRegistry.cs
Packages/NarrativeFramework/Cognition/Rolls/RollServiceState.cs
Packages/NarrativeFramework/Cognition/Rolls/IRandom.cs
Packages/NarrativeFramework/Cognition/Rolls/SystemRandom.cs
Packages/NarrativeFramework/Tests/EditMode/Rolls/ActiveRollTests.cs
Packages/NarrativeFramework/Tests/EditMode/Rolls/PassiveRollTests.cs
Packages/NarrativeFramework/Tests/EditMode/Rolls/RepeatableRollTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `ActiveRoll_Success_WhenTotalGteDc` | Fixed RNG seed |
| `ActiveRoll_Failure_SelectsFailureBranch` | OutcomeBranchId |
| `PassiveRoll_NoDice_ThresholdOnly` | RolledValue 0 |
| `Repeatable_LocksOnFail_Recovers` | CanAttempt false then true |
| `ModifierStack_IncludesEquipment` | Phase 12 integration |
| `SpecExample_MatchesTable` | Roadmap exit: spec examples |

**Exit criteria:** Roll outcomes match spec examples in cognition-roll.md; ≥25 tests combined with faculty.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) 2d6 locked; `IRandom` injectable; `>=` DC; passive feedback Phase 13 Presentation.

---

## Related documents

- [cognition-faculty.md](cognition-faculty.md) — values
- [story-fail-forward.md](story-fail-forward.md) — failure routing pattern
- [content-inventory.md](content-inventory.md) — equipment modifiers
- [rules-gate.md](rules-gate.md) — gated attempts
