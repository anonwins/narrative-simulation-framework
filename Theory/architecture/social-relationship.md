# Social Relationship Service — Implementation Architecture

- Spec: [systems/social-relationship.md](../systems/social-relationship.md)
- Roadmap: [Phase 7](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why relationships are multi-dimensional vectors

Single "friendship" scores cannot represent NSF social contradiction (high respect, low trust). `IRelationshipService` exposes named **metrics** (`metric_trust`, `metric_respect`, etc.) as floats so gates and dialogue query individual dimensions without combinatorial enum explosion.

### Why player ↔ actor is MVP scope

Actor ↔ actor and faction matrices add graph complexity; MVP stores one `RelationshipState` per `actor_*` target relative to implicit player source. `[FULL]` generalizes to `(sourceId, targetId)` pairs behind the same API shape.

### Why modifications go through events

Dialogue, rolls, and conduct publish social events; `RelationshipEventHandler` applies configured deltas from content. Direct `ModifyMetric` remains for scripts and tests — both paths publish `RelationshipChanged` for chronicle and companion reactions.

### Why Social assembly is separate from Simulation

Social modules depend on Simulation (actors, events) but Simulation must not depend on Social — avoids cycles. Core actor memory stays in `IActorService`; relationship metrics are Social layer interpretation of repeated interactions.

---

## Implementation summary

| MVP (Phase 7) | Full |
|---|---|
| `GetRelationship` / `ModifyMetric` / `GetMetric` | Actor ↔ actor edges `[FULL]` |
| Default metrics from glossary pattern | Relationship history log `[FULL]` |
| Event-driven deltas from content tables | Derived labels (Trusted Ally) `[FULL]` |
| `RelationshipChanged` event | Asymmetric player unknown trust `[FULL]` |
| Persistence `RelationshipServiceState` | Faction-influenced drift `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Social`
- **Namespace:** `NarrativeFramework.Social.Relationships`
- **References:** Runtime, Simulation (IEventBus, IActorService — interfaces only)
- **Tick phase:** **Social** (apply queued deltas) · **Events** (publish changes)

Registered as `IRelationshipService`. No Presentation references.

---

## Public API

See [contracts.md](contracts.md) — `IRelationshipService`.

| Method | Behavior |
|---|---|
| `RelationshipState GetRelationship(string actorId)` | Full metric bag for actor vs player |
| `void ModifyMetric(string actorId, string metricId, float delta)` | Clamp to configured min/max |
| `float GetMetric(string actorId, string metricId)` | Single dimension read |

`RelationshipState` DTO:

```csharp
class RelationshipState
{
    public Dictionary<string, float> Metrics;   // metric_trust → value
}
```

---

## Internal implementation

### RelationshipService

```csharp
sealed class RelationshipService : IRelationshipService, IStatefulService<RelationshipServiceState>
{
    readonly RelationshipRegistry _registry = new();
    readonly IContentStore _content;
    readonly IEventBus _events;

    public RelationshipState GetRelationship(string actorId)
        => _registry.GetOrCreate(actorId).ToDto();

    public float GetMetric(string actorId, string metricId)
        => _registry.GetOrCreate(actorId).GetMetric(metricId);

    public void ModifyMetric(string actorId, string metricId, float delta)
    {
        ContentIdValidator.IsValid(actorId, "actor_");
        ContentIdValidator.IsValid(metricId, "metric_");
        var inst = _registry.GetOrCreate(actorId);
        var prior = inst.GetMetric(metricId);
        inst.ApplyDelta(metricId, delta, _content.GetDefinition<MetricDefinition>(metricId));
        var next = inst.GetMetric(metricId);
        if (Math.Abs(next - prior) > float.Epsilon)
        {
            _events.Publish(new SimEvent {
                Type = EventTypes.RelationshipChanged,
                Payload = {
                    ["actorId"] = actorId,
                    ["metricId"] = metricId,
                    ["prior"] = prior.ToString(CultureInfo.InvariantCulture),
                    ["next"] = next.ToString(CultureInfo.InvariantCulture)
                }
            });
        }
    }
}
```

### RelationshipRegistry (internal)

```csharp
sealed class RelationshipRegistry
{
    readonly Dictionary<string, RelationshipInstance> _byActor = new();

    public RelationshipInstance GetOrCreate(string actorId)
    {
        if (!_byActor.TryGetValue(actorId, out var r))
            _byActor[actorId] = r = RelationshipInstance.CreateDefault(actorId);
        return r;
    }
}

sealed class RelationshipInstance
{
    readonly Dictionary<string, float> _metrics = new();

    public float GetMetric(string metricId)
        => _metrics.TryGetValue(metricId, out var v) ? v : 0f;

    public void ApplyDelta(string metricId, float delta, MetricDefinition def)
    {
        var v = GetMetric(metricId) + delta;
        _metrics[metricId] = Math.Clamp(v, def.Min, def.Max);
    }
}
```

### RelationshipEventHandler

Maps `DialogueChoiceSelected`, `RollResolved`, conduct events to content `RelationshipModifierTable` rows:

```json
{ "eventKey": "HELPED_ACTOR", "metric_trust": 2, "metric_respect": 1 }
```

Batch apply during **Social** phase to keep ordering before Events flush.

---

## Definition assets

```csharp
class MetricDefinition : ContentDefinition
{
    public float Min;
    public float Max;
    public float DefaultValue;
    public string DisplayNameKey;       // hidden from player in MVP UI
}

class RelationshipModifierTable : ContentDefinition
{
    public string EventKey;
    public List<MetricDelta> Deltas;
}

class MetricDelta
{
    public string MetricId;
    public float Delta;
}
```

Standard metrics per glossary: `metric_trust`, `metric_respect`, `metric_fear`, `metric_suspicion`, `metric_familiarity`.

Gate rules reference thresholds: `relationship(actor_x).metric_trust >= 5`.

---

## Runtime state

```csharp
class RelationshipServiceState
{
    public List<RelationshipInstanceSnapshot> Relationships;
}

class RelationshipInstanceSnapshot
{
    public string ActorId;
    public Dictionary<string, float> Metrics;
}
```

History log `[FULL]`:

```csharp
class RelationshipHistoryEntry
{
    public string EventKey;
    public GameTime At;
    public Dictionary<string, float> Deltas;
}
```

---

## Core algorithms

### Default initialization

On first `GetRelationship(actorId)`, seed metrics from `MetricDefinition.DefaultValue` for all metrics declared in content pack manifest.

### Clamping policy

All values clamped to `[Min, Max]` from definition. Overflow logged in debug — no wraparound.

### Social phase batch

1. Collect pending social events from queue (interpretation outputs)
2. Lookup modifier tables
3. Apply cumulative deltas per actor/metric
4. Publish single `RelationshipChanged` per changed metric

Prevents double-counting when dialogue fires multiple tags in one choice.

### Gate evaluation

`IGateService` calls `GetMetric` during **Gating**. Relationship service read-only in that phase — mutations only Social/Content.

### Companion overlap

`ICompanionService` may mirror subset of metrics for companion actor ID or delegate to `IRelationshipService` for same ID — single source of truth on relationship service.

---

## Event contracts + tick phase

| Event | Publisher | Subscribers | Phase |
|---|---|---|---|
| `RelationshipChanged` | IRelationshipService | ICompanionService, chronicle, dialogue tone | Events |
| `DialogueChoiceSelected` | IDialogueService | RelationshipEventHandler | Events → Social |
| `RollResolved` | IRollService | RelationshipEventHandler (failure embarrassment) | Events |

**Write phase:** Social. **Read phase:** Gating, Interpretation.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IEventBus | Change notifications |
| IContentStore | Metric defs + modifier tables |
| IActorService | Validate actor IDs exist `[FULL]` |

| Used by | Reason |
|---|---|
| IGateService | Threshold conditions |
| IDialogueService | Tone / topic availability |
| ICompanionService | Trust read for commentary |
| IFactionService | Indirect reputation `[FULL]` |
| IPersistenceService | Metric snapshot |
| IOutcomeService | Ending synthesis `[Phase 11]` |

**Forbidden:** Social.Relationships → Presentation.

---

## MVP scope (Phase 7)

- [ ] `IRelationshipService` + `RelationshipService`
- [ ] Five core metrics with content defaults
- [ ] `ModifyMetric` with clamp
- [ ] Handler wiring for dialogue choice event keys
- [ ] `RelationshipChanged` event
- [ ] `RelationshipServiceState` capture/restore
- [ ] Tests: delta, clamp, event, gate threshold slice

---

## Full scope

- `[FULL]` Actor ↔ actor relationship graph
- `[FULL]` History ring buffer per actor
- `[FULL]` Derived relationship labels for author tools
- `[FULL]` Relationship decay over time
- `[FULL]` Faction standing bleed into personal metrics

---

## File tree

```text
Packages/NarrativeFramework/Social/Relationships/IRelationshipService.cs
Packages/NarrativeFramework/Social/Relationships/RelationshipService.cs
Packages/NarrativeFramework/Social/Relationships/RelationshipRegistry.cs
Packages/NarrativeFramework/Social/Relationships/RelationshipState.cs
Packages/NarrativeFramework/Social/Relationships/MetricDefinition.cs
Packages/NarrativeFramework/Social/Relationships/RelationshipModifierTable.cs
Packages/NarrativeFramework/Social/Relationships/RelationshipEventHandler.cs
Packages/NarrativeFramework/Social/Relationships/RelationshipServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Social/RelationshipMetricTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/RelationshipEventTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/RelationshipGateTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `ModifyMetric_ClampsToMax` | Value == Max |
| `GetMetric_DefaultZero` | Unknown metric |
| `DialogueChoice_AppliesTableDelta` | Trust increased |
| `RelationshipChanged_PublishedOnce` | Event count |
| `CaptureRestore_PreservesMetrics` | Roundtrip |
| `Gate_BlocksBelowThreshold` | With Phase 5 kernel |
| `NegativeDelta_RespectsMin` | Floor clamp |

Contributes to Phase 7 ≥20 tests with faction/companion/ideology.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) C-06, SOC-01 (float API; Noir labels / SciFi numbers in Presentation).

---

## Related documents

- [social-companion.md](social-companion.md), [social-faction.md](social-faction.md)
- [sim-actor.md](sim-actor.md), [story-dialogue.md](../systems/story-dialogue.md)
- [rules-gate.md](rules-gate.md), [contracts.md](contracts.md)
