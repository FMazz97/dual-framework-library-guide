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

Regardless of how dependencies are resolved, every **ESP-IDF standalone project** needs a minimal set of **CMake** files:

- A root `CMakeLists.txt` defining the project name and importing ESP-IDF build logic:

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  include($ENV{IDF_PATH}/tools/cmake/project.cmake)
  project(my_example)
  ```

- A `main/` directory containing the entry point (e.g., `main.cpp`) and its own `main/CMakeLists.txt`:

  ```cmake
  idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
  )
  ```

The choice between pure CMake and the Component Manager simply dictates **how** the project links to your library component.

---

### Component resolution through `EXTRA_COMPONENT_DIRS`

In a pure **CMake** workflow, component discovery is entirely filesystem-driven.

- Register the requred component's path in the project's root `CMakeLists.txt`:

  ```cmake
  list(APPEND EXTRA_COMPONENT_DIRS "${CMAKE_CURRENT_LIST_DIR}/../path/to/<component>")
  # set(EXTRA_COMPONENT_DIRS "../path/to/<component>")
  ```

  ESP-IDF scans the specified directories, finds each component's `CMakeLists.txt`, and registers the component under the exact **name** of the folder that contains it.

  > Prefer `list(APPEND EXTRA_COMPONENT_DIRS)` with `${CMAKE_CURRENT_LIST_DIR}` over `set(EXTRA_COMPONENT_DIRS)` to avoid overwriting existing paths and to ensure robust, absolute path resolution.

- Declare the dependency in `main/CMakeLists.txt` to make public headers and symbols accessible:

  ```cmake
  idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
    REQUIRES <component>
  )
  ```

  Notice that the value passed to `REQUIRES` must match the exact **folder name** of the component registered in the previous step.

The complete resolution chain for a pure **CMake** setup is therefore structured as follows:

```text
Project
│
├── CMakeLists.txt (list(APPEND EXTRA_COMPONENT_DIRS ...))
│         ↓
│   components/foo/
│         ↓
│   CMakeLists.txt → idf_component_register(...)
│
└── main/
    └── CMakeLists.txt (idf_component_register(... REQUIRES foo))
```

In this workflow, component's `idf_component.yml` is completely ignored. The component builds successfully even if no manifest exists, because **discovery and naming are handled entirely by CMake based on the filesystem structure and explicit `REQUIRES` statements.**

---

### Component resolution through `idf_component.yml`

The flow changes when the project uses the **Component Manager**:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../components/foo"
```

Here, the first actor is no longer CMake, but the **Espressif Component Manager**, while the `CMakeLists.txt` files remain minimal and unchanged.

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
├── CMakeLists.txt (standard / minimal)
│
└── main/
    │
    └── idf_component.yml (override_path: "../path/to/foo")
              ↓
        components/foo/
              ↓
        idf_component.yml (manifest name validation)
              ↓
        CMakeLists.txt → idf_component_register(...)
