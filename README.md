# Multi-Clock UART System with Asynchronous FIFO

A fully synthesizable, dual-clock-domain **UART transceiver system** implemented in Verilog.  
Data is received on a fast RX clock, buffered through a **Gray-coded asynchronous FIFO** for safe clock-domain crossing (CDC), and re-transmitted on an independent TX clock — demonstrating industry-standard CDC practices in a complete, working datapath.

---

## Architecture Overview

```
                    RX Clock Domain                       TX Clock Domain
               ┌───────────────────────┐            ┌───────────────────────┐
  rx_in ──────►│       UART_RX         │            │       UART_TX         │──────► tx_out
               │  ┌─────────────────┐  │            │  ┌─────────────────┐  │
               │  │ RX_FSM          │  │  ┌──────┐  │  │ TX_FSM          │  │
               │  │ data_sampling   │  │  │Async │  │  │ serializer      │  │
               │  │ edge_bit_counter│  │  │FIFO  │  │  │ parity_calc     │  │
               │  │ deserializer    │──┼─►│      │──┼─►│ TX_mux          │  │
               │  │ parity_check    │  │  │ CDC  │  │  │                 │  │
               │  │ start_check     │  │  │Bridge│  │  │                 │  │
               │  │ stop_check      │  │  └──────┘  │  └─────────────────┘  │
               └───────────────────────┘            └───────────────────────┘
```

The **system_top** module wires the three major IP blocks together:

1. **UART_RX** — receives serial data, validates framing, and outputs parallel bytes.
2. **Async FIFO** — bridges the RX and TX clock domains using Gray-code pointer synchronization.
3. **UART_TX** — reads from the FIFO and transmits serial frames on the TX clock.

---

## Features

| Feature | Details |
|---|---|
| **Data Width** | Parameterized (default 8-bit) |
| **Oversampling** | Configurable prescale — 8×, 16×, or 32× |
| **Parity** | Even / Odd / None — runtime selectable |
| **Error Detection** | Start-bit glitch, parity error, and stop-bit error flags |
| **CDC** | Gray-coded pointers with double-flop synchronizers (`DF_Sync`) |
| **FIFO Depth** | Parameterized via `FIFO_P_WIDTH` (default depth = 8) |
| **Full / Empty Flags** | Safe generation in respective clock domains |
| **Synthesis** | Verified on **TSMC 130 nm** (Synopsys Design Compiler Ultra) |
| **Lint** | Clean SpyGlass reports (lint, CDC, clock-reset, STARC) |

---

## Repository Structure

```
.
├── UART_RX/                    # UART Receiver (RX clock domain)
│   ├── UART_RX.v               #   Top-level RX wrapper
│   ├── RX_FSM.v                #   Receiver state machine
│   ├── data_sampling.v         #   Mid-bit majority-vote sampler
│   ├── edge_bit_counter.v      #   Oversampling edge & bit counters
│   ├── deserializer.v          #   Serial-to-parallel shift register
│   ├── parity_check.v          #   Parity error checker
│   ├── start_check.v           #   Start-bit glitch detector
│   ├── stop_check.v            #   Stop-bit error detector
│   └── UART_RX_tb.v            #   Unit-level RX testbench
│
├── Async_FIFO/                 # Asynchronous FIFO (CDC bridge)
│   ├── async_fifo.v            #   Top-level FIFO wrapper
│   ├── fifo_mem.v              #   Dual-port register-file memory
│   ├── fifo_wr.v               #   Write pointer & full-flag generator
│   ├── fifo_rd.v               #   Read pointer & empty-flag generator
│   ├── DF_Sync.v               #   Double-flop synchronizer (generic)
│   └── async_fifo_tb.v         #   Unit-level FIFO testbench
│
├── UART_TX/                    # UART Transmitter (TX clock domain)
│   ├── UART_TX.v               #   Top-level TX wrapper
│   ├── TX_FSM.v                #   Transmitter state machine
│   ├── serializer.v            #   Parallel-to-serial shift register
│   ├── parity_calc.v           #   Parity bit generator
│   ├── TX_mux.v                #   Output multiplexer (start/data/parity/stop)
│   └── UART_TX_tb.v            #   Unit-level TX testbench
│
├── System_Top/                 # Full system integration
│   ├── system_top.v            #   Top-level instantiation (RX → FIFO → TX)
│   ├── system_top_tb.v         #   End-to-end loopback testbench
│   ├── run.do                  #   ModelSim/QuestaSim automation script
│   └── wave.do                 #   Pre-configured waveform viewer setup
│
├── Synthesis/                  # Synopsys Design Compiler output
│   ├── syn/
│   │   ├── syn_script.tcl      #   DC Ultra synthesis TCL script
│   │   └── syn.log             #   Full synthesis session log
│   ├── Netlist/
│   │   ├── system_top_netlist.v        #   Post-synthesis mapped netlist
│   │   └── system_top_netlist_GTECH.v  #   Generic technology (GTECH) netlist
│   └── Snippets/               #   Design Vision schematic screenshots
│
├── SpyGlass/                   # Synopsys SpyGlass lint & CDC reports
│   ├── UART_TX/                #   TX lint report (clean — 0 violations)
│   ├── UART_RX/                #   RX lint report (clean — 0 violations)
│   └── Async_FIFO/             #   FIFO lint report (clean — 0 violations)
│
└── Presentation/
    └── CDC Project.pdf         #   Project presentation slides
```

