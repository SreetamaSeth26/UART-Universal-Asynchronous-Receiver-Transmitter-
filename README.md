# UART (Universal Asynchronous Receiver/Transmitter)

A UART transceiver written in Verilog HDL. The design generates its own baud-rate clock from a system clock, transmits 8-bit data serially with start/stop framing, and receives serial data back into parallel form using 16x oversampling.

## Files

| File | Description |
|---|---|
| `uart.v` | Design source: `baud_rate_genrator`, `transmitter`, `receiver`, and the top-level `uart_top` |
| `uart_tb.v` | Testbench for `uart_top`, sends three bytes through the transmitter and checks them at the receiver |

## Module Overview

**baud_rate_genrator**
Divides the system clock down to two enable pulses: `tx_enable` at the baud rate, and `rx_enable` at 16x the baud rate (for oversampled receiver sampling). With `clk_freq = 100_000_000` and `baud_rate = 9600`, this gives `tx_divisor = 10416` and `rx_divisor = 651`.

**transmitter**
A 4-state Moore FSM that loads a byte on `wr_enable`, then shifts it out on `tx` one bit at a time, framed with a start bit (0) and a stop bit (1). `busy` is high whenever the FSM is outside `IDLE_STATE`.

| State | Behavior |
|---|---|
| `IDLE_STATE` | `tx` held high. Loads `data_in` into `data` and moves to `START_STATE` when `wr_enable` is asserted |
| `START_STATE` | Drives `tx` low (start bit) on the next `enable` pulse, moves to `DATA_STATE` |
| `DATA_STATE` | Shifts out `data[index]` on each `enable` pulse, advancing `index` from 0 to 7, then moves to `STOP_STATE` |
| `STOP_STATE` | Drives `tx` high (stop bit) on `enable`, returns to `IDLE_STATE` |

**receiver**
A 3-state FSM that oversamples the incoming `rx` line at 16x the baud rate to find the middle of each bit period, then reconstructs the byte.

| State | Behavior |
|---|---|
| `START_STATE` | Counts consecutive low samples on `rx`; once 16 samples confirm a start bit, moves to `DATA_OUT_STATE` |
| `DATA_OUT_STATE` | Samples `rx` at the midpoint (sample count 8) of each bit period, capturing 8 bits into `temp_register`, then moves to `STOP_STATE` |
| `STOP_STATE` | Waits out the stop bit period, latches `temp_register` into `data_out`, raises `ready`, and returns to `START_STATE` |

`ready` is cleared externally via `ready_clear`, decoupling the consumer's read timing from the receiver FSM.

**uart_top**
Wires the three blocks together: `baud_rate_genrator` drives both FSMs, the transmitter's `tx` output loops directly into the receiver's `rx` input (loopback), and `data_in`/`data_out` form the parallel-side interface.

## Simulation

Simulated with Icarus Verilog.

```
iverilog -Wall -g2012 -o sim uart.v uart_tb.v
vvp sim
```

The testbench resets the design, then sends three bytes (`0x41`, `0x55`, `0xAA`) through the transmitter in loopback, waiting on `busy` and `ready` between each one and clearing `ready` before the next send. Output:

```
Received data = 41
Received data = 55
Received data = aa
```

All three bytes round-trip correctly through the transmit/receive loopback.

## Synthesis (Yosys)

Synthesized with Yosys, mapped to generic logic cells.

```
yosys -p "read_verilog uart.v; hierarchy -top uart_top; proc; opt; techmap; opt; stat"
```

Resulting cell counts for the full hierarchy:

| Module | Cells | Notes |
|---|---|---|
| `baud_rate_genrator` | 155 | 34 flip-flops (two 16-bit counters), AND/OR/XOR/NOT logic for the divider compares |
| `transmitter` | 106 | 14 flip-flops (state, index, data, tx), muxes for the FSM datapath |
| `receiver` | 281 | 27 flip-flops (state, sample, index, data_out, temp_register), the largest block due to oversampled sampling logic |
| `uart_top` | 3 | submodule instances only (hierarchy not flattened) |
| **Total** | **542** | 226 wires / 971 wire bits across the design |

To produce a gate-level netlist file:

```
yosys -p "read_verilog uart.v; hierarchy -top uart_top; proc; opt; fsm; opt; memory; opt; techmap; opt; write_verilog uart_synth_netlist.v; write_json uart_netlist.json"
```

To visualize the top-level block connectivity with netlistsvg:

```
yosys -p "read_verilog uart.v; hierarchy -top uart_top; proc; opt; write_json uart_top_level.json"
netlistsvg uart_top_level.json -o uart_top_netlist.svg
```

This renders `uart_top` as three connected blocks (`baud_rate_genrator`, `transmitter`, `receiver`), showing the loopback wiring between the transmitter's `tx` and the receiver's `rx`.

## Notes

- The transmitter and receiver share no clock divider logic of their own; both rely on `baud_rate_genrator` for timing, which keeps the FSMs purely behavioral.
- `rx` is internally looped back to `tx` in `uart_top`, so this design is self-contained for simulation. Driving `uart_top` from an external pin would require breaking that loopback and exposing `tx`/`rx` as separate top-level ports.
- 16x oversampling in the receiver (rather than sampling once per bit) is what makes the receiver FSM the largest block in the design, it trades extra flip-flops and compare logic for tolerance to small clock-rate mismatches with the transmitting end.
