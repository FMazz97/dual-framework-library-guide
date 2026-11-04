# Chapter 1 — Ecosystem Constraints

Before discussing repository layouts, abstraction layers, or build integration, it's worth identifying the small set of rules that actually drive the entire architecture. These constraints are neither numerous nor particularly complex: most of the design decisions presented throughout this guide are simply the result of satisfying them simultaneously within a single repository.

This chapter summarizes those rules and their practical consequences on library structure, component identity, and source organization.

---

## 1.1 Arduino

### 1.1.1 Source discovery is structural

The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every source file (`.cpp` / `.c` / `.s`) found recursively under `src/`** (see: [Library specification - Layout of folders and files](https://docs.arduino.cc/arduino-cli/library-specification/#layout-of-folders-and-files)). There is no manifest listing compilation units: structure alone decides what gets built, so implementation files placed outside `src/` **are never compiled**.

**Library identity is driven by public headers**, not folder names. As long as the expected public include (e.g. `#include <Foo.h>`) is available under `src/`, the library works regardless of how its directory is named on disk.

For more details, see [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/).

---

## 1.2 ESP-IDF (CMake)

This guide targets the CMake-based ESP-IDF build system, the standard since ESP-IDF 4.x. Its fundamental building blocks are **components**: Espressif deliberately uses this term rather than *library* because components are self-contained units that can expose functionality, depend on other components, and participate directly in the build system.

Three concepts must be distinguished:

- **Component:** a reusable unit of source code registered through `idf_component_register()` in the component's root `CMakeLists.txt`. Its name comes from the folder that contains it.
- **Component Manager:** an optional dependency-resolution layer that can discover, validate, and download components before the build begins.
- **Component Registry:** the online distribution service used to publish and retrieve components.

For more details, see [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md) on how components are discovered, and Espressif's guide ["What is the ESP Component Registry?"](https://developer.espressif.com/blog/2024/10/what-is-the-esp-registry/).

### 1.2.1 Source discovery is explicit

Unlike Arduino, ESP-IDF does not discover source files automatically from directory conventions. Every component must explicitly declare its source files and public include in its `CmakeLists.txt`:

```cmake
idf_component_register(
    SRCS
        "foo.c"
    INCLUDE_DIRS
        "include"
)
```

Only files declared through this mechanism participate in the build.

### 1.2.2 Publication imposes the constraints

ESP-IDF itself places very few restrictions on how components are organized locally: as long as CMake can locate the component and `idf_component_register()` points to the correct files, any layout is technically valid.

The constraints come from publication. The commands that package and upload a component, `compote component pack` and `compote component upload` (see: [Compote CLI](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/compote_cli.html)), enforce additional expectations for **reproducible and reliable components**, and two of them shape the whole guide.

#### 1.2.2.1 Published components must be self-contained

The packaging commands run from the component's root folder and include only that folder itself, so any source file referenced outside that boundary will be missing once the component is downloaded from the **ESP Component Registry**. Publishable components must therefore be self-contained, with `CMakeLists.txt` in the component root and every required source inside it, as Espressif's [component files reference](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/component_files.html) specifies.

#### 1.2.2.2 Registry naming rules

The same packaging commands normalize names to lowercase and validate them against a restricted slug format.

Specifically, published component names:

- must be between **2 and 64 characters** long;
- may contain letters (`a-Z`); however, names are normalized to lowercase before validation, so the effective character set is (`a-z`);
- may contain digits (`0-9`);
- may contain underscores (`_`);
- may contain hyphens (`-`);
- cannot begin or end with `_` or `-`;
- cannot contain consecutive separators such as `__`, `--`, `_-`, or `-_`.

> [!NOTE]
> These rules were revealed by inspecting the Component Manager source code: [idf_component_manager/cli/validations.py](https://github.com/espressif/idf-component-manager/blob/main/idf_component_manager/cli/validations.py) and [idf_component_tools/manifest/constants.py](https://github.com/espressif/idf-component-manager/blob/main/idf_component_tools/manifest/constants.py).

**This can be surprising: the rules are enforced by the publication tooling, not by ESP-IDF's CMake configuration system.** The upload process silently converts names to lowercase before validation, without notifying the user. At the time of writing, the behavior is not documented.

### 1.2.3 The practical rule of thumb

When creating a component intended for reuse or publication, name it as if it were going to be published from day one. To maintain total consistency across toolchains, use the **exact same lowercase identifier**, in a 1-to-1 mapping, across every layer:

```text
Component folder name (<name>/)
└── Exposed component name (REQUIRES <name>)
    └── Component Manager manifest (idf_component.yml → name: "<name>")
```

This aligns local development and registry publication from the start, preventing breaking renames and identity mismatches down the road.

For more details, see [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md).

---

## 1.3 PlatformIO

PlatformIO is highly configurable by design and introduces no structural constraints on the library: as long as `platformio.ini` (and, for a library, its `library.json`) points to the correct sources and include paths, any layout is technically valid. As a result, the repository layout described in [Chapter 2 - Repository Structure](02-repo-structure.md) satisfies PlatformIO without any source reorganization, only a declaration of the folder holding the core headers.

### 1.3.1 `framework = arduino`

Under the Arduino framework, metadata, packaging, and examples are driven by PlatformIO's `library.json` manifest, including its structured `"examples"` field, with a fallback to the Arduino conventions when the manifest doesn't define them (see [Arduino examples on PlatformIO](04-examples.md#arduino-examples-on-platformio)).

### 1.3.2 `framework = espidf`

Under ESP-IDF, PlatformIO does more than simply invoke `idf.py`: it generates its own project scaffolding, driven by `platformio.ini` rather than by ESP-IDF's native project layout.

The convention therefore shifts from ESP-IDF's traditional `main/` directory to PlatformIO's `src_dir` (which defaults to `src/`). A project can still preserve a fully native ESP-IDF structure by explicitly setting `src_dir = main`; the example layout that builds under both toolchains is described in [Chapter 4 - Example Conventions](04-examples.md#esp-idf-examples-on-platformio).

How a dependency is built depends on how it enters the project:

- **Libraries linked through `lib_deps`** continue to follow PlatformIO's library model and are processed by its library builder, with the defaults described in [Chapter 2](02-repo-structure.md).
- **Components discovered through `EXTRA_COMPONENT_DIRS`, the project's `components/` folder, or the Component Manager (`idf_component.yml`)** follow ESP-IDF's native component model (see [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)).

For more details, see [PlatformIO Library Management](https://docs.platformio.org/en/latest/librarymanager/).

---

## 1.4 Where the constraints collide

Comparing these three ecosystems reveals that the true challenge is not managing different APIs, but harmonizing conflicting toolchain assumptions about source discovery, identity, and publication:

| Ecosystem | Structural constraint | Identity constraint |
| :--- | :--- | :--- |
| **Arduino** | All compilable sources **must** live under `src/` | Library identity is independent of the local folder name or `library.properties` manifest |
| **ESP-IDF** | Published components **must** be self-contained | Registry-compatible components require **lowercase** slug format (see [1.2.2.2](#1222-registry-naming-rules)) |
| **PlatformIO** | Introduces no additional structural constraints | Package identity comes from `library.json` |

The root conflict comes down to just two non-negotiable rules:

- **Arduino's source discovery** (everything compiled lives under `src/`)
- **ESP-IDF's publication requirements** (self-contained components with strict naming)

They are not inherently incompatible: the challenge is simply finding a repository layout that satisfies both simultaneously.

The layout used throughout this guide exploits exactly that: the ESP-IDF component files (`CMakeLists.txt` and `idf_component.yml`) are placed at the repository root, making the repository itself the component boundary, while `src/` remains the single source of truth for the implementation. [Chapter 2 - Repository Structure](02-repo-structure.md) shows the resulting layout and explains why it works.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[Introduction](../README.md) — [Chapter 2 - Repository Structure ➡️](02-repo-structure.md)

</div>
<!-- markdownlint-enable MD033 -->
