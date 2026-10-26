# Chapter 1 — The Two Ecosystems and Their Non-Negotiable Constraints

Before discussing repository layouts, abstraction layers, or build integration, it's worth identifying the small set of rules that actually drive the entire architecture.

Fortunately, the constraints imposed by Arduino and ESP-IDF are neither numerous nor particularly complex. Most of the design decisions presented throughout this guide are simply the result of satisfying those constraints simultaneously within a single repository.

This chapter summarizes those rules and the practical consequences they have on library structure, component identity, and source organization.

---

## 1.1 Arduino

### Source discovery is structural

The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every `.cpp` file found recursively under `src/`** ([See: Library specification - Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files)). There is no manifest that explicitly lists compilation units: if a source file exists under `src/`, it gets compiled. Because the Arduino library specification is built around this standardized layout, placing implementation files outside of `src/` **will not be compiled**.

**Library identity is driven by public headers**, not folder names. As long as the expected public include (e.g. `#include <Foo.h>`) is available under `src/`, the library continues to function regardless of how its directory is named on disk.

For more details, see [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/).

---

## 1.2 ESP-IDF (CMake)

This guide targets the modern CMake-based ESP-IDF build system, which became the standard build system for ESP-IDF projects starting from ESP-IDF 4.x and later. Components are the fundamental building blocks of this architecture.

Espressif deliberately uses the term component rather than library because components are designed to be self-contained units that can expose functionality, depend on other components, and participate directly in the build system.

To understand the repository structure developed throughout this guide, three concepts must be distinguished:

- **Component:** a reusable unit of source code registered through `idf_component_register()` within root's `CMakeLists.txt`. Its name come from the folder that contains it.
- **Component Manager:** an optional dependency-resolution layer that can discover, validate, and download components before the build begins.
- **Component Registry:** the online distribution service used to publish and retrieve components.

For more details, see [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md) for how discover a component and official Espressif guide ["What is the ESP Component Registry?"](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/)

### Source discovery is explicit

Unlike Arduino, ESP-IDF never discovers source files automatically based on directory conventions.

Every component must explicitly declare its source files and public include directories using `SRCS` and `INCLUDE_DIRS`:

```CMake
idf_component_register(
    SRCS
        "foo.c"
    INCLUDE_DIRS
        "include"
)
```

Only files declared through this mechanism participate in the build.

### *Publication* imposes the constraints

ESP-IDF itself places very few restrictions on how components are organized locally. As long as CMake can locate the component and `idf_component_register()` points to the correct files, all different layouts are technically valid.

Publication, however, introduces additional expectations for **reproducible and reliable components**.

#### Published components must be self-contained

The **ESP Component Registry** packages only the component itself via `compote component pack`/`compote component upload` command given in the root of the component's folder. Any source files referenced outside the component boundary will be unavailable once the component is downloaded independently througt the registry.

For this reason, publishable components should be structured as self-contained units, with their `CMakeLists.txt` located at the component root and all required sources residing within that package boundary.

#### Registry naming rules

Generally, component names require little attention during local development. As long as the component can be located by CMake and the underlying filesystem accepts the chosen name, ESP-IDF itself imposes very few restrictions.

Publication through the **ESP Component Registry**, however, introduces additional naming requirements enforced by the publication tooling.

During `compote component pack` and `compote component upload`, inspection of the Component Manager source code reveals that component names are normalized to lowercase and validated against a restricted slug format before publication.

Specifically, published component names:

- may contain letters (`a-z`);
- may contain digits (`0-9`);
- may contain underscores (`_`);
- may contain hyphens (`-`);
- cannot begin or end with `_` or `-`;
- cannot contain consecutive separators such as `__`, `--`, `_-`, or `-_`.

