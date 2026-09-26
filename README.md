# Audio Amplifier Design and Implementation

Design, simulation, and hardware implementation of a discrete multi-stage audio power amplifier, built to a fixed set of electrical specifications and validated both in **LTspice simulation** and on a **breadboard prototype**.

This project was completed for the **Electronics Workshop (EW)** course, IIIT Hyderabad, by Chandini Gayathri Jakku and Veekshita Sai Viswanadham (Dept. of ECE).

---

## Table of Contents
- [Overview](#overview)
- [Design Specifications](#design-specifications)
- [Amplifier Architecture](#amplifier-architecture)
  - [1. Pre-Amplifier — Differential Stage](#1-pre-amplifier--differential-stage)
  - [2. Gain Stage — Common-Emitter Amplifier](#2-gain-stage--common-emitter-amplifier)
  - [3. Active Band-Pass Filter](#3-active-band-pass-filter)
  - [4. Power Amplifier — Class AB Output Stage](#4-power-amplifier--class-ab-output-stage)
- [Performance Summary](#performance-summary)
- [Repository Structure](#repository-structure)
- [How to Simulate](#how-to-simulate)
- [Bill of Materials](#bill-of-materials)
- [Requirements / Tools Used](#requirements--tools-used)

---

## Overview

Raw audio signals from microphones and similar sources are typically only a few millivolts peak-to-peak — far too weak to drive a speaker directly. This project designs a **four-stage discrete BJT audio amplifier** that takes a 10-20 mV input, amplifies it by a factor of roughly 500, restricts the signal to the audible band (20 Hz - 20 kHz), and delivers at least 1.5 W into a 10 Ω load — all while preserving signal fidelity.

Each stage is analyzed by hand using hybrid-π small-signal models to derive its gain, impedance, and component values, then verified in LTspice, and finally confirmed on hardware with an oscilloscope.

## Design Specifications

| Parameter | Symbol / Condition | Value |
|---|---|---|
| Supply voltage | V_CC, V_EE | ±5 V |
| Input signal (peak-to-peak) | V_in,pp | 10-20 mV |
| Total voltage gain | G1 × G2 | ≥ 500 |
| Frequency response (pass-band) | — | 20 Hz - 20 kHz |
| Output power | P_out | ≥ 1.5 W |
| Load resistance | R_L | 10 Ω |
| Filter in-band attenuation | — | None (0 dB) |
| Power amplifier voltage gain | — | ≈ 1 (unity) |

## Amplifier Architecture

The amplifier splits the amplification and filtering tasks across four dedicated stages rather than trying to meet gain, noise, bandwidth, and power requirements with a single stage — this keeps each stage simple, stable, and easy to bias correctly, and lets the overall gain be distributed to avoid clipping.

```
Weak Input -> [1] Differential Pre-Amp -> [2] Common-Emitter Gain Stage -> [3] Active Band-Pass Filter -> [4] Class AB Power Amp -> 10ohm Load
   (~10-20mV)      (Gain ~ 20)                (majority of voltage gain)      (20Hz-20kHz, 0dB)            (Gain ~ 1, high current)
```

### 1. Pre-Amplifier — Differential Stage
**Circuit:** `Circuit/PREAMPLIFIER.asc` | **Transistors:** Q1, Q2 (BC547B, NPN)

- A classic differential pair (two NPN transistors with a shared emitter resistor `R_E`) provides the first stage of amplification directly on the weak input signal.
- The differential topology rejects common-mode noise and presents high input impedance, so it doesn't load down the signal source.
- Hand-derived via the hybrid-π small-signal model: `A_v = g_m·R_C`, targeting a gain of ≈20.
- Component values were solved analytically (`R_C ≈ 520 Ω`, `R_E ≈ 4.13·R_C`) then fine-tuned in simulation (`R_C` adjusted to 560 Ω) to hit the target gain, accounting for non-idealities not captured by the first-order equations.
- **Result:** simulated gain ≈20; hardware-measured gain ≈21.3; measured **CMRR ≈ 44 dB**, confirming good common-mode noise rejection.

### 2. Gain Stage — Common-Emitter Amplifier
**Circuit:** `Circuit/GAIN.asc` | **Transistor:** Q3 (BC547B, NPN)

- A common-emitter amplifier with voltage-divider biasing supplies the bulk of the remaining voltage gain needed to reach the overall ≥500 target.
- Biased at a low quiescent collector current (`I_C ≈ 0.2 mA`) to minimize power dissipation, with `R_C ≈ 22.5 kΩ` chosen so the collector sits near the midpoint of the supply rails for maximum symmetric swing.
- Input impedance `R_in = R1‖R2‖r_π` is kept high enough to avoid loading the pre-amplifier stage.

### 3. Active Band-Pass Filter
**Circuit:** `Circuit/FILTER.asc` | **Op-amp:** UA741

- An op-amp-based active band-pass filter restricts the amplified signal to the audible range (20 Hz - 20 kHz), removing out-of-band noise and unwanted frequency content before the power stage.
- Designed for **unity gain in the passband** (`R1 = R2`) so the filter performs pure frequency selection without adding extra amplification.
- Upper/lower cutoff frequencies, center frequency, bandwidth, and Q-factor were derived analytically from the RC time constants of the filter network.
- **Result:** simulated gain ≈448 (whole amplifier chain up to this stage); hardware-measured gain ≈435.

### 4. Power Amplifier — Class AB Output Stage
**Circuit:** `Circuit/POWERAMPLIFIER.asc` | **Transistors:** Q4 (TIP31A, NPN), Q5 (TIP32A, PNP)

- A complementary push-pull emitter-follower pair (TIP31A/TIP32A) drives the low-impedance 10 Ω load with the current the earlier voltage-gain stages cannot supply.
- **Class AB** biasing was chosen over Class A (poor efficiency) and Class B (crossover distortion): two diodes (D1, D2) bias both transistors slightly into conduction so each conducts for just over half the cycle, eliminating crossover distortion while keeping quiescent power dissipation moderate.
- Sized to supply peak load currents of ≈500 mA for a ≈5 V output swing into 10 Ω, with base current requirements calculated from the transistors' current gain (β ≈ 200).
- Input coupling capacitor (10 µF) sets a low cutoff (~15.9 Hz), well below the audio band, so the 20 Hz-20 kHz range passes unattenuated.
- **Result:** simulated gain ≈422; hardware-measured gain ≈410 (consistent with the near-unity current-boosting role of this stage relative to the filter stage).

## Performance Summary

The complete 4-stage amplifier (`Circuit/FINAL.asc`) was simulated end-to-end and then rebuilt on a breadboard, with performance validated against the design specs:

| Metric | Result |
|---|---|
| Overall frequency response | Flat gain across 20 Hz - 20 kHz (LTspice AC sweep + hardware FRA sweep both confirm) |
| Total Harmonic Distortion (THD) | ≈5.16% (LTspice Fourier/FFT analysis), consistent with hardware THD measurement |
| CMRR (pre-amp stage) | ≈43.9-44 dB |
| Slew Rate | ≈0.026 V/µs (≈25.9 V/ms), sufficient for full audio bandwidth |
| Output power into 10 Ω | Meets ≥1.5 W target |

Simulation results closely matched hand-calculated theoretical predictions and hardware measurements across every stage, validating the design methodology end-to-end.

## Repository Structure
```
Audio-Amplifier-Design-main/
├── EW report.pdf                  # Full project report: theory, derivations, simulation & hardware results
├── BILL.txt                       # Bill of Materials (BOM)
├── Circuit/
│   ├── PREAMPLIFIER.asc           # Stage 1: Differential pre-amplifier (standalone)
│   ├── GAIN.asc                   # Stage 2: Common-emitter gain stage (standalone)
│   ├── FILTER.asc                 # Stage 3: Active band-pass filter, uses UA741 op-amp (standalone)
│   ├── POWERAMPLIFIER.asc         # Stage 4: Class AB power amplifier (standalone)
│   └── FINAL.asc                  # Complete 4-stage amplifier, fully integrated
└── LTSPICE_models/
    ├── UA741.301                  # SPICE model for the UA741 op-amp (used in the filter stage)
    ├── tip31a.lib                 # SPICE model for the TIP31A NPN power transistor
    └── tip32a.lib                 # SPICE model for the TIP32A PNP power transistor
```

## How to Simulate

1. Install [LTspice](https://www.analog.com/en/resources/design-tools-and-calculators/ltspice-simulator.html).
2. Place the `.asc` schematic and the `LTSPICE_models/` files in the same working directory (the schematics `.include` the `UA741.lib`, `tip31a.lib`, and `tip32a.lib` model files directly).
3. Open any individual stage (`PREAMPLIFIER.asc`, `GAIN.asc`, `FILTER.asc`, `POWERAMPLIFIER.asc`) to simulate it in isolation, or open `FINAL.asc` for the complete integrated amplifier.
4. Run the transient simulation (`.tran 10m` directive is already included in each schematic) to view input/output waveforms, or set up an AC sweep / Fourier (`.four`) analysis to reproduce the frequency response and THD results reported in `EW report.pdf`.

## Bill of Materials

See `BILL.txt` for the full parts list. Key components:

| Ref | Part | Role |
|---|---|---|
| Q1, Q2, Q3 | BC547B (NPN) | Pre-amp differential pair + CE gain stage |
| Q4 | TIP31A (NPN) | Power amplifier — push half of Class AB pair |
| Q5 | TIP32A (PNP) | Power amplifier — pull half of Class AB pair |
| U1 | UA741 | Op-amp for the active band-pass filter |
| D1, D2 | 1N4148 | Class AB bias diodes (eliminate crossover distortion) |
| C1-C7 | 2.2 pF - 10 µF | Coupling, filter, and bypass capacitors |
| R1-R14 | 10 Ω - 515 kΩ | Biasing, gain-setting, and filter resistors |

## Requirements / Tools Used
- **LTspice** — schematic capture and circuit simulation (transient, AC sweep, Fourier/THD analysis)
- **Oscilloscope** (with waveform generator and Frequency Response Analysis feature) — hardware verification
- **Discrete components** (BJTs, op-amp, R/C/D) on breadboard — physical prototype
- **Heat sinks** — used on the Class AB power transistors to safely dissipate output-stage power

## Author
Electronics Workshop (EW) course project, IIIT Hyderabad.
