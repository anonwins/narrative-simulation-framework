# Presentation Audio — Implementation Architecture

- Spec: [present-audio.md](../systems/present-audio.md)
- Glossary: [AudioNarrativeService](../terminology-glossary.md)
- Roadmap: [Phase 13](../development-roadmap.md)
- Contracts: [contracts.md](contracts.md) — `IAudioNarrativeService`
- UI sync: [present-ui.md](present-ui.md)

---

## Design rationale

### Why audio is a Presentation bridge

Unity audio playback (`AudioSource`, FMOD, Wwise) is engine-facing. Narrative systems emit **intent** ("play voice line X for locale Y") — not clip references. `IAudioNarrativeService` translates intent into playback while keeping Story and Cognition free of `UnityEngine.AudioClip`.

### Why stub/procedural TTS for Phase 13

External TTS API keys violate agent-executable CI. Phase 13 ships a **procedural stub**: deterministic sine/beep patterns keyed by voice line ID so tests assert `PlayVoiceLine` was called with correct IDs without hearing real speech.

### Why channels matter

Dialogue voice, ambient beds, faculty stingers, and UI feedback compete. Channel-based stop/preempt avoids bark spam drowning main dialogue — mirrors spec audio director pattern without coupling to a specific middleware.

### Why core never references Presentation

`IVoiceService` (Story) selects **which** line logically; `IAudioNarrativeService` (Presentation) plays it. Story publishes `VoiceLineRequested` or calls audio service through registry interface declared in contracts — implementation lives in Presentation assembly only.

---

## Implementation summary

| MVP (Phase 13) | Full (Phase 15+) |
|---|---|
| `IAudioNarrativeService` + stub playback | Real clip assets in samples |
| Channel map (voice, ambient, fx) | FMOD/Wwise adapter `[FULL]` |
| Voice line ID → stub tone | Lip-sync hooks `[FULL]` |
| Event-driven triggers | Spatial audio `[FULL]` |
| Headless call log | Play Mode audio smoke |

---

## Assembly and namespace

- **Assembly:** `NarrativeFramework.Presentation`
- **Namespace:** `NarrativeFramework.Presentation.Audio`
- **References:** Runtime, Content, Story, Simulation

---

## Public API

See [contracts.md](contracts.md):

```csharp
interface IAudioNarrativeService
{
    void PlayVoiceLine(string voiceLineId, string localeId);
    void PlayAmbient(string ambientId);
    void StopChannel(string channelId);
}
```

Channel constants (Presentation):

```csharp
static class AudioChannels
{
    public const string Voice = "voice";
    public const string Ambient = "ambient";
    public const string Faculty = "faculty";
    public const string Ui = "ui";
}
```

---

## Internal implementation

### AudioNarrativeService

```csharp
sealed class AudioNarrativeService : IAudioNarrativeService
{
    readonly IContentStore _content;
    readonly IAudioPlaybackAdapter _playback;
    readonly AudioCallLog _log;  // headless test sink

    public void PlayVoiceLine(string voiceLineId, string localeId)
    {
        var def = _content.TryGet<VoiceLineDefinition>(voiceLineId, out var line)
            ? line
            : VoiceLineDefinition.CreateStub(voiceLineId);

        _playback.PlayOnChannel(AudioChannels.Voice, def.ResolveClipId(localeId), def.Volume);
        _log.Record(new AudioCall(AudioCallKind.VoiceLine, voiceLineId, localeId));
    }

    public void PlayAmbient(string ambientId)
    {
        var def = _content.GetRequired<AmbientDefinition>(ambientId);
        _playback.PlayLooping(AudioChannels.Ambient, def.ClipId, def.Volume);
        _log.Record(new AudioCall(AudioCallKind.Ambient, ambientId, null));
    }

    public void StopChannel(string channelId)
    {
        _playback.StopChannel(channelId);
        _log.Record(new AudioCall(AudioCallKind.Stop, channelId, null));
    }
}
```

### IAudioPlaybackAdapter

Abstraction over Unity audio vs stub:

```csharp
interface IAudioPlaybackAdapter
{
    void PlayOnChannel(string channel, string clipId, float volume);
    void PlayLooping(string channel, string clipId, float volume);
    void StopChannel(string channel);
}

sealed class StubAudioPlaybackAdapter : IAudioPlaybackAdapter
{
    public readonly Dictionary<string, string> ActiveClips = new();

    public void PlayOnChannel(string channel, string clipId, float volume)
        => ActiveClips[channel] = clipId;

    public void PlayLooping(string channel, string clipId, float volume)
        => ActiveClips[channel] = clipId;

    public void StopChannel(string channel)
        => ActiveClips.Remove(channel);
}
```

Samples provide `UnityAudioPlaybackAdapter` with `AudioSource` pool.

### AudioNarrativeCoordinator

Subscribes to narrative events:

```csharp
sealed class AudioNarrativeCoordinator
{
    public void OnDialogueAdvanced(DialogueNode node)
    {
        if (!string.IsNullOrEmpty(node.VoiceLineId))
            _audio.PlayVoiceLine(node.VoiceLineId, _locale.GetActiveLocale());
    }

    public void OnLocationEntered(string locationId)
    {
        var loc = _content.GetRequired<LocationNodeDefinition>(locationId);
        if (!string.IsNullOrEmpty(loc.DefaultAmbientId))
            _audio.PlayAmbient(loc.DefaultAmbientId);
    }

    public void OnRollResolved(RollResult result)
    {
        if (result.FacultyInterjectionId != null)
            _audio.PlayVoiceLine(result.FacultyInterjectionId, _locale.GetActiveLocale());
    }
}
```

Coordinator lives in Presentation — Story only emits events.