(see: [idf_component_manager/cli/validations.py](https://github.com/espressif/idf-component-manager/blob/main/idf_component_manager/cli/validations.py) and [idf_component_tools/manifest/constants.py](https://github.com/espressif/idf-component-manager/blob/main/idf_component_tools/manifest/constants.py))

**This behavior can be surprising because these constraints are enforced by the publication tooling rather than by ESP-IDF's CMake build system itself.** The upload process silently normalizes names to lowercase before validation, without reporting that conversion to the user and it also not documented anywhere.

### The practical rule of thumb

When creating a component intended for reuse or publication, name it as if it were going to be published from day one.

To maintain total consistency across toolchains, use the **exact same lowercase identifier** in a 1-to-1 mapping across every layer:

```text
  Component folder name (<name>/)
  └── Exposed component name (REQUIRES <name>)
      └── Component Manager manifest (idf_component.yml → name: "<name>")
```

In practice, this unified identifier strictly aligns local development and registry publication from the start, preventing breaking renames and identity mismatches down the road.

For more, see [IDF Component Manager](https://docs.espressif.com/projects/idf-component-manager/).

---

## 1.3 PlatformIO

PlatformIO introduces no additional structural constraints on the library itself. When used with Arduino, it follows Arduino's library conventions; when used with ESP-IDF, it follows ESP-IDF's component model. As a result, the repository layout established in the previous sections already satisfies PlatformIO without requiring any PlatformIO-specific source organization.

The only area that typically requires additional attention is the example/project structure on the ESP-IDF side, where the same examples may need to remain buildable under both native ESP-IDF and PlatformIO's ESP-IDF integration (see: [Chapter 4 - Example Conventions](04-examples.md)).

### `framework = arduino`

Source discovery relies on Arduino’s recursive `src/` compilation model, while examples are driven by PlatformIO’s `library.json` manifest, falling back to standard Arduino conventions when the manifest is absent. PlatformIO introduces its own metadata layer through `library.json` (including the structured `examples` field).

### `framework = espidf`

Here PlatformIO doesn't just wrap `idf.py` — it generates its **own** project scaffolding, driven by `platformio.ini` rather than IDF's native `CMakeLists.txt` flow. The convention shifts from IDF's `main/` folder to PlatformIO's own `src_dir` (default `src/`). A project can still use a hand-written, IDF-native `CMakeLists.txt`/`main/` layout under PlatformIO, but only by explicitly `src_dir = main`.

For more details, see [PlatformIO Library Management](https://docs.platformio.org/en/latest/librarymanager/).

---

## 1.4 Where the constraints collide

Comparing these three ecosystems reveals that the true challenge of a cross-ecosystem library is not managing different APIs, but harmonizing conflicting toolchain assumptions about source discovery, identity, and metadata immutability:

| Ecosystem | Primary structural constraint | Identity constraint |
| :--- | :--- | :--- |
| **Arduino** | All compilable sources **must** live under `src/` | Library identity is independent of the local folder name |
| **ESP-IDF** | Published components **must** be self-contained | Component identity depends on the mechanism in use; *Registry-compatible* components require **lowercase** `<name>` values |
| **PlatformIO** | Introduces no additional structural constraints beyond the selected framework | Package identity comes from `library.json` |

The root conflict lies in just two non-negotiable rules:

- **Arduino** expects the library's implementation to live under `src/`, automatically compiling every `.cpp` file found there.

- **ESP-IDF** expects published components to be self-contained and imposes stricter naming requirements when the Component Manager and Registry are involved.

Fortunately, **Arduino's source-discovery** requirements and **ESP-IDF's publication** requirements are not inherently incompatible. The challenge is simply finding a repository layout that satisfies both simultaneously.

The layout used throughout this guide exploits exactly that: the ESP-IDF manifest is placed at the repository root, making the repository itself the component boundary, while `src/` remains the single source of truth for the implementation. [Chapter 2 - Repository Structure](02-repo-structure.md) explains why this specific arrangement works and why alternative layouts fail once publication enters the picture.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository Structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md)
- [Chapter 4 - Example Conventions](04-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)
- [Appendix B - Yield & Watchdog](../appendix/B-yield-watchdog.md)
- [Appendix C - Naming Convention](../appendix/C-naming-convention.md)
