# Social Companion Service — Implementation Architecture

- Spec: [systems/social-companion.md](../systems/social-companion.md)
- Roadmap: [Phase 7](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why companion is a facade over one actor

The companion is a **narrative witness** implemented as a dedicated service wrapping a single `actor_*` ID (e.g. `actor_companion_kim`). `ICompanionService` exposes trust/availability APIs tuned for dialogue integration without forcing all consumers to know which actor is "the companion."

### Why trust metrics overlap relationship service

Companion-specific trust dimensions (`metric_companion_trust`, etc.) may mirror or alias `IRelationshipService` metrics for the companion actor ID. Companion service **delegates** `ModifyTrust` to relationship service in MVP to maintain one metric store — companion adds presence and availability semantics on top.

### Why presence gates dialogue

`IsAvailableForDialogue` combines actor availability, story flags, and companion state (Following vs Separated). Dialogue graphs query companion service before injecting commentary nodes or interrupt lines.

### Why companion never references Presentation

Commentary hooks emit events and content IDs; voice/UI layers subscribe in Phase 11+/13+. Core tests assert availability and trust deltas without Canvas.

---

## Implementation summary

| MVP (Phase 7) | Full |
|---|---|
| `GetCompanionActorId` from content config | Multiple companion handoff `[FULL]` |
| `GetTrustMetric` / `ModifyTrust` | Commentary trigger registry `[FULL]` |
| `IsAvailableForDialogue` | Schedule + location follow `[FULL]` |
| Subscribe dialogue/roll events for trust | Companion memory highlights `[FULL]` |
| `CompanionTrustChanged` event | Contextual barks content pipeline `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Social`
- **Namespace:** `NarrativeFramework.Social.Companions`
- **References:** Runtime, Simulation (IActorService, IEventBus), Social.Relationships
- **Tick phase:** **Social** (trust updates) · **Interpretation** (availability read for dialogue)

Registered as `ICompanionService`.

---

## Public API

See [contracts.md](contracts.md) — `ICompanionService`.

| Method | Semantics |
|---|---|
| `string GetCompanionActorId()` | Configured companion actor ID |
| `float GetTrustMetric(string metricId)` | Trust dimension for companion |
| `void ModifyTrust(string metricId, float delta)` | Adjust trust via relationship layer |
| `bool IsAvailableForDialogue()` | Present + not unavailable + not narrative locked |

Metric IDs use same `metric_*` prefix as relationship service.

---

## Internal implementation

### CompanionService

```csharp
sealed class CompanionService : ICompanionService, IStatefulService<CompanionServiceState>
{
    readonly CompanionConfig _config;
    readonly IRelationshipService _relationships;
    readonly IActorService _actors;
    readonly IStoryStateService _story;
    readonly IEventBus _events;

    public string GetCompanionActorId() => _config.CompanionActorId;

    public float GetTrustMetric(string metricId)
        => _relationships.GetMetric(_config.CompanionActorId, metricId);

    public void ModifyTrust(string metricId, float delta)
    {
        var prior = GetTrustMetric(metricId);
        _relationships.ModifyMetric(_config.CompanionActorId, metricId, delta);
        var next = GetTrustMetric(metricId);
        if (Math.Abs(next - prior) > float.Epsilon)
        {
            _events.Publish(new SimEvent {
                Type = EventTypes.CompanionTrustChanged,
                Payload = {
                    ["metricId"] = metricId,
                    ["prior"] = prior.ToString(CultureInfo.InvariantCulture),
                    ["next"] = next.ToString(CultureInfo.InvariantCulture)
                }
            });
        }
    }

    public bool IsAvailableForDialogue()
    {
        var id = _config.CompanionActorId;
        if (_story.GetFlag(_config.NarrativeLockedFlagId)) return false;
        var state = _actors.GetState(id);
        if (state.Availability == ActorAvailability.Unavailable) return false;
        return _state.Presence == CompanionPresence.Following
            || _state.Presence == CompanionPresence.Present;
    }
}
```

### CompanionConfig (content)

```csharp
class CompanionConfig : ContentDefinition
{
    public string CompanionActorId;
    public string NarrativeLockedFlagId;
    public List<string> TrustMetricIds;
}
```

### CompanionEventHandler

| Incoming event | Trust / commentary action |
|---|---|
| `DialogueChoiceSelected` (embarrassing) | `ModifyTrust(metric_companion_trust, -1)` |
| `RollResolved` (critical failure witnessed) | trust/respect deltas from table |
| `RelationshipChanged` (companion actor) | forward to `CompanionTrustChanged` if metric in config |

Commentary node injection `[FULL]` handled by dialogue service listening for `CompanionCommentaryRequested`.

### CompanionPresence state

```csharp
enum CompanionPresence { Following, Present, Separated, Unavailable, NarrativeLocked }

class CompanionServiceState
{
    public CompanionPresence Presence;
    public HashSet<string> FirstTimeReactionKeys;   // [FULL]
}
```

---

## Definition assets

```csharp
class CompanionCommentaryDefinition : ContentDefinition   // [FULL]
{
    public string TriggerKey;
    public string DialogueNodeId;
    public string RequiredLocationId;
    public float MinTrustMetric;
}

class CompanionReactionTable : ContentDefinition
{
    public string EventKey;
    public List<MetricDelta> TrustDeltas;
}
```

MVP: `CompanionConfig` only; reactions use shared `RelationshipModifierTable` keyed by companion actor.

---

## Runtime state

`CompanionServiceState` holds presence and first-time flags. Trust values **not** duplicated — persisted via `RelationshipServiceState` for companion actor ID.

Restore order: relationship before companion presence (persistence doc order).

---

## Core algorithms

### Availability evaluation

```text
narrative locked flag? → false
actor unavailable? → false
presence Separated/Unavailable? → false
else → true
```

Dialogue calls this before companion interjections; player-companion direct conversation still requires actor availability Idle/Talking.

### Trust modification path

All trust changes funnel through `IRelationshipService.ModifyMetric(companionActorId, ...)` so gates using relationship API and companion API stay consistent.

### Witness model

When companion presence is Following/Present and player performs witnessed action, content tags fire companion-specific modifiers **in addition to** standard relationship deltas for NPCs.

### Phase 7 exit integration

Companion trust change must alter dialogue gate: test registers low trust → gated choice denied; high trust → choice allowed (with Phase 4+5 dialogue/gate).

---

## Event contracts + tick phase

| Event | Publisher | Subscribers |
|---|---|---|
| `CompanionTrustChanged` | ICompanionService | IDialogueService, IVoiceService |
| `CompanionPresenceChanged` | `[FULL]` travel hooks | Exploration bridge |
| `CompanionCommentaryRequested` | `[FULL]` content triggers | Dialogue graph |

Trust writes: **Social**. Availability reads: **Interpretation**, **Gating**.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IRelationshipService | Metric storage |
| IActorService | Availability, location |
| IStoryStateService | Narrative lock flags |
| IEventBus | Trust notifications |
| IContentStore | Companion config |

| Used by | Reason |
|---|---|
| IDialogueService | Commentary + gates |
| IGateService | `companion.trust >= N` rules |
| ILocationService | Follow travel `[FULL]` |
| IPersistenceService | Presence state |
| IOutcomeService | Companion ending variants |

**Forbidden:** CompanionService → Presentation.

---

## MVP scope (Phase 7)

- [ ] `ICompanionService` + `CompanionService`
- [ ] `CompanionConfig` content binding
- [ ] Trust get/modify delegating to relationship
- [ ] `IsAvailableForDialogue` presence + availability
- [ ] `CompanionTrustChanged` event
- [ ] Handler for 2+ dialogue/roll event keys
- [ ] `CompanionServiceState` for presence
- [ ] Tests: trust delta, availability false when separated, gate integration

---

## Full scope

- `[FULL]` Auto-follow on player travel
- `[FULL]` Commentary catalog with contextual triggers
- `[FULL]` Companion knowledge separate from player (info flow)
- `[FULL]` Companion-initiated dialogue requests
- `[FULL]` Disapproval without simple approval meter

---

## File tree

```text
Packages/NarrativeFramework/Social/Companions/ICompanionService.cs
Packages/NarrativeFramework/Social/Companions/CompanionService.cs
Packages/NarrativeFramework/Social/Companions/CompanionConfig.cs
Packages/NarrativeFramework/Social/Companions/CompanionPresence.cs
Packages/NarrativeFramework/Social/Companions/CompanionEventHandler.cs
Packages/NarrativeFramework/Social/Companions/CompanionServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Social/CompanionTrustTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/CompanionAvailabilityTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/CompanionDialogueGateTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `ModifyTrust_DelegatesToRelationship` | Same value from GetMetric |
| `CompanionTrustChanged_EventFired` | Bus payload |
| `IsAvailableFalse_WhenSeparated` | Presence flag |
| `IsAvailableFalse_WhenActorUnavailable` | Actor state mock |
| `IsAvailableTrue_WhenFollowing` | Happy path |
| `Gate_BlocksChoiceWhenTrustLow` | Phase 7 exit criterion |
| `CaptureRestore_PreservesPresence` | Stateful roundtrip |

---

## Deferred decisions

**Resolved** — [decisions-log.md](../decisions-log.md):

| ID | Decision |
|---|---|
| C-02 | One active companion |
| C-03 | Companion side-channel dialogue allowed |

---

## Bootstrap and dialogue gate pattern

Phase 7 exit criterion test pattern — companion trust gates a dialogue choice:

```csharp
// Gate rule (conceptual)
Rule condition: companion.metric_trust >= 5
Rule context: ICompanionService.GetTrustMetric("metric_trust")

// Test flow
companion.ModifyTrust("metric_trust", -10);
Assert.False(gate.IsAllowed("gate_companion_confide", ctx));

companion.ModifyTrust("metric_trust", 20);
Assert.True(gate.IsAllowed("gate_companion_confide", ctx));
```

`CompanionConfig` loaded at startup from content id `companion_main`. Sample packs ship one companion; swapping config ID selects alternate co-star without code changes.

Presence defaults to `Following` on new game. Story beat may call internal `SetPresence(Separated)` via script action `[FULL]` or story flag side effect listening to `AdvanceBeat`.

Dialogue service checks `IsAvailableForDialogue()` before injecting companion lines — unavailable companion skips interjection nodes silently (fail-forward: main conversation continues).

Relationship history for companion actor ID visible via `IRelationshipService.GetRelationship(companionId)` for debug overlay Phase 14 — not player-facing MVP.

### Witnessed action modifier table

When companion presence is `Following` or `Present`, additional deltas apply from `CompanionReactionTable`:

| Event key | metric_trust | metric_respect |
|---|---|---|
| `witnessed_player_lie` | -2 | -1 |
| `witnessed_player_kindness` | +1 | 0 |
| `witnessed_critical_failure` | -1 | -2 |

Handler skips rows when `IsAvailableForDialogue()` false — separated companion does not judge off-screen actions in MVP (no remote memory `[FULL]`).

Companion commentary nodes in dialogue graphs use `companionAvailable: true` node metadata — validated at compile time against `ICompanionService` contract documentation in content-script spec Phase 12.

Trust metrics for companion use the same persistence envelope as other actors — no separate companion trust save blob.

---

## Related documents

- [social-relationship.md](social-relationship.md), [sim-actor.md](sim-actor.md)
- [story-dialogue.md](../systems/story-dialogue.md), [rules-gate.md](rules-gate.md)
- [social-ideology.md](social-ideology.md)
