# Chapter 4 — Example Conventions

With the core fully framework-agnostic and the HAL sitting between it and the framework (see: [Chapter 3](03-agnostic-core-hal-architecture.md)), the focus shifts to structuring practical `examples/`. These are essential for showcasing the API to beginners and offer a ready-to-use template for quick setup. Ultimately, the base code written to test the library provides the perfect foundation to help users hit the ground running.

---

## 4.1 Arduino

The Arduino IDE expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`: the sketch name must match its parent directory. Additional levels of nesting can group examples into categories (`examples/<Category>/<ExampleName>/<ExampleName>.ino`) without affecting discovery, as long as this naming convention is preserved.

ESP-IDF example projects don't follow this convention and are therefore automatically omitted from the Arduino IDE's Examples menu. No further setup is needed.

For more details, see the **'Library Examples'** section on [Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files).

### Arduino examples on PlatformIO

PlatformIO's example browser falls back to the same convention when `library.json` doesn't define `examples` explicitly. To make an Arduino example buildable as a PlatformIO project as well, it can carry its own `platformio.ini` next to the sketch:

```text
examples/BasicExample/
├── BasicExample.ino    ← Arduino IDE sketch
└── platformio.ini      ← PlatformIO project configuration
```

and simply adapt the source code from default `src/` folder to the example folder:

```ini
[platformio]
src_dir = . ; tipical 
```

The `examples` field of `library.json` specifies which files make up each example in the published package (see: [PlatformIO Manifest and Metadata](05-publishing#531-manifest-and-metadata)).

---

## 4.2 ESP-IDF

ESP-IDF examples are complete, self-contained projects, each with its own build scripts, configuration, and source directory structure.

Natively, an example can reach the library in two ways: through the filesystem, with `EXTRA_COMPONENT_DIRS` in its root `CMakeLists.txt`, or through the Component Manager, with a dependency declared in `main/idf_component.yml` (see [Appendix A - IDF Dependency Resolution](../appendices/A-esp-idf-dependency-resolution.md)). **The Component Manager is strongly recommended for publishable libraries**, because it is the mechanism end users will rely on:

```yaml
dependencies:
  foo:
    version: "*"
    override_path: "../../.." # repository root = component root (local development)
