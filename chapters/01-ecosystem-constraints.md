# Chapter 1 — Ecosystem Constraints

Before discussing repository layouts, abstraction layers, or build integration, it's worth identifying the small set of rules that actually drive the entire architecture. These constraints are neither numerous nor particularly complex: most of the design decisions presented throughout this guide are simply the result of satisfying them simultaneously within a single repository.

This chapter summarizes those rules and their practical consequences on library structure, component identity, and source organization.

---

## 1.1 Arduino

### Source discovery is structural

The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every source file (`.cpp` / `.c` / `.s`) file found recursively under `src/`** (see: [Library specification - Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files)). There is no manifest listing compilation units: structure alone decides what gets built, so implementation files placed outside `src/` **are never compiled**.

**Library identity is driven by public headers**, not folder names. As long as the expected public include (e.g. `#include <Foo.h>`) is available under `src/`, the library works regardless of how its directory is named on disk.

For more details, see [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/).

---

## 1.2 ESP-IDF (CMake)

This guide targets the CMake-based ESP-IDF build system, the standard since ESP-IDF 4.x. Its fundamental building blocks are **components**: Espressif deliberately uses this term rather than *library* because components are self-contained units that can expose functionality, depend on other components, and participate directly in the build system.

To understand the repository structure developed throughout this guide, three concepts must be distinguished:

- **Component:** a reusable unit of source code registered through `idf_component_register()` in the component's root `CMakeLists.txt`. Its name comes from the folder that contains it.
- **Component Manager:** an optional dependency-resolution layer that can discover, validate, and download components before the build begins.
- **Component Registry:** the online distribution service used to publish and retrieve components.

For more details, see [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md) on how components are discovered, and Espressif's guide ["What is the ESP Component Registry?"](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/).

### Source discovery is explicit

Unlike Arduino, ESP-IDF does not discover source files automatically from directory conventions. Every component must explicitly declare its source files and public include directories using `SRCS` and `INCLUDE_DIRS`:

```cmake
idf_component_register(
    SRCS
        "foo.c"
    INCLUDE_DIRS
        "include"
)
```

Only files declared through this mechanism participate in the build.

### Publication imposes the constraints

ESP-IDF itself places very few restrictions on how components are organized locally: as long as CMake can locate the component and `idf_component_register()` points to the correct files, any layout is technically valid.

Publication, however, introduces additional expectations for **reproducible and reliable components**.

#### Published components must be self-contained

The **ESP Component Registry** packages only the component itself: `compote component pack` and `compote component upload` are run from the component's root folder. Any source file referenced outside that boundary will be missing once the component is downloaded from the registry. Publishable components must therefore be self-contained, with `CMakeLists.txt` in the component root (as Espressif's [Component Files](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/component_files.html) reference specifies) and every required source inside it.

#### Registry naming rules

During local development, component names require little attention. Publication through the **ESP Component Registry**, however, enforces naming rules in the publication tooling: during `compote component pack` and `compote component upload`, component names are normalized to lowercase and validated against a restricted slug format (as revealed by inspecting the Component Manager source code).

Specifically, published component names:

- may contain letters (`a-z`);
- may contain digits (`0-9`);
- may contain underscores (`_`);
- may contain hyphens (`-`);
- cannot begin or end with `_` or `-`;
- cannot contain consecutive separators such as `__`, `--`, `_-`, or `-_`.

(see: [idf_component_manager/cli/validations.py](https://github.com/espressif/idf-component-manager/blob/main/idf_component_manager/cli/validations.py) and [idf_component_tools/manifest/constants.py](https://github.com/espressif/idf-component-manager/blob/main/idf_component_tools/manifest/constants.py))

**This can be surprising: the rules are enforced by the publication tooling, not by ESP-IDF's CMake build system.** The upload process silently converts names to lowercase before validation, without notifying the user, and at the time of writing, the behavior is not documented.

### The practical rule of thumb

When creating a component intended for reuse or publication, name it as if it were going to be published from day one. To maintain total consistency across toolchains, use the **exact same lowercase identifier**, in a 1-to-1 mapping, across every layer:

```text
Component folder name (<name>/)
└── Exposed component name (REQUIRES <name>)
    └── Component Manager manifest (idf_component.yml → name: "<name>")
```

This aligns local development and registry publication from the start, preventing breaking renames and identity mismatches down the road.

For more details, see [IDF Component Manager](https://docs.espressif.com/projects/idf-component-manager/).

---

## 1.3 PlatformIO

PlatformIO introduces no additional structural constraints on the library itself. When used with Arduino, it follows Arduino's library conventions; when used with ESP-IDF, it follows ESP-IDF's component model. As a result, the repository layout described in [Chapter 2 - Repository Structure](02-repo-structure.md) already satisfies PlatformIO without any other source organization.

The only area that typically requires additional attention is the example/project structure on the ESP-IDF side, where the same examples may need to remain buildable under both native ESP-IDF and PlatformIO's ESP-IDF integration (see: [Chapter 4 - Example Conventions](04-examples.md)).

### `framework = arduino`

Source discovery follows Arduino's recursive `src/` compilation model. Metadata and examples are driven by PlatformIO's `library.json` manifest (including its structured `examples` field, falling back to standard Arduino conventions when  is missing).

### `framework = espidf`

Here PlatformIO doesn't just wrap `idf.py`: it generates its **own** project scaffolding, driven by `platformio.ini` rather than IDF's native `CMakeLists.txt` flow. The convention shifts from IDF's `main/` folder to PlatformIO's own `src_dir` (default `src/`). A project can still use a hand-written, IDF-native `CMakeLists.txt`/`main/` layout under PlatformIO, but only by explicitly setting `src_dir = main`.

For more details, see [PlatformIO Library Management](https://docs.platformio.org/en/latest/librarymanager/).

---

## 1.4 Where the constraints collide

Comparing these three ecosystems reveals that the true challenge of a cross-ecosystem library is not managing different APIs, but harmonizing conflicting toolchain assumptions about source discovery, identity, and publication:

| Ecosystem | Primary structural constraint | Identity constraint |
| :--- | :--- | :--- |
| **Arduino** | All compilable sources **must** live under `src/` | Library identity is independent of the local folder name |
| **ESP-IDF** | Published components **must** be self-contained | Component identity depends on the mechanism in use; *Registry-compatible* components require **lowercase** `<name>` values |
| **PlatformIO** | Introduces no additional structural constraints beyond the selected framework | Package identity comes from `library.json` |

The root conflict comes down to just two non-negotiable rules: **Arduino's source discovery** (everything compiled lives under `src/`) and **ESP-IDF's publication requirements** (self-contained components with strict naming). They are not inherently incompatible: the challenge is simply finding a repository layout that satisfies both simultaneously.

The layout used throughout this guide exploits exactly that: the ESP-IDF component files (`CMakeLists.txt` and `idf_component.yml`) are placed at the repository root, making the repository itself the component boundary, while `src/` remains the single source of truth for the implementation. [Chapter 2 - Repository Structure](02-repo-structure.md) shows the resulting layout and explains why it works.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[Introduction](../README.md) — [Chapter 2 - Repository Structure ➡️](02-repo-structure.md)

</div>
<!-- markdownlint-enable MD033 -->
