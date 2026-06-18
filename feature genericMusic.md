# Generic Music

## Overview

Generic Music is a feature that enables Haven lighting controllers to choreograph lighting effects in real time to a song that is currently playing. The goal is to give users the sense of a live, music-driven light show — a party atmosphere — without requiring any deep analysis of the audio itself.

This feature represents the lightweight tier of music synchronization. It operates on a small set of generic song metadata rather than a fully studied, beat-mapped playlist file. A more advanced tier (playlist-based synchronization) exists separately and involves pre-analyzed song data downloaded to the controller ahead of playback.

All lighting controllers participating in a Generic Music session share synchronized UTC clocks, allowing a single broadcast of song metadata to drive coordinated, time-accurate effects across every device simultaneously.

---

## Section 1: Data Model and Runtime Behavior

### 1.1 `GenericMusic` Input Schema

When a song begins playing, the song server broadcasts a `GenericMusic` payload to all listening controllers. The payload has the following fields:

| Field | Type | Description |
|---|---|---|
| `duration` | `float` | Total length of the song in seconds (e.g., `133.0`) |
| `bpm` | `float` | Beats per minute of the song (e.g., `122.0`) |
| `transitions` | `float[]` | Timestamps in seconds where the song transitions between sections (e.g., `[15.0, 17.0, 202.0, 456.0]`) |
| `mood` | `float` | A 0–100 value representing song intensity. `0` = calm, `100` = aggressive. Controls the amplitude of brightness bumps in the lighting effect. |
| `colors` | `string[]` | A palette of color names to be used in effect calculations (e.g., `["red", "amber", "white"]`) |
| `epochStartTime` | `int` | UTC epoch time in milliseconds representing the exact moment the stereo began playing the song |

### 1.2 Song Sections

The `transitions` array divides the song into discrete sections. For a transitions array of `[15.0, 17.0, 202.0, 456.0]`, the controller derives the following sections:

| Section | Start | End |
|---|---|---|
| 1 | `0.0s` | `15.0s` |
| 2 | `15.0s` | `17.0s` |
| 3 | `17.0s` | `202.0s` |
| 4 | `202.0s` | `456.0s` |
| 5 | `456.0s` | end of song |

At each transition boundary, the controller **snaps immediately** to a new lighting effect. There is no blend or crossfade between sections.

### 1.3 Position Calculation

Because all controllers share synchronized UTC clocks and are given the same `epochStartTime`, each controller independently calculates its current position in the song at any moment:

```
currentPosition (seconds) = (nowUTC_ms - epochStartTime) / 1000.0
```

The controller uses `currentPosition` to determine which section it is in and which effect to apply.

### 1.4 End-of-Song Behavior

The controller may not always receive explicit notification that a song has ended. When `currentPosition` exceeds `duration`, or when no new `GenericMusic` command has been received, the controller transitions to a **calm idle state**:

- A gentle, low-intensity wavy lighting effect is played indefinitely.
- This state signals clearly that the system is in a wait condition between songs.
- The idle state persists until a new `GenericMusic` payload is received.

The idle state should be visually calm and unobtrusive — it must not resemble an active lighting effect or appear jarring.

---

## Section 2: Effect Calculations

*This section is reserved for future definition.*

The controller will maintain a library of approximately ten distinct lighting effects. Each effect will be defined by an equation that takes the following inputs:

- `bpm` — to synchronize effect timing to the beat
- `mood` (0–100) — to scale brightness bump amplitude
- `colors` — the active palette to render the effect with

During song playback, the controller rotates through the available effects at each transition point, applying a different effect per section.

The specific equations governing each lighting effect will be defined in a future revision of this document.

---

*Open items: Effect equation definitions (Section 2) are pending authoring.*