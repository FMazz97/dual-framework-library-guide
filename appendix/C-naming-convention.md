# Appendix C — Naming Convention: What a Name Promises

While refining the public API of the [CS123x](https://github.com/FMazz97/CS123x) library, a recurring pattern emerged: several method names were making promises their implementations didn't actually keep. What initially looked like a library-specific cleanup eventually turned into a more general naming convention that applies to many hardware drivers and communication libraries.

> [!NOTE]
> This appendix is a guideline, not a standard. It only covers names that follow a precise scheme (`get_`, `set_`, `read_`, `write_`) and the case style of the core and the Arduino facade. Other names are left to each library: some are universal and self-explanatory (`begin()`, `end()`), others depend on the device (`tare()`, `power_up()`).

---

## Method prefixes: what the name promises

| Prefix | Promise |
| :--- | :--- |
| **`get_*`** | Returns state already held in memory, or derived from it. **No hardware access**, always `const`, cannot fail or block. |
| **`set_*`** | Stores state and may push it to the device. Always has a `get_*` counterpart for the same state. If it touches the hardware, it **reports success or failure** through its return value. |
| **`read_*`** | Acquires data from the device. Not `const`: it can **block**, can **time out**, and **signals failure** with a sentinel value (e.g. `CS123X_TIMEOUT_ERROR`, or `NAN` for floating-point results). |
| **`write_*`** | Raw transfer primitive that sends data to the device. Reserved for operations with **no state counterpart** (here, the private `write_bits()`). |

The boundary between `set_*` and `write_*` is the existence of a counterpart: `set_config()` pairs with `get_config()`, while `write_bits()` has no `get_bits()`, because it is a transfer and not state. A primitive that reads and writes in a single transaction carries both prefixes, as in the private `read_val_and_write_register()`: it honours the `read_*` promise (it can time out and returns sentinel values) while also writing to the device.

The `read_*`/`write_*` pair also echoes the Arduino library style guide, which asks to use `read()` to read inputs and `write()` to write outputs, as in `digitalRead()` and `analogWrite()`.

Everything else (`begin()`, `power_up()`, `power_down()`, `tare()`, `is_ready()`, `calibrate_*`) sits outside the scheme: universal, self-explanatory names like `begin()` and `end()` need no convention, and the rest adapt to the device at hand, so imposing a rule on them would make little sense. The guideline only asks that such names don't borrow one of the four prefixes without keeping its promise.

### Where the convention came from

Version 1.1.0 of the library predates the convention, and auditing its public API for names that disagreed with their implementation is what produced it. The clearest cases (1.1.0 names are the Arduino API of that release; current names are shown in the core's `snake_case`):

| Version 1.1.0 | Current | What the old name hid |
| :--- | :--- | :--- |
| `getValue()` | `read_net_counts()` | A `get_` name that calls `read_average()`: it blocks, can time out, and isn't even `const`, unlike its neighbours `getOffset()` and `getScale()`. |
| `getUnits()` | `read_net_units()` | The same, plus a division by the scale. |
| `setTempCalibration(float)` | `calibrate_temp()` | A `set_` name that reads a live value from the chip to derive a calibration point. |
| `setTempCalibration(float, int32_t)` | `set_temp_calibration()` | Nothing: the genuinely pure overload keeps the `set_` name. |

The two `setTempCalibration()` overloads shared one name, so overload resolution was the only thing telling a hardware read from a plain write to memory; splitting them removed that ambiguity.

The lesson: **a convention found by auditing a real API is often more reliable than one designed in the abstract.** It only emerged once every method was listed side by side and asked the same question, "does this touch hardware, yes or no?", and once found it should be written down and treated as load-bearing. A name that lies is worth fixing in a deliberate breaking release: the longer it stays in a public API, the more code depends on it.

> [!NOTE]
> Small inconsistencies that don't break a promise are another matter. The library still has `set_ch()`/`get_ch()` instead of `set_channel()`/`get_channel()`: they stayed because it was published before this convention was devised, and a rename this small doesn't justify a major release.

---

## Case styles: `snake_case` for the core directly used by ESP-IDF, `camelCase` for the Arduino facade

> [!IMPORTANT]
> Each of the three systems already has its own guidelines:
>
> - [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-naming)
> - [Arduino library style guide](https://docs.arduino.cc/learn/contributions/arduino-library-style-guide/)
> - [ESP-IDF style guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/contribute/style-guide.html)
>
> They don't always agree with each other: this appendix sets out compromises that try to align them all.
<!-- markdownlint-disable MD028 -->
<!-- markdownlint-enable MD028 -->
> [!NOTE]
> **Terminology.** The ESP-IDF guide says `CamelCase` but specifies "with a capital letter as beginning", which is `PascalCase`, while the Arduino guide's "camel case" means `camelCase`, with a lowercase initial. The same word covers two styles, so this guide always writes `camelCase` (`readNetCounts()`) and `PascalCase` (`CS123x`).

| Item | Core / ESP-IDF | Arduino facade |
| :--- | :--- | :--- |
| Class | `cs123x` | `CS123x` |
| Header | `cs123x.h` | `CS123x.h` |
| Methods | `read_net_counts()` | `readNetCounts()` |
| Parameters | `known_weight` | `knownWeight` |
| Shared types and constants | `CS123X_Config`, `CS123X_GAIN_128`: identical in both | |

> [!WARNING]
> The core class is shown in lowercase (`cs123x`), which departs from ESP-IDF's C++ convention of `PascalCase` classes. It is deliberate: it keeps the core's calls close to the C style, where a function usually written `foo_method()` reads as `foo.method()`.

The mapping is mechanical: drop the underscores and capitalise the next letter (`get_timeout_ms()` → `getTimeoutMs()`), and the prefix semantics above carry over unchanged. In practice, **design every new method in `snake_case` first, in the core, and derive the facade name from it**, never the other way around.

**Why two spellings.** The core follows the conventions of the environment it targets natively: ESP-IDF's style guide asks for `snake_case` functions (`gpio_set_level()`) and, in C++, `snake_case` methods and members, which is also what the C++ standard library does, and what the C++ Core Guidelines prefer. The facade follows the Arduino style guide, which asks to "Use camel case function names, not underscore", a style it adopted from Processing.org for readability, as in `analogRead()`. Each audience sees one style only, and the two never appear side by side in a user's code (see [Chapter 3](../chapters/03-agnostic-core-hal-architecture.md)).

**Shared types and constants.** Enums, structs, and constants are shared by core and facade, so they keep a single spelling in both worlds, built on the class prefix: `CS123X_` followed by PascalCase for types (`CS123X_Config`, `CS123X_DualReading`) and ALL_CAPS for constants and enumerators (`CS123X_GAIN_128`, `CS123X_TIMEOUT_ERROR`). No single standard covers them, and the references disagree, so here is what each one says, for the record:

| Reference | Functions and methods | Classes and structs | Enums and constants |
| :--- | :--- | :--- | :--- |
| **Arduino style guide** | `camelCase` | no indication | long ALL_CAPS names are hard to read; simplify where possible |
| **ESP-IDF style guide (C)** | `snake_case` | type definitions in `snake_case_t` | namespaced ALL_CAPS constants (`MODULE_FOO_ONE`) and a `module_foo_t` type |
| **ESP-IDF style guide (C++)** | `snake_case` methods and members | `CamelCase`, with an underscore allowed if readability suffers (`GPIO_Output`) | no indication |
| **C++ Core Guidelines** | `underscore_style` (NL.10) | examples predominantly use `PascalCase` types | ALL_CAPS reserved for macros (NL.9) |

> [!NOTE]
> **Terminology.** The ESP-IDF guide says `CamelCase` but specifies "with a capital letter as beginning", which is `PascalCase`, while the Arduino guide's "camel case" means `camelCase`, with a lowercase initial. The same word covers two styles, so this guide always writes `camelCase` (`readNetCounts()`) and `PascalCase` (`CS123x`).

The shared names land in between: `CS123X_Config` follows the ESP-IDF C++ pattern (`CamelCase`, with an underscore after an all-caps prefix, as in `GPIO_Output`), and `CS123X_GAIN_128` mirrors ESP-IDF's namespaced ALL_CAPS constants. The class itself is the one real departure: the core class `cs123x` is in `snake_case`, like the C++ standard library's classes, whereas ESP-IDF's C++ guide asks for `CamelCase` classes, which the facade class `CS123x` happens to satisfy for both ecosystems.

**Enumerations.** Avoid `enum class`: use plain enums whose enumerators carry the class prefix (`CS123X_GAIN_128`), which gives the same namespacing without the drawbacks. Scoped enumerators require qualification and explicit casts, which is hard on Arduino beginners used to plain constants like `HIGH` and `INPUT`, and `enum class` is C++-only, so it would not carry over to a C core.

> [!IMPORTANT]
> These are conventions, not rules: the references disagree. To stay closer to ESP-IDF's C style, the scheme works just as well with the `_t` suffix on types (e.g. `CS123X_Config_t`). What matters is that every shared name is built the same way, from the class prefix (`CLASSNAME_Rest`): it keeps each name unambiguous and prevents clashes with macro names, which is the reason the C++ Core Guidelines reserve ALL_CAPS for macros.

**Files.** Core and HAL files are lowercase (`cs123x.h`, `cs123x_hal_espidf.h`); the facade keeps the library name (`CS123x.h`). The core header and the facade header differ only in case, so they must never share a folder: on a case-insensitive file system they would be the same file (here they live in `src/core/` and `src/`, see [Chapter 2](../chapters/02-repo-structure.md)).

---

## A deliberate exception: boolean `verify` arguments

The same Arduino style guide advises avoiding boolean arguments, suggesting two functions with distinguishing names instead. The library knowingly deviates with `bool verify = true`, found on `set_config()`, `set_ch()`, `set_gain()`, `set_rate()`, `set_ref()`, `read_temp()`, and `read_dual_channel()`. The flag is uniform across the whole API (always the last parameter, always defaulting to `true`, always meaning "read the register back to confirm"), so it reads as a convention rather than an oversight, and it avoids doubling every hardware-writing method into a verified and an unverified twin.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Appendix B - Yield & Watchdog](B-yield-watchdog.md) — [Introduction](../README.md) — [Appendix D - Testing During Development ➡️](D-testing-during-development.md)

</div>
<!-- markdownlint-enable MD033 -->
