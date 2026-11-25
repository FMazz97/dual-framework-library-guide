# References

External sources cited throughout the guide, grouped by ecosystem, including the framework source files read to check the behaviour described. Each entry names the chapter or appendix where it is first relied upon.

## Arduino

- [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/): folder layout, `library.properties` fields, `depends`, and the `examples/` convention (Ch. 1, 4, 5).
- [Arduino Library Manager](https://docs.arduino.cc/libraries/): the registry and its tag-driven indexing (Ch. 2, 5).
- [Arduino Lint](https://arduino.github.io/arduino-lint/latest/): `arduino-lint --library-manager submit` and `update` (Ch. 5).
- [arduino/library-registry](https://github.com/arduino/library-registry): the repository where the first submission is made through a Pull Request (Ch. 5).

## ESP-IDF and ESP Component Registry

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

## PlatformIO

- [PlatformIO Registry](https://registry.platformio.org/): the registry itself (Ch. 2, 5).
- [Library Management](https://docs.platformio.org/en/latest/librarymanager/): `lib_deps` and the library builder (Ch. 1).
- [`library.json` reference](https://docs.platformio.org/en/latest/manifests/library-json/index.html): manifest fields, `examples`, `export`, `build` (Ch. 2, 5).
- [`includeDir`](https://docs.platformio.org/en/stable/manifests/library-json/fields/build/includedir.html): default include paths of a library (Ch. 2).
- [Espressif 32 ESP-IDF framework](https://docs.platformio.org/en/latest/frameworks/espidf.html): project structure, `src_dir`, and limitations (Ch. 4, 5).
- [Package Management CLI](https://docs.platformio.org/en/latest/core/userguide/pkg/index.html): `pio pkg pack` and `pio pkg publish` (Ch. 5).
- [PlatformIO Issue #12672](https://github.com/platformio/platformio-home/issues/12672): PIO Home flattening multi-file `examples` entries (Ch. 5, 7).

## Reference implementation

- [CS123x](https://github.com/FMazz97/CS123x): the library on which this guide is based (Ch. 3, 5).
- [DualBlinkCpp](https://github.com/FMazz97/DualBlinkCpp): the minimal, buildable example of the guide (Ch. 6) and the template of the version check (App. E).

## Appendices

### Appendix A, IDF Dependency Resolution

- [ESP-IDF Build System](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html): component discovery, `EXTRA_COMPONENT_DIRS`, and the override order.
- [IDF Component Manager](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/tools/idf-component-manager.html): how the manager runs inside the configuration step and what it adds to the build system.
- [IDF Component Manager and ESP Component Registry Documentation](https://docs.espressif.com/projects/idf-component-manager/en/latest/): the manifest and registry reference.
- [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/index.html): the entry point to the official documentation.
- [What is the ESP Component Registry?](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/): already listed under ESP-IDF, also relied upon here.

### Appendix B, Yield & Interrupts

- Arduino-ESP32 core, `esp32-hal-misc.c`: the `yield()` wrapper around `vPortYield()` and `yieldIfNecessary()`. Read from the installed package through "Go to Definition" (see Appendix D).
- FreeRTOS kernel, `tasks.c`: the behaviour of `vTaskDelay(0)`, which only forces a reschedule. Read from the framework sources, in the same way.
- Arduino core for ESP8266, `core_esp8266_wiring_digital.cpp`: where the core protects its own GPIO handling with `esp8266::InterruptLock`, the guard that replaced the failed `xt_rsil` attempt. Read from the installed package, in the same way.
- [CS123x](https://github.com/FMazz97/CS123x): the library where the Task Watchdog failure was observed and fixed.

### Appendix C, Naming Convention

- [C++ Core Guidelines, Naming and layout](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-naming): `underscore_style` (NL.10) and ALL_CAPS reserved for macros (NL.9).
- [Arduino Library Style Guide](https://docs.arduino.cc/learn/contributions/arduino-library-style-guide/): `camelCase` functions and the advice on boolean arguments.
- [ESP-IDF Style Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/contribute/style-guide.html): `snake_case` in C and C++, and the type and constant conventions.

### Appendix D, Testing During Development

- No external sources beyond those already listed (PlatformIO and ESP-IDF documentation). The behaviour described was observed on hardware and checked in the framework sources.

### Appendix E, Version Check

- [Semantic Versioning 2.0.0, precedence](https://semver.org/#spec-item-11): the ordering rule applied to the versions.
- [`version` field of `library.json`](https://docs.platformio.org/en/latest/manifests/library-json/fields/version.html): allowed characters and the 20-character limit.
- [Git hooks](https://git-scm.com/docs/githooks): `core.hooksPath` and the `pre-push` hook.

<!--START_no_print_zone-->
---

<!-- markdownlint-disable MD033 -->
<div align="center">

[Introduction](../README.md)

</div>
<!-- markdownlint-enable MD033 -->
<!--STOP_no_print_zone-->
