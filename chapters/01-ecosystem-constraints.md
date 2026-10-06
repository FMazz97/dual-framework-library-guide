# Chapter 1 — The Two Ecosystems and Their Non-Negotiable Constraints

Before discussing abstraction layers, repository layouts, or build integration, it's important to understand a fundamental reality: **the hardest part of a cross-ecosystem library is rarely the code itself**.

Each ecosystem comes with its own assumptions about project structure, source discovery, dependency management, examples, packaging, and publication. **These are not conventions that can be bent or negotiated**: they are hard requirements enforced by the tooling. Ignore them, and the build breaks, the library fails validation, or the package never gets published.

The challenge, therefore, is not simply making Arduino and ESP-IDF compile the same code. It is designing a repository that simultaneously satisfies two independent ecosystems (and the cross-platform toolchains like PlatformIO) each with its own rules, directory structures, and developer expectations.

Everything described in this chapter was verified empirically through real builds, real hardware testing, and real publication attempts. Several assumptions that seemed perfectly reasonable when reading the documentation turned out to be incorrect once exercised in practice.

---

## 1.1 Arduino

### Source discovery is primarily structural

The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every `.cpp` file found recursively under `src/`** ([See: Library specification - Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files)). There is no manifest that explicitly lists compilation units: if a source file exists under `src/`, it gets compiled. Because the Arduino library specification is built around this standardized layout, placing implementation files outside of `src/` is **not permitted**.

**Library identity is driven by public headers**, not folder names. As long as the expected public include (e.g. `#include <Foo.h>`) is available under `src/`, the library continues to function regardless of how its directory is named on disk.

For more details, see [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/).

---

## 1.2 ESP-IDF (native)

### Header and source visibility is explicit

ESP-IDF requires every component to explicitly declare its source files and public include directories using `SRCS` and `INCLUDE_DIRS` inside `idf_component_register()` within `CMakeLists.txt`. A header or source file is only visible to the rest of the project if it is intentionally declared. This makes the API boundary completely explicit, but also means the component structure must be designed deliberately from the start:

```CMake
idf_component_register(
    SRCS
        "src/foo.cpp"
    INCLUDE_DIRS
        "src"
)
```

### Component identity is defined differently by `CMake` and the `Component Manager`

This turned out to be one of the most time-consuming constraints to pin down, precisely because it is not documented as a unified rule anywhere and only becomes obvious when each mechanism is tested in isolation:

- **`EXTRA_COMPONENT_DIRS` + `REQUIRES`**: purely `CMake`-*based*. Components are registered under the exact name of the folder that contains them, and `REQUIRES` must reference that name exactly. In this workflow, the component's identity comes entirely from the filesystem.

- **`idf_component.yml`** (via the **Component Manager**, either through `override_path` during development or via the `Registry`): introduces a second source of identity. The Component Manager reads the component's `name:` directly from the manifest and validates it **according to Registry rules**. Unlike a plain CMake-based workflow, the `name:` field must follow the Registry naming convention **containing only lowercase letters, numbers, and underscores**.

See [Section 4.1 - IDF dependency resolution](04-examples.md#41-esp-idf-dependency-resolution-cmake-and-component-manager-are-different-layers) for how these two mechanisms discover, validate, and build a component.

### A *published* component must be self-contained

While local development imposes few restrictions on component layout, publication introduces an additional constraint: the component must be self-contained. A component intended only for local use may be organized anywhere in the repository (for example under `components/<name>/`) and may freely reference sources outside its own directory, provided those paths exist on the local filesystem.

But, the ESP Component Registry packages **only the folder containing `idf_component.yml`**. Any `CMakeLists.txt` that references source files *outside* that folder via relative paths (e.g. `../../src/core/foo.cpp`) works fine during local development, but breaks as soon as the component is published and downloaded standalone: those paths simply do not exist in the extracted package.

**The practical rule of thumb is simple:** keep the component folder name and the manifest `name:` identical, using **lowercase** for both. This satisfies **CMake**'s filesystem-based resolution, the **Component Manager**'s validation rules, and the **ESP Component Registry**'s publication requirements with a single convention.

For more details, see [IDF Component Manager](https://docs.espressif.com/projects/idf-component-manager/).

---

## 1.3 PlatformIO

PlatformIO introduces no additional structural constraints on the library itself. When used with Arduino, it follows Arduino's library conventions; when used with ESP-IDF, it follows ESP-IDF's component model. As a result, the repository layout established in the previous sections already satisfies PlatformIO without requiring any PlatformIO-specific source organization.

The only area that typically requires additional attention is the example/project structure on the ESP-IDF side, where the same examples may need to remain buildable under both native ESP-IDF and PlatformIO's ESP-IDF integration.

### `framework = arduino`

Source discovery remains based on Arduino's recursive `src/` compilation model, and examples continue to follow Arduino's expectations. PlatformIO adds its own metadata layer through `library.json`, including the richer `examples` field, but this metadata is consumed by multiple internal PlatformIO surfaces whose behavior is similar, yet not entirely equivalent. Chapter 5 covers the resulting edge cases in detail.

### `framework = espidf`

Here PlatformIO doesn't just wrap `idf.py` — it generates its **own** project scaffolding, driven by `platformio.ini` rather than IDF's native `CMakeLists.txt`/`project()` flow. The convention shifts from IDF's `main/` folder to PlatformIO's own `src_dir` (default `src/`). A project can still use a hand-written, IDF-native `CMakeLists.txt`/`main/` layout under PlatformIO, but only by explicitly pointing `src_dir` at that folder.

For more details, see [PlatformIO Library Management](https://docs.platformio.org/en/latest/librarymanager/).

---

## 1.4 Where the constraints collide

Comparing these three ecosystems reveals that the true challenge of a cross-ecosystem library is not managing different APIs, but harmonizing conflicting toolchain assumptions about source discovery, identity, and metadata immutability:

| Ecosystem | Primary structural constraint | Identity constraint |
| :--- | :--- | :--- |
| **Arduino** | All compilable sources **must** live under `src/` | Library identity is independent of the local folder name |
| **ESP-IDF** | Published components **must** be self-contained | Component identity depends on the mechanism in use; *Registry-compatible* components require **lowercase** `name:` values |
| **PlatformIO** | Introduces no additional structural constraints beyond the selected framework | Package identity comes from `library.json` |

The root conflict lies in just two non-negotiable rules:

- **Arduino** expects the library's implementation to live under `src/`, automatically compiling every `.cpp` file found there.

- **ESP-IDF** expects published components to be self-contained and imposes stricter naming requirements when the Component Manager and Registry are involved.

Fortunately, **Arduino's source-discovery** requirements and **ESP-IDF's publication** requirements are not inherently incompatible. The challenge is simply finding a repository layout that satisfies both simultaneously.

The layout used throughout this guide exploits exactly that: the ESP-IDF manifest is placed at the repository root, making the repository itself the component boundary, while `src/` remains the single source of truth for the implementation. [Chapter 2 - Repository structure](02-repo-structure.md) explains why this specific arrangement works and why alternative layouts fail once publication enters the picture.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - Example conventions](04-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](06-bonus_yield-watchdog_naming-convention.md)
