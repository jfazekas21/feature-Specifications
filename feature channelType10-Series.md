# Ten Series

**Version:** 0.1 · **Last updated:** 2026-06-18

## Overview

Ten Series is a new channel type for X Series lighting controllers, designed for installations with **96-inch LED spacing**. It uses an **RGBWW** configuration — red, green, blue, plus two distinct white channels (W1 and W2).

Because five channels exceed the capacity of a single driver IC, each pixel requires **two LED driver ICs**:

- The **first IC** drives the RGB channels.
- The **second IC** drives the two white channels (W1, W2). Its third output is unused.

## Hardware Architecture

Each Ten Series pixel maps its five channels across two driver ICs as follows:

| Driver IC | Output | Channel |
|-----------|--------|---------|
| IC 1 | 1 | Red |
| IC 1 | 2 | Green |
| IC 1 | 3 | Blue |
| IC 2 | 1 | White 1 (W1) |
| IC 2 | 2 | White 2 (W2) |
| IC 2 | 3 | Unused |

## RGB vs. White LED Operation

Not all lighting behaviors engage the white channels:

- **Standard effects** (e.g., sparkle, ripple) and **HSV-based animations** operate on the **RGB LEDs only**. The white channels remain inactive.
- To drive the **white channels**, Ten Series relies on a **color table** with entries defined uniquely for this light type.

## Color Table Definition

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

## Blend Function and Cross-Channel Ramping

A new blend function enables **linear ramping between any two colors across all five channels**.

When transitioning from one color to another — for example, white to green — the channels that are active in the source color **scale linearly down to zero**, while the channels active in the target color **ramp linearly up**. In the white-to-green case, W1 and W2 fall to zero as G rises.

This is the same mathematical logic used for three-channel RGB ramping, extended seamlessly to all five RGBWW channels.