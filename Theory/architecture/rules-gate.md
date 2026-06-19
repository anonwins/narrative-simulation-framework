# Rules Gate Service — Implementation Architecture

- Spec: [rules-gate.md](../systems/rules-gate.md)
- Glossary: [Gate](../terminology-glossary.md) · [Fact vs Story flag](../terminology-glossary.md)
- Roadmap: [Phase 5](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Engine: [rules-engine.md](rules-engine.md)

---

## Design rationale

### Why gates wrap rules with identity and UX metadata

`IRuleEngine` evaluates anonymous IF/THEN trees. `IGateService` adds **gate IDs**, denial reasons, and content-pack-facing definitions so designers ask “who can access this?” Gates compose faculty, belief, fact, conduct, relationship, and beat conditions — all delegated to rule evaluation.

### Why gates run in Gating phase

Gates depend on facts registered in **Facts** phase same tick (see [sim-fact.md](sim-fact.md)). Kernel runs **Gating** after **Events** so scheduled state is visible. Dialogue choice checks call gate service during Gating before Content execution.

### Why gates never read chronicle state

Chronicle is a **read-only projection**. A task shown as “complete” in UI does not gate content — underlying flag/beat/fact does. This prevents projection lag from unlocking content incorrectly.

### Term split for gated content

| Gating on… | Service | Example gate ID |
|---|---|---|
| Narrative progression | Story beat stage | `gate_beat_act2_enter` |
| World truth | Fact active | `gate_fact_body_examined` |
| Investigation milestone | Thread resolution | `gate_thread_suspect_identified` |
| Player journal visibility | **Not a gate input** | Use flag/beat instead |

---

## Implementation summary

| MVP (Phase 5) | Full (Phase 8+) |
|---|---|
| `IsAllowed` / `GetDenialReason` | Belief phase gates `[FULL]` |
| Faculty threshold gates | Relationship / faction gates `[FULL]` |
| Fact + flag + beat gates | Political ideology gates `[FULL]` |
| Dialogue condition delegation | Inventory item gates Phase 12 |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Rules`
- **Namespace:** `NarrativeFramework.Rules.Gates`
- **Tick phase:** **Gating**

---

## Public API

See [contracts.md](contracts.md) — `IGateService`.

```csharp
interface IGateService
{
    bool IsAllowed(string gateId, GateContext context);
    GateDenialReason GetDenialReason(string gateId, GateContext context);
}

class GateContext
{
    public string ActorId;
    public string ContentId;      // optional: dialogue node, location
    public RuleContext RuleContext;
}

enum GateDenialReason
{
    None,
    FacultyTooLow,
    FactMissing,
    FlagFalse,
    BeatNotReady,
    ThreadUnresolved,
    ConductTooLow,
    Custom
}
```

---

## Internal implementation

### GateService

```csharp
sealed class GateService : IGateService
{
    readonly IContentStore _content;
    readonly IRuleEngine _rules;

    public bool IsAllowed(string gateId, GateContext context)
    {
        var def = _content.GetDefinition<GateDefinition>(gateId);
        return _rules.Evaluate(def.Rule, context.RuleContext);
    }

    public GateDenialReason GetDenialReason(string gateId, GateContext context)
    {
        var def = _content.GetDefinition<GateDefinition>(gateId);
        if (_rules.Evaluate(def.Rule, context.RuleContext))
            return GateDenialReason.None;

        return def.DenialReasonHint != GateDenialReason.None
            ? def.DenialReasonHint
            : InferDenial(def.Rule, context);
    }

    GateDenialReason InferDenial(Rule rule, GateContext ctx)
    {
        // Walk failed leaf for UX hint — MVP: first failing leaf
        /* ... */
        return GateDenialReason.Custom;
    }
}
```

### GateDefinition (content)

```csharp
class GateDefinition : ContentDefinition
{
    public string Id;              // gate_*
    public Rule Rule;
    public GateDenialReason DenialReasonHint;
    public string DenialTextKey;   // locale for UI
}
```

### GateRegistry (internal)

Optional index by content ID for reverse lookup `[FULL]`. MVP: forward lookup by `gate_*` only.

---

## Definition assets

| Asset | ID prefix | Typical conditions |
|---|---|---|
| `GateDefinition` | `gate_*` | Faculty, fact, flag, beat |
| Content references | on dialogue nodes, locations | `RequiredGateId` field |

Beat gates use `ConditionKind.BeatStage`, not chronicle section IDs.

---

## Runtime state

Gate service is **stateless**. Gate open/closed derives from other services every evaluation — no cached “unlocked” set in MVP (avoids stale cache bugs).

`[FULL]` optional memoization per tick keyed by `(gateId, snapshotVersion)`.

---

## Core algorithms

### IsAllowed

1. Load `GateDefinition` from content store
2. Merge `GateContext` into `RuleContext`
3. Return `_rules.Evaluate(def.Rule, ctx)` — no side effects

### GetDenialReason

1. If allowed, return `None`
2. Use authored hint if present
3. Else infer from first failing condition leaf for player-facing feedback

### Faculty gate example

```text
gate_logic_deduction_4:
  Rule: FacultyValue(logic) >= 4 AND FactActive(fact_clue_found)
  DenialHint: FacultyTooLow
```

### Thread gate example (Phase 9)

```text
gate_confront_suspect:
  Rule: ThreadResolved(thread_homicide, subject_suspect, Correct)
```

Thread resolution is authoritative; beat `beat_confrontation` may also need `Active` — compose with `All`.

### Kernel slice test (Phase 5 exit)

1. Tick **Facts** — register `fact_key_found`
2. Tick through **Gating** — `IsAllowed(gate_need_key)` true
3. Without fact — false, `GetDenialReason` → `FactMissing`

---

## Event contracts

Gate service publishes no events. When gate blocks dialogue, dialogue returns false — optional `GateDenied` debug event `[FULL]`.

Unblocking happens when underlying services mutate and publish domain events; chronicle refreshes separately.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IRuleEngine` | Evaluation |
| `IContentStore` | Gate definitions |
| `IFactService` | Via rule leaves |
| `IStoryStateService` | Via rule leaves |
| `IFacultyService` | Via rule leaves Phase 3+ |
| `IThreadService` | Thread resolution gates Phase 9 |

| Used by | Reason |
|---|---|
| `IDialogueService` | Choice/node visibility |
| `IExplorationService` | Location entry Phase 13 |
| `IDiscoveryService` | Discoverable unlock `[FULL]` |
| `IPacingService` | Orthogonal — pacing delays after gate passes |

---

## MVP scope (Phase 5)

- [ ] `GateService` with IsAllowed + GetDenialReason
- [ ] Gate definitions for fact, flag, beat, faculty
- [ ] Dialogue integration
- [ ] Kernel test: fact before gate same tick
- [ ] ≥15 tests with rule engine suite

---

## Full scope

- `[FULL]` Belief, relationship, faction, conduct gates
- `[FULL]` Denial text keys to locale
- `[FULL]` Gate debug overlay Phase 14
- `[FULL]` Item inventory gates

---

## File tree

```text
Packages/NarrativeFramework/Rules/Gates/IGateService.cs
Packages/NarrativeFramework/Rules/Gates/GateService.cs
Packages/NarrativeFramework/Rules/Gates/GateContext.cs
Packages/NarrativeFramework/Rules/Gates/GateDenialReason.cs
Packages/NarrativeFramework/Content/Definitions/GateDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Rules/GateServiceAllowDenyTests.cs
Packages/NarrativeFramework/Tests/EditMode/Rules/FactsBeforeGatingGateTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `IsAllowed_FactGate` | True when fact active |
| `IsAllowed_FlagGate` | Story flag condition |
| `IsAllowed_BeatStageGate` | Beat stage threshold |
| `GetDenialReason_Faculty` | Hint when faculty low |
| `Gate_NoChronicleDependency` | Chronicle mock never called |
| `Kernel_FactRegisteredSameTick_GatePasses` | Phase ordering |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) ST-06.

---

## Related documents

- [rules-engine.md](rules-engine.md), [story-dialogue.md](story-dialogue.md)
- [sim-fact.md](sim-fact.md), [runtime-kernel.md](runtime-kernel.md)
- [ledger-thread.md](ledger-thread.md)
- [contracts.md](contracts.md)
