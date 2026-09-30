# Chapter 4 — Examples Buildable from a Single Source Tree, via Either `idf.py` or PlatformIO

With the `src/` directory fully framework-agnostic [(Chapter 3 - Agnostic Core & HAL Architecture)](03-agnostic-core_hal-architecture.md) the focus shifts to structuring `examples/`.

- For **Arduino**, example layout is straightforward and strictly constrained: as detailed in [Arduino example conventio](01-ecosystem-constraints.md#examples-follow-a-strict-namenameino-convention), both the Arduino IDE and PlatformIO expect projects organized as `examples/<ExampleName>/<ExampleName>.ino`, where the sketch name must match its parent directory.

- For **ESP-IDF**, native `CMake` projects and PlatformIO’s `framework = espidf` integration follow different structural conventions. However, the actual differences are minimal: with a small configuration tweak, a single example folder can satisfy both toolchains seamlessly.

---

## 4.1 The layout that makes one example buildable by both toolchains

Simply add the `platfomrio.ini` in the ESP-IDF's examples:

```text
examples/my_example/
├── CMakeLists.txt        ← idf.py entry point (project-level)
├── platformio.ini        ← PlatformIO entry point
└── main/
  ├── CMakeLists.txt      ← idf_component_register(...)
  ├── idf_component.yml   ← dependency, via override_path for in-repo development
  └── main.cpp
```

### ESP IDF (native)

When built through `idf.py`, the example behaves like any other native **ESP-IDF project**. The build process is driven entirely by the project's CMake files and Component Manager configuration:

- The root `CMakeLists.txt` is a standard, native IDF project file — nothing PlatformIO-specific:

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  include($ENV{IDF_PATH}/tools/cmake/project.cmake)
  project(my_example)
  ```

- `main/idf_component.yml` declares the dependency once, resolved consistently regardless of which toolchain triggers the build:

  ```yaml
  dependencies:
    foo:
      version: "*"
      override_path: "../../../"   # repo root, for in-tree development
  ```

### PlatformIO

By default, PlatformIO expects application source code to reside in a `src/` directory at the project root. Rather than creating a redundant `src/` folder alongside ESP-IDF's native layout, PlatformIO can be configured to point directly to the existing `main/` directory instead.

This requires only a single additional setting in `platformio.ini` beyond the usual board and platform configuration:

```ini
[platformio]
src_dir = main

[env:esp32dev]
platform = espressif32
framework = espidf
board = esp32dev
```

With `src_dir = main`, PlatformIO builds the same hand-written, IDF-native project used by `idf.py`, reusing the existing `CMakeLists.txt` and `idf_component.yml`.

As a result, the same `main.cpp` builds unchanged whether the project is compiled through `idf.py build` or PlatformIO's **Build button**.

But, *which build system is responsible for resolving external dependencies?*

## 4.2 Dependency-*fetching* and component-*registration* are separate concepts

This distinction is easy to miss, because PlatformIO exposes both behind a seemingly unified experience (everything lands in `.pio/libdeps/`, regardless of framework).

- **`lib_deps`** is responsible for locating, downloading, and updating libraries.
- **`REQUIRES`** is an ESP-IDF build-system construct responsible for component registration inside CMake.
- The two mechanisms are independent and do not automatically satisfy each other's requirements.

**`lib_deps` does not make a library visible to IDF's native component resolver.** Under `framework = espidf`, a dependency fetched by PlatformIO is not automatically discoverable by an explicit `REQUIRES foo` in `main/CMakeLists.txt` — those are two independent systems.

**Declaring a dependency in both `lib_deps` and `REQUIRES` without `idf_component.yml` breaks the build.** This produces a resolution failure rather than harmless redundancy, as CMake's native resolver cannot find what PlatformIO fetched.

**The practical rule is simple:** under `framework = espidf`, choose a single authoritative dependency-resolution path whenever possible.

- **PlatformIO-only flow:** if the library is managed through PlatformIO (`library.json` or registry), leave dependency management to `lib_deps` and omit `REQUIRES`. *PlatformIO* automatically injects its fetched libraries into the build, avoiding manual CMake registration.

- **ESP-IDF Component Manager flow:** include a valid `idf_component.yml` and let *ESP-IDF's Component Manager* handle registration natively; `lib_deps` may fetch the same package alongside (causing minor redundancy), but the build proceeds cleanly without conflicts.

The practical rule remains the same: choose a ***single** authoritative dependency-resolution path* for a given dependency, rather than mixing PlatformIO's package management and ESP-IDF's component-registration mechanisms unnecessarily.

<!--This division of responsibilities at the project level directly informs how the library itself is published. In Chapter 5 — Publishing, we extend this principle to package registries—explaining how to configure manifest files (library.json and idf_component.yml) so that both the PlatformIO Registry and the ESP-IDF Component Manager resolve and register the library smoothly without conflicting version requirements or cross-registry ambiguity.-->

## 4.3 Note: Unidirectional build conflict (`idf.py` → `PlatformIO`)

While **PlatformIO** places its output isolated in `.pio/build/<env>`, native `idf.py` builds directly into `build/` at the project root.

If you compile using `idf.py` first, PlatformIO's CMake integration will pick up the existing `build/` cache and fail due to conflicting path configurations. Switching from PlatformIO to `idf.py` works fine, but going from `idf.py` to PlatformIO requires clearing the native build artifacts first:

```bash
idf.py fullclean # or manually remove the build/ directory
```

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - ESP-IDF dual buildable examples](04-ESP-IDF-dual-buildable-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](06-bonus_yield-watchdog_naming-convention.md)
