# Chapter 1 — The Three Ecosystems and Their Non-Negotiable Constraints

Before discussing abstraction layers, repository layouts, or build integration, it's important to understand a fundamental reality: **the hardest part of a cross-ecosystem library is rarely the code itself**.

Each ecosystem comes with its own assumptions about project structure, source discovery, dependency management, examples, packaging, and publication. **These are not conventions that can be bent or negotiated**; they are hard requirements enforced by the tooling. Ignore them, and the build breaks, the library fails validation, or the package never gets published.

The challenge, therefore, is not simply making Arduino and ESP-IDF compile the same code. The challenge is designing a repository that simultaneously satisfies two independent ecosystems (three if considering also PlatformIO), each with its own rules and expectations.

Everything described in this chapter was verified empirically through real builds, real hardware testing, and real publication attempts. Several assumptions that seemed perfectly reasonable when reading the documentation turned out to be incorrect once exercised in practice.

## 1.1 Arduino

- **Source discovery is primarily structure.** The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every `.cpp` file found recursively under `src/`**. There is no manifest that explicitly lists compilation units: if a source file exists under `src/`, it gets compiled. Because the *Arduino Library Manager* expects libraries to follow this standardized layout, reorganizing sources outside of src/ is generally not a practical solution. For multi-framework libraries, this has an important consequence: any source file that must **not** be compiled for a specific target (for example, an ESP-IDF-specific HAL implementation) must exclude itself through preprocessor conditionals, because the build system provides no structural mechanism to skip it.

- **Examples follow a strict `Name/Name.ino` convention.** The Arduino IDE (and PlatformIO's example browser for `.ino` projects; see Section 1.3) expects examples to be organized as `examples/<ExampleName>/<ExampleName>.ino`. The sketch name must match its parent directory. Additional levels of nesting can be used to organize examples into categories and do not affect discovery, provided this naming convention is preserved.

- **Library metadata in `library.properties`is taken from the tagged release, not the default branch.** During publication, the *Arduino Library Manager* indexes the content associated with the Git tag being released. Updating `library.properties` after the tag already exists has no effect on that release. Likewise, publishing before incrementing the version field in `library.properties` typically results in the release being ignored or treated as a duplicate. The version declared in `library.properties` and the *GitHub release tag* must refer to the same release version.

- **Dependency resolution is supported, but remains library-centric.** Arduino libraries can declare dependencies using the `depends` field in `library.properties`, enabling automatic installation of required libraries by the Library Manager.

## 1.2 ESP-IDF (native)

- **A component must be self-contained.** The IDF Component Registry packages **only the folder containing `idf_component.yml`** — any `CMakeLists.txt` that references source files *outside* that folder via a relative path (e.g. `../../src/core/foo.cpp`) works fine for local development, but breaks the moment the component is published and downloaded standalone: those paths simply don't exist in the extracted package on someone else's machine.
- **Name resolution depends entirely on *which* mechanism declares the dependency — and each mechanism has a different, independent source of truth for the name.** This turned out to be the single most time-consuming constraint to pin down, precisely because it isn't documented as a unified rule anywhere — it only becomes visible by testing each mechanism in isolation:
  - **`EXTRA_COMPONENT_DIRS` + `REQUIRES`**: purely filesystem-based. CMake scans the listed directories and registers each component **under the exact name of the folder that contains it** — case-sensitive, with no fallback. `REQUIRES cs123x` will fail to resolve a component sitting in a folder named `CS123x`, even though the two strings differ only in case. `library.json`/`library.properties` have no influence here whatsoever — this mechanism doesn't even know they exist.
  - **`idf_component.yml`** (via the Component Manager, either with `override_path` for local development or resolved from the Registry): resolves the name from the `name:` field **inside the manifest itself** — completely independent of the local folder's name, and independent of any Arduino-facing manifest. This makes it the only one of the three IDF-side mechanisms that doesn't require the local checkout folder to match a specific case.
  - The practical rule of thumb this leads to: **the case-sensitivity constraint never disappears — it just moves to wherever the mechanism you're using keeps its name.** Using the same lowercase name everywhere (folder, `idf_component.yml`, published package) is the only choice that satisfies every mechanism at once, not because case stops mattering, but because it happens to match all their independent sources of truth simultaneously.
- **`REQUIRES` and a build tool's own dependency-injection mechanism don't compose.** Declaring a dependency both explicitly (`REQUIRES foo`) and through an external resolver that isn't `idf_component.yml`-aware results in a resolution conflict, not a "last one wins" — see 1.3 and Chapter 6 for the concrete case this produced.

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