---

## Module Hierarchy

### UART Receiver (`UART_RX`)

| Sub-module | Function |
|---|---|
| `RX_FSM` | 6-state FSM (IDLE → START → DATA → PARITY → STOP → VALID) controlling all receiver datapath enables |
| `data_sampling` | Triple-sample majority-vote logic at the center of each bit period — supports 8×, 16×, and 32× oversampling |
| `edge_bit_counter` | Counts oversampling edges within a bit and bits within a frame; resets on FSM `Clear` |
| `deserializer` | Shift register that assembles sampled bits (LSB-first) into a parallel byte |
| `parity_check` | Compares expected vs. received parity at the mid-bit sample point |
| `start_check` | Validates the start bit at mid-point to reject glitches |
| `stop_check` | Validates the stop bit; flags `stp_err` on framing violation |

### Asynchronous FIFO (`async_fifo`)

| Sub-module | Function |
|---|---|
| `fifo_mem` | `2^(P_WIDTH-1)` deep register-based dual-port memory; synchronous write, asynchronous read |
| `fifo_wr` | Binary-to-Gray write pointer conversion and **full** flag generation (compares against synchronized read pointer) |
| `fifo_rd` | Binary-to-Gray read pointer conversion and **empty** flag generation (compares against synchronized write pointer) |
| `DF_Sync` | Parameterized 2-stage flip-flop synchronizer for safe pointer crossing between clock domains |

### UART Transmitter (`UART_TX`)

| Sub-module | Function |
|---|---|
| `TX_FSM` | 5-state FSM (IDLE → START → DATA → PARITY → STOP) driving mux select, serializer enables, and busy flag |
| `serializer` | Load-and-shift register outputting data LSB-first; asserts `ser_done` after 8 bits |
| `parity_calc` | Registered even/odd parity computation — latched on `Data_Valid` |
| `TX_mux` | 4:1 output multiplexer selecting between start bit, serial data, parity bit, and stop bit |

---

## Clock-Domain Crossing Strategy

The design follows the classic **Cummings-style** asynchronous FIFO methodology:

1. **Gray-code pointers** — Write and read pointers are converted to Gray code before crossing domains, ensuring only a single bit changes per increment and eliminating multi-bit metastability hazards.
2. **Double-flop synchronizers** (`DF_Sync`) — Each Gray-coded pointer is passed through a 2-stage synchronizer before being compared for full/empty generation.
3. **Conservative flag generation** — The full flag is generated in the write domain; the empty flag in the read domain. Both use the synchronized (and therefore potentially stale) partner pointer, guaranteeing safe operation with a minor over-report of fullness/emptiness.

---

## Verification

### Unit-Level Testbenches

| Testbench | Coverage |
|---|---|
| `UART_TX_tb.v` | 6 tests — Even parity, odd parity, no parity, edge-case data patterns (`0x00`, `0xFF`, `0x55`, `0xAA`, `0xA5`). Verifies start bit, each data bit, parity bit, and stop bit. |
| `UART_RX_tb.v` | 10 tests — Prescale sweep (8×/16×/32×), even & odd parity, parity error injection, stop error injection, back-to-back frames, reset verification, idle-line stability. |
| `async_fifo_tb.v` | 6 tests — Fill to full, overflow prevention, partial read/write, drain to empty, underflow prevention. Dual-clock (100 MHz write / 40 MHz read). |

