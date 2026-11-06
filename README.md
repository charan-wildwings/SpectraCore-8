# SpectraCore-8

SpectraCore-8 is an eight-point, fixed-point Fast Fourier Transform (FFT) processor written in Verilog. A host sends eight complex samples over a Serial Peripheral Interface (SPI), waits for the transform, and reads back eight frequency bins. Each 32-bit word carries both parts of one complex number. The design uses a single butterfly datapath for all three FFT stages.

**Project status:** All 24 directed simulation checks passed. Design Compiler mapped the processor to 3,943 SAED32 RVT cells and met the 50 MHz target on the constrained paths at the analyzed corner. IC Compiler II produced legal placement and a saved clock-tree synthesis (CTS) checkpoint. Three maximum-capacitance violations remain. Detailed routing, final physical verification, and GDSII export are pending.

## Design specification

| Property | Implementation |
|---|---|
| RTL top | `fft8_spi` |
| Transform | Eight-point forward radix-2 decimation-in-time FFT |
| Frame | Eight complex input samples and eight complex output bins |
| Word format | Bits `[31:16]` carry the real component. Bits `[15:0]` carry the imaginary component. |
| Number format | Signed Q1.15 per component. Divide the signed integer code by 32,768 to obtain its value. |
| SPI protocol | Mode 0, most significant bit first, one 32-bit word per chip-select assertion |
| FFT schedule | Three stages of four butterflies, one butterfly per system clock |
| Output scale | One half per stage, or one eighth across the transform |
| Implementation target | SAED32 RVT standard cells at a 20 ns system clock period (50 MHz) |

## How a frame moves through the processor

| State | Host action | Processor action |
|---|---|---|
| `LOAD` | Write eight samples in time order. | Collect each word on MOSI, separate its real and imaginary components, and write them to a bit-reversed address. |
| `CALC` | Wait for `result_ready`. | Assert `busy` and perform twelve butterfly operations on the stored samples. |
| `READ` | Send eight dummy words while reading MISO. | Assert `result_ready` and return bins 0 through 7 in natural order. Return to `LOAD` after the eighth read. |

For example, the word `0x40000000` represents **0.5 + j0**. Its upper half is 16,384, and its lower half is zero. If the host sends that word followed by seven zero words, each frequency bin should contain **0.0625 + j0**, packed as `0x08000000`. This example links the SPI word format directly to an observable FFT result.

### Butterfly arithmetic

Each butterfly reads two complex values, multiplies the second by a twiddle factor, and writes a scaled sum and difference. For operands *a* and *b* and twiddle factor *W*, the calculation is:

~~~text
t = b × W
upper output = (a + t) / 2
lower output = (a − t) / 2
~~~

The controller stores input indices 0 through 7 at addresses **0, 4, 2, 6, 1, 5, 3, 7**. This bit reversal allows the host to read the finished bins in ordinary order.

| FFT stage | Address pairs | Twiddle exponents for W₈ = exp(−j2π/8) |
|---|---|---|
| 1 | (0,1), (2,3), (4,5), (6,7) | 0, 0, 0, 0 |
| 2 | (0,2), (1,3), (4,6), (5,7) | 0, 2, 0, 2 |
| 3 | (0,4), (1,5), (2,6), (3,7) | 0, 1, 2, 3 |

A unity twiddle passes its operand through. Multiplication by −j maps (real, imaginary) to (imaginary, −real). The remaining rotations use signed coefficients, including 23,170 as the Q1.15 approximation to 1/√2. The RTL widens intermediate values and uses arithmetic shifts and saturation when storing the results. Truncation explains the one-code asymmetry seen in some positive and negative output components.

### SPI timing and throughput

The RTL samples SCLK, chip select, and MOSI in the system clock domain. It detects synchronized SCLK edges, receives on rising edges, and advances MISO on falling edges. The host provides guard time around chip select so the synchronized edge detector can recognize a new word. MISO is driven low while deselected, so sharing MISO with other devices requires output-enable handling at integration.

The testbench uses a **100 MHz system clock and a 5 MHz SPI clock**. Synthesis uses a separate **50 MHz system-clock target**. At that target, twelve arithmetic cycles take **240 ns**. At 5 MHz SPI, eight writes plus eight reads require **512 bit periods**, or **102.4 µs** before chip-select gaps. Serial transfers therefore account for most of the frame latency in this configuration.

## Functional verification

The self-checking VCS testbench sends complete SPI frames, reads the eight bins, and compares both signed components against the expected values. The tolerance is one least-significant bit (LSB) per component.

| Test | Input | Observed output | What the test exercises |
|---|---|---|---|
| Real impulse | Sample 0 is 0.5. All other samples are zero. | All eight bins returned `0x08000000`. | Word packing, readback, and overall 1/8 gain |
| Imaginary DC | All samples equal j0.25. | Bin 0 returned `0x00002000`. Bins 1–7 returned zero. | Imaginary data handling, DC accumulation, and cancellation |
| Delayed real impulse | Sample 1 is 0.5. All other samples are zero. | Phase changed across the bins as expected. Bin 1 returned signed codes (1448, −1449). | Twiddle factors, forward-transform sign, and bin order |

