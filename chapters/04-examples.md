# Chapter 4 — Example Conventions and Single-Source ESP-IDF Integration

With the `src/` directory fully framework-agnostic ([Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)) the focus shifts to structuring `examples/`.

## Arduino's examples follow a strict `Name/Name.ino` convention

The Arduino IDE (and PlatformIO's example browser for `.ino` projects; see Section 5.x) expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`. The sketch name must match its parent directory. Additional levels of nesting can be used to organize examples into categories `examples/<Category>/<ExampleName>/<ExampleName>.ino` and do not affect discovery, provided this naming convention is preserved.

Driven entirely by this simple layout, no further setup or explanation is needed.

For more details, see `Library Examples` section on [Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files).

## ESP-IDF Examples as Standalone Projects

Unlike Arduino sketches, ESP-IDF examples are complete, self-contained projects. Each example comes with its own build scripts, configuration, and source directory structure.

ESP-IDF examples are slightly more interesting because native `idf.py` projects and PlatformIO's `framework = espidf` integration expect different project layouts — a difference worth looking at closely.

For more details, see the official Espressif documentation how to [Reference an ESP Component](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/index.html) or refer to [Appendix A - IDF dependency resolution](../appendix/A-esp-idf-dependency-resolution.md) for a deep dive into component resolution mechanisms.

Fortunately, those differences are largely superficial. With a small amount of configuration, a single example can be built unchanged by both toolchains.

---

## 4.1 ESP-IDF (native)

When built natively through `idf.py`, an example operates as a self-contained project. While local examples can be wired using either a pure CMake setup or the ESP Component Manager ([Appendix A — IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)), **using the Component Manager is strongly recommended for publishable libraries**.

Declaring dependencies via `main/idf_component.yml`:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../path/to/foo" # for local development
```

mirrors the exact resolution and manifest validation rules enforced by the **ESP Component Registry**. This ensures that your examples test the exact same integration model that end users will experience once the library is published.

---

## 4.2 PlatformIO

By default, PlatformIO expects application source code to reside in a `src/` directory at the project root. Rather than creating a redundant `src/` folder alongside ESP-IDF's native layout, PlatformIO can be configured to point directly to the existing `main/` directory instead (See: [PlatformIO ESP-IDF Project Structure](https://docs.platformio.org/en/latest/frameworks/espidf.html#project-structure)).

This requires only a single additional setting in `platformio.ini` beyond the usual board and platform configuration:

```ini
[platformio]
src_dir = main

[env:esp32dev]
platform = espressif32
framework = espidf
board = esp32dev
```

With `src_dir = main`, PlatformIO builds the same hand-written, IDF-native project used by `idf.py`, seamlessly reusing the existing `CMakeLists.txt` and `idf_component.yml`.

As a result, the same `main.cpp` builds unchanged whether the project is compiled through `idf.py build` or PlatformIO's **Build button**.

At that point, an important question remains: *which system is responsible for resolving external dependencies?*

---

## 4.3 PlatformIO, Component Manager, and CMake are independent dependency-resolution systems

[Appendix A - IDF dependency resolution](../appendix/A-esp-idf-dependency-resolution.md) showed that ESP-IDF component discovery and component compilation are distinct stages of the build process. When **PlatformIO** is introduced, a third dependency-resolution mechanism enters the picture.

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

- **`EXTRA_COMPONENT_DIRS` + `REQUIRES` only:** works.
  Useful for purely local **CMake**-based development.

- **`lib_deps` + `idf_component.yml`:** works.
  Causes some redundancy because the same dependency may be fetched by PlatformIO while also being resolved by the Component Manager.

- **`lib_deps` + `EXTRA_COMPONENT_DIRS` + `REQUIRES`:** works.
  Also introduces redundancy because both PlatformIO and CMake are locating the same component.

- **`lib_deps` + `REQUIRES` without either `EXTRA_COMPONENT_DIRS` or `idf_component.yml` will fail**.

  **Dependency-*fetching* and component-*registration* are separate concepts.** PlatformIO successfully fetches the dependency, but ESP-IDF is never given a mechanism to discover and register it as a component. The package exists on disk, yet `REQUIRES` cannot resolve it because no component-resolution path has been established.

The practical recommendation is to use a single authoritative dependency-resolution mechanism whenever possible:

1. **PlatformIO (`lib_deps`)** when the project is intended primarily for *PlatformIO* users.
2. **ESP-IDF Component Manager (`idf_component.yml`)** when working with *ESP-IDF components and registries*.
3. **CMake (`EXTRA_COMPONENT_DIRS` + `REQUIRES`)** for local, *filesystem-based component* development.

## 4.4 The layout that makes one example buildable by both toolchains

To achieve full dual-toolchain compatibility, an ESP-IDF example simply places a `platformio.ini` file directly at its root alongside the native ESP-IDF project files:

```text
examples/my_example/
├── CMakeLists.txt        ← idf.py entry point (project-level)
├── platformio.ini        ← PlatformIO entry point with lib_deps
└── main/
  ├── CMakeLists.txt      ← example component registration
  ├── idf_component.yml   ← dependency via Component Manager
  └── main.cpp
```

> Note: Optional configuration files (such as `sdkconfig.defaults`, custom partition tables `partitions.csv`, or application-level `Kconfig.projbuild`) can be placed directly in the example's root or `main/` directory as needed. For complete guidelines on project customization files, see the [Example Project](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html#example-project) section on official [ESP-IDF Build System](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html) Documentation.

Adopting the **ESP-IDF Component Manager** through `idf_component.yml` as the primary dependency-resolution mechanism keeps the example aligned with Espressif's recommended workflow while remaining fully compatible with PlatformIO's ESP-IDF integration.

Although the dependency can also be declared in `platformio.ini` via `lib_deps` for convenience, having both manifests in place allows each toolchain to resolve dependencies using its own native system.

This dual-nature layout also sets up a clean boundary for publication. When registering and publishing packages to the respective ecosystems, each toolchain's manifest allows excluding files irrelevant to its registry:

- The component's `idf_component.yml` uses its `files.exclude` rules to ignore `platformio.ini` when published to the **ESP Component Registry**.

- The library's `library.json` uses its `export.exclude` rules to ignore project's `idf_component.yml` manifests when published to the **PlatformIO Registry**.

This reciprocal exclusion keeps the distributed packages clean and native to each ecosystem, without compromising the unified source layout during development.

How those manifests are structured is covered in [Chapter 5 - Publishing](05-publishing.md).

---

## 4.5 Note: Unidirectional build conflict (`idf.py` → `PlatformIO`)

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