### System-Level Loopback Testbench

`system_top_tb.v` performs **end-to-end loopback verification** where serial frames are injected at `rx_in`, pass through the UART RX → Async FIFO → UART TX pipeline, and are captured at `tx_out`:

| Test Category | Cases |
|---|---|
| Prescale sweeps | 8×, 16×, 32× oversampling — no parity |
| Parity hardware checks | Even parity (`0x55`), Odd parity (`0xAA`) |
| Fault injection | Intentional wrong parity — verifies `rx_parity_error` assertion |
| Burst streaming | 3 back-to-back frames (`0x11`, `0x22`, `0x33`) — validates FIFO buffering under continuous load |

**Clock configuration:**
- TX clock: ~115.2 kHz (half-period = 4340.278 ns)
- RX clock: ~921.6 kHz (half-period = 542.535 ns) — 8× oversampling of the TX baud rate

### Running Simulation

```bash
# Using ModelSim / QuestaSim (from System_Top/ directory)
vsim -do run.do
```

The `run.do` script compiles all sources, launches the testbench, loads the pre-configured `wave.do` waveform, and runs all tests to completion. Look for the `TEST SUMMARY REPORT` in the transcript.

---

## Synthesis

Synthesis was performed using **Synopsys Design Compiler Ultra (O-2018.06-SP1)** targeting the **TSMC 130 nm** standard cell library at the worst-case (SS) corner:

| Parameter | Value |
|---|---|
| Technology | TSMC CLN013G (130 nm) RVT |
| Target Library | `scmetro_tsmc_cl013g_rvt_ss_1p08v_125c.db` |
| Optimization | `compile_ultra` with full hierarchy flattening |
| Outputs | Gate-level netlist, GTECH netlist, DDC database |

### Running Synthesis

```tcl
# From Synopsys DC shell
source syn_script.tcl
```

The script generates:
- `system_top_netlist.v` — mapped gate-level netlist
- `system_top_netlist_GTECH.v` — generic technology netlist
- Area, timing, and power reports

---

## Linting & CDC Analysis

All three major IP blocks were analyzed with **Synopsys SpyGlass (vL-2016.06)** across multiple rule decks:

| Rule Deck | UART_TX | UART_RX | Async_FIFO |
|---|---|---|---|
| lint / morelint | ✅ Clean | ✅ Clean | ✅ Clean |
| clock-reset | ✅ Clean | ✅ Clean | ✅ Clean |
| erc | ✅ Clean | ✅ Clean | ✅ Clean |
| latch | ✅ Clean | ✅ Clean | ✅ Clean |
| STARC / STARC2005 | ✅ Clean | ✅ Clean | ✅ Clean |
| simulation | ✅ Clean | ✅ Clean | ✅ Clean |
| timing | ✅ Clean | ✅ Clean | ✅ Clean |

> **0 violations** reported across all modules and all policy checks.

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `DATA_WIDTH` | 8 | Width of the UART data field (bits) |
| `FIFO_P_WIDTH` | 4 | Pointer width for the async FIFO → FIFO depth = 2^(P_WIDTH−1) = **8 entries** |

Both parameters are propagated from `system_top` down through the full hierarchy.

---

## Tools Used

| Tool | Purpose |
|---|---|
| **ModelSim / QuestaSim** | RTL simulation & waveform analysis |
| **Synopsys Design Compiler Ultra** | Logic synthesis (TSMC 130 nm) |
| **Synopsys Design Vision** | Schematic visualization |
| **Synopsys SpyGlass** | Lint, CDC, clock-reset, and STARC analysis |

---

## References

- C. E. Cummings, "Simulation and Synthesis Techniques for Asynchronous FIFO Design," SNUG 2002
- C. E. Cummings & P. Alfke, "Simulation and Synthesis Techniques for Asynchronous FIFO Design with Gray Code Counters," SNUG 2014

---

## License

This project is provided for educational and reference purposes. Feel free to use and modify.
