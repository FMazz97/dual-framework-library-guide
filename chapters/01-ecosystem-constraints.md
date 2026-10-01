# Chapter 1 — The Two Ecosystems and Their Non-Negotiable Constraints

Before discussing abstraction layers, repository layouts, or build integration, it's important to understand a fundamental reality: **the hardest part of a cross-ecosystem library is rarely the code itself**.

Each ecosystem comes with its own assumptions about project structure, source discovery, dependency management, examples, packaging, and publication. **These are not conventions that can be bent or negotiated**: they are hard requirements enforced by the tooling. Ignore them, and the build breaks, the library fails validation, or the package never gets published.

The challenge, therefore, is not simply making Arduino and ESP-IDF compile the same code. It is designing a repository that simultaneously satisfies two independent ecosystems (and the cross-platform toolchains like PlatformIO) each with its own rules, directory structures, and developer expectations.

Everything described in this chapter was verified empirically through real builds, real hardware testing, and real publication attempts. Several assumptions that seemed perfectly reasonable when reading the documentation turned out to be incorrect once exercised in practice.

---

## 1.1 Arduino

### Source discovery is primarily structure

The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every `.cpp` file found recursively under `src/`**. There is no manifest that explicitly lists compilation units: if a source file exists under `src/`, it gets compiled. Because the *Arduino Library Manager* expects libraries to follow this standardized layout, reorganizing sources outside of `src/` is **not permetted**.

### Library metadata in `library.properties`is taken from the tagged release, not the default branch

During publication, the *Arduino Library Manager* indexes the content associated with the Git tag being released. Updating `library.properties` after the tag already exists has no effect on that release. Likewise, publishing before incrementing the version field in `library.properties` typically results in the release being ignored or treated as a duplicate. The version declared in `library.properties` and the *GitHub release tag* must refer to the same release version.

### Examples follow a strict `Name/Name.ino` convention

