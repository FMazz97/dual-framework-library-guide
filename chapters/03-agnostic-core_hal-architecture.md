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

## 3.2 The naming convention (`get_`/`read_`/`set_`) — and why it only emerged by iterating

This convention wasn't designed upfront. It was extracted, after the fact, from noticing that a few method names were making a promise their implementation didn't keep — and that inconsistency is worth explaining directly, because the fix pattern generalizes well beyond this one library.

The rule, once made explicit:

- **`get_*`** — reads **already-held state**. Zero hardware communication, always `const`, cannot fail, cannot block. `get_gain()`, `get_rate()`, `get_config()`.
- **`set_*`** — mutates **already-held state**, or writes it to hardware *and* reports success/failure explicitly through its return value and documented behavior. `set_offset()` (pure memory), `set_config()` (hardware write, but its `set_` name is earned by symmetry with the `get_config()` it pairs with and by being the library's canonical "configure the chip" verb).
- **`read_*`** — talks to the chip. Can block, can time out, returns a value that may be a sentinel error. `read()`, `read_average()`, `read_dual_channel()`.

Two renames from the library's own history show where the convention was violated before it was named:

- `getValue()`/`getUnits()` **looked** like pure state accessors — same `get_` prefix as `getOffset()`/`getScale()` sitting right next to them — but both internally called `read_average()`, with all the blocking and failure modes that implies. Renamed to `readNetCounts()`/`readNetUnits()`.
- `setTempCalibration(float)` (the single-argument overload) silently read a live value from the chip to derive a calibration point — a `set_` name hiding a `read_` operation, overloaded against a *second*, genuinely pure `setTempCalibration(float, int32_t)` that only wrote already-known values to memory. Splitting them into `calibrateTemp()` (hardware-reading) and `setTempCalibration()` (pure) removed the ambiguity that the shared name — and the overload resolution needed to tell them apart — had been carrying.

The lesson this generalizes into: **a naming convention discovered by auditing a real API for where the name and the implementation disagree is more reliable than one designed in the abstract before any method exists to test it against.** Both renames above were only visible once every method could be listed side by side and asked the same question — "does this touch hardware, yes or no?" — a question that's easy to get right on a small API and easy to drift on as one grows, unless the convention is written down and treated as load-bearing rather than cosmetic once it's found. It's also, not incidentally, the kind of change that's worth doing as a deliberate breaking version bump (Chapter 6 covers the resulting `v2.0.0` migration table) rather than deferring indefinitely — a naming inconsistency left in a public API only gets more expensive to fix as more code depends on the wrong name.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - Yield & Watchdog](04-yield-watchdog.md)
- [Chapter 5 - ESP-IDF dual buildable examples](05-dual-buildable-examples)
- [Chapter 6 - Publishing](06-publishing.md)
