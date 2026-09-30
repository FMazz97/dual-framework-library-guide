# Chapter 3 — The Agnostic Core, the HAL Layer, and the Arduino Facade

With the repository structure established ([Chapter 2 - A Folder Structure That Satisfies All Three at Once](02-repo-structure.md)), the next challenge is architectural: how do you organize the code so that the protocol logic exists only *once*, while still allowing Arduino and ESP-IDF to use their own native hardware APIs?

The resulting architecture separates responsibilities into three layers:

```text
Foo (Arduino facade, camelCase)
│ forwards every call 1:1, adds nothing
▼
foo (core, snake_case)
│ this IS the public API, also used by ESP-IDF
│ calls FOO_* macros, never Arduino/IDF APIs directly
▼
HAL macros (foo_hal_arduino.h / foo_hal_espidf.h)
resolved at compile time through:
#if defined(ARDUINO)
#elif defined(ESP_PLATFORM)
```

The **core** (styled in `snake_case`, following C++ and ESP-IDF standard) contains all *protocol logic*, *calculations*, *state handling*, and *device behavior*. The **HAL** provides *framework-specific hardware access*, while the **Arduino facade** exists solely to expose the naming style Arduino users typically expect forwarding every public method: `bool classMethod() { return _core.class_method(); }`.

*By isolating framework selection within the HAL, the same protocol implementation runs natively on both Arduino and ESP-IDF using their respective hardware access paths.*

The resulting repository layout becomes:

```text
repo/
├── idf_component.yml    ← ESP-IDF
├── CMakeLists.txt       ← ESP-IDF
├── library.properties   ← Arduino
├── library.json         ← PlatformIO
├── src/
│   ├── Foo.h            ← Arduino facade
│   ├── core/            ← Framework-agnostic core
│   │   ├── foo.h
│   │   └── foo.cpp
│   └── hal/             ← Hardware Abstraction Layer
│       ├── foo_hal_arduino.h
│       └── foo_hal_espidf.h
└── examples/
```

So, *the protocol implementation exists exactly **once***, while each framework retains its own **native hardware implementation**.

---

## 3.1 Why compile-time macros, not virtual dispatch or templates

Before settling on compile-time macros, two standard abstraction mechanisms — *virtual interfaces* and *C++ templates* — were evaluated and rejected. Both introduce overhead that cannot be justified in a low-level hardware-access layer designed to disappear at compile time.

## Virtual dispatch

A classic object-oriented solution defines a HAL interface and provides separate framework implementations behind virtual methods:

```cpp
class IHal {
public:
virtual void pinMode(...) = 0;
virtual void digitalWrite(...) = 0;
};
```

While clean, this approach introduces runtime indirection into the hottest part of the codebase. In tight bit-banging loops or high-frequency register transactions, clock and data lines are toggled repeatedly within strict microsecond-level timing budgets.

A virtual call's indirection (a few extra cycles per invocation) is negligible in high-level software, but when multiplied across every single bit operation on hardware where standard GPIO functions are already a bottleneck, it is an unnecessary performance tax. More importantly, it solves a **non-existent problem**: the target framework is fixed at build time, so paying a performance penalty for runtime flexibility provides zero benefit.

## Templates

Templates eliminate virtual dispatch and allow the compiler to inline every hardware call:

```cpp
template<typename Hal>
class Driver {
  // ...
};
```

From a performance perspective, this is optimal but the drawback is API *complexity* becouse templates rapidly propagate through the public API.

Users would either need to:

- instantiate explicit template types, such as `Driver<ArduinoHal>`, introducing a syntax burden for a library whose audience-facing goal is simplicity;

- rely on an additional wrapper layer whose sole purpose is hiding the template parameter, at which point the facade's responsibilities double for little practical benefit.

For a HAL consisting of just a *few low-level I/O primitives*, **the additional template machinery feels disproportionate to the value it provides**.

## Why macros won

Because the target platform is *known before compilation begins*, preprocessor macros allow the HAL implementation to be selected entirely at **compile time**:

```cpp
#if defined(ARDUINO)
  #include "foo_hal_arduino.h"
#elif defined(ESP_PLATFORM)
  #include "foo_hal_espidf.h"
#endif
```

This approach allows each hardware macro (such as `HARDWARE_SET_HIGH()`) to expand directly into the platform's fastest *primitive* — direct register access, a native framework API, or a standard I/O function.

The trade-off is clear: macros are unscoped, global, and lack language-level type safety across boundaries. However, since the target platform never changes at runtime, sacrificing dynamic flexibility yields a zero-cost abstraction: no vtables, no complex template syntax, and zero runtime overhead.

*The hardware abstraction exists for the developer, but **disappears completely for the machine.***
<!--
Macros resolved once, at preprocessing time, via `#if defined(ARDUINO) / #elif defined(ESP_PLATFORM)`, sidestep both costs: each `CS123X_SCLK_HIGH()`/`CS123X_DOUT_READ()` expands directly to the target platform's fastest available primitive (direct AVR register access, `gpio_set_level()`, or a plain `digitalWrite()`), with zero indirection and zero extra type parameters. The trade-off is real and worth naming honestly: macros are global, unscoped, and can't be swapped at runtime — exactly one HAL is compiled in per build. For a library where the target is always known at compile time (you're never deciding at runtime whether you're on Arduino or IDF), that's not a limitation in practice — it's the same assumption the constraint already makes.
-->

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - ESP-IDF dual buildable examples](04-ESP-IDF-dual-buildable-examples)
- [Chapter 5 - Publishing](05-publishing.md)
- [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](06-bonus_yield-watchdog_naming-convention.md)
