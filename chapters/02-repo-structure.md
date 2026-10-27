# Chapter 2 — Repository Structure

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) examined each ecosystem in isolation. Taken individually, none of those constraints is hard to satisfy; the challenge begins when all of them must coexist within the same repository.

This chapter documents how those requirements are reconciled into a single folder structure that can be consumed simultaneously by [Arduino Library Manager](https://docs.arduino.cc/libraries/), [ESP Component Registry](https://components.espressif.com/), and [PlatformIO Registry](https://registry.platformio.org/), without maintaining separate branches or framework-specific distributions.

---

## 2.1 The layout

```text
repo/                    ← repository root = component root = component name
├── idf_component.yml    ← ESP-IDF component manifest
├── CMakeLists.txt       ← SRCS "src/core/foo.cpp", INCLUDE_DIRS "src/core"
├── library.properties   ← Arduino
├── library.json         ← PlatformIO
├── src/
│   ├── Foo.h            ← Arduino public header (camelCase facade)
│   ├── core/            ← shared implementation
│   │   ├── foo.h        ← ESP-IDF public header
│   │   └── foo.cpp
│   └── hal/             ← framework-specific HAL (see Chapter 3)
└── examples/
```

ESP-IDF recognizes a component by its `CMakeLists.txt`, which must live in the component root (see [Published components must be self-contained](01-ecosystem-constraints.md#published-components-must-be-self-contained)). Placing `CMakeLists.txt` and `idf_component.yml` at the repository root therefore makes the repository itself the component: the ESP Component Registry packages the directory that contains the manifest, minus any exclusions declared under `files:` (see: [Manifest and Metadata](05-publishing.md#521-manifest-and-metadata)). The shared implementation under `src/core/` becomes part of the published package, while Arduino keeps discovering and compiling sources from `src/` exactly as expected.

The absence of a traditional ESP-IDF `include/` directory is intentional. Since Arduino already requires public headers and implementation files to live within the `src/` tree, duplicating the same headers under a separate `include/` directory would introduce redundancy without providing any architectural benefit.

As a result, the same source tree works unchanged for:

- Arduino Library Manager
- Local development via `EXTRA_COMPONENT_DIRS` and `idf_component.yml`
- ESP Component Registry installation via `idf.py add-dependency`
- PlatformIO Registry, across both **Arduino** and **ESP-IDF** frameworks

---

## 2.2 Local folder name vs. GitHub repository name

A related trap, once the [Registry naming rules](01-ecosystem-constraints.md#registry-naming-rules) are understood, is the instinct to rename the GitHub repository to match the *lowercase* component name, especially for libraries that originated in the Arduino ecosystem with a *PascalCase* repository name.

**This isn't necessary.**

The key observation is that `EXTRA_COMPONENT_DIRS` and the Component Manager (in local deployment) operate on **what exists locally on disk at build time**, while a component installed from the Registry is resolved from its registered package name. In neither case does the GitHub repository name play a role.

Using:

```bash
git clone https://github.com/owner/Foo.git foo
```

clones the exact same repository (same history, same tags, same remote URL) into a **locally chosen** folder named `foo`, independent of whatever the repository is called on GitHub. As long as the local folder and manifest names satisfy the conventions from [Chapter 1]((01-ecosystem-constraints.md)), the remote name is irrelevant.

Renaming the GitHub repository solves nothing that a documented `git clone <URL> <lowercase-name>` instruction doesn't already solve, while introducing real (if usually low) risk to existing registrations that may reference the current URL or name more strictly than expected.

The distinction becomes even more important once publication enters the picture: ***repository name, local folder name, and package name are three different things.*** The first two can usually be changed with limited consequences (a repository rename still carries the risks above); **the third often cannot.**

[Chapter 5 - Publishing](05-publishing.md) covers this in detail, including why package names on the Arduino Library Manager, PlatformIO Registry, and ESP Component Registry should be treated as **hard or costly to change** once published.

[Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md) builds on this layout, showing how to organize the code so that application logic stays shared across frameworks while framework-specific hardware access remains isolated.

---

**Previous:** [← Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) · **Next:** [Chapter 3 - Agnostic Core & HAL Architecture →](03-agnostic-core-hal-architecture.md)
