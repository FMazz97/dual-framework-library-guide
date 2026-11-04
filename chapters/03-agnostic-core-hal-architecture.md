# Chapter 3 — Agnostic Core & HAL Architecture

With the repository structure established ([Chapter 2 - Repository Structure](02-repo-structure.md)), the next challenge is architectural: how do you organize the code so that the protocol logic exists only *once*, while still allowing **Arduino** and **ESP-IDF** to use their own **native hardware APIs**?

The resulting architecture separates responsibilities into three layers:

```text
Foo (Arduino facade: camelCase methods)
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

The **core** (styled in `snake_case`, following the **C** and **C++** recommendation and **ESP-IDF** conventions; see [Case styles section in Appendix C](../appendix/C-naming-convention.md#case-styles-snake_case-for-the-core-directly-used-by-esp-idf-camelcase-for-the-arduino-facade)) contains all *protocol logic*, *calculations*, *state handling*, and *device behavior*. The **HAL** provides *framework-specific hardware access*, while the **Arduino facade** exists solely to expose the naming style Arduino users typically expect.

The facade's entire job is **forwarding** every public method, with no logic of its own:

```cpp
#include "core/foo.h"

class Foo {
  private:
    foo _core;

  public:
    Foo(int pin) : _core(pin) {}
    bool classMethod(int param) { return _core.class_method(param); }
    // ...
};
```

The rest of the repository is unchanged from [Chapter 2](02-repo-structure.md#21-the-layout); only the content of `src/` is now fully defined:

```text
src/
├── Foo.h                  ← Arduino facade
├── core/                  ← Framework-agnostic core
│   ├── foo.h
│   └── foo.cpp
└── hal/                   ← Hardware Abstraction Layer
    ├── foo_hal_arduino.h
    └── foo_hal_espidf.h
```

By isolating framework selection within the HAL, **the protocol implementation exists exactly once, while each framework retains its own native hardware implementation**.

---

## 3.1 Why compile-time macros, not virtual dispatch or templates

Before settling on compile-time macros, two standard abstraction mechanisms, *virtual interfaces* and *C++ templates*, were evaluated and rejected: the first introduces overhead that cannot be justified in a low-level hardware-access layer designed to disappear at compile time, the second adds API complexity without any benefit here.

Both are also C++-only, so neither would carry over to a C core, the hypothetical path mentioned at the end of the [Introduction](../README.md#-the-purpose-of-this-repository). C can only emulate virtual dispatch with function-pointer tables, which keep the same runtime indirection, while macros work unchanged in both languages.

### Virtual dispatch

A classic object-oriented solution defines a HAL interface and provides separate framework implementations behind virtual methods:

```cpp
class IHal {
public:
  virtual void pinMode(...) = 0;
  virtual void digitalWrite(...) = 0;
};
```

While clean, this approach introduces runtime indirection into the hottest part of the codebase. In tight bit-banging loops or high-frequency protocol transactions, clock and data lines are toggled repeatedly within strict microsecond-level timing budgets.

A virtual call's indirection (a few extra cycles per invocation) is negligible in high-level software, and would be an acceptable price if it bought something. Here it buys nothing: **the target framework is fixed at build time**, so paying even a small penalty, multiplied across every bit operation, for runtime flexibility that nobody needs solves a **non-existent problem**.

### Templates

Templates eliminate virtual dispatch and allow the compiler to inline every hardware call:

```cpp
template<typename Hal>
class Driver {
  // ...
};
```

From a performance perspective, this is optimal, but the drawback is API *complexity*: templates rapidly propagate through the public API. Users would either need to:

- instantiate explicit template types, such as `Driver<ArduinoHal>`, introducing a syntax burden for a library whose audience-facing goal is simplicity;
- or rely on an additional wrapper layer whose sole purpose is hiding the template parameter, at which point the facade's responsibilities double for little practical benefit.

For a HAL consisting of just a *few low-level I/O primitives* (such as GPIO setting, timing and yielding), **the additional template machinery feels disproportionate to the value it provides**.

### Why macros won

Because the target platform is *known before compilation begins*, preprocessor macros allow the HAL implementation to be selected entirely at **compile time** (the selection itself is shown in [3.2](#32-integrating-the-hal-into-the-core)). Each hardware macro (such as `FOO_PIN_HIGH()`) expands directly into the platform's fastest *primitive*, whether that is a native framework API, direct hardware access, or a standard I/O function.

The trade-off is clear: macros are unscoped, global, and lack language-level type safety across boundaries. However, since the target platform never changes at runtime, sacrificing dynamic flexibility yields a zero-cost abstraction: no vtables and no template syntax leaking into the public API.

*The hardware abstraction exists for the developer, but **disappears completely for the machine.***

---

## 3.2 Integrating the HAL into the Core

Once the abstraction boundary is established, integration into the core is minimal. The core header selects the appropriate HAL at compile time:

```cpp
// foo.h
#if defined(ARDUINO)
  #include "../hal/foo_hal_arduino.h"
