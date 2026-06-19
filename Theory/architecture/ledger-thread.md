# Ledger Thread Service — Implementation Architecture

- Spec: [ledger-thread.md](../systems/ledger-thread.md)
- Glossary: [Thread](../terminology-glossary.md) · [Evidence / Theory](../terminology-glossary.md)
- Roadmap: [Phase 9](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Chronicle: [ledger-chronicle.md](ledger-chronicle.md)

---

## Design rationale

### Why thread owns investigation logic

The player builds an explanation of reality — evidence, hypotheses, theories, subject resolution. That is **`ThreadEngine`** domain, not story beats (progression switches) and not chronicle entries (projected journal lines). `IThreadService` is the registry-facing facade; `ThreadEngine` evaluates resolution against facts.

### Why evidence lives in thread, not chronicle

**Thread evidence** is authoritative reasoning input. **Chronicle clues** are read-only descriptions of that evidence for the player. Adding evidence always goes through `IThreadService.AddEvidence`; chronicle updates via `RefreshProjection`.

### Why thread resolution is not beat completion

Resolving `subject_suspect` as `Correct` or `Incorrect` is thread state. A related **story beat** may simultaneously advance to `Complete` or `AltResolved` via rule actions — coordinated in content, not conflated in code.

### Term split summary

```text
Thread (thread_*)           — authoritative inquiry container
  ├─ ThreadEvidence          — reasoning objects
  ├─ ThreadTheory            — hypothesis graph
  └─ ThreadSubject           — resolution targets

Story beat (beat_*)         — IStoryStateService progression

Chronicle section (section_*) — projected journal grouping
  └─ Lead / Task / Clue      — entry types, not thread objects
```

---

## Implementation summary

| MVP (Phase 9) | Full (Phase 14+) |
|---|---|
| AddEvidence, AddTheory | Theory contradiction graph `[FULL]` |
| TryResolveSubject | Multi-subject dependency chains `[FULL]` |
| ThreadEngine.Evaluate | Automated theory suggestions `[FULL]` |
| Fact view integration | Info-flow weighted evidence `[FULL]` |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Ledger`
- **Namespace:** `NarrativeFramework.Ledger.Thread`
- **Tick phase:** Mutations during **Content**; evaluation during **Gating** or on demand

---

## Public API

See [contracts.md](contracts.md) — `IThreadService`, `ThreadEngine`.

```csharp
interface IThreadService
{
    void AddEvidence(string threadId, ThreadEvidence evidence);
    void AddTheory(string threadId, ThreadTheory theory);
    bool TryResolveSubject(string threadId, string subjectId, out ThreadResolution resolution);
    IReadOnlyList<ThreadEvidence> GetEvidence(string threadId); // query for chronicle `[FULL]` on interface
}

class ThreadEngine
{
    public ThreadState Evaluate(string threadId, IReadOnlyFactView facts);
}
```

MVP may add `GetEvidence` as explicit query helper on service for chronicle projector — not on chronicle service.

---

## Internal implementation

### ThreadService

```csharp
sealed class ThreadService : IThreadService, IStatefulService<ThreadServiceState>
{
    readonly ThreadRegistry _threads = new();
    readonly ThreadEngine _engine;
    readonly IFactService _facts;
    readonly IEventBus _events;

    public void AddEvidence(string threadId, ThreadEvidence evidence)
    {
        ContentIdValidator.IsValid(evidence.Id, "evidence_");
        var thread = _threads.GetOrCreate(threadId);
        thread.Evidence.Add(evidence);
        _events.Publish(new SimEvent {
            Type = EventTypes.ThreadEvidenceAdded,
            Payload = { ["threadId"] = threadId, ["evidenceId"] = evidence.Id }
        });
    }

    public void AddTheory(string threadId, ThreadTheory theory)
    {
        var thread = _threads.GetOrCreate(threadId);
        thread.Theories.Add(theory);
        _events.Publish(new SimEvent {
            Type = EventTypes.ThreadTheoryAdded,
            Payload = { ["threadId"] = threadId, ["theoryId"] = theory.Id }
        });
    }

    public bool TryResolveSubject(string threadId, string subjectId, out ThreadResolution resolution)
    {
        resolution = default;
        var thread = _threads.GetOrCreate(threadId);
        var facts = new ReadOnlyFactView(_facts);
        var state = _engine.Evaluate(threadId, facts);
        if (!state.CanResolve(subjectId, thread, out resolution))
            return false;

        thread.SubjectResolutions[subjectId] = resolution;
        _events.Publish(new SimEvent {
            Type = EventTypes.ThreadSubjectResolved,
            Payload = {
                ["threadId"] = threadId,
                ["subjectId"] = subjectId,
                ["kind"] = resolution.Kind.ToString()
            }
        });
        return true;
    }

    public IReadOnlyList<ThreadEvidence> GetEvidence(string threadId)
        => _threads.GetOrCreate(threadId).Evidence;
}
```

### ThreadEngine

```csharp
class ThreadEngine
{
    readonly IContentStore _content;

    public ThreadState Evaluate(string threadId, IReadOnlyFactView facts)
    {
        var def = _content.GetDefinition<ThreadDefinition>(threadId);
        var state = new ThreadState { ThreadId = threadId };

        foreach (var subject in def.Subjects)
        {
            state.SubjectEvaluations[subject.Id] = EvaluateSubject(
                subject, facts, def);
        }
        return state;
    }

    SubjectEvaluation EvaluateSubject(
        ThreadSubjectDefinition subject,
        IReadOnlyFactView facts,
        ThreadDefinition threadDef)
    {
        var supporting = subject.RequiredFactIds.Count(facts.IsActive);
        var contradictions = subject.ContradictingFactIds.Count(facts.IsActive);
        return new SubjectEvaluation {
            SubjectId = subject.Id,
            Confidence = supporting - contradictions,
            CanResolveCorrect = supporting >= subject.MinSupportingFacts
                && contradictions == 0,
            CanResolveIncorrect = contradictions > 0 && supporting == 0
        };
    }
}
```

### ThreadRegistry (internal)

```csharp
sealed class ThreadRegistry
{
    readonly Dictionary<string, ThreadRuntime> _threads = new();
    public ThreadRuntime GetOrCreate(string id) =>
        _threads.TryGetValue(id, out var t) ? t : (_threads[id] = new ThreadRuntime { ThreadId = id });
}
```

---

## Definition assets

| Asset | ID prefix |
|---|---|
| `ThreadDefinition` | `thread_*` |
| `ThreadSubjectDefinition` | `subject_*` |
| `ThreadEvidence` (runtime) | `evidence_*` |
| `ThreadTheory` (runtime) | `theory_*` |

Content links evidence items to fact IDs and subject hypotheses — not to `section_*` or `beat_*` directly.

---

## Runtime state

```csharp
class ThreadServiceState
{
    public Dictionary<string, ThreadRuntime> Threads;
}

class ThreadRuntime
{
    public string ThreadId;
    public List<ThreadEvidence> Evidence;
    public List<ThreadTheory> Theories;
    public Dictionary<string, ThreadResolution> SubjectResolutions;
}

class ThreadResolution
{
    public ThreadResolutionKind Kind; // Correct, Incorrect, Inconclusive
    public GameTime ResolvedAt;
    public List<string> SupportingEvidenceIds;
}

enum ThreadResolutionKind { Correct, Incorrect, Inconclusive }
```

---

## Core algorithms

### Add evidence

1. Validate IDs
2. Append to thread runtime list
3. Publish `ThreadEvidenceAdded`
4. Chronicle subscriber refreshes — thread does not write chronicle

### Add theory

1. Store theory node (links evidence IDs, target subject)
2. Publish `ThreadTheoryAdded`
3. `[FULL]` evaluate contradictions in theory graph

### TryResolveSubject

1. `ThreadEngine.Evaluate` with current fact view
2. Check subject evaluation + player-selected theory (dialogue action supplies theory ID)
3. Write `SubjectResolutions[subjectId]`
4. Publish `ThreadSubjectResolved`
5. Optional rule actions (via dialogue) set story flags — not automatic in thread service

### Evaluate (read-only)

1. Load thread definition
2. For each subject, score facts against required/contradicting sets
3. Return `ThreadState` for UI hints and gate conditions — no mutation

### Wrong resolution path

Player selects incorrect theory in dialogue → `TryResolveSubject` returns true with `Kind = Incorrect` → story beat may still `AdvanceBeat` to `AltResolved` via separate rule action.

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `ThreadEvidenceAdded` | After AddEvidence | Chronicle, debug |
| `ThreadTheoryAdded` | After AddTheory | Chronicle `[FULL]` |
| `ThreadSubjectResolved` | After successful resolve | Story rules, outcome, chronicle |

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `IFactService` | Resolution ground truth |
| `IContentStore` | Thread/subject definitions |
| `IEventBus` | Downstream projection |
| `ITimeService` | Resolution timestamps |

| Used by | Reason |
|---|---|
| `IRuleEngine` | AddEvidence action, thread gates |
| `IDialogueService` | Accusation / theory choices |
| `IChronicleService` | Read-only projection |
| `IOutcomeService` | Ending weights Phase 11 |
| `IGateService` | Thread resolution gates |

| Does not use | Reason |
|---|---|
| `IChronicleService` writes | No such API |

---

## MVP scope (Phase 9)

- [ ] `ThreadService` + `ThreadRegistry`
- [ ] `ThreadEngine.Evaluate` with fact-based subject scoring
- [ ] AddEvidence, AddTheory, TryResolveSubject
- [ ] Events for evidence and resolution
- [ ] Chronicle integration test (projection only)
- [ ] Wrong resolution path test

---

## Full scope

- `[FULL]` Theory graph contradictions
- `[FULL]` Info-flow weighted evidence
- `[FULL]` Multi-thread cross-subject dependencies
- `[FULL]` GetResolution query on interface

---

## File tree

```text
Packages/NarrativeFramework/Ledger/Thread/IThreadService.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadService.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadEngine.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadRegistry.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadRuntime.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadEvidence.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadTheory.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadResolution.cs
Packages/NarrativeFramework/Ledger/Thread/ThreadServiceState.cs
Packages/NarrativeFramework/Ledger/Thread/IReadOnlyFactView.cs
Packages/NarrativeFramework/Content/Definitions/ThreadDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Ledger/ThreadEvidenceTests.cs
Packages/NarrativeFramework/Tests/EditMode/Ledger/ThreadResolutionTests.cs
Packages/NarrativeFramework/Tests/EditMode/Ledger/ThreadChronicleIntegrationTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `AddEvidence_PublishesEvent` | Event payload |
| `Evaluate_RequiresFacts` | Cannot resolve without facts |
| `TryResolve_Correct` | Kind Correct |
| `TryResolve_Incorrect` | Wrong theory path |
| `Resolve_DoesNotWriteChronicle` | Chronicle only RefreshProjection |
| `Evidence_NotStoryBeat` | AddEvidence does not advance beat |
| `Gate_ThreadResolved` | Phase 5+ gate integration |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) thread UX via dialogue; inconclusive resolution allowed; duplicate evidence IDs + log.

---

## Related documents

- [ledger-chronicle.md](ledger-chronicle.md), [story-state.md](story-state.md)
- [rules-engine.md](rules-engine.md), [story-outcome.md](story-outcome.md)
- [sim-fact.md](sim-fact.md)
- [contracts.md](contracts.md)
