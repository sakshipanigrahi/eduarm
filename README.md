# RGB LED, Stopwatch & Menu-Driven Control System (TM4C123)


## System overview

Controls an RGB LED's color (8 options) and blinking speed (8 levels), and adds a SysTick-driven stopwatch. Input comes from three sources: the Tiva onboard switches SW1/SW2 (hardware interrupt, Port F), a 4x4 keypad (hardware interrupt, Port C), and a UART command line. Output is a 4-digit 7-segment display (multiplexed) that shows either RGB status or stopwatch time, depending on whether the stopwatch is enabled.

## Pin map

**Tiva onboard switches**
- `PF4` = SW1
- `PF0` = SW2

**RGB LED**
- `PF1` = RED
- `PF2` = BLUE
- `PF3` = GREEN

**7-segment display**
- `PA4`–`PA7` = digit select (digit 0–3)
- `PB0`–`PB7` = segments (`PB7` = decimal point)

**4x4 keypad (EduARM4)**
- `PC4`–`PC7` = columns 0–3 (interrupt, falling edge)
- `PE0`–`PE3` = rows 0–3 (open-drain outputs)

## Controls

**Tiva onboard SW1 (PF4) / SW2 (PF0)** — interrupt-driven, `GPIOPortF_Handler`:
- SW1 alone → next speed
- SW2 alone → next colour
- SW1 + SW2 → toggle RGB blink / continuous-on

Behaviour is unchanged from the non-interrupt version — only the mechanism of detecting the press changed.

**4x4 keypad** — interrupt-driven, `GPIOPortC_Handler` + `keypadScan()`:
- Key 1 (row 1, col 1) → stopwatch ENABLE / DISABLE (toggle)
- Key 2 (row 1, col 2) → stopwatch START / STOP (+reset)
- Key 3 (row 1, col 3) → stopwatch PAUSE / RESUME
- All other keys do nothing

**UART** — type the word or its shortcut, then press Enter. Leading/trailing spaces around the word are ignored.

| Command | Shortcut | Effect |
|---|---|---|
| `color` / `colour` | `c` | Next colour |
| `rate` | `r` | Next speed |
| `t` | — | Toggle RGB blink / continuous-on |
| `enable` | `e` | Stopwatch ENABLE / DISABLE (toggle) |
| `disable` | `d` | Stopwatch DISABLE (forced, regardless of state) |
| `start` | — | Stopwatch START / STOP (+reset) |
| `pause` | — | Stopwatch PAUSE / RESUME |
| `h` | — | Print the command list |

## 7-segment display layout

**Stopwatch DISABLED** — shows RGB status (digit 0 = leftmost = `PA4`):

| Digit | Pin | Shows |
|---|---|---|
| 0 | `PA4` | `R` (running/blinking) or `P` (paused/continuous-on) |
| 1 | `PA5` | Colour index, 0–7 |
| 2 | `PA6` | `S` (label) |
| 3 | `PA7` | Speed index, 1–8 |

**Stopwatch ENABLED** — shows elapsed time as `SS.CC` (seconds.hundredths):

| Pin | Shows |
|---|---|
| `PA7` | Seconds, tens digit |
| `PA6` | Seconds, units digit (decimal point lit here) |
| `PA5` | Hundredths, tens digit |
| `PA4` | Hundredths, units digit |

## LED behaviour

**Colours** (colour index 0–7): `0` OFF · `1` RED · `2` BLUE · `3` GREEN · `4` MAGENTA · `5` YELLOW · `6` CYAN · `7` WHITE

**Speeds** (speed index 1–8): `1` slowest → `8` fastest. Timed by main-loop pass count, so this is an approximate rate, not a fixed real-time interval.

**Logic**
- `state = 0` → LED blinks on/off at the selected speed
- `state = 1` → LED stays continuously on at the selected colour

## Stopwatch behaviour

Timed by `SysTick_Handler`, which fires every 1 ms and increments `stopwatchMs` whenever the stopwatch is enabled, running, and not paused. Wraps back to 0 after 59.99 seconds.

- **ENABLE / DISABLE** (toggle) — Enable: stopwatch turns on, resets to `0:00`, not yet running. Disable: stopwatch turns off, resets to `0:00`.
- **START / STOP** (toggle, only has effect if enabled) — Start: begins counting up from `0:00`. Stop: stops counting **and** resets back to `0:00`.
- **PAUSE / RESUME** (toggle, only has effect if running) — Pause: freezes the display at the current value. Resume: continues counting up from where it was paused.

Display automatically switches to stopwatch mode while enabled, and back to RGB status once disabled.

## Startup file changes (`Startup_Lab4_Extended.c`)

Based on the unedited `Actual_tiva_startup_file`, with exactly two kinds of change:

**1. Three `extern` declarations added**, so the vector table can reference functions defined in `Lab4_Extended_Stopwatch.c`:

```c
extern void SysTick_Handler(void);
extern void GPIOPortC_Handler(void);
extern void GPIOPortF_Handler(void);
```

**2. Three vector table entries repointed** from the default handler (`IntDefaultHandler`, an infinite empty loop) to the real handlers:

| Vector table slot | Before | After |
|---|---|---|
| The SysTick handler | `IntDefaultHandler` | `SysTick_Handler` |
| GPIO Port C | `IntDefaultHandler` | `GPIOPortC_Handler` |
| GPIO Port F | `IntDefaultHandler` | `GPIOPortF_Handler` |

Every other vector table entry, the stack size, and every other function in the file are unchanged from the original. Only these three interrupt sources are armed in `main()` (SysTick, GPIO Port C, GPIO Port F), so only these three entries needed changing — any slot left pointing at `IntDefaultHandler` corresponds to an interrupt this program never enables.

