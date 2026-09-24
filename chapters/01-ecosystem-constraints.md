# Chapter 1 — The Three Ecosystems and Their Non-Negotiable Constraints

Before any code gets shared between frameworks, it's worth mapping out exactly what each build system *requires* — not as a matter of convention, but as a hard constraint that, if violated, breaks the build. Everything in this chapter was verified empirically, on real hardware and real publish attempts — several assumptions that looked reasonable on paper turned out to be wrong once tested.

## 1.1 Arduino

- **Source discovery is purely structural.** The Arduino build system (and PlatformIO's Arduino framework support, which follows the same convention) compiles **every `.cpp` file found recursively under `src/`**, with no manifest listing them individually. If a file exists under `src/`, it gets compiled — full stop. This has a direct consequence for multi-file architectures: any file that must **not** be compiled for a given target (e.g. a native ESP-IDF HAL implementation) needs to guard itself with preprocessor conditionals, because there's no way to exclude it structurally within `src/`.
- **Examples are Name/Name.ino, one level deep.** The IDE (and PlatformIO's example browser, in the case of `.ino` files — see 1.3) expects `examples/<ExampleName>/<ExampleName>.ino`: the file name must match its containing folder. Anything nested deeper, or with a mismatched name, either doesn't get recognized as a sketch or shows up in unexpected places in the IDE's Examples menu.
- **`library.properties`/`library.json` are read only from the tagged release**, not from the latest commit on the default branch. The Arduino Library Manager indexer snapshots the manifest at the Git tag you publish — moving a tag after the fact, or publishing before bumping the version fields, produces a mismatch that the indexer either silently ignores or flags as a duplicate/no-op version.
- **No dependency resolution beyond name + version.** Arduino's library model has no concept of "component," no build-system-level linking between libraries — a `#include` either finds the header somewhere in the flattened include path, or it doesn't.

## 1.2 ESP-IDF (native)

- **A component must be self-contained.** The IDF Component Registry packages **only the folder containing `idf_component.yml`** — any `CMakeLists.txt` that references source files *outside* that folder via a relative path (e.g. `../../src/core/foo.cpp`) works fine for local development, but breaks the moment the component is published and downloaded standalone: those paths simply don't exist in the extracted package on someone else's machine.
- **Name resolution depends entirely on *which* mechanism decl