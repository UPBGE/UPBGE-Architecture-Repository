# UPBGE MIDI System Architecture

## Overview

This document describes the architecture for adding MIDI support to UPBGE. The system provides both MIDI file playback and real-time procedural MIDI generation, exposed through Python API and Logic Bricks. A single `KX_MidiManager` instance per scene acts as the central hub, ensuring a single synthesis pipeline and a single Audaspace callback.

---

## Directory Structure

```
[blender_root]/
├── extern/
│   ├── tml/
│   │   └── tml.h                        # MIDI parser (header-only, MIT)
│   └── tsf/
│       └── tsf.h                        # SF2 synthesizer (header-only, MIT)
│
├── datafiles/
│   ├── gamecontrollerdb/                # Existing
│   │   └── gamecontrollerdb.txt
│   └── midifiles/                       # NEW
│       └── default.sf2                  # General MIDI SoundFont (bundled)
│
├── source/blender/makesdna/DNA_scene_types.h   # Modified (GameData)
├── source/blender/makesrna/intern/rna_scene.c  # Modified (RNA exposure)
│
└── source/gameengine/                   # Main work area
    ├── Ketsji/
    │   ├── KX_MidiManager.h
    │   ├── KX_MidiManager.cpp
    │   ├── KX_Scene.h                   # Add KX_MidiManager* member
    │   └── KX_Scene.cpp                 # Init/shutdown MidiManager
    │
    ├── GameLogic/
    │   ├── Actuators/
    │   │   ├── KX_MidiActuator.h
    │   │   └── KX_MidiActuator.cpp
    │   └── Sensors/
    │       ├── KX_MidiEventSensor.h
    │       └── KX_MidiEventSensor.cpp
    │
    └── PyAPI/
        └── KX_PythonMidi.cpp            # bge.sound.midi* bindings
```

---

## Blender DNA Changes (Minimal)

Only `GameData` (already inside `Scene`) is extended. No new DNA blocks are created.

**`source/blender/makesdna/DNA_scene_types.h`**

```c
typedef struct GameData {
    /* ... existing fields ... */

    /* MIDI Settings */
    char midi_soundfont[256];   /* Path to .sf2 file. Empty = use bundled default.sf2 */
    int  midi_max_voices;       /* Polyphony limit. Default: 256 */
    int  midi_sample_rate;      /* 44100 or 48000 */
    int  midi_default_tempo;    /* BPM. Default: 120 */
    short midi_enabled;         /* 0 = disabled, 1 = enabled */
    short midi_pad;             /* padding */
} GameData;
```

**`source/blender/makesrna/intern/rna_scene.c`** — RNA exposure for the UI panel.

---

## Blender Editor Panel

Added as a new panel inside **Properties → Scene** (same area as Physics, Logic, etc.).

```
Scene Properties
  ├── Scene
  ├── World
  ├── Physics
  ├── Navigation
  └── MIDI                              ← New panel
        [ x ] Enable MIDI
        SoundFont  [ default.sf2  ] 📁  ← empty = bundled default
        Max Voices [ 256 ]
        Sample Rate [ ● 44100  ○ 48000 ]
        Default BPM [ 120 ]
```

The panel is registered in `scripts/startup/bl_ui/properties_scene.py` (or the UPBGE equivalent), touching Blender UI minimally.

---

## Runtime Architecture

### Initialization Flow

```
KX_Scene::Start()
  └─ if (scene->gm.midi_enabled)
       └─ m_midiManager = new KX_MidiManager(scene->gm)
               ├─ Resolve soundfont path
               │     └─ empty path → load bundled datafiles/midifiles/default.sf2
               ├─ tsf_load_filename(soundfont_path)   // load SF2
               ├─ tsf_set_output(TSF_STEREO_INTERLEAVED, sample_rate, 0, -10.0f)
               └─ Register AUD_MidiSound callback with Audaspace

KX_Scene::End()
  └─ delete m_midiManager
        ├─ AUD_removeMidiSound()
        └─ tsf_close(m_synth)
```

### Audio Thread / Game Logic Thread Communication