```

The Component Manager reads this manifest, resolves `foo`, and hands its folder to the configuration system, exactly as it does when end users install the library from the ESP Component Registry, adding the dependency to the component's requirements along the way. The only difference is the source: `override_path` points to the local repository root instead of a registry download, and that folder must be named like the dependency `foo`. The example therefore exercises the same integration model that users will experience once the library is published.

For more details, see Espressif's documentation on how to [Reference an ESP Component](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/index.html), and [Appendix A - IDF Dependency Resolution](../appendices/A-esp-idf-dependency-resolution.md) for a deep dive into component resolution mechanisms.

### ESP-IDF examples on PlatformIO

By default, PlatformIO expects the application sources in a `src/` folder at the project root (see [1.3.2 `framework = espidf`](01-ecosystem-constraints.md#132-framework--espidf)). In an ESP-IDF example, however, `main/` is more than the default source folder: it is a component in its own right, the one holding the entry point, added to the build automatically and requiring every other component (see [Appendix A](../appendices/A-esp-idf-dependency-resolution.md)). Rather than duplicating it in a `src/` folder, PlatformIO can be pointed at it with a single setting in `platformio.ini`, on top of the usual board and platform configuration:

```ini
[platformio]
src_dir = main
```

see: [PlatformIO ESP-IDF Project Structure](https://docs.platformio.org/en/latest/frameworks/espidf.html#project-structure)

With `src_dir = main`, PlatformIO builds the same hand-written, IDF-native project used by `idf.py`, reusing the existing `main/CMakeLists.txt` and `main/idf_component.yml`. As a result, the same `main.cpp` builds unchanged whether the project is compiled through `idf.py build` or PlatformIO's **Build button**.

The example now builds under both toolchains, but its dependency on the library can be declared in two ways: in `idf_component.yml`, as in the native flow, or in PlatformIO's own `lib_deps`. Which system is responsible for resolving it?

---

## 4.3 PlatformIO, Component Manager, and CMake are independent dependency-resolution systems

As [Appendix A - IDF Dependency Resolution](../appendices/A-esp-idf-dependency-resolution.md) explains, ESP-IDF finds components in two ways, both within CMake's single configuration step: the build system discovers directories on the filesystem (`EXTRA_COMPONENT_DIRS`, the project's `components/` folder), and the Component Manager adds more from `idf_component.yml` manifests. PlatformIO adds a third, `lib_deps`, handled by PlatformIO itself rather than by CMake:

- **PlatformIO (`lib_deps`)** fetches libraries into `.pio/libdeps/` and builds them with its own library builder, using the defaults described in [2.1.3 PlatformIO Source & Include Path](02-repo-structure.md#213-platformio-source--include-path).
- **ESP-IDF Component Manager (`idf_component.yml`)** resolves dependencies and hands their folders to the build system, inside CMake's configuration step (see: [Appendix A](../appendices/A-esp-idf-dependency-resolution.md)).
- **ESP-IDF CMake (`EXTRA_COMPONENT_DIRS`)** discovers components on the filesystem; `REQUIRES` then connects them to each other (see: [Appendix A](../appendices/A-esp-idf-dependency-resolution.md)).

Each mechanism works correctly on its own, but combining them can introduce redundancy or, in some cases, resolution failures. The following combinations make the distinction easiest to see:

| Mechanism(s) | Result | Notes |
| :--- | :--- | :--- |
| `lib_deps` only | Works | Recommended when the library is distributed through the **PlatformIO Registry** |
| `idf_component.yml` only | Works | Recommended with the **ESP Component Registry** or local `override_path` development |
| `EXTRA_COMPONENT_DIRS` + `REQUIRES` only | Works | Useful for purely local **CMake**-based development |
| `lib_deps` + `idf_component.yml` | Works | Redundant: the same dependency may be fetched by PlatformIO and resolved by the Component Manager |
| `lib_deps` + `EXTRA_COMPONENT_DIRS` + `REQUIRES` | Works | Redundant: PlatformIO and CMake both locate the same component |
| `lib_deps` + `REQUIRES`, without `EXTRA_COMPONENT_DIRS` or `idf_component.yml` | **Fails** | See below |

**Dependency-*fetching* and component-*registration* are separate concepts.** In the failing case, PlatformIO fetches the dependency and builds it with its own library builder, which never reads `idf_component_register()` in the library's `CMakeLists.txt`: the package exists on disk and compiles, but it never becomes an ESP-IDF component. `REQUIRES` only connects components that the build system already knows, so it has nothing to resolve (see [2.1.3 PlatformIO Source & Include Path](02-repo-structure.md#213-platformio-source--include-path) for why PlatformIO handles such libraries on its own).

The practical recommendation is to give each toolchain a single authoritative dependency-resolution mechanism:

1. **PlatformIO (`lib_deps`)** when the project is intended primarily for *PlatformIO* users.
2. **ESP-IDF Component Manager (`idf_component.yml`)** when working with *ESP-IDF components and registries*.
3. **CMake (`EXTRA_COMPONENT_DIRS` + `REQUIRES`)** for local, *filesystem-based component* development.

---

## 4.4 The layout that makes one example buildable by both toolchains

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

> [!IMPORTANT]
> "The esp-idf build system does not support spaces in the path to either esp-idf or to projects." (see: [Step 5. Start a Project](https://docs.espressif.com/projects/esp-idf/en/v3.3/get-started-cmake/index.html#step-5-start-a-project))
<!-- markdownlint-disable MD028 -->
<!-- markdownlint-enable MD028 -->
> [!TIP]
> Optional configuration files (such as `sdkconfig.defaults`, a custom partition table `partitions.csv`, or an application-level `Kconfig.projbuild`) can be placed directly in the example's root or `main/` directory as needed. For complete guidelines on project customization files, see the [Example Project](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html#example-project) section of the official [ESP-IDF Build System](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html) documentation.

The example therefore carries one dependency declaration per toolchain, each authoritative in its own context: `idf_component.yml` for native `idf.py` builds, aligned with Espressif's recommended workflow, and `lib_deps` in `platformio.ini` for PlatformIO builds, so that **whenever PlatformIO is involved, it resolves dependencies through its own system**. Within the repository, PlatformIO sees both declarations, which produces the redundancy described in [4.3](#43-platformio-component-manager-and-cmake-are-independent-dependency-resolution-systems); the combination still works, and it is the price of building one unchanged example with both toolchains.

This layout also sets up a clean boundary for publication: each registry's manifest can exclude the files that belong to the other toolchain.

- The component's `idf_component.yml` uses its `files.exclude` rules to ignore `platformio.ini` when published to the **ESP Component Registry**.
- The library's `library.json` uses its `export.exclude` rules to ignore the project's `idf_component.yml` manifests when published to the **PlatformIO Registry**.

This reciprocal exclusion keeps the distributed packages clean and native to each ecosystem, without compromising the unified source layout during development. How those manifests are structured, and what each package ends up containing, is covered in [Chapter 5 - Publishing](05-publishing.md).

---

## 4.5 Unidirectional build conflict (`idf.py` → PlatformIO)

While **PlatformIO** isolates its output in `.pio/build/<env>`, native `idf.py` builds directly into `build/` at the project root.

If you compile with `idf.py` first, PlatformIO's CMake integration will pick up the existing `build/` cache and fail due to conflicting path configurations. Switching from PlatformIO to `idf.py` works fine, but going from `idf.py` to PlatformIO requires clearing the native build artifacts first:

```bash
idf.py fullclean
# or manually remove the build/ directory
```

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md) — [Introduction](../README.md) — [Chapter 5 - Publishing ➡️](05-publishing.md)

</div>
<!-- markdownlint-enable MD033 -->
