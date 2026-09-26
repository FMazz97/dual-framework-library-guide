# Chapter 2 — A Folder Structure That Satisfies All Three at Once

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) examined each ecosystem in isolation and identified the rules that cannot be ignored. Taken individually, none of those constraints is particularly difficult to satisfy. The challenge begins when all three ecosystems are expected to coexist within the same repository.

This chapter documents how those requirements were reconciled into a single folder structure that can be consumed simultaneously by Arduino Library Manager, PlatformIO Registry, and the ESP Component Registry without maintaining separate branches or framework-specific distributions.

It also covers an architectural dead end that initially appeared to satisfy every requirement on paper, yet failed when subjected to real publication and validation workflows. That failure is worth documenting precisely because it is difficult to predict from the official documentation alone and only becomes apparent when the repository is exercised across all three ecosystems.

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

## 2.2 The solution: IDF manifest at the repository root, pointing *toward* `src/core`

The resolution is to stop treating the component folder as a subfolder that must physically contain its source, and instead let **the whole repository be the self-contained unit** the Registry packages:

```text
repo/ ← same folder as idf_component.yml
├── idf_component.yml
├── CMakeLists.txt ← SRCS "src/core/foo.cpp" (no ../..)
├── library.properties ← Arduino
├── library.json ← PlatformIO
├── src/
│   ├── Foo.h ← Arduino public include
│   └── core/ ← shared implementation
│       ├── foo.h
│       └── foo.cpp
└── examples/
```

With `idf_component.yml` at the root, publishing packages the **entire repository** — `src/core/foo.cpp` is now *inside* the packaged unit, referenced with a plain relative path (`src/core/foo.cpp`, no `..`) that resolves identically whether you're building from the live repository or from a downloaded, extracted package. Arduino continues to see exactly what it always expected — everything it needs under `src/` — untouched by any of this.

The rule this generalizes to: **a component's `CMakeLists.txt` may reference paths *down and into* its own package root, never *up and out* of it.** Placing the manifest at the root, rather than in a nested subfolder, is what makes "down and into" cover the entire shared core.

## 2.3 Local folder name vs. GitHub repository name: independent, and only the local one matters

A related trap, easy to fall into once Chapter 1.2's case-sensitivity constraint is understood: the instinct to **rename the GitHub repository itself** to match the lowercase component name.

This isn't necessary, and it's worth understanding precisely why. `EXTRA_COMPONENT_DIRS` and the Component Manager both resolve names from **what's physically on the local disk at build time** — the repository's name on GitHub has no bearing on this at all. When someone runs:

```bash
git clone https://github.com/owner/Foo.git foo
```

Git clones the exact same repository — same history, same tags, same remote URL — into a **locally chosen** folder name, independent of whatever the repository is called on GitHub. Every registry registration tied to the repository's URL (Arduino Library Manager, PlatformIO Registry entries referencing the Git remote) remains valid regardless of what any individual clone is named on someone's disk.

The only thing that matters for the constraints in Chapter 1.2 is the folder name **at the moment `EXTRA_COMPONENT_DIRS` or a manual checkout is used** — which is a per-user, per-clone choice, not a property of the repository itself. Renaming the GitHub repository solves nothing that a documented `git clone ... <lowercase-name>` instruction doesn't already solve, while introducing real (if usually low) risk to existing registrations that may reference the current URL or name more strictly than expected.

Chapter 6 covers the one place this distinction *does* have a hard consequence: package names on the Registries themselves (as opposed to folder/repository names) are not free to change after first publication without losing accumulated download history — a different, and non-reversible, kind of "name" entirely.