#elif defined(ESP_PLATFORM)
  #include "../hal/foo_hal_espidf.h"
#else
  #error "Foo core: unsupported platform (expected ARDUINO or ESP_PLATFORM)"
#endif
```

> [!IMPORTANT]
> The order of these checks matters. When building for ESP32 using the Arduino framework, both `ARDUINO` and `ESP_PLATFORM` are defined. For this reason, `ARDUINO` must be tested first; otherwise Arduino builds would incorrectly select the ESP-IDF HAL implementation.

From that point onward, the implementation uses only **framework-neutral macros**:

```cpp
// foo.cpp — the core only sees neutral macros
FOO_PIN_HIGH();
FOO_PIN_LOW();
FOO_DELAY_US(us);
```

Each HAL then provides the *platform-specific* implementation of those operations:

```cpp
// foo_hal_arduino.h
#define FOO_PIN_HIGH()      digitalWrite(_pin, HIGH)
#define FOO_PIN_LOW()       digitalWrite(_pin, LOW)
#define FOO_DELAY_US(us)    delayMicroseconds(us)
```

```cpp
// foo_hal_espidf.h
#define FOO_PIN_HIGH()      gpio_set_level(static_cast<gpio_num_t>(_pin), 1)
#define FOO_PIN_LOW()       gpio_set_level(static_cast<gpio_num_t>(_pin), 0)
#define FOO_DELAY_US(us)    esp_rom_delay_us(us)
```

From the core's perspective, there is no concept of Arduino `digitalWrite()`, ESP-IDF `gpio_set_level()`, or any other *framework-specific* API: **the only contract it knows is the set of HAL macros.** The protocol code can therefore focus exclusively on *device behavior*, *timing requirements*, *state management*, and *calculations*, while framework-specific code stays confined to a small number of HAL headers and the core implementation remains identical across every supported ecosystem.

> [!NOTE]
> Real implementations may occasionally require small leaks across that boundary. In [CS123x](https://github.com/FMazz97/CS123x), the library on which this guide is based, AVR-specific fast-I/O fields (register pointers and bit masks) remain declared in the core class under `#if defined(__AVR__)`, because the HAL macros expand directly inside the core methods. This does not change the architectural separation described above, but it is a practical exception worth documenting rather than pretending the abstraction is perfectly pure.

With the architecture defined under `src/`, **the library source code is complete.** What remains is the examples: Arduino examples work naturally across Arduino IDE and PlatformIO, while ESP-IDF examples need additional structure to stay buildable under both native ESP-IDF and PlatformIO. [Chapter 4 - Example Conventions](04-examples.md) covers that shared layout.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 2 - Repository Structure](02-repo-structure.md) — [Introduction](../README.md) — [Chapter 4 - Example Conventions ➡️](04-examples.md)

</div>
<!-- markdownlint-enable MD033 -->
