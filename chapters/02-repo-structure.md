# Chapter 2 — Repository Structure

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) examined each ecosystem in isolation. Taken individually, none of those constraints is hard to satisfy; the challenge begins when all of them must coexist within the same repository.

This chapter documents how those requirements are reconciled into a single folder structure that can be consumed simultaneously by [Arduino Library Manager](https://docs.arduino.cc/libraries/), [ESP Component Registry](https://components.espressif.com/), and [PlatformIO Registry](https://registry.platformio.org/), without maintaining separate branches or framework-specific distributions.

---

## 2.1 The Layout

```text
repo/                    ← repository root = component root = component name
├── idf_component.yml    ← ESP-IDF component manifest
├── CMakeLists.txt       ← SRCS "src/core/foo.cpp", INCLUDE_DIRS "src/core"
├── library.properties   ← Arduino
├── library.json         ← PlatformIO
├── src/
│   ├── Foo.h            ← Arduino public header (facade, camelCase methods)
│   ├── core/            ← shared implementation
│   │   ├── foo.h        ← ESP-IDF public header
│   │   └── foo.cpp
│   └── hal/             ← framework-specific HAL (see Chapter 3)
└── examples/
```

ESP-IDF recognizes a component by its `CMakeLists.txt`, which must live in the component root (see [Published components must be self-contained](01-ecosystem-constraints.md#1221-published-components-must-be-self-contained)). Placing `CMakeLists.txt` and `idf_component.yml` at the repository root therefore makes the repository itself the component: the ESP Component Registry packages the directory that contains the manifest, minus any exclusions declared under `files:` (see: [Manifest and Metadata](05-publishing.md#521-manifest-and-metadata)). The shared implementation under `src/core/` becomes part of the published package, while Arduino keeps discovering and compiling sources from `src/` exactly as expected.

The absence of a traditional `include/` directory, the usual home of public headers in both ESP-IDF components and PlatformIO libraries, is intentional. Since Arduino already requires public headers and implementation files to live within the `src/` tree, duplicating the same headers under a separate `include/` directory would introduce redundancy without providing any architectural benefit.

A layout with a separate `include/` directory is feasible too, but discouraged: the Arduino IDE only searches `src/`, so every file there has to reach the header through relative paths (`#include "../include/foo.h"` from the facade `Foo.h`, `#include "../../include/foo.h"` from `foo.cpp`), and these references multiply with every header and every level of the tree.

### 2.1.1 Arduino IDE Source & Include Path

As described in [1.1 Arduino](01-ecosystem-constraints.md#11-arduino), the Arduino IDE compiles everything under `src/` and searches only that folder for headers, so nothing needs to be configured: the facade `Foo.h` sits in `src/` and reaches the core with a path relative to it (`#include "core/foo.h"`).

### 2.1.2 ESP-IDF Source & Include Path

As described in [1.2.1](01-ecosystem-constraints.md#121-source-discovery-is-explicit), ESP-IDF source discovery is explicit, so the component's `CMakeLists.txt` lists the sources and points `INCLUDE_DIRS` straight at the core folder, where the public header lives:

```cmake
idf_component_register(
    SRCS
        "src/core/foo.cpp"
    INCLUDE_DIRS
        "src/core"
)
```

### 2.1.3 PlatformIO Source & Include Path

PlatformIO builds a library linked through `lib_deps` with its own library builder, whatever the framework. By default it compiles the sources under `src/` and its subfolders (recursively) and automatically adds both `src/` and `include/` (if present) to the include search path, but not their subdirectories (see [`includeDir`](https://docs.platformio.org/en/stable/manifests/library-json/fields/build/includedir.html) in the [`library.json`](https://docs.platformio.org/en/latest/manifests/library-json/index.html) reference). Headers located in a subfolder such as `src/core/` should be exposed through `library.json` (see [PlatformIO Manifest and Metadata](05-publishing.md#531-manifest-and-metadata)), for both frameworks:

```json
  "build": {
    "flags": ["-I src/core"]
  }
```

This plays a role similar to ESP-IDF's `INCLUDE_DIRS`: it extends the include search path without affecting the library's structure.

The library's `CMakeLists.txt` plays no part in this, not even under `framework = espidf`: `idf_component_register()` belongs to ESP-IDF's component model, while a library linked through `lib_deps` follows its `library.json`, so the `INCLUDE_DIRS "src/core"` it declares is never seen.

### 2.1.4 Layout conclusions

At this point, each toolchain finds sources and headers in its own way:

| | Sources | Include path | Where it is declared |
| :--- | :--- | :--- | :--- |
| **Arduino IDE** | everything under `src/`, recursively | `src/` only | nowhere: the layout is the configuration |
| **ESP-IDF** | listed explicitly in `SRCS` | `INCLUDE_DIRS` | the component's `CMakeLists.txt` |
| **PlatformIO** | everything under `src/`, recursively | `src/` and `include/`, without their subfolders, plus `build.flags` | `library.json` |

And the same source tree works unchanged for:

- Arduino Library Manager and Arduino IDE
- ESP-IDF local development via `EXTRA_COMPONENT_DIRS` and `idf_component.yml`
- ESP Component Registry installation via `idf.py add-dependency`
- PlatformIO local development and Registry, across both **Arduino** and **ESP-IDF** frameworks

---

## 2.2 Local folder name vs. GitHub repository name

A related trap, once the [Registry naming rules](01-ecosystem-constraints.md#registry-naming-rules) are understood, is the instinct to rename the GitHub repository to match the *lowercase* component name, especially for libraries that originated in the Arduino ecosystem with a *PascalCase* repository name.

**This isn't necessary.**

The key observation is that `EXTRA_COMPONENT_DIRS` and the Component Manager (in local deployment) operate on **what exists locally on disk at build time**, while a component installed from the Registry is resolved from its registered package name. In neither case does the GitHub repository name play a role: a component is named after its **local folder** ([Appendix A](../appendix/A-esp-idf-dependency-resolution.md)), and the clone command lets you choose it:

```bash
git clone https://github.com/<owner>/<repository>.git <component_name>
```

The same repository (history, tags, remote URL) lands in a folder named `<component_name>`, whatever the repository is called on GitHub. Renaming the repository would therefore solve nothing, while introducing real (if usually low) risk to existing registrations that may reference the current URL or name more strictly than expected.

The distinction becomes even more important once publication enters the picture: ***repository name, local folder name, and package name are three different things.*** The first two can usually be changed with limited consequences (a repository rename still carries the risks above); **the third often cannot.**

[Chapter 5 - Publishing](05-publishing.md) covers this in detail, including why package names on the Arduino Library Manager, PlatformIO Registry, and ESP Component Registry should be treated as **hard or costly to change** once published.

[Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md) builds on this layout, showing how to organize the code so that application logic stays shared across frameworks while framework-specific hardware access remains isolated.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) — [Introduction](../README.md) — [Chapter 3 - Agnostic Core & HAL Architecture ➡️](03-agnostic-core-hal-architecture.md)

</div>
<!-- markdownlint-enable MD033 -->
