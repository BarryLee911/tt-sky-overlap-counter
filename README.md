# Sign-Based Signal Demodulator

A digital signal-demodulation core that combines phase-sensitive sign-agreement counting with approximate amplitude tracking.

[![gds](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/gds.yaml/badge.svg)](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/gds.yaml) [![docs](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/docs.yaml/badge.svg)](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/docs.yaml) [![test](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/test.yaml/badge.svg)](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/test.yaml) [![equivalence](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/equivalence.yaml/badge.svg)](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/workflows/equivalence.yaml)

## Features

| Property | Implementation |
| --- | --- |
| Input | Unsigned 8-bit samples, centered on 128 |
| COUNT | Sign agreement with an internal reference over 2048 samples; output 0–2047 |
| MAGNITUDE | Approximate peak magnitude over about 1024 samples; output 0–128 codes |
| Sampling | 24 levels; one capture every `2^LEVEL` clocks |
| Output | 11-bit bus with alternating COUNT / MAGNITUDE, identified by TYPE |

**Target:** Tiny Tapeout SKY26d · SKY130 · 6×2 tiles · 80 MHz build target.

## Simulation highlights

With a frequency-matched sinusoid, **COUNT follows relative phase while MAGNITUDE tracks input amplitude**. Example: LEVEL 5, 80 MHz ideal clock, amplitude 64 codes, ideal 14-bit ADC → upper 8 bits:

| Input phase | COUNT | MAGNITUDE |
| --- | ---: | ---: |
| 0° | 0 | 64 |
| 45° | 512 | 64 |
| 90° | 1024 | 64 |
| 135° | 1536 | 64 |
| 180° | 2047 | 64 |

At 90°, tested amplitudes of 16, 32 and 127 codes produced matching MAGNITUDE values. **All 27 local waveform cases passed**, including accelerated tests at slow levels. COUNT is a phase-sensitive statistic, not a signed angle; these results depend on the stated input model.

RTL, gate-level and cycle-equivalence checks are documented in the [validation record](docs/validation.md). All-corner timing closure and silicon operation are not established; 80 MHz is a build target, not measured performance.

[Architecture](docs/architecture.md) · [Pinout & operation](docs/info.md) · [ADC interface](docs/adc-interface.md) · [Validation & reproduction](docs/validation.md) · [Layout viewer](https://barrylee911.github.io/tt-sky-overlap-counter/)

[Citation](CITATION.cff) — cite the exact commit used. · [Apache-2.0](LICENSE)
