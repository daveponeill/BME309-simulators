# Bode Plot / S-Domain Filter Explorer — Design Doc
**Date:** 2026-06-01  
**Course:** BME 309 — Northwestern University

## Overview

A single-file HTML interactive simulator for exploring continuous-time filter design via s-plane pole/zero placement with live Bode plot feedback. Companion to `zplane_explorer_3.html`.

## Learning Objectives

- Understand how s-plane pole/zero locations shape Bode magnitude and phase
- Build intuition for filter types (LP, HP, BP, notch) through interactive manipulation
- Connect geometric s-plane interpretation to frequency-domain response

## Layout

Two-column plot area:
- **Left:** s-plane canvas (tall, square) — draggable poles and zeros
- **Right:** Bode magnitude (top) + Bode phase (bottom), vertically stacked, sharing the same log frequency axis

Controls above plots follow the same card/slider pattern as `zplane_explorer_3.html`.

## Controls

**Pole/Zero panels (side by side):**
- Count toggles: 0 / 1 pair / 2 pairs
- σ slider: −2000 to 0 rad/s
- Ω slider: 0 to 2000 rad/s (0 = real pole/zero; >0 = conjugate pair ±jΩ)

**Toggle card:**
- Hz ↔ rad/s (rescales Bode x-axis labels only; internal math stays in rad/s)
- Linear ↔ dB (Bode magnitude axis)
- Bode display: **Real** / **Asymptotic approximation** / **Both** — shows exact curve, straight-line approximation, or both overlaid

## Interaction

- Sliders update plots live on `oninput`
- Canvas drag: mousedown within 12px of a marker starts drag; mousemove maps pixel coords to σ/Ω and updates sliders; conjugate pair moves together
- All internal state in rad/s

## Bode Computation

- H(s) = ∏(s − zᵢ) / ∏(s − pᵢ), evaluated at s = jΩ
- 500 log-spaced points from 0.1 to 10,000 rad/s
- Magnitude: 20·log₁₀|H(jΩ)| dB (or linear |H(jΩ)|)
- Phase: ∠H(jΩ) in degrees, unwrapped
- Dashed vertical markers on Bode magnitude at each pole/zero frequency
- **Asymptotic approximation:** piecewise straight-line Bode approximation computed from pole/zero corner frequencies — +20 dB/dec per zero, −20 dB/dec per pole, with ±3 dB correction option omitted for clarity. Phase approximation: 0° far below corner, ±45° at corner, ±90° far above (per decade approximation). Overlaid in a lighter color when "Both" is selected.

## Presets

| Name               | Poles                        | Zeros                        |
|--------------------|------------------------------|------------------------------|
| Butterworth 1st LP | σ=−100, Ω=0                  | none                         |
| Butterworth 2nd LP | σ=−71, Ω=±71                 | none                         |
| Butterworth 2nd HP | σ=−71, Ω=±71                 | double zero at origin (s=0)  |
| ECG bandpass       | σ=−3, Ω=±628 (~100 Hz)       | zeros at origin              |
| 60 Hz notch        | σ=−38, Ω=±377; zeros Ω=±377  | —                            |
| Reset              | σ=−100, Ω=0 (1 pair)         | none                         |

## Note Box

Describes each pole/zero: σ, Ω, stability status, and which frequency region it influences on the Bode plot.

## Style

Matches `zplane_explorer_3.html` exactly: same CSS variables, dark mode support, Georgia serif header, system-ui body text, card/surface pattern.
