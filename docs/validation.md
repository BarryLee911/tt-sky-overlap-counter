# Validation and reproducibility

This record separates implementation-flow results, digital functional checks and supplementary ideal-input observations. It does not report fabricated-silicon measurements or characterize an analog signal chain.

## Fixed hardware and build record

The reference implementation for this document is repository commit [`1f28d70b04c08c91fa98e1ee7355492fa2102d49`](https://github.com/BarryLee911/tt-sky-overlap-counter/tree/1f28d70b04c08c91fa98e1ee7355492fa2102d49). The documentation update retains its HDL, implementation configuration, regression tests and workflows. The title and description in `info.yaml` are presentation metadata.

Source: `src/project.v`, SHA-256 `279db397eecc4afd91e2f172e8bf21a465506f964cbf1438a8e865b14531323e`.

| Check | Recorded result | Evidence |
| --- | --- | --- |
| GDS implementation, precheck, gate-level regression, viewer | All four jobs passed | [GDS run 38596543490](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/runs/38596543490) |
| Pin-level RTL regression | Passed | [Test run 38596543503](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/runs/38596543503) |
| Cycle-equivalence simulation and provenance checks | Passed | [Equivalence run 38596543584](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/runs/38596543584) |
| Documentation build | Passed | [Docs run 38596543579](https://github.com/BarryLee911/tt-sky-overlap-counter/actions/runs/38596543579) |

The RTL and gate-level tests use the same pin-level regression. The gate-level workflow uses functional standard-cell models with unit delays; it is not an SDF-backannotated timing simulation. Cycle equivalence uses Icarus Verilog, seeds 101/202/303 and a shortened 40-clock handoff. The pin-level regression separately checks the normal 80000-clock handoff. The optional formal and all-level divider checks are not executed by the equivalence CI job. No formal proof is claimed here.

### Implementation metrics and limits

Values below were extracted from that GDS run's `GDS_logs` artifact, using `scripts/assess_build.py`. The [machine-readable summary](validation/build-summary.json) preserves all nine corners and the resolved checker settings.

| Metric | Recorded value |
| --- | ---: |
| Clock-period constraint | 12.5 ns (80 MHz target) |
| Standard-cell area | 124984 µm² |
| Worst setup slack across nine corners | −0.110671 ns, `max_ss_100C_1v60` |
| Minimum hold slack across nine corners | +0.110229 ns, `min_ff_n40C_1v95` |
| Maximum reported slew-violation count among corners | 555 |
| Reported maximum-capacitance violations | 0 at each corner |
| Reported maximum-fanout violations | 162 at each corner |
| Reported Magic DRC, routing DRC, LVS and antenna errors | 0 in the corresponding metrics |

Standard-cell area is not the allocated tile area. These results do not establish all-corner setup or electrical-constraint closure. In the resolved flow, timing acceptance selected `*tt*`, hold selected `*`, and slew/capacitance checker corner lists contained an empty string. A successful workflow therefore must not be interpreted as zero violations at every reported corner.

[Tiny Tapeout GPIO specifications](https://tinytapeout.com/specs/gpio/) list a 66 MHz input limit and a separate 33 MHz output limit, with output behavior dependent on loading and waveform interpretation. The 80 MHz target exceeds the input rating. Internal STA, pad limits and board timing are separate constraints; RTL simulation establishes none of them as measured hardware performance.

## Supplementary waveform observations

Local RTL experiments recorded on 2026-10-21 used Icarus Verilog 12.0 devel, an 80 MHz ideal clock and the following ideal ADC/adapter model:

```text
x[k] = 128 + A*sin(2*pi*(k+0.5)/2048 + phi)
B[k] = clamp(floor(64*x[k] + 0.5), 0, 16383)
SAMPLE[k] = B[k] >> 6
```

`k = 0` is the first captured sample and `phi` is in radians. The byte is presented half a clock before its scheduled capture and held until the next update. Each case is reset, allowed to settle and observed for two reference periods (4096 captures). The local reference model checks cycle behavior; separate checks recompute the quantized sample codes and sign-agreement totals. This is not a device model of the ADC3544: conversion latency, DCLK, noise, gain/offset error and analog filtering are absent.

At LEVEL 5, with real 32-clock capture intervals:

| Phase | Input amplitude A (codes) | COUNT | MAGNITUDE |
| --- | ---: | ---: | ---: |
| 0° | 64 | 0 | 64 |
| 45° | 64 | 512 | 64 |
| 90° | 64 | 1024 | 64 |
| 135° | 64 | 1536 | 64 |
| 180° | 64 | 2047 | 64 |
| 90° | 16 | 1024 | 16 |
| 90° | 32 | 1024 | 32 |
| 90° | 127 | 1024 | 127 |
| 90° | 0 | 1024 | 0 |

A zero amplitude means constant code 128: COUNT=1024 then follows from the reference duty cycle and does not represent a measured phase.

A further sweep repeated `(phase, amplitude) = (0°,64), (90°,32), (180°,127)` at each level below. All six levels returned `(COUNT,MAGNITUDE) = (0,64), (1024,32), (2047,127)` respectively:

| LEVEL | Mode | Nominal reference frequency at 80 MHz |
| --- | --- | ---: |
| 0 | Real divider | 39.0625 kHz |
| 3 | Real divider | 4.8828125 kHz |
| 5 | Real divider | 1.220703125 kHz |
| 8 | Real divider | 152.587890625 Hz |
| 12 | Accelerated capture | 9.536743164 Hz |
| 23 | Accelerated capture | 4.656612873 mHz |

The 27 cases passed their local checks; each reported statistic was constant during its observation interval. The accelerated cases force the core sampling strobe after configuration, capturing every master clock. They check processing under that stimulus, **not the real divider wait** at LEVEL 12 or 23. Their listed frequencies are calculated nominal rates, not simulated rates.

The [result export](validation/waveform-results.json) records conditions, output ranges, frame counts and testbench/reference hashes. These are supplementary observations, not an exhaustive accuracy study or CI tests. The local Windows waveform launcher and full VCD files are not included in this repository; the export is a summary rather than a self-contained reproduction package. In particular, these selected results do not imply exact amplitude tracking for arbitrary waveforms or zero phase bias under other quantizers.

## Reproducing the repository checks

Use a fixed checkout and record `git rev-parse HEAD`. The following shell commands mirror the existing RTL workflow on a system with Python 3.11, Icarus Verilog and GNU Make:

```sh
python -m pip install -r test/requirements.txt
python scripts/verify_inputs.py
cd test
make clean
make SIM=icarus
python -m cocotb_tools.check_results results.xml
cd ..
```

See [cycle-equivalence instructions](../test/equivalence/README.md) for the three simulation seeds and optional checks. The provenance script validates locked sources/configuration/workflows; it does not evaluate timing closure.

For implementation results, retain the GDS workflow URL, commit, artifact and tool versions recorded by that run. After extracting `GDS_logs`, run from the repository root:

```sh
python scripts/assess_build.py --run-dir /path/to/extracted/runs/wokwi --output verification/build/diagnostics.json
```

This is a diagnostic summary, not an independent signoff decision. Workflow action references and upstream tools can evolve, and Actions artifacts have finite retention; preserve the exact artifacts used for a publication. Rebuilding unchanged RTL does not by itself guarantee identical placement, routing or timing metrics.
