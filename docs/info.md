# How it works

Sign-based signal demodulation with phase-sensitive counting and amplitude tracking.

The input is an unsigned 8-bit sample centered on 128. Values below 128 have sign 0; values at or above 128 have sign 1. The internal square reference repeats every 2048 captured samples, with 1024 samples in each half-period.

COUNT reports sign agreement over the most recent 2048 samples. Unfilled positions count as zero at startup, and the 11-bit result saturates at 2047. MAGNITUDE tracks the approximate peak of `abs(SAMPLE - 128)` over about 1024 samples, in the range 0-128. It is not an exact sliding maximum.

For a frequency-matched sinusoid, COUNT is related to relative phase; it does not directly encode a signed phase angle or an I/Q pair. Quantization, offset and the reference phase convention affect its interpretation. See the [architecture description](https://github.com/BarryLee911/tt-sky-overlap-counter/blob/main/docs/architecture.md) for the numerical definitions.

# How to test

## Pins

| Pin | Direction | Function |
| --- | --- | --- |
| `ui[7:0]` | Input | Unsigned 8-bit SAMPLE |
| `uo[7:0]` | Output | Result bits 7:0 |
| `uio[7:5]` | Output | Result bits 10:8; do not drive externally |
| `uio[4:1]` | Input | LEVEL bits 4:1 during configuration |
| `uio[0]` | Input, then output | LEVEL bit 0 during configuration; TYPE after handoff |
| `clk` | Input | Clock |
| `rst_n` | Input | Low-active synchronous reset |
| `ena` | Input | Not used as an operation control |

## Reset and configuration

1. Apply a clock and hold `rst_n` low for at least two rising edges.
2. Set LEVEL on `uio[4:0]` to a value from 0 to 23, then release reset. The first rising edge with a valid level latches it. Levels 24-31 do not start operation.
3. Keep SAMPLE stable around each capture edge. The first sample is captured `2^LEVEL` clocks after the configuration edge; subsequent samples use the same interval. LEVEL 0 captures on every clock, starting on the next edge.
4. Release external drive on `uio[0]` after configuration and before the 80000th clock after the configuration edge. At that edge the chip drives TYPE on this pin. At the 80 MHz build target this interval is 1 ms.

The level remains latched until reset. Reset is required to change it. Sampling rate is `f_clk / 2^LEVEL` and reference frequency is `f_clk / (2048 * 2^LEVEL)`; window lengths are measured in captured samples rather than clocks. The external sample source must be synchronized to this capture schedule. There is no data-ready input or backpressure.

## Reading results

Combine `{uio[7:5], uo[7:0]}` as an 11-bit value and read TYPE at the same time:

| TYPE | Result | Range |
| --- | --- | --- |
| 0 | COUNT | 0-2047 |
| 1 | MAGNITUDE | 0-128 |

The handoff edge presents COUNT; the next edge presents MAGNITUDE. Results then alternate every clock, independently of the sampling interval. Capture the registered outputs after they settle following each rising edge.

A sample captured at edge N updates statistics at N+1 and can appear from N+2 in the next slot for its result type. Read TYPE with each result rather than treating adjacent words as a simultaneous pair, especially at LEVEL 0. With slow sampling, handoff can occur before the first sample; zero results during this wait are normal.

## Build target

The design targets 80 MHz on SKY130 with 6x2 tiles. In the recorded build at repository revision `1f28d70`, the worst setup slack was -0.110671 ns; slew and fanout violations were also reported. This is an implementation target, not a demonstrated operating frequency. It exceeds the input-pad rating in the [Tiny Tapeout GPIO specifications](https://tinytapeout.com/specs/gpio/), which also specify a separate output-frequency limit.

See the [validation record](https://github.com/BarryLee911/tt-sky-overlap-counter/blob/main/docs/validation.md) for the exact build, test scope and reproduction commands. Passing CI does not establish all-corner timing closure or operation on fabricated hardware. No silicon measurements are reported.

# External hardware

A clock source, an 8-bit digital sample source, configuration drivers and a device to capture the result pins. Synchronize SAMPLE to the clock and release the shared configuration/TYPE pin before handoff.

Suggested external ADCs are TI ADC3544 with parallel data adaptation, or ADI AD4052 with an external SPI bridge. See the [ADC interface guide](https://github.com/BarryLee911/tt-sky-overlap-counter/blob/main/docs/adc-interface.md) for UK suppliers and connection requirements. These are datasheet-based candidates; the external hardware has not yet been bench tested.
