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

- **Header visibility is explicit.** Unlike Arduino's relatively permissive library model, ESP-IDF exposes only the include directories explicitly declared through `INCLUDE_DIRS` in `idf_component_register()`. A header is part of the component's public interface only if it is intentionally exported. This makes the API boundary explicit, but also means the component structure must be designed deliberately from the start.

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

PlatformIO plays two structurally different roles depending on `framework`, and conflating them is the source of most confusion when a library needs to support both.

- **`framework = arduino`**: PlatformIO is a thin wrapper around the Arduino build convention described in 1.1 — same `src/` recursive compilation, same example discovery patterns (plus its own richer `examples` field in `library.json`, which is read by two *different* internal surfaces with two *different* — and not fully equivalent — behaviors, covered in Chapter 5).
- **`framework = espidf`**: here PlatformIO doesn't just wrap `idf.py` — it generates its **own** project scaffolding, driven by `platformio.ini` rather than IDF's native `CMakeLists.txt`/`project()` flow. The convention shifts from IDF's `main/` folder to PlatformIO's own `src_dir` (default `src/`). A project can still use a hand-written, IDF-native `CMakeLists.txt`/`main/` layout under PlatformIO, but only by explicitly pointing `src_dir` at that folder — otherwise PlatformIO's own generated build takes over and ignores it.
- **`lib_deps` is a dependency-*fetching* mechanism, not a component-*registration* one.** Under `framework = espidf`, `lib_deps` resolves and downloads a library's source into `.pio/libdeps/`, but that alone does **not** make it visible to `REQUIRES` in `main/CMakeLists.txt` — those are two independent systems. Declaring the same dependency through both `lib_deps` *and* an explicit `REQUIRES` (with no `idf_component.yml` in between) causes a resolution failure, not redundancy: CMake's native resolver doesn't know how to find what PlatformIO fetched. Removing `REQUIRES` and leaving `lib_deps` as the sole source works cleanly — PlatformIO wires its own fetched libraries into the generated build automatically. Coexisting with `idf_component.yml` also works cleanly: the Component Manager resolves the dependency and the build proceeds through it, with `lib_deps` fetching the same source (redundant, but not conflicting) alongside.

## 1.4 Where the constraints collide

| | Source layout expected | Name resolved from | Case-sensitive? |
| :--- | :--- | :--- | :--- |
| **Arduino** (IDE / PlatformIO) | Everything under `src/`, compiled recursively | `library.properties` / `library.json` `name` | No practical constraint observed |
| **ESP-IDF**: `EXTRA_COMPONENT_DIRS` + `REQUIRES` | Component folder, referenced directly | **Local folder name** | **Yes** — must match exactly |
| **ESP-IDF**: `idf_component.yml` | Self-contained component folder | `name:` field inside the manifest | **Yes** — but decoupled from the local folder and from `library.json` |
| **PlatformIO**: `lib_deps` alone (`espidf`) | Whatever the fetched package contains | PlatformIO's own package name | No — bypasses IDF's native resolver entirely |

The layout that satisfies all three simultaneously — used throughout this guide — places the IDF component manifest **at the repository root** (not nested under a `components/<name>/` folder pointing back out with `../..`), with `src/core/` as the single, self-contained source of truth referenced by relative paths that never leave the repository. Chapter 2 covers why that specific placement, and what breaks if the component manifest lives anywhere else.