### Headless AudioCallLog

```csharp
sealed class AudioCallLog
{
    readonly List<AudioCall> _calls = new();
    public IReadOnlyList<AudioCall> Calls => _calls;
    public void Record(AudioCall call) => _calls.Add(call);
    public void Clear() => _calls.Clear();
}
```

Tests assert call sequence without speaker hardware.

### Faculty and bark hooks

`VoiceBarkDefinition` with cooldown + priority evaluated in Presentation when subscribing to world events. Competing barks: lower priority dropped if voice channel busy — MVP: queue or drop policy in `AudioNarrativeService` config.

---

## Definition assets

Content pack owned:

```csharp
class VoiceLineDefinition : ContentDefinition
{
    public string SpeakerId;
    public string ClipId;
    public string SubtitleKey;
    public LocaleClipOverride[] LocaleOverrides;
}

class AmbientDefinition : ContentDefinition
{
    public string ClipId;
    public float Volume;
    public bool Loop;
}

class VoiceBarkDefinition : ContentDefinition
{
    public string TriggerEventId;
    public int Priority;
    public float CooldownSeconds;
    public string VoiceLineId;
}
```

`FrameworkTestPack`: `vo_test_greeting`, `ambient_test_office`.

---

## Runtime state

| State | Owner | Persisted |
|---|---|---|
| Active channel clips | Playback adapter | No |
| Bark cooldown timers | Audio coordinator | No |
| Call log | Test adapter | No |

No simulation truth in audio layer.

---

## Core algorithms

### Play voice line

1. Resolve `VoiceLineDefinition` (fallback stub if missing — test-friendly)
2. Pick clip for locale via `LocaleClipOverride` or default
3. Stop current voice channel if interruptible
4. Play via adapter
5. Log call; optionally notify dialogue presenter for subtitle timing `[FULL]`

### Ambient on location enter

1. `LocationEnteredEvent` received
2. Stop previous ambient unless crossfade `[FULL]`
3. Start looping ambient for new location
4. Log

### Stop channel

Used on dialogue skip, location exit, menu open — Presentation policy; core unaffected.

---

## Event contracts

| Event | Audio reaction |
|---|---|
| `DialogueAdvanced` | Voice line on node |
| `LocationEntered` | Ambient bed |
| `LocationExited` | Stop ambient |
| `RollResolved` | Faculty interjection |
| `InteractionExecuted` | World bark lookup |

Audio does not publish simulation events except optional `AudioPlaybackStarted` for debug `[FULL]`.

---

## Integration matrix

| Depends on | Usage |
|---|---|
| `IContentStore` | Line + ambient defs |
| `ILocaleService` | Locale clip selection |
| `IEventBus` | Triggers |
| Dialogue nodes (via events) | VoiceLineId |

| Consumed by | Usage |
|---|---|
| Samples | Real AudioSource wiring |
| `HeadlessDialoguePresenterTests` | Parallel subtitle-only path |
| Debug | Audio call trace |

**Forbidden:** Story → `AudioClip`; Cognition → Unity audio types.

---

## MVP scope (Phase 13)

- [ ] `IAudioNarrativeService`, `AudioNarrativeService`
- [ ] `StubAudioPlaybackAdapter` + `AudioCallLog`
- [ ] `AudioNarrativeCoordinator` core subscriptions
- [ ] `VoiceLineDefinition`, `AmbientDefinition` in test pack
- [ ] Channel stop/preempt for voice
- [ ] Headless tests assert call log order
- [ ] No external API keys

---

## Full scope

- `[FULL]` Unity `AudioSource` pool adapter
- `[FULL]` Bark cooldown priority queue
- `[FULL]` Lip-sync timing export to UI presenter
- `[FULL]` Crossfade ambient transitions
- Default TTS via `ITextToSpeechProvider` (A-01, A-07) — bundled neutral voice profiles

---

## File tree

```text
Packages/NarrativeFramework/Presentation/Audio/IAudioNarrativeService.cs
Packages/NarrativeFramework/Presentation/Audio/AudioNarrativeService.cs
Packages/NarrativeFramework/Presentation/Audio/AudioNarrativeCoordinator.cs
Packages/NarrativeFramework/Presentation/Audio/IAudioPlaybackAdapter.cs
Packages/NarrativeFramework/Presentation/Audio/StubAudioPlaybackAdapter.cs
Packages/NarrativeFramework/Presentation/Audio/AudioCallLog.cs
Packages/NarrativeFramework/Presentation/Audio/AudioChannels.cs
Packages/NarrativeFramework/Content/Definitions/VoiceLineDefinition.cs
Packages/NarrativeFramework/Content/Definitions/AmbientDefinition.cs
Packages/NarrativeFramework/Tests/EditMode/Presentation/AudioNarrativeServiceTests.cs
Samples~/FrameworkTestPack/Audio/vo_test_greeting.asset
```

---

## Test plan

| Test | Asserts |
|---|---|
| `PlayVoiceLine_LogsCall` | voiceLineId + locale |
| `PlayAmbient_LoopsOnChannel` | Active clip on ambient |
| `StopChannel_ClearsVoice` | Channel empty |
| `DialogueEvent_TriggersVoice` | Coordinator wiring |
| `MissingDefinition_UsesStub` | No throw |
| `Core_NoAudioClipReference` | Asmdef/content scan |

---

## Deferred decisions

**Locked** — [decisions-log.md](../decisions-log.md) A-01, A-04, A-07.

---

## Related documents

- [present-ui.md](present-ui.md) — subtitle sync `[FULL]`
- [present-exploration.md](present-exploration.md) — location ambient
- [story-voice.md](story-voice.md) — logical voice selection
- [integration.md](integration.md) — audio in full loop