Audaspace runs on a **dedicated audio thread**. All game logic (Actuators, Python API) runs on the **main game thread**. Communication between them uses a **lock-free SPSC ring buffer** (Single Producer Single Consumer) to avoid mutex overhead in the hot audio path.

```
Game Logic Thread                     Audio Thread (Audaspace callback)
─────────────────                     ─────────────────────────────────
MidiActuator::Update()                KX_MidiManager::FillBuffer()
  └─ noteOn(ch, note, vel)              └─ drain event ring buffer
       └─ push MidiEvent                     └─ tsf_note_on / tsf_note_off
            → LockFreeRingBuffer             └─ advance file sequencer
                                             └─ tsf_render_float → PCM out
```

**`MidiEvent` struct (cache-line friendly, 8 bytes):**

```cpp
struct MidiEvent {
    uint8_t  type;       // NOTE_ON, NOTE_OFF, PROGRAM_CHANGE, TEMPO, ALL_NOTES_OFF
    uint8_t  channel;
    uint8_t  note;
    uint8_t  velocity;
    uint32_t reserved;   // future use / padding to 8 bytes
};
```

---

## KX_MidiManager

Central singleton owned by `KX_Scene`. Responsible for synthesis, sequencing, and thread-safe event dispatch.

```cpp
class KX_MidiManager {
public:
    KX_MidiManager(const GameData& gm);
    ~KX_MidiManager();

    // Called from game logic thread (Actuators / Python)
    void NoteOn(int channel, int note, int velocity);
    void NoteOff(int channel, int note);
    void SetInstrument(int channel, int program);
    void SetTempo(int bpm);
    void AllNotesOff();

    // MIDI file playback
    bool LoadFile(const char* path);
    void Play();
    void Stop();
    void Pause();
    void SetPlaybackSpeed(float speed);   // tempo multiplier

    // Called from audio thread ONLY (Audaspace callback)
    void FillBuffer(float* buffer, int samples);

    // Event subscription for MidiEventSensor
    void SubscribeSensor(KX_MidiEventSensor* sensor);
    void UnsubscribeSensor(KX_MidiEventSensor* sensor);

private:
    tsf*            m_synth;
    tml_message*    m_midiFile;          // null if no file loaded
    double          m_currentTime;       // ms, audio thread only
    float           m_playbackSpeed;

    LockFreeRingBuffer<MidiEvent, 1024> m_eventQueue;  // game→audio
    LockFreeRingBuffer<MidiEvent, 256>  m_sensorQueue; // audio→sensors

    std::vector<KX_MidiEventSensor*> m_sensors;        // read from main thread
    std::atomic<bool> m_playing;
    std::atomic<bool> m_paused;
    int m_sampleRate;
};
```

---

## Audaspace Integration

A custom Audaspace `ISound` / factory generates the PCM stream by delegating to `KX_MidiManager::FillBuffer()`. This keeps the MIDI system decoupled from Audaspace internals.

```
Audaspace
  └─ AUD_Sound (custom MidiSound factory)
        └─ MidiReader : IReader
              └─ read(samples) → KX_MidiManager::FillBuffer()
```

No Audaspace source files are modified. The integration uses the existing Audaspace plugin/factory API that UPBGE already exposes.

---

## Logic Bricks

### MidiActuator

Analogous to the existing `SoundActuator`. Handles both file playback and real-time note generation.

| Mode | Parameters |
|---|---|
| `PLAY_FILE` | File path, loop, playback speed |
| `STOP` | — |
| `NOTE_ON` | Channel, Note (or Property), Velocity (or Property) |
| `NOTE_OFF` | Channel, Note (or Property) |
| `SET_INSTRUMENT` | Channel, Program (0–127) |
| `SET_TEMPO` | BPM (or Property) |
| `ALL_NOTES_OFF` | — |

Dynamic values (e.g. procedural note generation) can be driven by **Game Object Properties**, consistent with existing UPBGE actuator patterns.

```
[ MidiActuator ]
  Mode:      [ Note On        ▼ ]
  Channel:   [ 0  ]
  Note:      [ Property: "midi_note"     ]
  Velocity:  [ Property: "midi_velocity" ]
```

### MidiEventSensor

Fires a positive pulse when a matching MIDI event occurs during file playback. Enables gameplay synchronization (Guitar Hero style) without Python.