```

This explains why naming rules become strict when using the **Component Manager**. Unlike the pure `EXTRA_COMPONENT_DIRS` workflow, the component must first pass manifest identity validation before CMake ever processes its directory.

### The common ground

Both workflows ultimately converge at the same endpoint in the component `CMakeLists.txt`:

```cmake
idf_component_register(...)
```

This macro remains the **sole mechanism that tells ESP-IDF which source files, include directories, and compile-time requirements belong to the component.** The structural difference lies entirely in how the component is discovered, named, and validated before it reaches the compiler.

---

## ESP-IDF (native)

When built natively through `idf.py`, an example operates as a self-contained project. Based on the resolution mechanics detailed in [Section 4.1](#41-esp-idf-dependency-resolution-cmake-and-component-manager-are-different-layers), you simply choose the workflow that best fits your development cycle:

- **Pure CMake Workflow (`EXTRA_COMPONENT_DIRS` + `REQUIRES`):** Ideal for quick local prototyping where no manifest validation is required. The root `CMakeLists.txt` appends the component directory, and `main/CMakeLists.txt` declares `REQUIRES <component>`.
- **Component Manager Workflow (`idf_component.yml`):** The recommended approach for registry readiness and production-like environments. The CMake files remain completely standard, while `main/idf_component.yml` uses `override_path` to resolve the local repository during development.

---

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

## 4.3 PlatformIO, Component Manager, and CMake are independent dependency-resolution systems

[Section 4.1](#41-esp-idf-dependency-resolution-cmake-and-component-manager-are-different-layers) showed that ESP-IDF component discovery and component compilation are distinct stages of the build process. When **PlatformIO** is introduced, a third dependency-resolution mechanism enters the picture.

Under `framework = espidf`, three independent systems can participate in component resolution:

- **PlatformIO (`lib_deps`)** fetches libraries and injects them into the generated build environment (`.pio/libdeps/`).
- **ESP-IDF Component Manager (`idf_component.yml`)** resolves and validates component dependencies before handing them to CMake.
- **ESP-IDF CMake (`EXTRA_COMPONENT_DIRS` + `REQUIRES`)** performs filesystem-based component discovery and registration.

Each mechanism works correctly on its own, but combining them can introduce redundancy or, in some cases, resolution failures.

The distinction becomes easiest to see through the following combinations:

- **`lib_deps` only:** works.
  Recommended when the library is distributed through the **PlatformIO Registry**.

- **`idf_component.yml` only:** works.
  Recommended when using the **ESP-IDF Component Registry** or local `override_path` development.

- **`EXTRA_COMPONENT_DIRS` + `REQUIRES` only** works.
  Useful for purely local **CMake**-based development.

- **`lib_deps` + `idf_component.yml`** works.
  Causes some redundancy because the same dependency may be fetched by PlatformIO while also being resolved by the Component Manager.

- **`lib_deps` + `EXTRA_COMPONENT_DIRS` + `REQUIRES`** works.
  Also introduces redundancy because both PlatformIO and CMake are locating the same component.

- **`lib_deps` + `REQUIRES` without either `EXTRA_COMPONENT_DIRS` or `idf_component.yml` will fail**.
  **Dependency-*fetching* and component-*registration* are separate concepts.** PlatformIO successfully fetches the dependency, but ESP-IDF is never given a mechanism to discover and register it as a component. The package exists on disk, yet `REQUIRES` cannot resolve it because no component-resolution path has been established.

The practical recommendation is to use a single authoritative dependency-resolution mechanism whenever possible:

1. **PlatformIO (`lib_deps`)** when the project is intended primarily for *PlatformIO* users.
2. **ESP-IDF Component Manager (`idf_component.yml`)** when working with *ESP-IDF components and registries*.
3. **CMake (`EXTRA_COMPONENT_DIRS` + `REQUIRES`)** for local, *filesystem-based component* development.

## 4.4 The layout that makes one example buildable by both toolchains

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

The same separation of responsibilities extends beyond build systems and into package registries themselves. Through their respective manifests (`library.json` and `idf_component.yml`).

How those manifests are structured is covered in [Chapter 5 - Publishing](05-publishing.md).

## 4.4 Note: Unidirectional build conflict (`idf.py` → `PlatformIO`)

While **PlatformIO** places its output isolated in `.pio/build/<env>`, native `idf.py` builds directly into `build/` at the project root.

If you compile using `idf.py` first, PlatformIO's CMake integration will pick up the existing `build/` cache and fail due to conflicting path configurations. Switching from PlatformIO to `idf.py` works fine, but going from `idf.py` to PlatformIO requires clearing the native build artifacts first:

```bash
idf.py fullclean
# or manually remove the build/ directory
```

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - Example conventions](04-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Appendix A - IDF dependency resolution](../appendix/A-esp-idf-dependency-resolution.md)
- [Appendix B - Yield & Watchdog](../appendix/B-yield-watchdog.md)
- [Appendix C - Naming Convention](../appendix/C-naming-convention.md)
