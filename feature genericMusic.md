# Generic Music

| | |
|---|---|
| **Version** | 0.2 |
| **Status** | Draft |
| **Last updated** | 2026-06-18 |
| **Owner** | Jonathan, Haven Lighting |
| **Target / scope** | Haven lighting controllers — lightweight music sync tier |
| **Classification** | Internal |

---

## 1. Overview

Generic Music is a feature that enables Haven lighting controllers to choreograph lighting effects in real time to a song that is currently playing. The goal is to give users the sense of a live, music-driven light show — a party atmosphere — without requiring any deep analysis of the audio itself.

This feature represents the lightweight tier of music synchronization. It operates on a small set of generic song metadata rather than a fully studied, beat-mapped playlist file. A more advanced tier (playlist-based synchronization) exists separately and involves pre-analyzed song data downloaded to the controller ahead of playback.

All lighting controllers participating in a Generic Music session share synchronized UTC clocks, allowing a single broadcast of song metadata to drive coordinated, time-accurate effects across every device simultaneously.

## 2. Goals & Non-Goals

- **Goals** — drive coordinated, time-accurate lighting from lightweight broadcast song metadata using synchronized UTC clocks, with graceful idle behavior between songs.
- **Non-Goals** — playlist-based synchronization with pre-analyzed beat-mapped data (separate tier); on-device audio analysis; the effect equations themselves (deferred, see §4).

---

## 3. Data Model and Runtime Behavior

### 3.1 `GenericMusic` Input Schema

When a song begins playing, the song server broadcasts a `GenericMusic` payload to all listening controllers. The payload has the following fields:

| Field | Type | Description |
|---|---|---|
| `duration` | `float` | Total length of the song in seconds (e.g., `133.0`) |
| `bpm` | `float` | Beats per minute of the song (e.g., `122.0`) |
| `transitions` | `float[]` | Timestamps in seconds where the song transitions between sections (e.g., `[15.0, 17.0, 202.0, 456.0]`) |
| `mood` | `float` | A 0–100 value representing song intensity. `0` = calm, `100` = aggressive. Controls the amplitude of brightness bumps in the lighting effect. |
| `colors` | `string[]` | A palette of color names to be used in effect calculations (e.g., `["red", "amber", "white"]`) |
| `epochStartTime` | `int` | UTC epoch time in milliseconds representing the exact moment the stereo began playing the song |

### 3.2 Song Sections

The `transitions` array divides the song into discrete sections. For a transitions array of `[15.0, 17.0, 202.0, 456.0]`, the controller derives the following sections:

| Section | Start | End |
|---|---|---|
| 1 | `0.0s` | `15.0s` |
| 2 | `15.0s` | `17.0s` |
| 3 | `17.0s` | `202.0s` |
| 4 | `202.0s` | `456.0s` |
| 5 | `456.0s` | end of song |

At each transition boundary, the controller **snaps immediately** to a new lighting effect. There is no blend or crossfade between sections.

### 3.3 Position Calculation

Because all controllers share synchronized UTC clocks and are given the same `epochStartTime`, each controller independently calculates its current position in the song at any moment:

```
currentPosition (seconds) = (nowUTC_ms - epochStartTime) / 1000.0
```

The controller uses `currentPosition` to determine which section it is in and which effect to apply.

### 3.4 End-of-Song Behavior

The controller may not always receive explicit notification that a song has ended. When `currentPosition` exceeds `duration`, or when no new `GenericMusic` command has been received, the controller transitions to a **calm idle state**:

- A gentle, low-intensity wavy lighting effect is played indefinitely.
- This state signals clearly that the system is in a wait condition between songs.
- The idle state persists until a new `GenericMusic` payload is received.

The idle state should be visually calm and unobtrusive — it must not resemble an active lighting effect or appear jarring.

---

## 4. Effect Calculations

*This section is reserved for future definition.*

The controller will maintain a library of approximately ten distinct lighting effects. Each effect will be defined by an equation that takes the following inputs:

- `bpm` — to synchronize effect timing to the beat
- `mood` (0–100) — to scale brightness bump amplitude
- `colors` — the active palette to render the effect with

During song playback, the controller rotates through the available effects at each transition point, applying a different effect per section.

The specific equations governing each lighting effect will be defined in a future revision of this document.

---

## 5. Open Questions

| # | Question | Owner |
|---|---|---|
| 1 | Effect equation definitions (§4) — the ~ten effects and their equations are pending authoring. | Firmware / product |
| 2 | How is the effect-per-section rotation order determined (fixed sequence, mood-driven, random seeded)? | Firmware / product |
| 3 | What clock-skew tolerance is acceptable across controllers before sync looks visibly off? | Firmware |
| 4 | What are the exact parameters of the calm idle "wavy" effect? | Firmware / product |

## Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 2026-06-18 | Jonathan | Initial draft |
| 0.2 | 2026-06-18 | Jonathan | Standardized to common spec template (metadata table, numbered sections, Goals/Non-Goals, Open Questions, Revision History) |