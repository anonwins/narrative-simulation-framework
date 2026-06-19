# Story Voice Service — Implementation Architecture

- Spec: [story-voice.md](../systems/story-voice.md)
- Glossary: [Voice](../terminology-glossary.md) · [Faculty vs Voice](../terminology-glossary.md)
- Roadmap: [Phase 11](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) · Model: [data-model.md](data-model.md)

---

## Design rationale

### Why voice is separate from dialogue

Dialogue is **what characters say** in structured conversation graphs. Voice is **how the world is narrated** — narrator, internal monologue, environmental description — often outside dialogue UI. Merging them creates unmaintainable graphs and blocks headless testing of narration logic.

### Why voice is separate from faculty

Faculties are cognitive agents with stats and rolls. Voice channels may *reference* faculty perspective (internal monologue) but `IVoiceService` does not evaluate rolls or modify beliefs. Faculty speaks through dialogue and cognition modules; voice orchestrates presentation-ready text keys.

### Why channels, not a single narrator

NSF uses layered interpretation. `VoiceChannel` enum partitions concurrent streams so Presentation can bind audio/UI per channel without collision.

### Term split

Voice output may **describe** thread or beat progress but never **author** it. Announcing “the case deepens” is locale text triggered by `StoryBeatAdvanced` events — beat advancement remains in `IStoryStateService`.

---

## Implementation summary

| MVP (Phase 11) | Full (Phase 14+) |
|---|---|
| Speak/Stop per channel | Perspective rules from conduct `[FULL]` |
| Text key + actor/faculty ID | Stacked channel priority / ducking `[FULL]` |
| Event-driven passive lines | Environmental voice tied to location `[FULL]` |
| Headless: no Unity Audio | `IAudioNarrativeService` bridge Phase 13 |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Story`
- **Namespace:** `NarrativeFramework.Story.Voice`
- **Tick phase:** **Interpretation** (passive narration hooks)

---

## Public API

See [contracts.md](contracts.md) — `IVoiceService`.

```csharp
interface IVoiceService
{
    void Speak(VoiceChannel channel, string textKey, string actorOrFacultyId);
    void Stop(VoiceChannel channel);
}

enum VoiceChannel
{
    Narrator,
    InternalMonologue,
    Environmental,
    FacultyCommentary,
    CompanionAside
}
```

---

## Internal implementation

### VoiceService

```csharp
sealed class VoiceService : IVoiceService, IStatefulService<VoiceServiceState>
{
    readonly ILocaleService _locale;
    readonly IEventBus _events;
    readonly Dictionary<VoiceChannel, ActiveVoiceLine> _active = new();

    public void Speak(VoiceChannel channel, string textKey, string actorOrFacultyId)
    {
        var line = new ActiveVoiceLine {
            Channel = channel,
            TextKey = textKey,
            SpeakerId = actorOrFacultyId,
            StartedAt = /* ITimeService.Now */
        };
        _active[channel] = line;
        _events.Publish(new SimEvent {
            Type = EventTypes.VoiceLineStarted,
            Payload = {
                ["channel"] = channel.ToString(),
                ["textKey"] = textKey,
                ["speakerId"] = actorOrFacultyId
            }
        });
    }

    public void Stop(VoiceChannel channel)
    {
        if (!_active.Remove(channel)) return;
        _events.Publish(new SimEvent {
            Type = EventTypes.VoiceLineStopped,
            Payload = { ["channel"] = channel.ToString() }
        });
    }

    public VoiceViewModel BuildViewModel(VoiceChannel channel)
    {
        if (!_active.TryGetValue(channel, out var line)) return null;
        return new VoiceViewModel {
            Text = _locale.Resolve(line.TextKey, /* locale */, null),
            SpeakerId = line.SpeakerId,
            Channel = channel
        };
    }
}
```

### VoiceEventSubscriber (internal)

Subscribes to simulation events for passive narration — does not mutate story/thread/chronicle:

```csharp
sealed class VoiceEventSubscriber : IEventHandler
{
    readonly IVoiceService _voice;
    readonly IContentStore _content;

