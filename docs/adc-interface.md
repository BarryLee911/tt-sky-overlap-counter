# External ADC options

For the current Overlap Count interface, use **TI ADC3544** when a synchronous parallel source is the priority. **ADI AD4052** is a newer precision, low-power alternative when an external SPI bridge is available. Neither is a pin-for-pin connection to this chip. This guide is a datasheet-based selection, not a tested board design; no adapter PCB or controller firmware is included.

## Parts and UK availability

Distributor listings checked on 2026-10-14:

| Part | UK source | Listed availability and single-piece price | Package |
| --- | --- | --- | --- |
| TI ADC3544IRSBT | [Mouser UK, 595-ADC3544IRSBT](https://www.mouser.co.uk/en/ProductDetail/Texas-Instruments/ADC3544IRSBT?qs=sGAEpiMZZMtgJDuTUz7Xu4QwBSNmNJrvpClvQjzVosnIFMll87BqGA%3D%3D) | 291 in stock; about GBP 76 | 40-pin WQFN, 5 x 5 mm |
| ADI AD4052BCPZ-R2 | [DigiKey UK, cut tape](https://www.digikey.co.uk/en/products/detail/analog-devices-inc/AD4052BCPZ-R2/24764407) | 377 in stock; GBP 14.51 excluding VAT | 14-pin LFCSP, 2 x 2.6 mm |

These are component prices; check current stock, tax and delivery at checkout. Both packages need a PCB with the correct land pattern and exposed-pad soldering.

| Selection factor | ADC3544 | AD4052 |
| --- | --- | --- |
| Product status / age evidence | TI Active; data sheet initially released December 2022 | ADI Recommended for New Designs; evaluation design files available from 2024 |
| Resolution / maximum conversion rate | 14 bit / 125 MSPS | 16 bit / 2 MSPS |
| Manufacturer SNR example | 74 dBFS at 5 MHz input and 125 MSPS | 86.1 dB with a 3.3 V reference |
| Manufacturer power example | 97 mW typical at 125 MSPS, external reference, SDR | 1.35 mW at 1 MSPS in sample mode |
| Integration cost | 1.8 V supplies and logic translation; parallel SDR configuration | SPI acquisition, conversion to a byte and scheduled parallel output |

The SNR and power entries use different manufacturer test conditions; they are not a measured system comparison. ADC3544 offers throughput headroom for the 80 MHz build target. AD4052 is the lower-power choice for slower operation. Neither specification qualifies the complete board for 80 MHz.

## Common input contract

The chip accepts `SAMPLE = ui[7:0]`, unsigned 0-255 with a center of 128. It has no ADC clock output, conversion-start output, data-ready input or serial ADC receiver. External hardware must present a complete, stable byte at each scheduled capture edge.

First normalize the ADC result to unsigned offset binary `B`, then take:

```text
ADC3544, 14-bit raw mode: SAMPLE = B >> 6
AD4052, 16-bit sample mode: SAMPLE = B >> 8
```

This maps midscale to 128. If the ADC is set to two's complement, convert its signed result to offset binary first. Gain and zero-level calibration belong in the external signal path or controller. Reducing a 14/16-bit ADC to a byte preserves the existing circuit interface; it does not give the chip 14/16-bit input precision.

## ADC3544 parallel connection

Configure raw 14-bit parallel **SDR**, bypass the ADC's digital processing, and select **offset binary**. The power-up coding is two's complement; follow TI's FORMAT A and digital-bypass instructions before using the bus. The following mapping assumes the default 14-bit SDR output bit mapper; ADC pin names differ from logical bit numbers.

| ADC3544 pin name | Package pin | Raw result bit | Destination through 1.8 V to 3.3 V translation |
| --- | --- | --- | --- |
| D8 | 29 | B[6] | ui[0] |
| D9 | 28 | B[7] | ui[1] |
| D10 | 27 | B[8] | ui[2] |
| D11 | 24 | B[9] | ui[3] |
| D12 | 23 | B[10] | ui[4] |
| D13 | 22 | B[11] | ui[5] |
| D14 | 21 | B[12] | ui[6] |
| D15 | 20 | B[13] | ui[7] |

ADC3544 analog and I/O supplies are 1.8 V. The Tiny Tapeout demo board uses 3.3 V I/O. An eight-channel translator such as [SN74AXC8T245](https://www.ti.com/product/SN74AXC8T245), configured from 1.8 V to 3.3 V, is a candidate for this data bus. Match ADC clock, reset and SPI-control voltages as well; a 3.3 V controller must not drive the ADC's 1.8 V inputs directly. Connect grounds and follow TI's reference, input-driver, decoupling and layout requirements.

An external clock controller must align the ADC stream and chip clock. Use the ADC's DCLK and timing diagram to place chip capture edges inside the translated bus's valid window; include translator delay, skew and board flight time. Start with a modest clock rate and measure setup/hold before increasing it. Keep the chip in reset until ADC configuration, reference settling and output validity are established. Record the ADC and adapter latency when aligning input samples with the internal reference.

## AD4052 bridge connection

```text
Analog input -> AD4052 -> SPI controller / FPGA
             -> offset-binary conversion and byte register -> ui[7:0]
```

The host configures the ADC, schedules conversions, reads complete SPI results and updates all eight output bits together. Use 3.3 V host outputs, or translate their voltage, for the Tiny Tapeout input. Keep conversion timing and byte presentation synchronized to the chip's capture schedule; independent free-running clocks are insufficient. The bridge must sustain the complete conversion/read/output cycle, not just the ADC's headline rate.

At the 80 MHz build target, LEVEL 5 requests 2.5 MSPS and exceeds AD4052's 2 MSPS ceiling; LEVEL 6 requests 1.25 MSPS. A bridge may need a slower setting. This rate calculation is not validation of an 80 MHz external interface.

## Sampling and analog conditioning

The chip captures every `2^LEVEL` clocks after configuration, with LEVEL 0-23. It has no backpressure: repeated bytes are counted as repeated samples. An ADC's maximum/minimum conversion clock and the chip's effective capture rate are separate limits; do not assume a slow LEVEL permits arbitrary ADC clock gating.

Use a DC-capable input path for slow signals, with the ADC's required common mode, reference and input range. Adjust the input zero level to code 128. Band-limit before the chip's effective sampling rate `f_clk / 2^LEVEL`; selecting every nth sample is not an anti-alias filter. Preserve the configuration and TYPE handoff described in the [operating guide](info.md); ADC control belongs to the external controller, not to the chip's result pins.

The existing RTL and gate-level tests exercise digital samples. They do not verify the ADC, analog input path, translator or adapter controller. Measure clipping, zero-code stability, timing and end-to-end latency before reporting board-level results.

## Sources

- [TI ADC3544 product page](https://www.ti.com/product/ADC3544) and [data sheet SBASA45](https://www.ti.com/lit/ds/symlink/adc3544.pdf), especially pin functions, SDR timing, output coding and initialization.
- [ADI AD4052 product page](https://www.analog.com/en/products/ad4052.html) and [AD4052/AD4058 data sheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ad4052-ad4058.pdf).
- [Tiny Tapeout GPIO specifications](https://tinytapeout.com/specs/gpio/): demo-board I/O voltage and pad frequency limits.
