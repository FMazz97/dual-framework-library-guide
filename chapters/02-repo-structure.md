# Chapter 2 — A Folder Structure That Satisfies Both at Once

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) examined each ecosystem in isolation and identified the rules that cannot be ignored. Taken individually, none of those constraints is particularly difficult to satisfy. The challenge begins when both ecosystems are expected to coexist within the same repository.

This chapter documents how those requirements were reconciled into a single folder structure that can be consumed simultaneously by [Arduino Library Manager](https://docs.arduino.cc/libraries/), [ESP Component Registry](https://components.espressif.com/), and also by [PlatformIO Registry](https://registry.platformio.org/) without maintaining separate branches or framework-specific distributions.

---

## 2.1 A repository layout that satisfies both ecosystems

```text
repo/                    ← same folder as idf_component.yml
├── idf_component.yml    ← ESP-IDF Manifest
├── CMakeLists.txt       ← SRCS "src/core/foo.cpp" (no ../ relative path. see Chapter 1)
├── library.properties   ← Arduino
├── library.json         ← PlatformIO
├── src/
│   ├── Foo.h            ← Arduino public include
│   ├── core/            ← shared implementation
│   │   ├── foo.h
│   │   └── foo.cpp
│   └── hal/             ← framework-specific HAL (see Chapter 3)
└── examples/
```

With `idf_component.yml` at the repository root, the ESP Component Registry packages the the directory that contains the manifest  minus `files:` exclusions declared in it (see: ESP-IDF [Manifest and Metadata](05-publishing.md#521-manifest-and-metadata) for more details). The shared implementation under `src/core/` becomes part of the published package itself, while Arduino continues to discover and compile sources from `src/` exactly as expected.

The absence of a traditional ESP-IDF `include/` directory is intentional. Since Arduino already requires public headers and implementation files to live within the `src/` tree, duplicating the same headers under a separate `include/` directory would introduce redundancy without providing any architectural benefit.

As a result, the same source tree works unchanged for:

- Arduino Library Manager.
- Local development via `EXTRA_COMPONENT_DIRS` and `idf_component.yml`.
- ESP Component Registry installation via `idf.py add-dependency`.
- PlatformIO Registry across both **Arduino** and **ESP-IDF** frameworks.

---

## 2.2 Local folder name vs. GitHub repository name: independent, and only the local one matters

A related trap, once the [case-sensitivity constraints for the ESP-IDF framework](01-ecosystem-constraints.md#component-identity-is-defined-differently-by-cmake-and-the-component-manager) were understood, is the instinct to rename the GitHub repository itself to match the *lowercase* component name, especially for libraries that originated in the Arduino ecosystem with a *PascalCase* repository name.

**This isn't necessary.**

The key observation is that `EXTRA_COMPONENT_DIRS` and the *Component Manager* (in local deployment) operate on **what exists locally on disk at build time**. Meanwhile, a component installed from the **Registry** is resolved from its registered package name. In neither case does the GitHub repository name play a role.

Using:

```bash
git clone https://github.com/owner/Foo.git foo
```

clones the exact same repository — same history, same tags, same remote URL — into a **locally chosen** folder named `foo`, independent of whatever the repository is called on GitHub.

From ESP-IDF's perspective the component-resolution mechanisms see only the local checkout, not the GitHub repository page. As long as the component folder and manifest names satisfy the expected conventions, the remote repository name is irrelevant.

Renaming the GitHub repository solves nothing that a documented `git clone <URL> <lowercase-name>` instruction doesn't already solve, while introducing real (if usually low) risk to existing registrations that may reference the current URL or name more strictly than expected.

The distinction becomes even more important once publication enters the picture: ***repository name, local folder name, and package name are three different things.*** The first two can usually be changed with limited consequence (a repository rename still carries the small risks above), **the third one often cannot.**

[Chapter 5 - Publishing](05-publishing.md) covers this in detail, including why package names on the *Arduino Library Manager*, *PlatformIO Registry*, and *ESP Component Registry* should be treated as effectively **hard or costly to chang** once published.

With the repository structure established, [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md) focuses on how that layout is combined with a code organization that keeps the application logic shared across frameworks while isolating framework-specific hardware access.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md)
- [Chapter 2 - Repository Structure](02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md)
- [Chapter 4 - Example Conventions](04-examples.md)
- [Chapter 5 - Publishing](05-publishing.md)
- [Appendix A - IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)
- [Appendix B - Yield & Watchdog](../appendix/B-yield-watchdog.md)
- [Appendix C - Naming Convention](../appendix/C-naming-convention.md)
