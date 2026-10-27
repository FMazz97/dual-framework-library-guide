# Chapter 4 — Example Conventions

With the core fully framework-agnostic and the HAL sitting between it and the framework (see: [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md)), the focus shifts to structuring `examples/`.

---

## 4.1 Arduino

The Arduino IDE expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`: the sketch name must match its parent directory. Additional levels of nesting can group examples into categories (`examples/<Category>/<ExampleName>/<ExampleName>.ino`) without affecting discovery, as long as this naming convention is preserved. PlatformIO's example browser falls back to the same convention when `library.json` doesn't define `examples` explicitly (see [5.3.1 Manifest and Metadata](05-publishing.md#531-manifest-and-metadata)). <!--cambio di link alla pubblicazione-->

ESP-IDF example projects don't follow this convention and are therefore automatically omitted from the Arduino IDE's Examples menu. No further setup is needed.

For more details, see the `Library Examples` section on [Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files).

---

## 4.2 ESP-IDF (native)

Unlike Arduino sketches, ESP-IDF examples are complete, self-contained projects, each with its own build scripts, configuration, and source directory structure. Native `idf.py` projects and PlatformIO's `framework = espidf` integration expect different layouts, but those differences are largely superficial: with a small amount of configuration, a single example can be built unchanged by both toolchains.

Natively, local examples can be wired using either a pure CMake setup or the ESP Component Manager, but **using the Component Manager is strongly recommended for publishable libraries**. Declaring dependencies via `main/idf_component.yml`:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../../.." # repository root = component root (local development)
```

mirrors the resolution and manifest validation rules enforced by the **ESP Component Registry**.  <!--lla frase è un po' forte, non so se la validazione del cmponenete passa anche sulle dipendenze dei sui esempi--> This ensures that your examples test the exact same integration model that end users will experience once the library is published.

For more details, see Espressif's documentation on how to [Reference an ESP Component](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/index.html), and [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md) for a deep dive into component resolution mechanisms.

---

## 4.3 PlatformIO

