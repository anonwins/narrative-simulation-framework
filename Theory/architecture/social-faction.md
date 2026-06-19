# Social Faction Service — Implementation Architecture

- Spec: [systems/social-faction.md](../systems/social-faction.md)
- Roadmap: [Phase 7](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why factions are political actors, not reputation bars

Organized groups in NSF have agendas, secrets, and inter-faction relations. `IFactionService` tracks **player standing** per `faction_*` plus stance derivation for dialogue and world reactions — not a single "+5 Union" number.

### Why standing is separate from personal relationship

An actor may like the player while their faction distrusts them. Faction standing gates institutional access (records, backup); `IRelationshipService` gates personal disclosure. Layers stay separate to avoid conflating systems.

### Why member lists are queryable

Dialogue and gates need "any `faction_police` member present?" `GetMemberActorIds` reads from content definitions merged with runtime membership overrides `[FULL]`.

### Why faction lives in Social assembly

Faction logic consumes events and affects gates but does not mutate simulation facts directly — it publishes events and modifies standing for story/content to react.

---

## Implementation summary

| MVP (Phase 7) | Full |
|---|---|
| `GetStanding` / `ModifyStanding` | Multi-axis standing (trust/hostility) `[FULL]` |
| `GetStance` derived from thresholds | Faction ↔ faction relations `[FULL]` |
| `GetMemberActorIds` from content | Dynamic membership changes `[FULL]` |
| Faction observation event handler | Faction goals / resources simulation `[FULL]` |
| `FactionStandingChanged` event | Indirect standing (help A hurts B) `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Social`
- **Namespace:** `NarrativeFramework.Social.Factions`
- **References:** Runtime, Simulation.Events, Content
- **Tick phase:** **Social** (standing updates) · **Events** (notifications)

Registered as `IFactionService`.

---

## Public API

See [contracts.md](contracts.md) — `IFactionService`.

| Method | Purpose |
|---|---|
| `int GetStanding(string factionId)` | Player standing score |
| `void ModifyStanding(string factionId, int delta)` | Clamped integer adjustment |
| `FactionStance GetStance(string factionId)` | Derived enum from standing bands |
| `IReadOnlyList<string> GetMemberActorIds(string factionId)` | Actor IDs with membership |

```csharp
enum FactionStance { Hostile, Wary, Neutral, Friendly, Allied }
```

---

## Internal implementation

### FactionService

```csharp
sealed class FactionService : IFactionService, IStatefulService<FactionServiceState>
{
    readonly Dictionary<string, int> _standing = new();
    readonly FactionRegistry _factions = new();
    readonly IContentStore _content;
    readonly IEventBus _events;

    public int GetStanding(string factionId)
        => _standing.TryGetValue(factionId, out var v) ? v : GetDefaultStanding(factionId);

    public void ModifyStanding(string factionId, int delta)
    {
        ContentIdValidator.IsValid(factionId, "faction_");
        var prior = GetStanding(factionId);
        var def = _content.GetDefinition<FactionDefinition>(factionId);
        var next = Math.Clamp(prior + delta, def.StandingMin, def.StandingMax);
        _standing[factionId] = next;
        if (next != prior)
        {
            _events.Publish(new SimEvent {
                Type = EventTypes.FactionStandingChanged,
                Payload = {
                    ["factionId"] = factionId,
                    ["prior"] = prior.ToString(),
                    ["next"] = next.ToString(),
                    ["stance"] = GetStance(factionId).ToString()
                }
            });
        }
    }

    public FactionStance GetStance(string factionId)
    {
        var s = GetStanding(factionId);
        var bands = _content.GetDefinition<FactionDefinition>(factionId).StanceBands;
        return StanceResolver.Resolve(s, bands);
    }

    public IReadOnlyList<string> GetMemberActorIds(string factionId)
        => _factions.GetMembers(factionId);
}
```

### FactionRegistry (internal)

Loads `FactionDefinition.MemberActorIds` at init. `[FULL]` runtime overrides from story flags stored in `FactionServiceState.MembershipOverrides`.

### FactionObservationHandler

Listens for content-tagged events (`SupportedFactionGoal`, `UnderminedFaction`, `HelpedMember`). Applies primary delta + `[FULL]` cross-faction ripple table:

```text
Help faction_guild → faction_guild +2, faction_enterprise +1 suspicion
```

MVP: primary faction only.

### StanceResolver (internal)

Maps integer standing to `FactionStance` using content-defined breakpoints (e.g. ≤-20 Hostile, ≥40 Allied).

---

## Definition assets

```csharp
class FactionDefinition : ContentDefinition
{
    public string DisplayNameKey;
    public int StandingMin;
    public int StandingMax;
    public int DefaultStanding;
    public StanceBands StanceBands;
    public List<string> MemberActorIds;
    public List<string> AllyFactionIds;     // [FULL]
    public List<string> RivalFactionIds;    // [FULL]
}

class StanceBands
{
    public int HostileMax;
    public int WaryMax;
    public int NeutralMax;
    public int FriendlyMax;
    // above FriendlyMax → Allied
}

class FactionStandingModifierTable : ContentDefinition
{
    public string EventKey;
    public string FactionId;
    public int Delta;
    public List<CrossFactionRipple> Ripples;   // [FULL]
}
```

---

## Runtime state

```csharp
class FactionServiceState
{
    public Dictionary<string, int> Standings;
    public Dictionary<string, List<string>> MembershipOverrides;   // [FULL]
    public Dictionary<string, FactionRelationSnapshot> FactionRelations;  // [FULL]
}
```

MVP persists `Standings` only.

---

## Core algorithms

### Default standing

Unmentioned faction returns `FactionDefinition.DefaultStanding` (usually 0 Neutral band).

### ModifyStanding clamp

Integer arithmetic only. Delta zero short-circuits without event.

### GetStance caching

Stance is computed on read — not stored — to avoid desync when bands change in content updates mid-dev.

### Member query

Union of static `MemberActorIds` and runtime overrides. Validates each ID has `actor_*` prefix.

### Actor default faction link

`ActorDefinition.DefaultFactionId` is content metadata; faction service does not duplicate. Observation events may check actor's faction via content lookup when handling `HelpedMember`.

### Gate integration

Rules: `faction(faction_police).stance >= Friendly` or `faction(faction_guild).standing >= 10`.

---

## Event contracts + tick phase

| Event | Publisher | Subscribers |
|---|---|---|
| `FactionStandingChanged` | IFactionService | Dialogue, chronicle, outcome |
| `FactionStanceCrossed` | `[FULL]` on band boundary | Story beats |
| Player action events | Various | FactionObservationHandler |

Standing writes in **Social** phase; gates read in **Gating**.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| IContentStore | Faction defs + modifier tables |
| IEventBus | Standing notifications |

| Used by | Reason |
|---|---|
| IGateService | Institutional access |
| IDialogueService | Faction-tagged lines |
| IActorService | Member identity (content) |
| IInfoFlowService | Faction knowledge `[FULL]` |
| IOutcomeService | Political endings |
| IPersistenceService | Standing snapshot |

**Forbidden:** FactionService → Presentation.

---

## MVP scope (Phase 7)

- [ ] `IFactionService` + `FactionService`
- [ ] Standing get/modify with clamp
- [ ] `GetStance` from content bands
- [ ] `GetMemberActorIds` from definitions
- [ ] Observation handler stub with 2–3 event keys
- [ ] `FactionStandingChanged` event
- [ ] `FactionServiceState` capture/restore
- [ ] Tests: standing delta, stance bands, members list

---

## Full scope

- `[FULL]` Multi-dimensional faction reputation (trust vs hostility)
- `[FULL]` Faction ↔ faction dynamic relations
- `[FULL]` Cross-faction ripple on standing changes
- `[FULL]` Faction knowledge bank separate from actors
- `[FULL]` Influence resources affecting story outcomes

---

## File tree

```text
Packages/NarrativeFramework/Social/Factions/IFactionService.cs
Packages/NarrativeFramework/Social/Factions/FactionService.cs
Packages/NarrativeFramework/Social/Factions/FactionRegistry.cs
Packages/NarrativeFramework/Social/Factions/FactionDefinition.cs
Packages/NarrativeFramework/Social/Factions/FactionStance.cs
Packages/NarrativeFramework/Social/Factions/StanceResolver.cs
Packages/NarrativeFramework/Social/Factions/FactionObservationHandler.cs
Packages/NarrativeFramework/Social/Factions/FactionServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Social/FactionStandingTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/FactionStanceTests.cs
Packages/NarrativeFramework/Tests/EditMode/Social/FactionMemberTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `ModifyStanding_ClampsMax` | At StandingMax |
| `GetStance_NeutralBand` | Mid-range standing |
| `GetStance_AlliedAtHighStanding` | Band boundary |
| `GetMemberActorIds_ReturnsContentList` | Count + IDs |
| `StandingChange_PublishesEvent` | Payload factionId |
| `CaptureRestore_PreservesStandings` | Roundtrip |
| `ObservationHandler_AppliesDelta` | Event → standing |

Phase 7 exit: faction standing + relationship trust tests combined.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) C-04, SOC-02 (multi-faction; standing decay when time active).

---

## Bootstrap and observation event catalog

Phase 7 MVP ships a minimal observation catalog in content:

| Event key | Typical trigger | Default delta |
|---|---|---|
| `faction_helped_member` | Dialogue / script | +2 primary faction |
| `faction_undermined_goal` | Failed gate / choice | -3 primary faction |
| `faction_supported_goal` | Story beat completion | +4 primary faction |

Handler resolves primary faction from payload `factionId` or from helped actor's `DefaultFactionId` in content when omitted.

```csharp
// FactionObservationHandler — Social phase batch entry
public void OnSocialPhase(IReadOnlyList<SimEvent> queued)
{
    foreach (var e in queued.Where(x => x.Type == EventTypes.FactionObservation))
        ApplyModifierTable(e.Payload["eventKey"], e.Payload.GetValueOrDefault("factionId"));
}
```

Cross-faction ripples `[FULL]` read `CrossFactionRipple` rows: `{ "targetFactionId": "faction_enterprise", "delta": 2 }` applied after primary delta in same Social batch.

Institutional dialogue tags in content use `requiresFactionStance(faction_police, Friendly)` — gate engine maps to `GetStance` comparison, not raw standing, so authors think in narrative stance bands.

Standing values are integers for author mental model; no fractional faction reputation in MVP. Debug overlay may show numeric standing to writers in Phase 14 tooling only.

Faction service never references Unity or Presentation assemblies — institutional UI badges bind in Phase 13 via view models.

---

## Related documents

- [social-relationship.md](social-relationship.md), [sim-actor.md](sim-actor.md)
- [social-ideology.md](social-ideology.md), [contracts.md](contracts.md)
- [story-outcome.md](../systems/story-outcome.md)