    public void Handle(SimEvent e, SimulationSnapshot state)
    {
        if (e.Type == EventTypes.StoryBeatAdvanced)
        {
            var beatId = e.Payload["beatId"];
            if (_content.TryGetDefinition(beatId, out StoryBeatDefinition def) && def.NarrationKey != null)
                _voice.Speak(VoiceChannel.Narrator, def.NarrationKey, "narrator");
        }
    }
}
```

---

## Definition assets

| Asset | Purpose |
|---|---|
| Voice profile definitions | Tone, channel defaults `[FULL]` |
| Beat `NarrationKey` | Passive line on beat advance |
| Environmental voice triggers | Location + time rules `[FULL]` |

Text keys use `ILocaleService`; voice service stores keys only.

---

## Runtime state

```csharp
class VoiceServiceState
{
    public Dictionary<VoiceChannel, ActiveVoiceLine> ActiveLines;
}

class ActiveVoiceLine
{
    public VoiceChannel Channel;
    public string TextKey;
    public string SpeakerId;
    public GameTime StartedAt;
}
```

Ephemeral presentation state — safe to clear on load; re-trigger from story events if needed `[FULL]`.

---

## Core algorithms

### Speak

1. Resolve channel conflict policy (MVP: replace active line on same channel)
2. Store active line; publish `VoiceLineStarted`
3. Presentation/audio subscribers render — core never touches Unity

### Stop

1. Remove channel entry; publish `VoiceLineStopped`

### Passive narration (Interpretation phase)

1. Kernel runs Interpretation after Facts
2. Voice subscriber reacts to queued events from prior tick
3. Speak calls are idempotent per `(channel, textKey)` per tick `[FULL]` dedup

### Layer stacking `[FULL]`

Priority table: Dialogue UI > InternalMonologue > Narrator > Environmental. Implementation in Presentation; voice service exposes active channel map only.

---

## Event contracts

| Event | When | Subscribers |
|---|---|---|
| `VoiceLineStarted` | After Speak | UI, audio, debug |
| `VoiceLineStopped` | After Stop | UI, audio |

Voice does **not** publish domain mutations.

---

## Integration matrix

| Depends on | Reason |
|---|---|
| `ILocaleService` | Text resolution for view models |
| `IEventBus` | Passive triggers |
| `IContentStore` | Narration key lookup |

| Used by | Reason |
|---|---|
| `IDialogueService` | Optional pre/post line `[FULL]` |
| `IAudioNarrativeService` | Phase 13 audio playback |
| Presentation presenters | Headless view models |

**Forbidden:** Voice → mutation of story flags, thread evidence, or chronicle.

---

## MVP scope (Phase 11)

- [ ] `VoiceService` Speak/Stop
- [ ] `VoiceLineStarted/Stopped` events
- [ ] Beat-advance passive narration subscriber
- [ ] Headless view model builder for tests
- [ ] Tests: channel isolation, event on speak, no state mutation

---

## Full scope

- `[FULL]` Conduct-driven perspective selection
- `[FULL]` Environmental voice by location/time
- `[FULL]` Channel priority / ducking contract with audio service
- `[FULL]` Faculty commentary channel tied to cognition (read-only)

---

## File tree

```text
Packages/NarrativeFramework/Story/Voice/IVoiceService.cs
Packages/NarrativeFramework/Story/Voice/VoiceService.cs
Packages/NarrativeFramework/Story/Voice/VoiceChannel.cs
Packages/NarrativeFramework/Story/Voice/VoiceViewModel.cs
Packages/NarrativeFramework/Story/Voice/VoiceEventSubscriber.cs
Packages/NarrativeFramework/Story/Voice/VoiceServiceState.cs
Packages/NarrativeFramework/Tests/EditMode/Story/VoiceServiceTests.cs
```

---

## Test plan

| Test | Asserts |
|---|---|
| `Speak_PublishesStartedEvent` | Event payload keys |
| `Stop_ClearsChannel` | Second Speak replaces |
| `BeatAdvance_TriggersNarration` | Subscriber calls Speak |
| `Speak_DoesNotSetStoryFlag` | No story state mutation |
| `BuildViewModel_ResolvesLocale` | Mock locale text |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) A-01, A-04, A-05 (TTS in Presentation; queue; multi-channel stack Phase 11).

---

## Related documents

- [story-dialogue.md](story-dialogue.md), [story-pacing.md](story-pacing.md)
- [present-audio.md](../systems/present-audio.md)
- [contracts.md](contracts.md)
