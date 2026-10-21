# Signal-demodulation architecture

The implementation provides two digital statistics for phase-sensitive signal demodulation: a sign-agreement count relative to an internal reference, and approximate input peak magnitude. The definitions below describe the RTL behavior independently of any analog application.

## Sample-domain definitions

Let `x[k]` be the unsigned input code at the kth capture after configuration, starting with `k = 0`:

```text
p[k] = k mod 2048
s[k] = 1 if x[k] >= 128, otherwise 0
r[k] = 0 for p[k] = 0..1023; 1 for p[k] = 1024..2047
a[k] = 1 if s[k] == r[k], otherwise 0
m[k] = abs(x[k] - 128)
```

The running count is the sum of `a[k]` over the latest 2048 captured samples. Before the first window fills, absent samples contribute zero. The internal sum can reach 2048; the output is `COUNT = min(sum, 2047)`. The window advances at each capture rather than producing only one result per block.

MAGNITUDE uses two ranked candidates and the previous sample, with position-based expiry on a 1024-sample scale. It is an approximation to recent peak magnitude, not an exact sliding maximum, RMS value or calibrated analog amplitude. The output range is 0–128 codes; code 0 is 128 codes from center, while code 255 is 127 codes from center.

## Phase interpretation

For an ideal zero-centered sinusoid at the reference frequency, with sample phase defined by `sin(2*pi*(k+0.5)/2048 + phi)`, the corresponding continuous sign-overlap relation is approximately:

```text
unsaturated COUNT ≈ 2048 * abs(phi_wrapped) / pi
phi_wrapped in [-pi, pi]
```

Thus 0°, 90° and 180° correspond approximately to 0, 1024 and 2048 agreements, with the last value reported as 2047. This convention follows the reference being low during its first half-period. COUNT alone does not distinguish positive from negative phase, and different waveforms can produce the same count.

Finite sampling, ADC quantization, offset, noise, clipping and frequency mismatch affect this relation. In particular, an input code of exactly 128 is classified as sign 1. The phase statistic is therefore not a guarantee of phase accuracy or a complete I/Q measurement. The [waveform observations](validation.md#supplementary-waveform-observations) state the input model under which their numerical results were obtained.

## Sampling and reference frequency

LEVEL is latched from the first valid configuration value, 0–23, and remains fixed until reset. For master clock frequency `f_clk`:

```text
capture interval = 2^LEVEL clocks
f_sample = f_clk / 2^LEVEL
f_reference = f_clk / (2048 * 2^LEVEL)
```

| LEVEL | Capture rate at an 80 MHz clock | Reference frequency |
| --- | ---: | ---: |
| 0 | 80 MS/s | 39.0625 kHz |
| 3 | 10 MS/s | 4.8828125 kHz |
| 5 | 2.5 MS/s | 1.220703125 kHz |
| 8 | 312.5 kS/s | 152.587890625 Hz |
| 12 | 19.53125 kS/s | 9.536743164 Hz |
| 23 | 9.536743164 S/s | 4.656612873 mHz |

These are rate calculations at the build target, not measured hardware bandwidths. Changing the master clock scales both frequencies proportionally. Decimation by the capture interval does not provide an anti-alias filter.

## Pipeline and interface

A sample captured on edge N is processed on N+1. Its updated statistic can appear from N+2, in the next output slot of the appropriate type. Magnitude comparisons and candidate-validity decisions are prepared in the capture stage for the following update.

The bus alternates COUNT and MAGNITUDE every master clock after handoff, independently of the capture interval. TYPE must be sampled with the data. These words are separately registered observations, not an atomic pair. Handoff is timed in master clocks and is independent of window filling; at slow LEVEL settings it can occur before the first input capture.

Both modules remain in [src/project.v](../src/project.v): `tt_um_sine_area_detector` contains the processing logic, and `tt_um_barrylee911_sine_area_v6` is the Tiny Tapeout integration top. Their existing names and hierarchy are retained. See the [operating instructions](info.md) for the complete pin contract.