**Result:** Three frames and 24 complex output words passed within tolerance. The delayed impulse gives the strongest bin-order check. Moving the impulse one sample changes the phase from bin to bin, whereas a misplaced bin can go undetected when all eight expected values are identical.

## Logic synthesis results

Synopsys Design Compiler used the **SAED32 RVT slow library at 0.95 V and 125 °C**. Timing values below apply to the constrained paths in that synthesis run.

| Metric | Reported result |
|---|---:|
| Clock period | 20 ns |
| Setup / hold uncertainty | 0.10 ns / 0.01 ns |
| Total mapped cells | 3,943 |
| Combinational / sequential cells | 3,598 / 345 |
| Total cell area | 11,415.64 library area units |
| Critical setup slack | +13.24 ns |
| Minimum displayed hold slack | +0.16 ns |
| Violating setup / hold paths | 0 / 0 |
| Maximum transition / capacitance violations | 0 / 0 |
| Estimated dynamic power | 99.68 µW |
| Estimated leakage power | 509.73 µW |
| Estimated total power | 609.41 µW |

The power estimate uses tool-estimated switching activity. External SPI and status-output timing budgets, reset-release checks, and fast-corner hold analysis remain open.

## Physical implementation

| Stage | Recorded outcome |
|---|---|
| Netlist import | ICC2 linked 3,943 instances, 4,526 nets, and eight top-level ports. |
| Floorplan and power planning | The script specifies 70% initial core utilization, an M7/M8 supply ring, an M5/M6 mesh, and M1 standard-cell rails. |
| Placement | The placement legality report lists zero violations in the checks performed. |
| Clock-tree synthesis | ICC2 saved the `3_cts` checkpoint and flagged an incomplete final clock-optimization phase. |
| Electrical checks | The `func_slow` scenario reported three maximum-capacitance violations and zero maximum-transition violations at the recorded checkpoint. |
| Routing and export | The script stopped at an unsupported routing option before detailed routing. A verified routed netlist, extracted SPEF, and final GDSII stream remain pending. |

## Challenges and resolutions

| Challenge | How it appeared | Resolution or next step |
|---|---|---|
| Inconsistent simulation timescales | VCS rejected the design because only the testbench declared a timescale. | Adding the same 1 ns / 1 ps declaration to the RTL allowed simulation to run. |
| Fixed-point accuracy and bin ordering | Stage scaling changed the expected output amplitude, and a single impulse at index zero could conceal a bin permutation. | We checked three complementary frames and compared signed real and imaginary codes separately. |
| Sampling asynchronous SPI signals | The controller must recognize an external edge and capture the corresponding data bit even though SPI and system clocks are independent. | Synchronized inputs passed the directed tests. External timing and clock-domain-crossing review remain open. |
| Missing synthesis library | The unavailable target library left the circuit unmapped. Generic cells produced zero area and 679 apparent hold violations. | Loading `saed32rvt_ss0p95v125c.db` produced a mapped netlist with nonzero area and zero reported hold violations at the analyzed corner. |
| Excessive loading after CTS | Nets `n2895`, `n3420`, and `n1294` exceeded their capacitance limits. Net `n2895` showed 8.01 against a limit of 8.00. | We saved a checkpoint before repair. Inspect the drivers, fanout, and wire loads, select a suitable repair, and recheck timing and capacitance. |
| ICC2 routing compatibility | ICC2 U-2022.12-SP3 rejected `route.detail.drc_convergence_mode` and stopped before `route_auto`. | Review clock optimization, correct the options for the installed release, then complete routing, extracted timing, physical checks, and GDSII export. |

## Project files

| File | Purpose |
|---|---|
| [rtl/fft8_spi.v](rtl/fft8_spi.v) | FFT storage, butterfly arithmetic, SPI interface, and controller |
| [tb/tb_fft8_spi.v](tb/tb_fft8_spi.v) | SPI stimulus, expected output values, comparisons, and simulation timeout |
| [syn/synth.tcl](syn/synth.tcl) | Design Compiler synthesis and report generation |
| [const/fft8_spi.sdc](const/fft8_spi.sdc) | Exported synthesis timing constraints |
| [pd1/p1.tcl](pd1/p1.tcl) | ICC2 physical implementation flow used for the recorded run |

## Run the RTL tests

Run these commands from the repository root in a Linux environment with Synopsys VCS available:

~~~bash
mkdir -p sim
cd sim
vcs -full64 -sverilog ../rtl/fft8_spi.v ../tb/tb_fft8_spi.v -top tb_fft8_spi -o simv -l compile.log
./simv -l simulation.log
~~~

The expected completion message is `PASS: all FFT/SPI tests passed`.

The ASIC flow scripts reference the lab's SAED32 libraries under `/opt/PDK`. Review those paths when using another installation. To resume physical implementation, reopen the saved NDM checkpoint. Rerunning the initialization section of `pd1/p1.tcl` deletes the existing `fft8_spi.ndm` design library.
