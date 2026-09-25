# Chapter 1 — The Three Ecosystems and Their Non-Negotiable Constraints

Before discussing abstraction layers, repository layouts, or build integration, it's important to understand a fundamental reality: **the hardest part of a cross-ecosystem library is rarely the code itself**.

Each ecosystem comes with its own assumptions about project structure, source discovery, dependency management, examples, packaging, and publication. **These are not conventions that can be bent or negotiated**; they are hard requirements enforced by the tooling. Ignore them, and the build breaks, the library fails validation, or the package never gets published.

The challenge, therefore, is not simply making Arduino and ESP-IDF compile the same code. The challenge is designing a repository that simultaneously satisfies two independent ecosystems (three if considering also PlatformIO), each with its own rules and expectations.

Everything described in this chapter was verified empirically through real builds, real hardware testing, and real publication attempts. Several assumptions that seemed perfectly reasonable when reading the documentation turned out to be incorrect once exercised in practice.

## 1.1 Arduino

- **Source discovery is primarily structure.** The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every `.cpp` file found recursively under `src/`**. There is no manifest that explicitly lists compilation units: if a source file exists under `src/`, it gets compiled. Because the *Arduino Library Manager* expects libraries to follow this standardized layout, reorganizing sources outside of src/ is generally not a practical solution. For multi-framework libraries, this has an important consequence: any source file that must **not** be compiled for a specific target (for example, an ESP-IDF-specific HAL implementation) must exclude itself through preprocessor conditionals, because the build system provides no structural mechanism to skip it.

- **Library metadata in `library.properties`is taken from the tagged release, not the default branch.** During publication, the *Arduino Library Manager* indexes the content associated with the Git tag being released. Updating `library.properties` after the tag already exists has no effect on that release. Likewise, publishing before incrementing the version field in `library.properties` typically results in the release being ignored or treated as a duplicate. The version declared in `library.properties` and the *GitHub release tag* must refer to the same release version.

