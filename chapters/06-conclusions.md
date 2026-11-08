# Chapter 6 — Conclusions

The guide started from a question: why accept the complexity of supporting both Arduino and ESP-IDF natively, instead of choosing one ecosystem or relying on a compatibility layer? This chapter collects what the answer turned out to be, what remains open, and where to go from here.

---

## 6.1 What the guide settled

Five decisions, each following from the previous one:

1. **Constraints first** ([Chapter 1](01-ecosystem-constraints.md)). Arduino compiles everything under `src/`, ESP-IDF needs explicit, self-contained components, and registry naming rules are enforced at publication. Everything else comes from satisfying them together.
2. **One repository, one component boundary** ([Chapter 2](02-repo-structure.md)). The repository root is the component root and `src/` is the single source of truth: no separate branches, no framework-specific distributions.
3. **A framework-agnostic core** ([Chapter 3](03-agnostic-core-hal-architecture.md)). Protocol logic is written once, hardware access is selected at compile time through a HAL, and the Arduino facade only forwards.
4. **Examples that build with both toolchains** ([Chapter 4](04-examples.md)). One project layout for `idf.py` and PlatformIO, with one dependency declaration per toolchain.
5. **Publication as a design concern** ([Chapter 5](05-publishing.md)). Three registries with different manifests, filters, and immutability rules, and a package name decided before the first publish.

The [Final Checklist](07-final-checklist.md) condenses them, in the order in which they come up.

---

## 6.2 What remains open

