# Final Checklist

Every item below is a rule that came out of the previous chapters and of the development of CS123x, in the order in which they come up.

## Before writing code

- [ ] Fix the **final package name** for every registry before the first publish. ESP component: lowercase, 2–64 characters, `a-z`, `0-9`, `-`, `_`, no leading, trailing or consecutive separators (`compote` lowercases `--name` anyway). PlatformIO: the same lowercase name. Arduino library and GitHub repository: `PascalCase`. ([Ch. 1](01-ecosystem-constraints.md#registry-naming-rules), [Ch. 5](05-publishing.md#55-the-generalizable-naming-rule), [App. C](../appendices/C-naming-convention.md))
- [ ] Fix the naming scheme: `snake_case` in the core, `camelCase` methods in the Arduino facade, shared types and constants identical in both; prefix semantics `get_` / `set_` / `read_` / `write_`. ([App. C](../appendices/C-naming-convention.md))
- [ ] Use plain enums whose enumerators carry the class prefix; avoid `enum class`. ([App. C](../appendices/C-naming-convention.md))
- [ ] Hardware access goes through **compile-time macros** (HAL), not virtual interfaces or templates. ([Ch. 3](03-agnostic-core-hal-architecture.md))
- [ ] The core calls no framework API directly. Test `ARDUINO` before `ESP_PLATFORM`. ([Ch. 3](03-agnostic-core-hal-architecture.md))
- [ ] In headers shared with AVR, include the C headers (`<stdint.h>`, `<math.h>`): `<cstdint>` and `<cmath>` broke the AVR build in this project.

## Repository layout

- [ ] IDF manifest and `CMakeLists.txt` at the **repository root**; sources under `src/`, referenced by paths that only go down (never `..`). ([Ch. 2](02-repo-structure.md))
- [ ] Arduino: everything that must compile lives under `src/`; examples are `examples/<Name>/<Name>.ino`. ([Ch. 1](01-ecosystem-constraints.md), [Ch. 4](04-examples.md))
- [ ] The facade header and the core header differ only in case, so they never share a folder (for now: see the warning in the [Introduction](../README.md)). ([App. C](../appendices/C-naming-convention.md))
- [ ] Check what the package exclusion lists remove: excluding the Arduino HAL header breaks anyone building the IDF package with `ARDUINO` defined. ([Ch. 5](05-publishing.md))
- [ ] Each IDF example is a native project: `main/`, `main/idf_component.yml` (with `override_path`), and a `platformio.ini` with `src_dir = main` and `lib_deps`. ([Ch. 4](04-examples.md))

## Manifests

- [ ] `library.properties` (Arduino): `name`, `version`, `includes`.
- [ ] `library.json` (PlatformIO): `name` lowercase, `"frameworks": ["arduino", "espidf"]`, `"build": {"flags": ["-I src/core"]}` for headers in subfolders, example `name`s **without parentheses and spaces**.
- [ ] `idf_component.yml` (ESP): `version`, `description`, `url`/`repository`, `idf` constraint, `examples`, `files: exclude`.
- [ ] **One version number, identical in all three.**

## Dependencies under ESP-IDF

- [ ] Give each toolchain **one** authoritative resolution path: `idf_component.yml` for native `idf.py`, `lib_deps` for PlatformIO. Never `lib_deps` and `REQUIRES` together without a manifest or `EXTRA_COMPONENT_DIRS`: PlatformIO fetches the package, but nothing registers it as a component. ([Ch. 4](04-examples.md#43-platformio-component-manager-and-cmake-are-independent-dependency-resolution-systems))
- [ ] With `EXTRA_COMPONENT_DIRS`: the `list(APPEND …)` goes **before** `include(…project.cmake)`; the component is named after its folder. ([App. A](../appendices/A-esp-idf-dependency-resolution.md))
- [ ] `REQUIRES` is optional in `main`, which requires every component automatically, and needed in any other component. ([App. A](../appendices/A-esp-idf-dependency-resolution.md))
- [ ] `REQUIRES <name>` accepts the short name for Registry components; the long `<namespace>__<name>` is rejected (in tests). ([App. A](../appendices/A-esp-idf-dependency-resolution.md))
- [ ] Test the example **as a user gets it**: `idf.py create-project-from-example "<namespace>/<name>^<version>:<example>"`, then build.

## Cooperating with the scheduler

- [ ] No portable `yield()`: AVR does nothing, ESP8266 needs the real call, ESP32 (Arduino and IDF) needs `vTaskDelay(1)`, not `taskYIELD()` or Arduino's `yield()`, which are only reschedule requests. ([App. B](../appendices/B-yield-watchdog.md))
- [ ] Polling loops: stay tight during the normal conversion window, yield after it, stop yielding near the timeout.
- [ ] The guard before the timeout is **larger than one yield** (up to one tick period), so a yield can never cause a timeout.
- [ ] Very long averages at high rates throttle their outer loop too, at an interval well below the watchdog timeout.

## Testing during development

- [ ] The PlatformIO project directory is **outside** the library repository; `src_dir` may point inside (the example folder, or its `main/` for ESP-IDF). Library linked with `symlink://`. ([App. D](../appendices/D-testing-during-development.md))
- [ ] One PlatformIO environment per wiring or USB port; `pio run` (all environments) before every release.
- [ ] Native build: `idf.py -C <example> -B <output-folder> build`; check `git status` afterwards.
- [ ] Arduino IDE: copy the whole repository into `libraries/`, keep one copy.
- [ ] Never mix `idf.py` and PlatformIO builds in one folder without cleaning (`idf.py fullclean`). ([Ch. 4](04-examples.md))
- [ ] When framework behaviour matters, open its source with F12 (Go to Definition, works with PlatformIO) instead of trusting memory. ([App. D](../appendices/D-testing-during-development.md))

## Release, in order

1. [ ] Build every environment and flash at least one board.
2. [ ] Bump `version` in all three manifests.
3. [ ] Commit and **push** the bump. The tag snapshots the commit it points to.
4. [ ] `arduino-lint --library-manager submit`.
5. [ ] `pio pkg pack`, then list the archive (`tar -tzf`).
6. [ ] `compote component upload --name <name> --dry-run`, then list the archives in `dist/`. The dry run validates without creating a version and needs no token.
7. [ ] Tag and publish the GitHub Release.
8. [ ] Arduino: the first release goes through a Pull Request to the registry; updates are automatic after a new tag.
9. [ ] `pio pkg publish` (logged in with `pio account login`). `accepted` is not `published`: wait for the email and check the package page.
10. [ ] `compote component upload --name <name> --namespace <namespace>` (token in `IDF_COMPONENT_API_TOKEN`). Processing is asynchronous; check `upload-status`.
11. [ ] Install from each registry in a **clean project** and build.

See [Chapter 5](05-publishing.md) for the details of each registry.

---

## References

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

[⬅️ Chapter 6 - Conclusions](06-conclusions.md) — [Introduction](../README.md)

</div>
<!-- markdownlint-enable MD033 -->