- **Examples follow a strict `Name/Name.ino` convention.** The Arduino IDE (and PlatformIO's example browser for `.ino` projects; see Section 1.3) expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`. The sketch name must match its parent directory. Additional levels of nesting can be used to organize examples into categories and do not affect discovery, provided this naming convention is preserved.

- **Dependency resolution is supported, but remains library-centric.** Arduino libraries can declare dependencies using the `depends` field in `library.properties`, enabling automatic installation of required libraries by the Library Manager.

## 1.2 ESP-IDF (native)

- **A published component must be self-contained.** The IDF Component Registry packages **only the folder containing `idf_component.yml`**. Any `CMakeLists.txt` that references source files *outside* that folder via relative paths (e.g. `../../src/core/foo.cpp`) work fine during local development, but breaks as soon as the component is published and downloaded standalone: those paths simply do not exist in the extracted package.

- **Header visibility is explicit.** ESP-IDF exposes only the include directories explicitly declared through `INCLUDE_DIRS` in `idf_component_register()` on `CMakeLists.txt`. A header is part of the component's public interface only if it is intentionally exported. This makes the API boundary explicit, but also means the component structure must be designed deliberately from the start.

- **Name resolution depends entirely on *which* mechanism declares the dependency — and each mechanism has a different, independent source of truth for the name.** This turned out to be the single most time-consuming constraint to pin down, precisely because it isn't documented as a unified rule anywhere — it only becomes visible by testing each mechanism in isolation:

- **Name resolution depends entirely on *which* mechanism declares the dependency, and each mechanism has its own source of truth.** This turned out to be one of the most time-consuming constraints to pin down, precisely because it is not documented as a unified rule anywhere and only becomes obvious when each mechanism is tested in isolation:
  
  - **`EXTRA_COMPONENT_DIRS` + `REQUIRES`**: purely filesystem-based. CMake scans the listed directories and registers each component **under the exact name of the folder that contains it** — case-sensitive and with no fallback. `REQUIRES foo` will fail to resolve a component sitting in a folder named `Foo`, even though the two strings differ only in case.

  - **`idf_component.yml`** (via the Component Manager, either through `override_path` during development or via the Registry): resolves the component's identity from the `name:` field **inside the manifest itself**. As with `EXTRA_COMPONENT_DIRS`, keeping the `name:` value identical to the component folder name avoids resolution inconsistencies across dependency mechanisms.

  - The practical rule of thumb is straightforward: **the component folder name and the component name declared in `idf_component.yml` should match.** For components intended to be published through the ESP-IDF Component Registry, keeping both names lowercase provides the most consistent behavior across local development, dependency resolution, and package distribution.
<!--
- **Standalone `REQUIRES` and a build tool's own dependency-injection mechanism do not compose automatically.** Declaring the same dependency both explicitly (`REQUIRES foo`) and through an external resolver that is not `idf_component.yml`-aware can lead to resolution conflicts rather than a predictable "last one wins" behavior. See 1.3 and Chapter 6 for the concrete case that exposed this limitation.
-->
- **Component metadata is taken from the published package version, not from the current repository state.** Once a version is published to the Component Registry, the corresponding `idf_component.yml` and associated metadata are immutable from the consumer's perspective. Updating the manifest on the default branch has no effect on already-published versions; a new version must be released for dependency or metadata changes to become visible.

- **Dependency management is component-based.** Components declare their dependencies through mechanisms such as `REQUIRES`, `PRIV_REQUIRES`, or `idf_component.yml`, allowing ESP-IDF to construct an explicit dependency graph at build time.

## 1.3 PlatformIO

PlatformIO occupies a unique position in this discussion because it is neither a framework nor a standalone build system in the traditional sense. Its behavior changes significantly depending on the selected `framework`, and many integration problems originate from assuming that it behaves the same way in all cases.

- **`framework = arduino`: PlatformIO largely follows Arduino's conventions.** Source discovery remains based on Arduino's recursive `src/` compilation model, and examples continue to follow Arduino's expectations. PlatformIO adds its own metadata layer through `library.json`, including the richer `examples` field, but this metadata is consumed by multiple internal PlatformIO surfaces whose behavior is similar, yet not entirely equivalent. Chapter 5 covers the resulting edge cases in detail.

- **`framework = espidf`**: here PlatformIO doesn't just wrap `idf.py` — it generates its **own** project scaffolding, driven by `platformio.ini` rather than IDF's native `CMakeLists.txt`/`project()` flow. The convention shifts from IDF's `main/` folder to PlatformIO's own `src_dir` (default `src/`). A project can still use a hand-written, IDF-native `CMakeLists.txt`/`main/` layout under PlatformIO, but only by explicitly pointing `src_dir` at that folder.

- **Package metadata is taken from the published package version, not from the current repository state.** Once a version is published to PlatformIO Registry, the corresponding manifest `library.json` and associated metadata are immutable from the consumer's perspective. Updating the repository's default branch has no effect on already-published versions; a new version must be released for changes to become visible. Consequently, early naming decisions are permanent, making it best practice to adopt the strictest convention — such as ESP-IDF's lowercase style — from day one.

- **Dependency management is package-based.** Dependencies are declared in `library.json` and resolved through the PlatformIO Registry, allowing PlatformIO to automatically fetch and install the required packages.

<!-- DA METTERE NEI PROGETTI / ESEMPI
DA CLAUDE:
- **`lib_deps` is a dependency-*fetching* mechanism, not a component-*registration* one.** Under `framework = espidf`, `lib_deps` resolves and downloads a library's source into `.pio/libdeps/`, but that alone does **not** make it visible to `REQUIRES` in `main/CMakeLists.txt` — those are two independent systems. Declaring the same dependency through both `lib_deps` *and* an explicit `REQUIRES` (with no `idf_component.yml` in between) causes a resolution failure, not redundancy: CMake's native resolver doesn't know how to find what PlatformIO fetched. Removing `REQUIRES` and leaving `lib_deps` as the sole source works cleanly — PlatformIO wires its own fetched libraries into the generated build automatically. Coexisting with `idf_component.yml` also works cleanly: the Component Manager resolves the dependency and the build proceeds through it, with `lib_deps` fetching the same source (redundant, but not conflicting) alongside.

DA COPILOT:
- **Dependency fetching and dependency registration are separate concepts.** This distinction is easy to miss because PlatformIO exposes both behind a seemingly unified experience.
  - **`lib_deps`** is responsible for locating, downloading, and updating libraries.
  - **`REQUIRES`** is an ESP-IDF build-system construct responsible for component registration and dependency declaration inside CMake.
  - The two mechanisms are independent and do not automatically satisfy each other's requirements.

- **`lib_deps` alone does not make a library visible to IDF's native component resolver.** Under `framework = espidf`, PlatformIO downloads dependencies into `.pio/libdeps/`, but a dependency fetched this way is not automatically discoverable by an explicit `REQUIRES foo` inside `main/CMakeLists.txt`. From ESP-IDF's perspective, fetching a dependency and registering a component are distinct operations.

- **`REQUIRES` and PlatformIO-managed dependencies only coexist cleanly when a valid `idf_component.yml` sits between them.** Declaring a dependency through both `lib_deps` and an explicit `REQUIRES` without a Component Manager-aware manifest typically produces a resolution failure rather than harmless redundancy: CMake attempts to resolve a component that PlatformIO merely downloaded, but never registered within IDF's native dependency graph.

- **The practical rule is simple:** under `framework = espidf`, choose a single authoritative dependency-resolution path whenever possible.
  - If the dependency is managed through PlatformIO, let `lib_deps` drive the process.
  - If the dependency is managed through ESP-IDF's Component Manager, let `idf_component.yml` drive the process.
  - Mixing multiple independent dependency-resolution systems without a clear source of truth usually introduces ambiguity rather than flexibility.
-->

## 1.4 Where the constraints collide

Comparing these three ecosystems reveals that the true challenge of a cross-ecosystem library is not managing different APIs, but harmonizing conflicting toolchain assumptions about source discovery, identity, and metadata immutability:

| | Source layout expected | Name resolved from | Case-sensitive? |
| :--- | :--- | :--- | :--- |
| **Arduino** (IDE / PlatformIO) | Everything under `src/`, compiled recursively | `name` field in `library.properties` / `library.json` | No practical constraint observed |
| **ESP-IDF**: `EXTRA_COMPONENT_DIRS` + `REQUIRES` | Component folder, referenced directly | **Local folder name** | **Yes** — must match exactly |
| **ESP-IDF**: `idf_component.yml` | Self-contained component folder | `name:` field inside the manifest | **Yes** — but decoupled from the local folder and from `library.json` |
| **PlatformIO**: `lib_deps` alone (`espidf`) | Whatever the fetched package contains | PlatformIO's own package name | No — bypasses IDF's native resolver entirely |

The layout that satisfies all three simultaneously — used throughout this guide — places the IDF component manifest **at the repository root** (not nested under a `components/<name>/` folder pointing back out with `../..`), with `src/core/` as the single, self-contained source of truth referenced by relative paths that never leave the repository. Chapter 2 covers why that specific placement, and what breaks if the component manifest lives anywhere else.
