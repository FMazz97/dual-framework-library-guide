# Chapter 2 — A Folder Structure That Satisfies All Three at Once

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) examined each ecosystem in isolation and identified the rules that cannot be ignored. Taken individually, none of those constraints is particularly difficult to satisfy. The challenge begins when all three ecosystems are expected to coexist within the same repository.

This chapter documents how those requirements were reconciled into a single folder structure that can be consumed simultaneously by Arduino Library Manager, PlatformIO Registry, and the ESP Component Registry without maintaining separate branches or framework-specific distributions.

It also covers an architectural dead end that initially appeared to satisfy every requirement on paper, yet failed when subjected to real publication and validation workflows. That failure is worth documenting precisely because it is difficult to predict from the official documentation alone and only becomes apparent when the repository is exercised across all three ecosystems.

---

## 2.1 Why `components/<name>/` with a `CMakeLists.txt` pointing outward (`../..`) breaks publishing

A natural first instinct, especially for libraries that originated in the Arduino ecosystem with a *PascalCase* repository name, is to leave all implementation files in Arduino's `src/` tree once it becomes the single source of truth ([Chapter 1.1](01-ecosystem-constraints.md#11-arduino)), and simply have the ESP-IDF component "reach into" it:

```text
repo/
├── src/
│   ├── Foo.h ← Arduino public include
│   └── core/ ← shared implementation
│       ├── foo.h
│       └── foo.cpp
└── components/
    └── foo/
        ├── idf_component.yml
        └── CMakeLists.txt ← SRCS "../../src/core/foo.cpp"
```

At first glance, the approach appears elegant. There is only one copy of the source code, Arduino remains happy because everything lives under `src/`, and ESP-IDF can successfully build the component through a relative path.

In fact, it works perfectly during *local development*.

When the repository is added through `EXTRA_COMPONENT_DIRS`, the entire project is available on disk. The relative path resolves correctly, idf.py compiles without complaint, and nothing suggests that the architecture is flawed.

The problem only appears when the component is **published**.

The ESP Component Registry packages only the directory that contains `idf_component.yml`, making the component self-contained from the consumer's perspective. Everything outside that directory is excluded from the published artifact. As a result, a user installing the component through `idf.py add-dependency` (or through PlatformIO's resolution of the same manifest) receives only the contents of `components/foo/`.

At that point, the previously valid reference in `CMakeLists.txt`:

```text
../../src/core/foo.cpp
```

now points to a path that **doesn't exist anywhere in what was downloaded**.

The build fails, typically with an generic "file not found" error from CMake rather than a clear indication that the component is attempting to reference files outside the published package.

This is easy to miss because **the failure mode never appears during local development**. `EXTRA_COMPONENT_DIRS` always sees the whole repository, so nothing ever signals that the component isn't actually self-contained until the exact moment it's packaged and consumed by someone else.

The obvious mirror-image solution seems to be moving the real source code *into* `components/foo/` and exposing it to Arduino through headers located under `src/`.

**That fails for the opposite reason.**

As established in [Chapter 1.1](01-ecosystem-constraints.md#1.1-Arduino), Arduino's build system discovers source files structurally and compiles only implementation files located under src/. A .cpp file living exclusively inside components/ is simply invisible to the Arduino build, regardless of what headers are included.

In other words, **neither direction works**.

Keeping the implementation under `src/` violates the ESP-IDF packaging model once the component is published. Keeping the implementation under `components/` violates Arduino's source-discovery model before the build even begins.

Neither ecosystem is satisfied by simply referencing the other's sources.

---

## 2.2 The solution: IDF manifest at the repository root, pointing *toward* `src/core`

The resolution is to stop treating the ESP-IDF component as a nested subdirectory with its own isolated source tree, and instead make the entire **repository the self-contained unit** consumed by all three ecosystems.

```text
repo/                    ← same folder as idf_component.yml
├── idf_component.yml    ← ESP-IDF Manifest
├── CMakeLists.txt       ← SRCS "src/core/foo.cpp" (no ../..)
├── library.properties   ← Arduino
├── library.json         ← PlatformIO
├── src/
│   ├── Foo.h            ← Arduino public include
│   └── core/            ← shared implementation
│       ├── foo.h
│       └── foo.cpp
└── examples/
```

At first glance, this may look like a small adjustment. In practice, it changes the packaging boundary completely.

With `idf_component.yml` at the repository root, the ESP Component Registry packages the entire **repository rather** than a nested subdirectory. The shared implementation under `src/core/` is now part of the published package itself, not something referenced from outside it.

As a result, the component's `CMakeLists.txt` can reference:

```text
src/core/foo.cpp
```

using an ordinary relative path entirely contained within the packaged artifact.

The same path resolves correctly in every scenario:

- local development through `EXTRA_COMPONENT_DIRS`;
- installation through `idf.py` add-dependency;
- handled through **PlatformIO's Component Manager** integration.

Arduino remains equally satisfied. Its source-discovery model still finds everything under src/, exactly where it expects it to be, and the presence of ESP-IDF-specific metadata at the repository root does not interfere with Arduino Library Manager or PlatformIO's Arduino framework.

So, the rule this generalizes to is simple:

- **A component's `CMakeLists.txt` may reference paths *down and into* its own package root, but never up and out of it.**
- **Likewise, Arduino expects the library's source tree to be *rooted at the repository level*, with all compilable sources living under `src/`.**

Placing `idf_component.yml` at the repository root is what allows both requirements to be satisfied simultaneously.

---

## 2.3 Local folder name vs. GitHub repository name: independent, and only the local one matters

A related trap, once the [case-sensitivity constraints for ESP-IDF framework](01-ecosystem-constraints.md#12-esp-idf-native) was understood, is the instinct to rename the GitHub repository itself to match the *lowercase* component name, especially for libraries that originated in the Arduino ecosystem with a *PascalCase* repository name.

**This isn't necessary.**

The key observation is that `EXTRA_COMPONENT_DIRS` and the Component Manager operate on **what exists locally on disk at build time**. They do not care what the repository is called on GitHub.

Using:

```bash
git clone https://github.com/owner/Foo.git foo
```

clones the exact same repository into a local folder named `foo` — same history, same tags, same remote URL — into a **locally chosen** folder name, independent of whatever the repository is called on GitHub.

From ESP-IDF's perspective the component-resolution mechanisms see only the local checkout, not the GitHub repository page. As long as the component folder and manifest names satisfy the expected conventions, the remote repository name is irrelevant.

Renaming the GitHub repository solves nothing that a documented `git clone ... <lowercase-name>` instruction doesn't already solve, while introducing real (if usually low) risk to existing registrations that may reference the current URL or name more strictly than expected.

The distinction becomes even more important once publication enters the picture: ***repository name, local folder name, and package name are three different things.*** The first two can usually be changed with little consequence; **the third often cannot.**

[Chapter 6 - Publishing](06-publishing.md) covers this in detail, including why package names on the *Arduino Library Manager*, *PlatformIO Registry*, and *ESP Component Registry* should be treated as effectively **permanent** once published.

With the repository structure established, [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md) focuses on how that layout is combined with a code organization that keeps the application logic shared across frameworks while isolating framework-specific hardware access.

---

## Index

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core_hal-architecture.md)
- [Chapter 4 - Yield & Watchdog](04-yield-watchdog.md)
- [Chapter 5 - ESP-IDF dual buildable examples](05-dual-buildable-examples)
- [Chapter 6 - Publishing](06-publishing.md)
