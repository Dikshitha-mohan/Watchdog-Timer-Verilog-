# Watchdog Timer (Verilog)

A synthesizable RTL watchdog timer (WDT) that monitors a system for unresponsiveness and asserts a latched timeout signal if it isn't "fed" (kicked) within a configurable window — a core building block in embedded and safety-critical digital systems.

## Overview

A watchdog timer is a hardware safety mechanism used to detect and recover from system hangs or software faults. Once armed, it continuously counts clock cycles; the monitored system must periodically send a **kick** (feed) pulse to reset the counter before it reaches its limit. If the kick doesn't arrive in time, the watchdog **times out** — typically triggering a system reset or interrupt in real hardware.

This project implements a parameterized watchdog counter with enable/disable control, a feed (kick) input, and a latched timeout output, verified against multiple operating scenarios in a self-checking testbench.

## Features

- **Configurable timeout window** — set via the `TIMEOUT_LIMIT` parameter (number of clock cycles allowed between kicks).
- **Enable/disable control** (`wdt_en`) — counter holds at zero and timeout stays clear while disabled.
- **Kick/feed input** (`kick`) — resets the counter and clears any active timeout.
- **Latched timeout output** (`timeout`) — once asserted, stays high until the watchdog is kicked again or disabled, mirroring real hardware behavior where a timeout must be explicitly acknowledged/cleared.
- **Debug counter output** (`count`) — exposes the live counter value for observation in simulation or hardware debug.
- **Self-checking testbench** covering:
  - No timeout while disabled
  - No timeout while regularly fed
  - Correct timeout when starved of kicks
  - Timeout clears on kick
  - Timeout clears on disable

## Files

| File | Description |
|---|---|
| `watchdog_timer.v` | RTL design — counter, enable logic, latched timeout |
| `tb_watchdog_timer.v` | Self-checking testbench covering 5 test scenarios |

## Module Ports

**`watchdog_timer`**

| Port | Direction | Width | Description |
|---|---|---|---|
| `clk` | input | 1 | System clock |
| `rst_n` | input | 1 | Active-low async reset |
| `wdt_en` | input | 1 | Enable/arm the watchdog |
| `kick` | input | 1 | Pulse to feed/reset the watchdog |
| `timeout` | output | 1 | Latched high on timeout, cleared by kick or disable |
| `count` | output | 32 | Current counter value (debug/observation) |

**Parameter:** `TIMEOUT_LIMIT` — number of consecutive clock cycles allowed without a kick before timeout asserts. In simulation this is set low (20) for fast test runs; in real hardware, it's sized to the desired watchdog window (e.g., `TIMEOUT_LIMIT` = clock_freq × timeout_seconds, often 1–2 seconds).

## How It Works

1. **Arming** — while `wdt_en` is low, the counter is held at 0 and `timeout` is forced low, effectively disabling monitoring.
2. **Counting** — once enabled, the counter increments every clock cycle.
3. **Feeding** — a `kick` pulse resets the counter to 0 and clears `timeout`, signaling the monitored system is alive.
4. **Timeout** — if the counter reaches `TIMEOUT_LIMIT - 1` without a kick, `timeout` asserts and **latches high**, holding until the next kick or until the watchdog is disabled.

## Simulation

Tested on **EDA Playground** using **Icarus Verilog 12.0**.

1. Place the design file in the **Design** pane and the testbench in the **Testbench** pane.
2. Select **Icarus Verilog 12.0** as the simulator.
3. Run the simulation — check the console for the 5-test PASS/FAIL sequence.
4. Open **EPWave** → click **Get Signals** → select `clk`, `rst_n`, `wdt_en`, `kick` (top-level scope) and `timeout`, `count` (`.dut` scope) to visualize the counter ramping, resetting on each kick, and latching on timeout.

## Example Console Output

```
---- Test 1: Watchdog disabled ----
PASS: no timeout while WDT disabled (count=0)
---- Test 2: Watchdog enabled, fed regularly ----
PASS: no timeout while regularly fed (count=0)
---- Test 3: Watchdog enabled, not fed (expect timeout) ----
PASS: timeout correctly asserted after 20 cycles
---- Test 4: Kick clears timeout ----
PASS: timeout cleared after kick
---- Test 5: Disable clears timeout ----
Timeout occurred again at count=20, now disabling WDT
PASS: timeout cleared after disabling WDT
ALL TESTS PASSED
```

## Possible Extensions

- Add a **windowed watchdog** mode (rejects kicks that arrive too early, not just too late)
- Drive an actual `system_reset` output on timeout instead of just a status flag
- Add a **timeout interrupt** with separate acknowledge logic distinct from the kick path
- Port to an FPGA dev board and tie `timeout` to an LED, with a physical button as `kick`, for hardware validation

## Author
Dikshitha M
