# Chapter 4 — Examples Buildable from a Single Source Tree, via Either `idf.py` or PlatformIO

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) introduced that `EXTRA_COMPONENT_DIRS` and PlatformIO's own `framework = espidf` scaffolding follow different conventions. This chapter covers how a single example folder was made to satisfy both — the same `main.cpp`, the same dependency, built by whichever toolchain a user already has installed, with no manual setup on either side.

## Dependency-*fetching* and component-*registration* are separate concepts

This distinction is easy to miss, because PlatformIO exposes both behind a seemingly unified experience (everything lands in `.pio/libdeps/`, regardless of framework).

- **`lib_deps`** is responsible for locating, downloading, and updating libraries.
- **`REQUIRES`** is an ESP-IDF build-system construct responsible for component registration inside CMake.
- The two mechanisms are independent and do not automatically satisfy each other's requirements.

**`lib_deps` does not make a library visible to IDF's native component resolver.** Under `framework = espidf`, a dependency fetched by PlatformIO is not automatically discoverable by an explicit `REQUIRES foo` in `main/CMakeLists.txt` — those are two independent systems.

**Declaring a dependency in both `lib_deps` and `REQUIRES` without `idf_component.yml` breaks the build.** This produces a resolution failure rather than harmless redundancy, as CMake's native resolver cannot find what PlatformIO fetched.

**The practical rule is simple:** under `framework = espidf`, choose a single authoritative dependency-resolution path whenever possible.

- **PlatformIO-only flow:** if the library is available on the PlatformIO Registry, leave dependency management to `lib_deps` and omit `REQUIRES`. PlatformIO automatically injects its fetched libraries into the build, avoiding manual CMake registration.

- **ESP-IDF Component Manager flow:** include a valid `idf_component.yml` and let ESP-IDF's Component Manager handle registration natively; `lib_deps` may fetch the same package alongside (causing minor redundancy), but the build proceeds cleanly without conflicts.

## The layout that makes one example buildable by both toolchains

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

`platformio.ini` needs exactly one line beyond the usual board/platform configuration — pointing PlatformIO's own scaffolding at the same `main/` folder IDF already expects, instead of generating its own `src/`-based layout:

```ini
[platformio]
src_dir = main

[env:esp32dev]
platform = espressif32
framework = espidf
board = esp32dev
```

With `src_dir = main`, PlatformIO uses the same hand-written, IDF-native `CMakeLists.txt` and `idf_component.yml` already used by `idf.py`, rather than generating a separate `src/`-based project structure. The same `main.cpp` therefore builds unchanged whether invoked through `idf.py build` or PlatformIO's Build button.

At that point, two dependency-resolution strategies are possible:

- **ESP-IDF Component Manager flow:** the dependency is declared in `idf_component.yml`, allowing both `idf.py` and PlatformIO to rely on ESP-IDF's native component-resolution mechanism. This is the preferred choice when developing locally against components that already provide an ESP-IDF manifest.

- **PlatformIO Registry flow:** if the library is published on the PlatformIO Registry, the dependency can be declared through `lib_deps` alone. In this scenario, PlatformIO manages both package retrieval and build integration automatically, making `idf_component.yml` and explicit CMake registration unnecessary for the consumer project.

The practical rule remains the same: choose a single authoritative dependency-resolution path for a given dependency, rather than mixing PlatformIO's package management and ESP-IDF's component-registration mechanisms unnecessarily.

## The one caveat: build caches don't mix

Switching between `idf.py` and PlatformIO **on the same example folder** reliably breaks the second toolchain used, because each writes its own `build/` cache — absolute paths, toolchain-specific configuration — that the other doesn't recognize and can't safely reuse. The fix is mechanical, not a sign of a deeper incompatibility:

```bash
idf.py fullclean
# or: delete the build/ directory manually
```

This is worth stating explicitly in any example's own README — it's the single most common point of confusion for someone trying both toolchains back to back on the same checkout, and it has nothing to do with the dependency-resolution design above; it would happen even for a trivial hello-world example with no external component at all.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - ESP-IDF dual buildable examples](04-ESP-IDF-dual-buildable-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](06-bonus_yield-watchdog_naming-convention.md)
