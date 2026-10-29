# Appendix B — Yield & Watchdog: Cooperating with the Scheduler

`yield()` looks like the most portable, boring function in embedded C. It isn't.

While developing the [CS123x](https://github.com/FMazz97/CS123x) library across Arduino and ESP-IDF, a seemingly trivial translation of `yield()` caused a real watchdog reset on hardware. The broader lesson: any function that can block for a long time (averaging thousands of ADC readings is the typical case) has to cooperate with the operating system underneath, and what "cooperating" means differs on every platform. Left unchecked, such loops starve whatever else the MCU has to do: on some platforms nothing bad happens, on others a watchdog resets the whole device.

> [!IMPORTANT]
> A long blocking function must not assume it owns the CPU. It should periodically hand control back to the platform, throttled by elapsed time and tuned to what the platform actually needs, never blindly on every iteration.

---

## The four environments considered

**AVR (Uno, Nano, Mega) — no scheduler.** `setup()` and `loop()` *are* the whole program, and `yield()` is a weak, empty stub kept for source-level compatibility. The Arduino AVR core enables no watchdog by default.

**ESP8266 (Arduino core) — a cooperative event loop, not a real RTOS.** The NON-OS SDK needs control handed back periodically to service Wi-Fi/TCP-IP and feed the **Soft WDT** (a software watchdog with a timeout in the low single-digit seconds), so `yield()` does real, necessary work.

**Arduino-ESP32 — FreeRTOS underneath, and `yield()` is only a reschedule request.** In the core's source it is a thin wrapper around `vPortYield()` (`esp32-hal-misc.c`): the caller stays ready, so lower-priority tasks are not guaranteed to run. The core knows this: on single-core configurations (`CONFIG_FREERTOS_UNICORE`) its own main loop calls `yieldIfNecessary()`, which blocks with `vTaskDelay(5)` at most once every 2 seconds. A long blocking call inside `loop()` cannot reach that code until it returns.

**Native ESP-IDF — same FreeRTOS, no Arduino core.** The obvious analogue of `yield()` is `taskYIELD()`, which is also a reschedule request. This is the one that caused the real failure.

| Environment | What `yield()` (or its equivalent) does | Cost | If omitted for too long |
| :--- | :--- | :--- | :--- |
| **AVR** | nothing (empty weak stub) | a few cycles | nothing (no watchdog by default) |
| **ESP8266** | hands control back to the SDK's event loop | cheap, necessary | Soft WDT reset |
| **Arduino-ESP32** | reschedule request (`vPortYield()`) | cheap, but lower-priority tasks are not guaranteed to run | Task WDT, if the starved idle task is monitored |
| **Native ESP-IDF** | `taskYIELD()`: reschedule request; `vTaskDelay(1)`: blocks the caller | reschedule cheap; blocking up to one tick | Task WDT, naming the stalled idle task |

---

## The bug: a reschedule request starves the IDLE task

The first native-IDF HAL used `taskYIELD()` as the direct translation of Arduino's `yield()`, a faithful one, since both are reschedule requests. On hardware it triggered the **Task Watchdog** almost exactly 5 seconds after boot (the default `CONFIG_ESP_TASK_WDT_TIMEOUT_S`), and the log named the stalled task: `IDLE0` had never been scheduled.

**The mechanism:** a reschedule request leaves the caller ready, so the scheduler keeps choosing among ready tasks of equal or higher priority. `IDLE0` runs at the lowest priority, is never chosen, and never feeds its watchdog check. The same holds for `vTaskDelay(0)`: the FreeRTOS kernel source itself states that a zero delay "just forces a reschedule" (`tasks.c`).

**The fix:** `vTaskDelay(1)`, which blocks the caller until the next tick and lets the scheduler run lower-priority tasks, `IDLE0` included.

```cpp
// cs123x_hal_espidf.h
// vTaskDelay(1), not taskYIELD(): a reschedule request never lets the lower-priority IDLE task run,
// starving it and tripping the Task Watchdog (observed: IDLE0 never scheduled, TWDT at ~5 s).
#define CS123X_YIELD() vTaskDelay(1)
```

```cpp
// cs123x_hal_arduino.h
#if defined(ARDUINO_ARCH_ESP32)
#define CS123X_YIELD() vTaskDelay(1)   // yield() is only a reschedule request here: same risk as native IDF
#else
#define CS123X_YIELD() yield()         // AVR: no-op. ESP8266: real, necessary cooperation.
#endif
```

Whether a missing yield actually trips a watchdog also depends on configuration (which core the code runs on, which idle tasks the Task Watchdog monitors, what the platform does between iterations), and a library cannot assume any of it.

---

## Cooperation has a cost: when and how often to yield

A blocking yield is not free: `vTaskDelay(1)` returns at the next tick, so it can cost up to one tick period (10 ms at the ESP-IDF default of 100 Hz), plus any time other ready tasks keep the CPU. For a chip whose conversion takes under a millisecond at its highest rate, an unconditional yield in the polling loop would cost more than the wait itself. Four rules follow:

1. **Yield only when the wait is long** compared to the cost of yielding.
2. **Throttle by elapsed time**, not by iteration count.
3. **Keep the throttle interval well below the watchdog timeout.**
4. **Never let a yield turn a healthy wait into a timeout.**

CS123x applies them with two independent throttles.

- **Inside `cs123x::wait_ready()`**: tight polling during the normal conversion window, then periodic yields, but only while ample time remains before the timeout:

  ```cpp
  bool cs123x::wait_ready(bool status) {
      uint32_t timeout = get_timeout_ms();
      uint32_t start = CS123X_MILLIS();
      uint32_t last_yield = start;
      while (is_ready() != status) {
          uint32_t now = CS123X_MILLIS();
          uint32_t elapsed = now - start;
          if (elapsed >= timeout) return false;
          // Poll tightly for the first ~2 ms (the whole normal conversion window at
          // 640/1280 Hz), then yield periodically, but only while more than 50 ms of
          // the timeout remain, so a yield can never push the wait past its timeout.
          if ((now - last_yield >= 2) && (timeout - elapsed > 50)) {
              CS123X_YIELD();
              last_yield = CS123X_MILLIS();
          }
      }
      return true;
  }
  ```

  The 50 ms guard is rule 4 in practice: a yield lasts at most one tick period, so as long as the tick is well below 50 ms it cannot, by itself, cause a timeout. At 10/40 Hz a conversion lasts 100/25 ms, so yielding past the 2 ms window happens by design.

- **Inside `read_average()`**: a second throttle on the *outer* loop, needed only at 640/1280 Hz, where every individual `wait_ready()` resolves within its normal window and never yields, so a long average (thousands of samples) could run for tens of seconds with no yield anywhere in the call stack:

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

  The 1s interval stays well below the default 5s watchdog timeout (rule 3), at a cost of at most one tick per second, about 1% at 100 Hz. At 10/40 Hz the throttle is skipped: `wait_ready()` already yields on nearly every call.

---

## The generalizable lesson

Three different mechanisms hide behind one familiar name, each with its own cost model and failure mode when misused. The failure was observed on native ESP-IDF; what explained it was not a single platform's documentation but the FreeRTOS kernel and Arduino core sources, read to account for what the hardware had shown.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Appendix A - IDF Dependency Resolution](A-esp-idf-dependency-resolution.md) — [Introduction](../README.md) — [Appendix C - Naming Convention ➡️](C-naming-convention.md)

</div>
<!-- markdownlint-enable MD033 -->
