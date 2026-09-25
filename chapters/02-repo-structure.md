# Chapter 2 — A Folder Structure That Satisfies All Three at Once

Chapter 1 established the constraints in isolation. This chapter walks through how they were reconciled into a single repository layout — including a dead end that looked correct on paper and only failed once actually published, which is worth documenting precisely because it isn't obvious from reading the tooling docs alone.

## 2.1 Why `components/<name>/` with a `CMakeLists.txt` pointing outward (`../..`) breaks publishing

A natural first instinct, once Arduino requires `src/core/` as the single source of truth (Chapter 1.1), is to keep it there and have the IDF component "reach into" it:

```text
repo/
    ├── src/
    │   └── core/
    │       ├── foo.cpp
    │       └── foo.h
    └── components/
        └── foo/
            ├── idf_component.yml
            └── CMakeLists.txt ← SRCS "../../src/core/foo.cpp"
```

This compiles and works perfectly for **local development** — `idf.py`, pointed at the repository via `EXTRA_COMPONENT_DIRS`, resolves the relative path just fine, because the whole repository is present on disk.

It breaks the moment the component is **published**. The Registry packages only `components/foo/` — the folder containing `idf_component.yml` — and nothing outside it. Someone who downloads the published package (via `idf.py add-dependency`, or PlatformIO's own resolution of the same manifest) gets a self-contained archive of just that folder. The `../../src/core/foo.cpp` reference in its `CMakeLists.txt` now points to a path that doesn't exist anywhere in what they downloaded — the build fails, not with a helpful "source outside package" error, but with whatever generic "file not found" CMake produces for a missing source.

This is easy to miss because **the failure mode never appears during local development** — `EXTRA_COMPONENT_DIRS` always sees the whole repository, so nothing ever signals that the component isn't actually self-contained until the exact moment it's packaged and consumed by someone else.

The mirror-image alternative — moving the real source *into* `components/foo/` and having `src/CS123x.h` (the Arduino facade) reach *into* it instead — fails for the opposite reason: Arduino/PlatformIO's Arduino framework only compiles `.cpp` files found under `src/` (Chapter 1.1). A `.cpp` file living under `components/` is invisible to that build entirely, regardless of what any header includes.

Neither direction works, because each side of the asymmetry has an absolute requirement about *where its own source lives*, not just about what it can reference.

## 2.2 The solution: IDF manifest at the repository root, pointing *toward* `src/core`

The resolution is to stop treating the component folder as a subfolder that must physically contain its source, and instead let **the whole repository be the self-contained unit** the Registry packages:

```text
repo/ ← same folder as idf_component.yml
    ├── idf_component.yml
    ├── CMakeLists.txt ← SRCS "src/core/foo.cpp" (no ../..)
    ├── library.properties ← Arduino
    ├── library.json ← PlatformIO
    ├── src/
    │   ├── Foo.h ← Arduino facade (PascalCase)
    │   ├── core/
    │   │   ├── foo.h ← framework-agnostic core (snake_case)
    │   │   └── foo.cpp
    │   └── hal/
    │       ├── foo_hal_arduino.h
    │       └── foo_hal_espidf.h
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