The guide rests on a single library and a single design (see the warning at the end of the [Introduction](../README.md#-the-purpose-of-this-repository)). Where it reports an observation instead of a documented fact, it says so in the text: this section collects what was not verified and the limits found along the way, and each point is a good starting point for a contribution.

### Not verified in this guide

- **Premises observed, not measured**
  - That a C API is the dominant design choice for ESP-IDF components: an observation, and the premise that could reshape the architecture of [Chapter 3](03-agnostic-core-hal-architecture.md).
- **Not tested**
  - Arduino tag with a `v` prefix: only tags without it were tested.
  - Version immutability on both Registries: stated from the documentation, never tested.
- **Observed, not explained**
  - How the build system accepts `<name>` in `REQUIRES` for a component registered as `<namespace>__<name>` ([Appendix A](../appendices/A-esp-idf-dependency-resolution.md)).
  - Why the Arduino-ESP32 build never tripped the Task Watchdog: tick rate and core of `loop()` were not compared ([Appendix B](../appendices/B-yield-watchdog.md)).
  - The hypothesis on the unstable PlatformIO layout ([Appendix D](../appendices/D-testing-during-development.md)): the files created while building appear to relaunch PlatformIO's reconfiguration, not verified in its source.

### Known limits

- The PIO Home → Examples panel ignores `examples[].name` for libraries and lists files one by one; the web Registry shows the names ([PlatformIO Issue #12672](https://github.com/platformio/platformio-home/issues/12672)).
- A PlatformIO package cannot be renamed from the CLI; it takes a support request ([5.4](05-publishing.md#54-the-case-sensitivity-trap-platformios-immutable-package-binding)).

---

## 6.3 Beyond Arduino & ESP-IDF: Extending and Contributing

While this guide focused on bridging Arduino, ESP-IDF, and PlatformIO, the underlying patterns (unified source layouts, explicit dependency management, and reciprocal registry exclusions) are likely applicable to the wider embedded ecosystem. The appendices go deeper on the technical details:

- **[Appendix A: IDF Dependency Resolution](../appendices/A-esp-idf-dependency-resolution.md)**: how ESP-IDF discovers components, and what the Component Manager adds to the build system.
- **[Appendix B: Yield & Watchdog](../appendices/B-yield-watchdog.md)**: how long blocking operations cooperate with the scheduler across AVR, ESP8266, and FreeRTOS-based ESP32 targets.
- **[Appendix C: Naming Convention](../appendices/C-naming-convention.md)**: what a method name promises, and how case styles map across the core and the Arduino facade.
- **[Appendix D: Testing During Development](../appendices/D-testing-during-development.md)**: one example tested on many boards with PlatformIO, and the layouts that keep it reliable.

The same approach should carry over to other families and frameworks, such as STM32 with its Arduino and STM32Cube frameworks, where a new HAL would be the only framework-specific addition, or Zephyr (not tested here). The embedded landscape keeps evolving, with new microcontrollers and alternative build systems continuing to emerge. If you want to extend this framework-agnostic approach to other platforms, refine the appendices, or share your own experience, issues and pull requests are always welcome: embedded development thrives on shared knowledge.

---

## 6.4 References

External sources cited throughout the guide, grouped by ecosystem. Each entry names the chapter or appendix where it is first relied upon

### Arduino

- [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/): folder layout, `library.properties` fields, `depends`, and the `examples/` convention (Ch. 1, 4, 5).
- [Arduino Library Manager](https://docs.arduino.cc/libraries/): the registry and its tag-driven indexing (Ch. 2, 5).
- [Arduino Lint](https://arduino.github.io/arduino-lint/latest/): `arduino-lint --library-manager submit` and `update` (Ch. 5).
- [arduino/library-registry](https://github.com/arduino/library-registry): the repository where the first submission is made through a Pull Request (Ch. 5).

### ESP-IDF and ESP Component Registry

- [What is the ESP Component Registry?](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/): components, Component Manager, and Registry (Ch. 1).
- [ESP Component Registry](https://components.espressif.com/): the registry itself (Ch. 2, 5).
- [Component files reference](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/component_files.html): what a published component must contain and how `files` filters are applied (Ch. 1, 5).
- [Compote CLI](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/compote_cli.html): `compote component pack` and `upload`, including `--dry-run` (Ch. 1, 5).
- [Reference an ESP Component](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/index.html): dependencies and `override_path` in `idf_component.yml` (Ch. 4).
- [Publish components](https://docs.espressif.com/projects/idf-component-manager/en/latest/publish/index.html): the publication workflow and token authentication (Ch. 5).
- [How to Package and Upload Examples](https://docs.espressif.com/projects/idf-component-manager/en/latest/publish/how_to_examples.html): the `examples` key in `idf_component.yml` (Ch. 5).
- [ESP-IDF Build System, Example Project](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html#example-project): optional project files such as `sdkconfig.defaults` and `partitions.csv` (Ch. 4).
- [ESP-IDF v3.3, Get Started (CMake), Step 5. Start a Project](https://docs.espressif.com/projects/esp-idf/en/v3.3/get-started-cmake/index.html#step-5-start-a-project): the restriction on spaces in project paths (Ch. 4).
- Component Manager source, where the naming rules were found (Ch. 1):
  - [`idf_component_manager/cli/validations.py`](https://github.com/espressif/idf-component-manager/blob/main/idf_component_manager/cli/validations.py)
  - [`idf_component_tools/manifest/constants.py`](https://github.com/espressif/idf-component-manager/blob/main/idf_component_tools/manifest/constants.py)

### PlatformIO

- [PlatformIO Registry](https://registry.platformio.org/): the registry itself (Ch. 2, 5).
- [Library Management](https://docs.platformio.org/en/latest/librarymanager/): `lib_deps` and the library builder (Ch. 1).
- [`library.json` reference](https://docs.platformio.org/en/latest/manifests/library-json/index.html): manifest fields, `examples`, `export`, `build` (Ch. 2, 5).
- [`includeDir`](https://docs.platformio.org/en/stable/manifests/library-json/fields/build/includedir.html): default include paths of a library (Ch. 2).
- [Espressif 32 ESP-IDF framework](https://docs.platformio.org/en/latest/frameworks/espidf.html): project structure, `src_dir`, and limitations (Ch. 4, 5).
- [Package Management CLI](https://docs.platformio.org/en/latest/core/userguide/pkg/index.html): `pio pkg pack` and `pio pkg publish` (Ch. 5).
- [PlatformIO Issue #12672](https://github.com/platformio/platformio-home/issues/12672): PIO Home flattening multi-file `examples` entries (Ch. 5, 6).

### Reference implementation

- [CS123x](https://github.com/FMazz97/CS123x): the library on which this guide is based (Ch. 3, 5).

### Appendices

- **Appendix A, IDF Dependency Resolution**
  - [ESP-IDF Build System](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html): component discovery, `EXTRA_COMPONENT_DIRS`, and the override order.
  - [IDF Component Manager](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/tools/idf-component-manager.html): how the manager runs inside the configuration step and what it adds to the build system.
  - [IDF Component Manager and ESP Component Registry Documentation](https://docs.espressif.com/projects/idf-component-manager/en/latest/): the manifest and registry reference.
  - [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/index.html): the entry point to the official documentation.
  - [What is the ESP Component Registry?](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/): already listed under ESP-IDF, also relied upon here.
- **Appendix B, Yield & Watchdog**
  - Arduino-ESP32 core, `esp32-hal-misc.c`: the `yield()` wrapper around `vPortYield()` and `yieldIfNecessary()`. Read from the installed package through "Go to Definition" (see Appendix D).
  - FreeRTOS kernel, `tasks.c`: the behaviour of `vTaskDelay(0)`, which only forces a reschedule. Read from the framework sources, in the same way.
  - [CS123x](https://github.com/FMazz97/CS123x): the library where the Task Watchdog failure was observed and fixed.
- **Appendix C, Naming Convention**
  - [C++ Core Guidelines, Naming and layout](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-naming): `underscore_style` (NL.10) and ALL_CAPS reserved for macros (NL.9).
  - [Arduino Library Style Guide](https://docs.arduino.cc/learn/contributions/arduino-library-style-guide/): `camelCase` functions and the advice on boolean arguments.
  - [ESP-IDF Style Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/contribute/style-guide.html): `snake_case` in C and C++, and the type and constant conventions.
- **Appendix D, Testing During Development**
  - No external sources beyond those already listed (PlatformIO and ESP-IDF documentation). The behaviour described was observed on hardware and checked in the framework sources.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 5 - Publishing](05-publishing.md) — [Introduction](../README.md) — [Final Checklist ➡️](07-final-checklist.md)

</div>
<!-- markdownlint-enable MD033 -->
