# Chapter 4 — Example Conventions and Single-Source ESP-IDF Integration

With the `src/` directory fully framework-agnostic ([Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)) the focus shifts to structuring `examples/`.

## Arduino's examples follow a strict `Name/Name.ino` convention

The Arduino IDE (and PlatformIO's example browser for `.ino` projects; see Section 5.x) expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`. The sketch name must match its parent directory. Additional levels of nesting can be used to organize examples into categories `examples/<Category>/<ExampleName>/<ExampleName>.ino` and do not affect discovery, provided this naming convention is preserved.

Driven entirely by this simple layout, no further setup or explanation is needed.

For more details, see `Library Examples` section on [Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files).

## ESP-IDF Examples as Standalone Projects

Unlike Arduino sketches, ESP-IDF examples are complete, self-contained projects. Each example comes with its own build scripts, configuration, and source directory structure.

ESP-IDF examples are slightly more interesting because native `idf.py` projects and PlatformIO's `framework = espidf` integration expect different project layouts — a difference worth looking at closely.

Fortunately, those differences are largely superficial. With a small amount of configuration, a single example can be built unchanged by both toolchains.

For more details, see how to [Reference](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/index.html) an ESP Component.

---

## 4.1 ESP-IDF dependency resolution: CMake and Component Manager are different layers

### Component resolution through `EXTRA_COMPONENT_DIRS`

In a pure **CMake** workflow, component discovery is entirely filesystem-driven.

When the project's `CMakeLists.txt` contains:

```cmake
set(EXTRA_COMPONENT_DIRS "../path/to/<component>")
  ```

ESP-IDF scans the specified directories, finds each component's `CMakeLists.txt`, and registers the component under the exact **name** of the folder that contains it.

```text
Project
│
└── CMakeLists.txt
          ↓
  EXTRA_COMPONENT_DIRS
          ↓
    components/foo/
          ↓
    CMakeLists.txt
          ↓
idf_component_register(...)
```

In this workflow, component's `idf_component.yml` is completely ignored. The component builds successfully even if no manifest exists, because **discovery and naming are handled entirely by CMake based on the filesystem structure.**

### Component resolution through `idf_component.yml`

The flow changes when the project uses the **Component Manager**:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../components/foo"
```

Here, the first actor is no longer CMake, but the **Espressif Component Manager** extension.

The Component Manager performs the following sequential steps:

- reads the project's `idf_component.yml`;
- locates the local dependency through `override_path`;
- loads the component's own `idf_component.yml`;
- validates the component's `name:` field (enforcing strict lowercase validation rules);
- injects validated component into the **CMake** build system environment.

Only after this validation sequence is successful does CMake take over.

```text
Project
│
└── idf_component.yml
            ↓
      override_path
            ↓
      components/foo/
            ↓
    idf_component.yml (validation)
            ↓
     CMakeLists.txt
            ↓
  idf_component_register(...)
```

This explains why naming rules become strict when using the **Component Manager**. Unlike the pure `EXTRA_COMPONENT_DIRS` workflow, the component must first pass manifest identity validation before CMake ever processes its directory.

### The common ground

Both workflows ultimately converge at the same endpoint:

```cmake
idf_component_register(...)
```

This macro remains the **sole mechanism that tells ESP-IDF which source files, include directories, and compile-time requirements belong to the component.** The structural difference lies entirely in how the component is discovered, named, and validated before it reaches the compiler.

---

## 4.2 The layout that makes one example buildable by both toolchains

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

### ESP-IDF (native)

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

By default, PlatformIO expects application source code to reside in a `src/` directory at the project root. Rather than creating a redundant `src/` folder alongside ESP-IDF's native layout, PlatformIO can be configured to point directly to the existing `main/` directory instead ([See: PlatformIO ESP-IDF Project Structure](https://docs.platformio.org/en/latest/frameworks/espidf.html#project-structure)).

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

At that point, an important question remains: *which system is responsible for resolving external dependencies?*

---

## 4.3 Dependency-*fetching* and component-*registration* are separate concepts

This distinction is easy to miss, because PlatformIO exposes both behind a seemingly unified experience (everything lands in `.pio/libdeps/`, regardless of framework).

- **`lib_deps`** is responsible for locating, downloading, and updating libraries.
- **`REQUIRES`** is an ESP-IDF build-system construct responsible for component registration inside CMake.
- The two mechanisms are independent and do not automatically satisfy each other's requirements.

**`lib_deps` does not make a library visible to IDF's native component resolver.** Under `framework = espidf`, a dependency fetched by PlatformIO is not automatically discoverable by an explicit `REQUIRES foo` in `main/CMakeLists.txt` — those are two independent systems.

**Declaring a dependency in both `lib_deps` and `REQUIRES` without `idf_component.yml` breaks the build.** This produces a resolution failure rather than harmless redundancy, as CMake's native resolver cannot find what PlatformIO fetched.

- **PlatformIO-only flow:** if the library is managed through PlatformIO (`library.json` or registry), leave dependency management to `lib_deps` and omit `REQUIRES`. *PlatformIO* automatically injects its fetched libraries into the build, avoiding manual CMake registration.

- **ESP-IDF Component Manager flow:** include a valid `idf_component.yml` and let *ESP-IDF's Component Manager* handle registration natively; `lib_deps` may fetch the same package alongside (causing minor redundancy), but the build proceeds cleanly without conflicts.

**The practical rule is simple:** under `framework = espidf`, choose a ***single** authoritative dependency-resolution path*, rather than mixing PlatformIO's package management and ESP-IDF's component-registration mechanisms unnecessarily.

The same separation of responsibilities extends beyond build systems and into package registries themselves. Through their respective manifests (`library.json` and `idf_component.yml`).

How those manifests are structured is covered in [Chapter 5 - Publishing](05-publishing.md).

## 4.4 Note: Unidirectional build conflict (`idf.py` → `PlatformIO`)

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
- [Chapter 4 - Example conventions](04-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](06-bonus_yield-watchdog_naming-convention.md)
