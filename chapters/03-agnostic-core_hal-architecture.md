# Chapter 3 — The Agnostic Core, the HAL Layer, and the Arduino Facade

With the repository structure settled ([Chapter 2 - Repository structure](02-repo-structure.md)), the remaining challenge is no longer where the code lives, but how it is organized.

The goal is simple: write the protocol logic **once** and have it run natively on both Arduino and ESP-IDF frameworks, without runtime translation layers, virtual dispatch, duplicated implementations, or framework-specific code leaking into the application's core logic.

The architecture that emerged consists of three clearly separated layers/folders:

- **framework-agnostic `core/`** containing all *protocol logic*, *calculations*, *state handling*, and *device behavior* (`snake_case` as C++ recommended);
- **lightweight Hardware Abstraction Layer `hal/`** responsible only for *hardware* access and resolved entirely at compile time (`snake_case` as C++ recommended);
- **Arduino facade** whose sole purpose is to expose the class's naming and API in *Arduino* `camelCase` style that users typically expect.

The key design decision is that only the *HAL* knows which **framework is being used**, allowing the protocol implementation to exist exactly **once** while each framework retains its own native hardware access path and the resulting repository layout becomes:

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

The Arduino facade's entire job is forwarding: `bool classMethod() { return _core.class_method(); }`, repeated for every public method.

The facade intentionally contains *no business logic*, *state management*, or *hardware interaction*. This is precisely what makes it safe to expose two naming styles (`Foo` and `foo`) without risking divergence: there is no behavior in the facade to diverge. Every call is mechanically forwarded to the same underlying implementation.

```text
    Foo (Arduino facade, camelCase)
    │ forwards every call 1:1, adds nothing
    ▼
    foo (core, snake_case — this IS the core public API used direcly by ESP-IDF)
    │ calls FOO_* macros, never touches Arduino/IDF APIs directly
    ▼
    HAL macros (foo_hal_arduino.h / foo_hal_espidf.h)
      resolved by #if defined(ARDUINO) / #elif defined(ESP_PLATFORM)
```

so, ***the protocol implementation exists exactly once.***

---

## 3.1 Why compile-time macros, not virtual dispatch or templates

Before settling on compile-time macros, two more conventional abstraction mechanisms were evaluated and ultimately rejected. The decision was not driven by unfamiliarity with virtual interfaces or templates, but by the fact that both introduce trade-offs that become hard to justify in low-level hardware-access code.

**Virtual dispatch** (an abstract `IHal` interface, implemented separately per platform) would let the core hold a pointer to a HAL implementation and call through it — clean, familiar, and completely reasonable for most code. It falls apart specifically in `readBits()`/`writeBits()`, which toggle a clock line **once per bit**, called up to 46 times per register transaction, at a data rate where the entire transaction budget is measured in microseconds (Chapter 4 covers the 1280 Hz timing budget in detail). A virtual call's indirection — a few extra instructions per invocation — is negligible in most contexts, but multiplied by every single bit of every single transaction, on hardware where an ordinary `digitalWrite()` was *already* the bottleneck being fought (see the AVR direct-register-access work that predates this architecture), it's not a cost worth reintroducing for the sake of a cleaner interface.

**Templates** (`template<typename Hal> class cs123x`) avoid the virtual-call cost — the compiler can inline everything — but push the abstraction into the class's *type*. The public class Arduino users instantiate would need to either carry an explicit template parameter (`CS123x<ArduinoHal> adc(...)`, a syntax burden for a library whose whole audience-facing goal on that side is simplicity) or be wrapped in yet another layer to hide it — at which point the facade's job doubles for no benefit.

Macros resolved once, at preprocessing time, via `#if defined(ARDUINO) / #elif defined(ESP_PLATFORM)`, sidestep both costs: each `CS123X_SCLK_HIGH()`/`CS123X_DOUT_READ()` expands directly to the target platform's fastest available primitive (direct AVR register access, `gpio_set_level()`, or a plain `digitalWrite()`), with zero indirection and zero extra type parameters. The trade-off is real and worth naming honestly: macros are global, unscoped, and can't be swapped at runtime — exactly one HAL is compiled in per build. For a library where the target is always known at compile time (you're never deciding at runtime whether you're on Arduino or IDF), that's not a limitation in practice — it's the same assumption the constraint already makes.

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

## Index

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - Yield & Watchdog](04-yield-watchdog.md)
- [Chapter 5 - ESP-IDF dual buildable examples](05-dual-buildable-examples)
- [Chapter 6 - Publishing](06-publishing.md)
