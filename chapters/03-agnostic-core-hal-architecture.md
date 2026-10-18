# Chapter 3 — The Agnostic Core, the HAL Layer, and the Arduino Facade

With the repository structure established ([Chapter 2 - Repository Structure](02-repo-structure.md)), the next challenge is architectural: how do you organize the code so that the protocol logic exists only *once*, while still allowing **Arduino** and **ESP-IDF** to use their own **native hardware APIs**?

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

The **core** (styled in `snake_case`, following the **C++** standard library and **ESP-IDF** conventions; see [Appendix C - Naming Convention](../appendix/C-naming-convention.md)) contains all *protocol logic*, *calculations*, *state handling*, and *device behavior*. The **HAL** provides *framework-specific hardware access*, while the **Arduino facade** exists solely to expose the naming style Arduino users typically expect.

The facade's entire job is **forwarding** every public method:

```cpp
#include "core/foo.h"

class Foo {
  private:
    foo _core;

  public:
    Foo (int pin) : _core(pin){}
    bool classMethod(int param) { return _core.class_method(param); }
    // ...
};
```

No logic lives inside the facade itself.

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

So, **the protocol implementation exists exactly once, while each framework retains its own native hardware implementation**.

---

## 3.1 Why compile-time macros, not virtual dispatch or templates

Before settling on compile-time macros, two standard abstraction mechanisms — *virtual interfaces* and *C++ templates* — were evaluated and rejected. The first one introduce overhead that cannot be justified in a low-level hardware-access layer designed to disappear at compile time and the second one add API complexity without adding any benefit here.

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

A virtual call's indirection (a few extra cycles per invocation) is negligible in high-level software, but when multiplied across every single bit operation on hardware where standard GPIO functions are already a bottleneck, it is an unnecessary performance tax. More importantly, it solves a **non-existent problem**: the target framework is fixed at build time, so paying a performance penalty for runtime flexibility provides zero benefit.

### Templates

Templates eliminate virtual dispatch and allow the compiler to inline every hardware call:

```cpp
template<typename Hal>
class Driver {
  // ...
};
```

From a performance perspective, this is optimal, but the drawback is API *complexity* because templates rapidly propagate through the public API.

Users would either need to:

- instantiate explicit template types, such as `Driver<ArduinoHal>`, introducing a syntax burden for a library whose audience-facing goal is simplicity.

- rely on an additional wrapper layer whose sole purpose is hiding the template parameter, at which point the facade's responsibilities double for little practical benefit.

For a HAL consisting of just a *few low-level I/O primitives* (such as GPIO setting, timing and yielding), **the additional template machinery feels disproportionate to the value it provides**.

### Why macros won

Because the target platform is *known before compilation begins*, preprocessor macros allow the HAL implementation to be selected entirely at **compile time**:

```cpp
#if defined(ARDUINO)
  #include "../hal/foo_hal_arduino.h"
#elif defined(ESP_PLATFORM)
  #include "../hal/foo_hal_espidf.h"
#endif
```

> **Note:** The order of these checks matters. When building for ESP32 using the Arduino framework, both `ARDUINO` and `ESP_PLATFORM` are defined. For this reason, `ARDUINO` must be tested first; otherwise Arduino builds would incorrectly select the ESP-IDF HAL implementation.

This approach allows each hardware macro (such as `FOO_PIN_HIGH()`) to expand directly into the platform's fastest *primitive*, whether that is a native framework API, direct hardware access, or a standard I/O function.

The trade-off is clear: macros are unscoped, global, and lack language-level type safety across boundaries. However, since the target platform never changes at runtime, sacrificing dynamic flexibility yields a zero-cost abstraction: no vtables, no complex template syntax, and zero runtime overhead.

*The hardware abstraction exists for the developer, but **disappears completely for the machine.***

---

## 3.2 Integrating the HAL into the Core

Once the abstraction boundary is established, integration into the core is minimal.

The core header selects the appropriate HAL at compile time:

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

From that point onward, the implementation uses only **framework-neutral macros**:

```cpp
// foo.cpp — the core only sees neutral macros
FOO_PIN_HIGH();
FOO_PIN_LOW();
FOO_DELAY_US(us);
```

Each HAL then provides the *platform-specific* implementation of those operations.

- Arduino:

  ```cpp
  // foo_hal_arduino.h
  #define FOO_PIN_HIGH()      digitalWrite(_pin, HIGH)
  #define FOO_PIN_LOW()       digitalWrite(_pin, LOW)
  #define FOO_DELAY_US(us)    delayMicroseconds(us)
  ```

- ESP-IDF:

  ```cpp
  // foo_hal_espidf.h
  #define FOO_PIN_HIGH()      gpio_set_level(static_cast<gpio_num_t>(_pin), 1)
  #define FOO_PIN_LOW()       gpio_set_level(static_cast<gpio_num_t>(_pin), 0)
  #define FOO_DELAY_US(us)    esp_rom_delay_us(us)
  ```

From the core's perspective, there is no concept of **Arduino** `digitalWrite()` or **ESP-IDF** `gpio_set_level()`, or any other *framework-specific* API. **The only contract it knows is the set of HAL macros.**

This separation has an important consequence: the protocol code can **focus exclusively** on *device behavior*, *timing requirements*, *state management*, and *calculations*, while the HAL remains responsible for translating those operations into the *platform's native hardware primitives*.

> **Note:** Real implementations may occasionally require small leaks across that boundary.In [CS123x](https://github.com/FMazz97/CS123x), the library on which this guide is based, AVR-specific fast-I/O fields (register pointers and bit masks) remain declared in the core class under `#if defined(__AVR__)`, because the HAL macros expand directly inside the core methods. This does not change the architectural separation described above, but it is a practical exception worth documenting rather than pretending the abstraction is perfectly pure.

As a result, framework-specific code is confined to a small number of HAL headers, while the core implementation remains identical across every supported ecosystem.

The *core* never calls **Arduino** or **ESP-IDF** APIs directly.

With the architecture defined under `src/` directory, **the library source code is complete.**

The remaining challenge is no longer the source code, but the examples that demonstrate it. While Arduino examples work naturally across Arduino IDE and PlatformIO, ESP-IDF examples require additional structure to remain buildable across both native ESP-IDF and PlatformIO workflows.

That shared example layout is the subject of [Chapter 4 - Example Conventions](04-examples.md).

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository Structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md)
- [Chapter 4 - Example Conventions](04-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)
- [Appendix B - Yield & Watchdog](../appendix/B-yield-watchdog.md)
- [Appendix C - Naming Convention](../appendix/C-naming-convention.md)