By default, PlatformIO expects application source code in a `src/` directory at the project root (see [`framework = espidf`](01-ecosystem-constraints.md#framework--espidf)). Rather than creating a redundant `src/` folder alongside ESP-IDF's native layout, PlatformIO can be configured to point directly to the existing `main/` directory instead (see: [PlatformIO ESP-IDF Project Structure](https://docs.platformio.org/en/latest/frameworks/espidf.html#project-structure)).

This requires only a single additional setting in `platformio.ini` beyond the usual board and platform configuration:

```ini
[platformio]
src_dir = main

[env:esp32dev]
platform = espressif32
framework = espidf
board = esp32dev
```

With `src_dir = main`, PlatformIO builds the same hand-written, IDF-native project used by `idf.py`, reusing the existing `CMakeLists.txt` and `idf_component.yml`. As a result, the same `main.cpp` builds unchanged whether the project is compiled through `idf.py build` or PlatformIO's **Build button**.

At that point, an important question remains: *which system is responsible for resolving external dependencies?*

---

## 4.4 PlatformIO, Component Manager, and CMake are independent dependency-resolution systems

[Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md) showed that ESP-IDF component discovery and component compilation are distinct stages of the build process. <!--ricontrollare una volta completata l'appendice A--> When **PlatformIO** is introduced, a third dependency-resolution mechanism enters the picture.

Under `framework = espidf`, three independent systems can participate in component resolution:

- **PlatformIO (`lib_deps`)** fetches libraries and injects them into the generated build environment (`.pio/libdeps/`).
- **ESP-IDF Component Manager (`idf_component.yml`)** resolves through component manager the dependencies before handing them to CMake.
- **ESP-IDF CMake (`EXTRA_COMPONENT_DIRS` + `REQUIRES`)** performs filesystem-based component discovery and registration.

Each mechanism works correctly on its own, but combining them can introduce redundancy or, in some cases, resolution failures. The following combinations make the distinction easiest to see:

| Mechanism(s) | Result | Notes |
| :--- | :--- | :--- |
| `lib_deps` only | Works | Recommended when the library is distributed through the **PlatformIO Registry** |
| `idf_component.yml` only | Works | Recommended with the **ESP Component Registry** or local `override_path` development |
| `EXTRA_COMPONENT_DIRS` + `REQUIRES` only | Works | Useful for purely local **CMake**-based development |
| `lib_deps` + `idf_component.yml` | Works | Redundant: the same dependency may be fetched by PlatformIO and resolved by the Component Manager |
| `lib_deps` + `EXTRA_COMPONENT_DIRS` + `REQUIRES` | Works | Redundant: PlatformIO and CMake both locate the same component |
| `lib_deps` + `REQUIRES`, without `EXTRA_COMPONENT_DIRS` or `idf_component.yml` | **Fails** | See below |

**Dependency-*fetching* and component-*registration* are separate concepts.** In the failing case, PlatformIO successfully fetches the dependency, but ESP-IDF is never given a mechanism to discover and register it as a component. The package exists on disk, yet `REQUIRES` cannot resolve it because no component-resolution path has been established.

The practical recommendation is to give each toolchain a single authoritative dependency-resolution mechanism:

1. **PlatformIO (`lib_deps`)** when the project is intended primarily for *PlatformIO* users.
2. **ESP-IDF Component Manager (`idf_component.yml`)** when working with *ESP-IDF components registries*.
3. **CMake (`EXTRA_COMPONENT_DIRS` + `REQUIRES`)** for local, *filesystem-based component* development.

---

## 4.5 The layout that makes one example buildable by both toolchains

To achieve full dual-toolchain compatibility, an ESP-IDF example simply places a `platformio.ini` file directly at its root alongside the native ESP-IDF project files:

```text
examples/my_example/
├── CMakeLists.txt        ← idf.py entry point (project-level)
├── platformio.ini        ← PlatformIO entry point (src_dir = main, lib_deps)
└── main/
    ├── CMakeLists.txt      ← example component registration
    ├── idf_component.yml   ← dependency via Component Manager
    └── main.cpp
```

> **Note:**
>
> - "The esp-idf build system does not support spaces in the path to either esp-idf or to projects." (see: [Step 5. Start a Project](https://docs.espressif.com/projects/esp-idf/en/v3.3/get-started-cmake/index.html#step-5-start-a-project))
> - Optional configuration files (such as `sdkconfig.defaults`, a custom partition table `partitions.csv`, or an application-level `Kconfig.projbuild`) can be placed directly in the example's root or `main/` directory as needed. For complete guidelines on project customization files, see the [Example Project](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html#example-project) section of the official [ESP-IDF Build System](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html) documentation.

The example therefore carries one dependency declaration per toolchain, each authoritative in its own context: `idf_component.yml` for native `idf.py` builds, aligned with Espressif's recommended workflow, and `lib_deps` in `platformio.ini` for PlatformIO builds, so that **whenever PlatformIO is involved, it resolves dependencies through its own system**. Under PlatformIO both declarations are visible, which produces the redundancy described in [4.4](#44-platformio-component-manager-and-cmake-are-independent-dependency-resolution-systems); the combination still works, and it is the price of building one unchanged example with both toolchains.

This layout also sets up a clean boundary for publication: each registry's manifest can exclude the files that belong to the other toolchain.

- The component's `idf_component.yml` uses its `files.exclude` rules to ignore `platformio.ini` when published to the **ESP Component Registry**.
- The library's `library.json` uses its `export.exclude` rules to ignore the project's `idf_component.yml` manifests when published to the **PlatformIO Registry**.

This reciprocal exclusion keeps the distributed packages clean and native to each ecosystem, without compromising the unified source layout during development. How those manifests are structured is covered in [Chapter 5 - Publishing](05-publishing.md).

---

## 4.6 Unidirectional build conflict (`idf.py` → PlatformIO)

While **PlatformIO** isolates its output in `.pio/build/<env>`, native `idf.py` builds directly into `build/` at the project root.

If you compile with `idf.py` first, PlatformIO's CMake integration will pick up the existing `build/` cache and fail due to conflicting path configurations. Switching from PlatformIO to `idf.py` works fine, but going from `idf.py` to PlatformIO requires clearing the native build artifacts first:

```bash
idf.py fullclean
# or manually remove the build/ directory
```

---

**Previous:** [← Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md) · **Next:** [Chapter 5 - Publishing →](05-publishing.md)