```
[ MidiEventSensor ]
  Source file:  [ track.mid  ] 📁   (optional — monitors global playback if empty)
  Event type:   [ Note On    ▼ ]   Note On | Note Off | Beat | Bar | Program Change
  Channel:      [ Any        ▼ ]
  Note filter:  [ Any        ▼ ]   or specific MIDI note 0–127
```

The sensor subscribes to `KX_MidiManager` on scene start. Events generated in the audio thread are forwarded via `m_sensorQueue` (lock-free) and consumed on the main logic tick, keeping sensor evaluation on the correct thread.

---

## Python API

Accessed via `bge.sound` namespace, consistent with existing UPBGE audio API.

```python
import bge

# Access the scene MIDI manager
midi = bge.logic.getCurrentScene().midi  # KX_MidiManager wrapper

# File playback
midi.load("//music/track.mid")
midi.play()
midi.stop()
midi.pause()
midi.playback_speed = 1.5

# Real-time generation
midi.note_on(channel=0, note=60, velocity=100)   # Middle C
midi.note_off(channel=0, note=60)
midi.set_instrument(channel=0, program=40)        # Violin
midi.set_tempo(140)                               # BPM
midi.all_notes_off()

# Query state
print(midi.is_playing)     # bool
print(midi.current_time)   # float, ms

# Event callback (alternative to MidiEventSensor)
def on_note(event):
    print(event.type, event.channel, event.note, event.velocity)

midi.set_event_callback(on_note)
```

---

## File Playback Sequencer

When a `.mid` file is loaded, `tml.h` parses it into a linked list of `tml_message` events. The audio thread advances `m_currentTime` by `(samples / sample_rate) * 1000 * playback_speed` on each `FillBuffer()` call and dispatches all events whose `time <= m_currentTime`.

```
FillBuffer(buffer, num_samples):
  time_advance_ms = (num_samples / m_sampleRate) * 1000.0 * m_playbackSpeed
  m_currentTime += time_advance_ms

  while (m_midiFile && m_midiFile->time <= m_currentTime):
      dispatch(m_midiFile)          // tsf_note_on / tsf_note_off / etc.
      push to m_sensorQueue         // notify MidiEventSensors
      m_midiFile = m_midiFile->next

  // drain game-logic events from ring buffer
  while (m_eventQueue.pop(ev)):
      apply(ev)

  tsf_render_float(m_synth, buffer, num_samples, 0)
```

---

## Bundled SoundFont

`datafiles/midifiles/default.sf2` is a compact General MIDI SoundFont (target: < 5 MB, e.g. GeneralUser GS or MuseScore's bundled SF3 converted). It is installed alongside the engine data and used automatically when the scene's SoundFont path is left empty.

The path resolution at runtime:

```cpp
const char* KX_MidiManager::ResolveSoundFont(const char* configured_path) {
    if (configured_path && configured_path[0] != '\0')
        return configured_path;  // user-specified

    // Fall back to bundled default
    return BKE_appdir_folder_id(BLENDER_DATAFILES, "midifiles/default.sf2");
}
```

---

## Implementation Phases

| Phase | Scope | Touches Blender? |
|---|---|---|
| 1 | Add `tml.h` + `tsf.h` to `/extern`, CMake wiring | Minimal (extern CMake only) |
| 2 | `KX_MidiManager` + Audaspace callback + lock-free queue | No |
| 3 | `DNA_scene_types.h` GameData extension + RNA + UI panel | Yes (minimal) |
| 4 | `MidiActuator` + `MidiEventSensor` Logic Bricks | No |
| 5 | Python API bindings | No |
| 6 | Bundle default.sf2, CMake install rule | Minimal |

---

## Dependencies Summary

| Library | Location | License | Size | Integration |
|---|---|---|---|---|
| `tml.h` | `extern/tml/tml.h` | MIT | ~600 lines | Header-only |
| `tsf.h` | `extern/tsf/tsf.h` | MIT | ~2500 lines | Header-only |
| `default.sf2` | `datafiles/midifiles/` | varies (permissive) | < 5 MB | Data file |

No new runtime shared library dependencies are introduced.