The Arduino IDE (and PlatformIO's example browser for `.ino` projects; see Section 1.3) expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`. The sketch name must match its parent directory. Additional levels of nesting can be used to organize examples into categories and do not affect discovery, provided this naming convention is preserved.

### Dependency resolution is supported, but remains library-centric

Arduino libraries can declare dependencies using the `depends` field in `library.properties`, enabling automatic installation of required libraries by the Library Manager.

---

## 1.2 ESP-IDF (native)

### A published component must be self-contained

The IDF Component Registry packages **only the folder containing `idf_component.yml`**. Any `CMakeLists.txt` that references source files *outside* that folder via relative paths (e.g. `../../src/core/foo.cpp`) works fine during local development, but breaks as soon as the component is published and downloaded standalone: those paths simply do not exist in the extracted package.

### Header visibility is explicit

ESP-IDF exposes only the include directories explicitly declared through `INCLUDE_DIRS` in `idf_component_register()` on `CMakeLists.txt`. A header is part of the component's public interface only if it is intentionally exported. This makes the API boundary explicit, but also means the component structure must be designed deliberately from the start.

### Name resolution depends entirely on *which* mechanism declares the dependency, and each mechanism has its own source of truth

This turned out to be one of the most time-consuming constraints to pin down, precisely because it is not documented as a unified rule anywhere and only becomes obvious when each mechanism is tested in isolation:
  
- **`EXTRA_COMPONENT_DIRS` + `REQUIRES`**: purely filesystem-based. CMake scans the listed directories and registers each component **under the exact name of the folder that contains it** — case-sensitive and with no fallback. `REQUIRES foo` will fail to resolve a component sitting in a folder named `Foo`, even though the two strings differ only in case.

- **`idf_component.yml`** (via the Component Manager, either through `override_path` during development or via the Registry): resolves the component's identity from the `name:` field **inside the manifest itself**. As with `EXTRA_COMPONENT_DIRS`, keeping the `name:` value identical to the component folder name avoids resolution inconsistencies across dependency mechanisms.

The practical rule of thumb is straightforward: **the component folder name and the component name declared in `idf_component.yml` should match.** For components intended to be published through the ESP-IDF Component Registry, keeping both names lowercase provides the most consistent behavior across local development, dependency resolution, and package distribution.

### Component metadata is taken from the published package version, not from the current repository state

Once a version is published to the Component Registry, the corresponding `idf_component.yml` and associated metadata are immutable from the consumer's perspective. Updating the manifest on the default branch has no effect on already-published versions; a new version must be released for dependency or metadata changes to become visible.

### Dependency management is component-based

Components declare their dependencies through mechanisms such as `REQUIRES`, `PRIV_REQUIRES`, or `idf_component.yml`, allowing ESP-IDF to construct an explicit dependency graph at build time.

---

## 1.3 PlatformIO

PlatformIO introduces no additional structural constraints on the library itself. When used with Arduino, it follows Arduino's library conventions; when used with ESP-IDF, it follows ESP-IDF's component model. As a result, the repository layout established in the previous sections already satisfies PlatformIO without requiring any PlatformIO-specific source organization.

The only area that typically requires additional attention is the example/project structure on the ESP-IDF side, where the same examples may need to remain buildable under both native ESP-IDF and PlatformIO's ESP-IDF integration.

### `framework = arduino`

Source discovery remains based on Arduino's recursive `src/` compilation model, and examples continue to follow Arduino's expectations. PlatformIO adds its own metadata layer through `library.json`, including the richer `examples` field, but this metadata is consumed by multiple internal PlatformIO surfaces whose behavior is similar, yet not entirely equivalent. Chapter 5 covers the resulting edge cases in detail.

### `framework = espidf`

Here PlatformIO doesn't just wrap `idf.py` — it generates its **own** project scaffolding, driven by `platformio.ini` rather than IDF's native `CMakeLists.txt`/`project()` flow. The convention shifts from IDF's `main/` folder to PlatformIO's own `src_dir` (default `src/`). A project can still use a hand-written, IDF-native `CMakeLists.txt`/`main/` layout under PlatformIO, but only by explicitly pointing `src_dir` at that folder.

### Package metadata is taken from the published package version, not from the current repository state

Once a version is published to PlatformIO Registry, the corresponding manifest `library.json` and associated metadata are immutable from the consumer's perspective. Updating the repository's default branch has no effect on already-published versions; a new version must be released for changes to become visible. Consequently, early naming decisions are permanent, making it best practice to adopt the strictest convention — such as ESP-IDF's lowercase style — from day one.

### Dependency management is package-based

Dependencies are declared in `library.json` and resolved through the PlatformIO Registry, allowing PlatformIO to automatically fetch and install the required packages.

---

## 1.4 Where the constraints collide

Comparing these three ecosystems reveals that the true challenge of a cross-ecosystem library is not managing different APIs, but harmonizing conflicting toolchain assumptions about source discovery, identity, and metadata immutability:

| | Source layout expected | Name resolved from | Case-sensitive? |
| :--- | :--- | :--- | :--- |
| **Arduino** (IDE / PlatformIO) | Everything under `src/`, compiled recursively | `name` field in `library.properties` / `library.json` | No practical constraint observed |
| **ESP-IDF**: `EXTRA_COMPONENT_DIRS` + `REQUIRES` | Component folder, referenced directly | `REQUIRES name` in CMake & local folder *name* | **Yes** — must match exactly |
| **ESP-IDF**: `idf_component.yml` | Self-contained component folder | `name:` field inside the manifest & local folder *name* | **Yes** — must match exactly |
| **PlatformIO**: `lib_deps` alone (`espidf`) | Whatever the fetched package contains | PlatformIO's own package name | No — bypasses IDF's native resolver entirely |

The root conflict lies in these non-negotiable rules:

- **Arduino** demands that all source code live under `src/` and automatically compiles everything it finds there without a manifest.

- **ESP-IDF** requires published components to be strictly self-contained within their manifest directory, so that meaning `CMakeLists.txt` cannot reference files outside via relative paths (like `../../src/`) without breaking once packaged and downloaded standalone.

- **Component Identity & Metadata** are handled independently by each toolchain. ESP-IDF's local filesystem resolution depends strictly on case-sensitive folder names, while package registry resolutions rely on immutable manifest metadata captured at release time.

The layout that satisfies all three simultaneously — used throughout this guide — places the IDF component manifest **at the repository root** (not nested under a `components/<name>/` folder pointing back out with `../..`), with `src/core/` as the single, self-contained source of truth. [Chapter 2 - Repository structure](02-repo-structure.md) covers why that specific placement, and what breaks if the component manifest lives anywhere else.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - ESP-IDF dual buildable examples](04-ESP-IDF-dual-buildable-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](06-bonus_yield-watchdog_naming-convention.md)
