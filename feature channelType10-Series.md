# Ten Series

| | |
|---|---|
| **Version** | 0.2 |
| **Status** | Draft |
| **Last updated** | 2026-06-18 |
| **Owner** | Jonathan, Haven Lighting |
| **Target / scope** | X Series lighting controllers — new channel type |
| **Classification** | Internal |

---

## 1. Overview

Ten Series is a new channel type for X Series lighting controllers, designed for installations with **96-inch LED spacing**. It uses an **RGBWW** configuration — red, green, blue, plus two distinct white channels (W1 and W2).

Because five channels exceed the capacity of a single driver IC, each pixel requires **two LED driver ICs**:

- The **first IC** drives the RGB channels.
- The **second IC** drives the two white channels (W1, W2). Its third output is unused.

## 2. Goals & Non-Goals

- **Goals** — define the RGBWW channel type, its two-IC hardware mapping, the five-channel color table, and the cross-channel blend function for X Series controllers at 96-inch spacing.
- **Non-Goals** — standard effect and HSV animation behavior (unchanged, RGB-only); color table content authoring beyond the worked example; other channel types or LED spacings.

## 3. Hardware Architecture

Each Ten Series pixel maps its five channels across two driver ICs as follows:

| Driver IC | Output | Channel |
|-----------|--------|---------|
| IC 1 | 1 | Red |
| IC 1 | 2 | Green |
| IC 1 | 3 | Blue |
| IC 2 | 1 | White 1 (W1) |
| IC 2 | 2 | White 2 (W2) |
| IC 2 | 3 | Unused |

## 4. RGB vs. White LED Operation

Not all lighting behaviors engage the white channels:

- **Standard effects** (e.g., sparkle, ripple) and **HSV-based animations** operate on the **RGB LEDs only**. The white channels remain inactive.
- To drive the **white channels**, Ten Series relies on a **color table** with entries defined uniquely for this light type.

## 5. Color Table Definition

Unlike standard color tables that specify only RGB values, Ten Series color table entries define blends across **all five channels** (R, G, B, W1, W2).

**Example — Index 27 (Warm White):**

| Channel | Value |
|---------|-------|
| Red | 0% |
| Green | 0% |
| Blue | 0% |
| White 1 (W1) | 5% |
| White 2 (W2) | 95% |

This entry produces warm white entirely from the two white channels, with no RGB contribution.

## 6. Blend Function and Cross-Channel Ramping

A new blend function enables **linear ramping between any two colors across all five channels**.

When transitioning from one color to another — for example, white to green — the channels that are active in the source color **scale linearly down to zero**, while the channels active in the target color **ramp linearly up**. In the white-to-green case, W1 and W2 fall to zero as G rises.

This is the same mathematical logic used for three-channel RGB ramping, extended seamlessly to all five RGBWW channels.

## 7. Open Questions

| # | Question | Owner |
|---|---|---|
| 1 | What is the full color table for Ten Series beyond the warm-white example, and who authors it? | Firmware / product |
| 2 | How are W1 and W2 spec'd (color temperature, binning) for the two physical white LEDs? | Hardware |
| 3 | Is the unused third output on IC 2 reserved for a future channel or permanently dark? | Hardware |

## Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 2026-06-18 | Jonathan | Initial draft |
| 0.2 | 2026-06-18 | Jonathan | Standardized to common spec template (metadata table, numbered sections, Goals/Non-Goals, Open Questions, Revision History) |