# Ledger Chronicle Service — Implementation Architecture

- Spec: [ledger-chronicle.md](../systems/ledger-chronicle.md)
- Glossary: [Chronicle section](../terminology-glossary.md) · [When to use which term](../terminology-glossary.md#when-to-use-which-term)
- Roadmap: [Phase 9](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Thread: [ledger-thread.md](ledger-thread.md)

---

## Design rationale

### Why chronicle is read-only projection — CRITICAL

`IChronicleService` **never mutates** authoritative narrative state. It does not advance story beats, add thread evidence, or register facts. Players experience chronicle as their journal; engineers treat it as a **materialized view** rebuilt from Thread + StoryState + Facts + domain events. Violating this creates desync where UI shows progress the simulation denies (or vice versa).

### Why chronicle is not a quest log

Quest logs store objectives as authoritative. NSF chronicle answers “what is happening in the character’s life?” — Leads, Tasks, Clues are **entry types** projected from inquiry and progression state, not checkboxes that complete quests.

### Why chronicle sections are not story beats or threads

| Term | What it is | ID pattern |
|---|---|---|
| **Chronicle section** | Player journal grouping for one storyline | `section_*` |
| **Story beat** | Progression unit in `IStoryStateService` | `beat_*` |
| **Thread** | Investigation container with evidence/theories | `thread_*` |

One homicide **thread** may map to one chronicle **section**, but section IDs ≠ beat IDs ≠ thread IDs. Projection rules join them.

### Why RefreshProjection instead of incremental mutation

MVP rebuilds sections from authoritative services on event batch — simpler to prove read-only invariant. `[FULL]` incremental projection with version stamps for performance.

---

## Implementation summary

| MVP (Phase 9) | Full (Phase 14+) |
|---|---|
| `GetSections` / `GetEntries` | People/locations/statistics sections `[FULL]` |
| `RefreshProjection` on domain events | Incremental projection cache `[FULL]` |
| Entry types: Lead, Task, Clue | Political/personal history timelines `[FULL]` |
| Test: chronicle never mutates authority | Completed entry archive `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Ledger`
- **Namespace:** `NarrativeFramework.Ledger.Chronicle`
- **Tick phase:** **Events** (subscriber refresh after domain events)

**CRITICAL:** No `SetEntry`, `CompleteTask`, or `AddSection` on public API.

---

## Public API

See [contracts.md](contracts.md) — `IChronicleService`.

```csharp
interface IChronicleService
{
    IReadOnlyList<ChronicleSection> GetSections();
    IReadOnlyList<ChronicleEntry> GetEntries(string sectionId);
    void RefreshProjection();
}
```

Presentation binds via `IChroniclePresenter` + `ChronicleView` — not via mutation hooks.

---

## Internal implementation

### ChronicleService

```csharp
sealed class ChronicleService : IChronicleService
{
    readonly INarrativeServiceRegistry _registry;
    readonly ChronicleProjector _projector;
    List<ChronicleSection> _sections = new();
    Dictionary<string, List<ChronicleEntry>> _entriesBySection = new();

    public IReadOnlyList<ChronicleSection> GetSections() => _sections;

    public IReadOnlyList<ChronicleEntry> GetEntries(string sectionId)
        => _entriesBySection.TryGetValue(sectionId, out var e) ? e : Array.Empty<ChronicleEntry>();

    public void RefreshProjection()
    {
        var snapshot = BuildProjectionInput(_registry);
        var result = _projector.Project(snapshot);
        _sections = result.Sections;
        _entriesBySection = result.EntriesBySection;
    }

    ProjectionInput BuildProjectionInput(INarrativeServiceRegistry reg)
    {
        return new ProjectionInput {
            Facts = reg.GetRequired<IFactService>().GetAll(),
            Story = reg.GetRequired<IStoryStateService>(),
            Threads = reg.GetRequired<IThreadService>(),
            // `[FULL]` relationships, beliefs for diary sections
        };
    }
}
```

### ChronicleProjector (internal)

```csharp
sealed class ChronicleProjector
{
    readonly IContentStore _content;

    public ProjectionResult Project(ProjectionInput input)
    {
        var sections = new List<ChronicleSection>();
        var entries = new Dictionary<string, List<ChronicleEntry>>();

        foreach (var mapping in _content.GetAllIds<ChronicleSectionMapping>())
        {
            var def = _content.GetDefinition<ChronicleSectionMapping>(mapping);
            if (!ShouldShowSection(def, input)) continue;

            sections.Add(new ChronicleSection {
                Id = def.SectionId,
                TitleKey = def.TitleKey,
                SortOrder = def.SortOrder
            });
            entries[def.SectionId] = BuildEntries(def, input);
        }
        return new ProjectionResult { Sections = sections, EntriesBySection = entries };
    }

    List<ChronicleEntry> BuildEntries(ChronicleSectionMapping def, ProjectionInput input)
    {
        var list = new List<ChronicleEntry>();

        // Tasks from story beat stages
        foreach (var beatLink in def.BeatLinks)
        {
            var stage = input.Story.GetBeatState(beatLink.BeatId).Stage;
            if (stage >= beatLink.MinStageForTask)
                list.Add(ChronicleEntry.Task(beatLink.TaskTextKey, beatLink.BeatId));
        }

        // Leads/clues from thread evidence
        foreach (var threadLink in def.ThreadLinks)
        {
            var evidence = input.Threads.GetEvidence(threadLink.ThreadId);
            foreach (var ev in evidence)
                list.Add(MapEvidenceToEntry(ev, threadLink));
        }

        return list;
    }
}
```

### ChronicleEventSubscriber (internal)

```csharp
sealed class ChronicleEventSubscriber : IEventHandler
{
    readonly IChronicleService _chronicle;

    public void Handle(SimEvent e, SimulationSnapshot state)
    {
        switch (e.Type)
        {
            case EventTypes.StoryFlagChanged:
            case EventTypes.StoryBeatAdvanced:
            case EventTypes.FactRegistered:
            case EventTypes.ThreadEvidenceAdded:
            case EventTypes.ThreadSubjectResolved:
                _chronicle.RefreshProjection();
                break;
        }
    }
}
```

Subscriber **only** calls `RefreshProjection` — never story or thread mutators.

---

## Definition assets

| Asset | Purpose |
|---|---|
| `ChronicleSectionMapping` | Joins `section_*` to `beat_*` and `thread_*` |
| Entry text keys | Locale-resolved in Presentation |
| `ChronicleEntryType` | Lead, Task, Clue, `[FULL]` Person, Location |

Authoring maps beats → Task entries, thread evidence → Lead/Clue — explicit projection rules, not implicit ID equality.

---

## Runtime state

```csharp
class ChronicleServiceState
{
    public List<ChronicleSection> Sections;
    public Dictionary<string, List<ChronicleEntry>> Entries;
    public int ProjectionVersion; // `[FULL]`
}
```

Projected cache only — safe to discard and rebuild from authoritative services on load. Persistence optional: rebuild on load preferred for invariant proof.

---

## Core algorithms

### RefreshProjection

1. Snapshot read-only inputs from registry services
2. For each section mapping, evaluate visibility rules (story flags, beat stages)
3. Build entries:
   - **Task** — from beat stage thresholds (“talk to X” when beat Active)
   - **Lead** — from thread evidence with lead role
   - **Clue** — from facts + thread evidence linkage
4. Replace in-memory projection caches
5. Notify presenter via event `ChronicleProjectionRefreshed` `[FULL]`

### Entry type selection

```text
Thread evidence (authoritative) → Lead or Clue chronicle entry (projection)
Story beat stage (authoritative) → Task chronicle entry (projection)
Fact registered (authoritative) → Clue chronicle entry if mapping exists
```

Reverse path **forbidden**: completing a Task in UI does not call services — UI triggers dialogue/world actions that mutate authority; chronicle refreshes.

### Wrong resolution path (Phase 9 test)

1. Player resolves thread subject incorrectly
2. `ThreadEngine` records `Incorrect` resolution
3. RefreshProjection shows alternate Lead entries — thread state authoritative

---

## Event contracts

| Event | Handler action |
|---|---|
| `StoryFlagChanged` | RefreshProjection |
| `StoryBeatAdvanced` | RefreshProjection |
| `FactRegistered` | RefreshProjection |
| `ThreadEvidenceAdded` | RefreshProjection |
| `ThreadSubjectResolved` | RefreshProjection |

Chronicle publishes **no** domain mutation events.

---

## Integration matrix

| Reads (read-only) | Purpose |
|---|---|
| `IStoryStateService` | Beat stages, flags for tasks |
| `IThreadService` | Evidence, resolutions for leads |
| `IFactService` | Clue visibility |
| `IContentStore` | Section mappings |

| Never calls | Reason |
|---|---|
| `IStoryStateService.SetFlag/AdvanceBeat` | CRITICAL |
| `IThreadService.AddEvidence` | CRITICAL |
| `IFactService.Register` | CRITICAL |

| Used by | Purpose |
|---|---|
| `IChroniclePresenter` | UI Phase 13 |
| Debug tools | Projection inspection |

---

## MVP scope (Phase 9)

- [ ] `ChronicleService` with Get* + RefreshProjection only
- [ ] `ChronicleProjector` with section mappings
- [ ] Entry types: Lead, Task, Clue
- [ ] Event subscriber triggers refresh
- [ ] Test: evidence add → chronicle entry appears
- [ ] Test: **no public mutator**; mock authority unchanged after GetEntries
- [ ] Test: wrong thread resolution → projected leads differ

---

## Full scope

- `[FULL]` People, locations, statistics sections
- `[FULL]` Incremental projection with version
- `[FULL]` Completed entry history archive
- `[FULL]` Belief/political diary sections

---

## File tree

```text
Packages/NarrativeFramework/Ledger/Chronicle/IChronicleService.cs
Packages/NarrativeFramework/Ledger/Chronicle/ChronicleService.cs
Packages/NarrativeFramework/Ledger/Chronicle/ChronicleProjector.cs
Packages/NarrativeFramework/Ledger/Chronicle/ChronicleSection.cs
Packages/NarrativeFramework/Ledger/Chronicle/ChronicleEntry.cs
Packages/NarrativeFramework/Ledger/Chronicle/ChronicleEventSubscriber.cs
Packages/NarrativeFramework/Content/Definitions/ChronicleSectionMapping.cs
Packages/NarrativeFramework/Tests/EditMode/Ledger/ChronicleProjectionTests.cs
Packages/NarrativeFramework/Tests/EditMode/Ledger/ChronicleReadOnlyInvariantTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Refresh_AfterEvidence_ShowsLead` | Projection content |
| `Refresh_AfterBeatActive_ShowsTask` | Beat → task mapping |
| `GetEntries_DoesNotMutateThread` | Evidence count unchanged |
| `GetEntries_DoesNotAdvanceBeat` | Beat stage unchanged |
| `WrongResolution_DifferentEntries` | Incorrect vs correct projection |
| `NoPublicMutators` | Reflection API scan |
| `FactClue_AppearsInSection` | Fact-driven clue |

**Exit:** ≥20 ledger tests combined with thread; chronicle never mutates authoritative state.

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) SIM-03; section sort from content `SortOrder`.

---

## Related documents

- [ledger-thread.md](ledger-thread.md), [story-state.md](story-state.md)
- [sim-fact.md](sim-fact.md), [story-dialogue.md](story-dialogue.md)
- [contracts.md](contracts.md)
