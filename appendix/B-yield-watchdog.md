# Cooperating with the Scheduler: Yield and Watchdogs Across Four Environments

`yield()` looks like the most portable, boring function in embedded C. It isn't.

While developing the [CS123x](https://github.com/FMazz97/CS123x) library across Arduino and ESP-IDF, a seemingly trivial translation of `yield()` ultimately caused a real watchdog reset on hardware. Investigating that failure revealed a broader lesson: across the four environments considered in this guide, `yield()` (or its closest equivalent) means four genuinely different things.

---

## Why it matters here specifically

Many embedded drivers contain blocking wait loops: polling a ready signal, waiting for a conversion to complete, or waiting for a peripheral state change. Left completely unchecked, these loops can monopolize the CPU for extended periods, starves whatever else the MCU needs to do in the background.
On some platforms that's cosmetic (nothing bad happens, just impolite to see); on others, it's fatal (a watchdog resets the whole device). The fix has to be *specific* to what each environment actually needs, not a single call assumed to behave the same everywhere.

---

## The four environments tested, and what "yielding" actually does in each

**AVR (Uno, Nano, Mega) — no scheduler at all.** `setup()` and `loop()` *is* the entire program; there's nothing else waiting for CPU time. `yield()` is a weak, empty stub provided purely for source-level compatibility with sketches written against boards that do have one. Calling it costs a few clock cycles and does nothing. No watchdog risk exists here by construction.

**ESP8266 (Arduino core) — a cooperative event loop, not a real RTOS.** The underlying NON-OS SDK needs periodic control handed back to service Wi-Fi/TCP-IP housekeeping and reset the **Soft WDT** (a software watchdog with a default timeout in the low single-digit seconds). `yield()` here does real, necessary work — skipping it for too long resets the device.

**Arduino-ESP32 — FreeRTOS underneath, but `yield()` isn't what it looks like.** Reading the core's own source (`cores/esp32/main.cpp`) settles what documentation doesn't state plainly: `yield()` is implemented as `vTaskDelay(0)` — not a lightweight reschedule, but a real (if zero-length) delay call that hands control to the FreeRTOS scheduler in a way that allows lower-priority tasks to actually run.

**Native ESP-IDF — same FreeRTOS, no Arduino core wrapping it.** The obvious analogue, `taskYIELD()`, is **not equivalent** to what Arduino's `yield()` does, and this is the one that caused a real failure.

---

## The bug: `taskYIELD()` starves the IDLE task

The first native-IDF HAL used `taskYIELD()` as the direct translation of Arduino's `yield()`. On hardware, this reliably triggered the **Task Watchdog** at almost exactly 5 seconds after boot — the default `CONFIG_ESP_TASK_WDT_TIMEOUT_S`. The watchdog log named the stalled task explicitly: `IDLE0` had never been scheduled once since startup.

**The mechanism:** `taskYIELD()` only requests a re-election among tasks of **equal or higher** priority than the caller. `IDLE0` runs at the lowest possible priority — `taskYIELD()` gives it no path to ever be chosen, so it never runs, and never feeds its own watchdog check.

**The fix:** `vTaskDelay(1)`, which forces the calling task into a genuinely blocked state for at least one tick, which *is* enough to let the scheduler drop down to `IDLE0`. This is, not coincidentally, functionally closer to what Arduino-ESP32's `vTaskDelay(0)` already does — the native-IDF HAL ended up converging on the same mechanism Arduino had been quietly relying on the whole time, once the actual cause was traced instead of assumed.

```cpp
// cs123x_hal_espidf.h
// vTaskDelay(1), not taskYIELD(): taskYIELD() only reschedules among tasks of
// equal/higher priority and never guarantees the IDLE task actually runs,
// starving it and tripping the Task Watchdog (confirmed on hardware — IDLE0
// never scheduled, TWDT fired at ~5s). Arduino-ESP32's own yield() avoids this
// the same way, calling vTaskDelay(0) internally (see cores/esp32/main.cpp).
#define CS123X_YIELD() vTaskDelay(1)
```

```cpp
// cs123x_hal_arduino.h
#if defined(ARDUINO_ARCH_ESP32)
#define CS123X_YIELD() vTaskDelay(1)   // same FreeRTOS starvation risk as native IDF
#else
#define CS123X_YIELD() yield()         // AVR: no-op. ESP8266: real, necessary cooperation.
#endif
```

---

## The next problem: `vTaskDelay(1)` isn't free, and "always yield" isn't safe either

A FreeRTOS tick is not one millisecond by definition — it's `1000 / configTICK_RATE_HZ`, commonly **10ms** at the default 100Hz tick rate, and FreeRTOS rounds *up* to the next tick boundary, so a single `vTaskDelay(1)` can cost close to **20ms** in the worst case. Calling it unconditionally, on every iteration of a polling loop, is catastrophic at high output data rates: at 1280Hz the chip's entire conversion cycle is under 1ms — a single yield call can cost 20× the wait it was meant to interrupt.

This produces two separate, complementary throttles, tuned against the chip's own timing rather than a fixed guess.

- **Inside `cs123x::wait_ready()`** — stay in tight polling while the wait is still within a normal conversion window (chip should answer within roughly 2ms across all supported rates), and only start yielding periodically once that window has passed without a response — which is exactly the situation (an unusually slow or unresponsive chip) where ceding control actually matters:

    ```cpp
    bool cs123x::wait_ready(bool status) {
        uint32_t timeout = get_timeout_ms();
        uint32_t start = CS123X_MILLIS();
        uint32_t last_yield = start;
        while (is_ready() != status) {
            uint32_t now = CS123X_MILLIS();
            uint32_t elapsed = now - start;
            if (elapsed >= timeout) return false;
            // Tight polling below ~2ms (normal conversion window at every
            // supported rate) — yielding here costs more than the wait itself.
            // Past that, resume yielding until timeout: something is likely
            // wrong, and the watchdog must stay fed for the rest of the wait.
            if ((now - last_yield >= 2) && (timeout - elapsed > 50)) {
                CS123X_YIELD();
                last_yield = CS123X_MILLIS();
            }
        }
        return true;
    }
    ```

- **Inside `read_average()`** — a second, independent throttle on the *outer* loop, needed only at 640/1280Hz. At those rates, each individual `wait_ready()` call resolves within its own normal conversion window and never triggers the throttle above — so a very long average (thousands of samples) could otherwise run for tens of seconds without a single yield anywhere in the call stack, since the inner throttle's "nothing looks wrong" case is exactly the case being hit over and over:

    ```cpp
    int32_t cs123x::read_average(uint16_t samples) {
        // ... init local variables ...
        bool need_yield = (_rate == CS123X_RATE_640Hz || _rate == CS123X_RATE_1280Hz);
        uint32_t last_yield = CS123X_MILLIS();
        for (uint16_t i = 0; i < samples; ++i) {
            int32_t raw = read();
            // ... accumulate ...
            if (need_yield && (CS123X_MILLIS() - last_yield >= 1000)) {
                CS123X_YIELD();
                last_yield = CS123X_MILLIS();
            }
        }
        // ... return average
    }
    ```

  At 10/40Hz this outer throttle is skipped entirely: `wait_ready()`'s own wider window already yields on nearly every call at those rates, and adding a second throttle on top would be pure overhead for no additional safety.

Finally, the per-rate timeout table itself needed a margin added specifically to absorb the tick-rounding cost above — without it, a single yield firing near a rate/gain switch could itself push the elapsed time past a timeout that was sized only for the chip's own datasheet timing, misreporting a hardware failure that was actually just scheduler overhead:

```cpp
// 640Hz/1280Hz timeouts include a ~20–30ms margin above the datasheet's own
// settling time, specifically to absorb the worst-case cost of a single
// CS123X_YIELD() call on FreeRTOS (tick-boundary rounding, ~20ms at the
// default 100Hz tick rate) — without it, a yield firing during a rate/gain
// switch could itself exceed the timeout and be misreported as a chip failure.
static constexpr uint32_t timeouts[4] = {350, 125, 55, 53};  // 10/40/640/1280 Hz
```

## The generalizable lesson

`yield()` — or whatever a given environment calls its equivalent — is never a portable no-op abstraction to translate mechanically from one platform to another. Each of the four environments above has a **different actual mechanism** underneath (no-op, cooperative event loop, `vTaskDelay(0)`, raw FreeRTOS primitives), a **different cost model** (free, cheap-but-necessary, tick-rounded and potentially expensive), and a **different failure mode** when got wrong (nothing, a Soft WDT reset, a Task WDT reset with a named stalled task). None of this is discoverable by reading a single platform's documentation in isolation — it only surfaced by testing the same code path on real hardware across all four, and by reading the reference implementation's own source when its documented behavior didn't fully explain what was observed.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](../chapters/01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](../chapters/02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](../chapters/03-agnostic-core_hal-architecture.md)
- [Chapter 4 - Example conventions](../chapters/04-examples.md)
- [Chapter 5 - Publishing](../chapters/05-publishing.md)
- [Appendix A - IDF dependency resolution](A-esp-idf-dependency-resolution.md)
- [Appendix B - Yield & Watchdog](B-yield-watchdog.md)
- [Appendix C - Naming Convention](C-naming-convention.md)
