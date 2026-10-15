# Chapter 5 — Publishing Across Ecosystems and Package Registry Mechanics

Publishing a framework-agnostic C++ library requires interacting with three independent package registries: the [Arduino Library Manager](https://docs.arduino.cc/libraries/), the [ESP Component Registry](https://components.espressif.com/), and the [PlatformIO Registry](https://registry.platformio.org/). Each ecosystem uses its own manifest format, indexing pipeline, and rule set for package immutability and naming.

Rather than treating publishing as a single monolithic step, this chapter breaks down the publication mechanics for each registry individually. It details how each ecosystem's manifest enforces metadata and file filtering (as introcuced in [Chapter 4 - Example Conventions](04-examples.md)), followed by local pre-validation, release workflows, and lifecycle maintenance.

---

## 5.1 Arduino Library Manager

The Arduino Library Manager uses an automated, tag-driven indexing model. Rather than hosting binary archives or accepting direct uploads via CLI, the registry periodically scans registered Git repositories.

### 5.1.1 Manifest and Metadata

The single source of truth for Arduino library metadata is the `library.properties` file located at the repository root. The indexer reads the manifest from the commit attached to the **tag**, not from the default branch. Updating `library.properties` on `main` after creating a release tag will not update the published release.

- **Version Indexing:** Arduino's indexer relies strictly on Git tags (e.g., `v1.0.0` or `1.0.0`). The `version` field in `library.properties` **must match the Git tag name exactly**.

- **Dependency Declaration:** Dependencies resolution is supported, but remains library-centric. Refer to other Arduino libraries using the `depends` field in `library.properties` (see `depends` point in Arduino [library format](https://docs.arduino.cc/arduino-cli/library-specification/#library-metadata) section).

- **File Management:** Arduino Library Manager does not provide manifest-level file filtering. The published package always reflects the contents of the tagged release, making repository structure the primary mechanism for controlling what is distributed.

For more details, see [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/) official documentation.

### 5.1.2 Local Pre-Verification

Before submitting a library to the Arduino Library Manager, validate the layout and metadata locally using:

```Bash
arduino-lint --library-manager submit
```

This command runs the exact same automated checks that the Arduino indexing bot performs, catching missing fields, incorrect directory structures, or invalid dependency declarations before initiating a review.

For more details, see [Arduino Lint](https://arduino.github.io/arduino-lint/latest/) official documentation.

### 5.1.3 Initial Publication

Initial inclusion requires a Pull Request against [arduino/library-registry](https://github.com/arduino/library-registry) GitHub repository, adding the library repository URL to the registry list. The submission is then validated automatically through the same checks performed by `arduino-lint`.

Before submitting, **at least one GitHub Release must already exist, and the release version must match the `version` field declared in `library.properties`.**

### 5.1.4 Updates

After acceptance, updates are fully automatic. To publish a new release:

- Bump the `version` field in `library.properties`.
- Create and publish a matching Git tag (and corresponding GitHub Release).

The Arduino indexer detects the new release and updates the library. No additional Pull Requests are required.

---

## 5.2 ESP Component Registry

The ESP Component Registry is Espressif's official package manager for ESP-IDF components. It operates under a self-service, push-based model.

### 5.2.1 Manifest and Metadata

The single source of truth for ESP-IDF component metadata is `idf_component.yml`. The registry always indexes the metadata associated with the published version, not the current state of the repository.

- **Version Indexing:** Versioning is driven explicitly by the `version:` field inside `idf_component.yml` following Semantic Versioning (SemVer).

- **Dependency Declaration:** External component dependencies are listed in the `dependencies:` section:

  ```yaml
  dependencies:
    <autor>/<component>: "^<version>"
  ```

- **File Management & Example Inclusion:** To ensure the published component package remains clean of foreign toolchain files and explicitly selects which examples to package, combine `examples` with the `files.exclude` key in `idf_component.yml`:

  ```yaml
  examples:
    - path: ./examples/basic_example
  
  files:
    exclude:
      - "library.properties"          # Arduino's manifest
      - "library.json"                # PlatformIO's manifest
      - "src/Foo.h"                   # Arduino's facade
      - "src/hal/foo_hal_arduino.h"   # Arduino's hal 
      - "examples/*/*.ino"            # Arduino's examples
      - "examples/*/platformio.ini"   # ESP-IDF PlatformIO's examples configuration
      - "assets/**"                   # other documentation
      # add aslo other element that doesn't matter for ESP-IDF component
  ```

For more details, see IDF Component Manager [`idf_component.yml` Manifest File](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/manifest_file.html) official documentation.

### 5.2.2 Local Pre-Verification

Before uploading to the Espressif Component Registry, validate the manifest and simulate the archive creation using `compote` on Espressif's Component Manager CLI:

```bash
compote component upload --name <component_name> --dry-run
```

This command parses `idf_component.yml`, checks SemVer compliance, verifies exclusion rules, and confirms that the archive package builds correctly without actually uploading it to the server.

### 5.2.3 Initial Publication

Publishing is performed directly via the CLI:

```bash
compote component upload --name <component_name> --namespace <your_user_name>
```

The package becomes live immediately upon command completion.

> Note: The upload command requires authentication through an ESP Component Registry API token. Ensure the `IDF_COMPONENT_API_TOKEN` environment variable is set (`$env:IDF_COMPONENT_API_TOKEN="<token>"`) before running `compote component upload`.

Then, the component can be published with:

For more details, see IDF Component Manager [Compote CLI](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/compote_cli.html) official documentation.

### 5.2.4 Updates

To publish bug fixes or changes, bump `version:` in `idf_component.yml` and upload the component again:

```bash
compote component upload --name <component_name> --namespace <your_user_name>
```

Published versions (`<namespace>/<name>^<version>`) on the ESP Component Registry are permanent and immutable. Once a version is uploaded, Espressif's backend prevents re-uploading or modifying that exact version string.

> Note: Updates use the same authentication mechanism as the initial publication. Ensure a valid ESP Component Registry access token is configured before publishing a new version.

---

## 5.3 PlatformIO Registry

The PlatformIO Registry is a self-service, CLI-driven package registry supporting Arduino, ESP-IDF, and other embedded frameworks.

### 5.3.1 Manifest and Metadata

PlatformIO package metadata is declared in `library.json` at the root of the project. The registry always indexes the metadata associated with the published version, not the current state of the repository. Remember to add both `"arduino"` and `"espidf"` framework on supported `"framework":` field.

- **Version Indexing:** Versioning is controlled by the `"version":` string inside `library.json`.

- **Dependency Declaration:** Package dependencies are declared inside the `"dependencies"` object:

  ```json
  "dependencies": {
    "<owner>/<name>": "^<version>"
  }
  ```

- **File Management:** To omit ESP-IDF component manifests or `asset/` folder from the PlatformIO package archive, use the `"export": { "exclude": [...] }` block in `library.json`:

- **File Management & Example Inclusion:** To omit ESP-IDF component manifests or `asset/` folder and explicitly declare which example structures and files will package for PlatformIO registry, use the `"export"` exclusion block alongside the `"examples"` configuration in `library.json`:

  ```json
  {
  "examples": [
      {
        "name": "Arduino - BasicExample",
        "base": "examples/BasicExample",
        "files": [
          "BasicExample",
          "platformio.ini"
        ]
      },
      {
        "name": "ESP-IDF - basic_example",
        "base": "examples/basic_example",
        "files": [
          "main/main.cpp",
          "main/CMakeLists.txt",
          "CMakeLists.txt",
          "platformio.ini",
          "README.md"
        ]
      }
    ]
  "export": {
      "exclude": [
        "assets",
        "library.properties"
        "idf_component.yml",
        "examples/*/main/idf_component.yml"
      ]
    }
  }
  ```

> ⚠️ **Note on PIO Home Example Rendering:**  
> When declaring structured multi-file examples in `library.json` (using `name`, `base`, and `files`), you may notice that the **PIO Home GUI panel** in VS Code does not group files correctly. Instead of showing a single project entry labeled with `example.name`, PIO Home currently flattens the array, rendering each source file as an individual entry (e.g., separate items named `main`, `CMakeLists`, etc.) due to a known rendering limitation in the desktop interface (reported in [PlatformIO Issue #12672](https://github.com/platformio/platformio-home/issues/12672)). Rest assured that the metadata and structure are correct: the examples render and index properly on the online PlatformIO Registry website.

For more details, see PlatformIO [library.json](https://docs.platformio.org/en/latest/manifests/library-json/index.html) official documentation.

### 5.3.2 Local Pre-Verification

To verify which files will be included in the published package, generate a local package archive using the PlatformIO Core CLI:

```bash
pio pkg pack
```

This creates a `.tar.gz` archive in the project root. Inspecting its contents allows you to confirm that `export.exclude` rules functioned properly and that no unwanted build files or manifests were included.

### 5.3.3 Initial Publication

To publish the package to the PlatformIO Registry, execute:

```bash
pio pkg publish
```

The package becomes live immediately upon command completion.

> Note: The publish command requires an authenticated PlatformIO account. Ensure you are logged in locally before running `pio pkg publish`.

For more details, see PlatformIO [Package Management CLI](https://docs.platformio.org/en/latest/core/userguide/pkg/index.html) official documentation.

### 5.3.4 Updates

To publish bug fixes or changes, bump `"version":` in `library.json` and upload the package again:

```bash
pio pkg publish
```

Published versions (`<owner>/<name> @ ^<version>`) on PlatformIO Registry are permanent and immutable. Once a version is uploaded, PlatformIO core backend prevents re-uploading or modifying that exact version string.

> Note: Updates use the same authentication mechanism as the initial publication. Ensure your PlatformIO account is authenticated before publishing a new version.

---

## 5.4 The case-sensitivity trap: PlatformIO's immutable package binding

While Arduino and ESP-IDF handle their respective registries with straightforward rules, **PlatformIO** acts as a multi-framework bridge, making it the primary place where casing inconsistencies can trigger hidden registry traps.

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md) established that ESP-IDF's native component-resolution mechanisms are case-sensitive. What's easy to miss is that PlatformIO's registry registry backend enforces a strict immutability binding on package names: **the name and casing recorded at the moment of your very first publish become a permanent backend identity (`pkg_id`).**

This friction typically happens when a cross-framework library is published first to match Arduino conventions (e.g., using uppercase characters like `Foo`). Later, when native ESP-IDF support is added, you naturally want to align `library.json`'s `"name"` field to lowercase (`foo`) to match ESP-IDF component naming rules also if it is not needed becauses PlatformIO depandency resolution resolve ESP-IDF package case-insensitive.

When you update `library.json` and attempt to publish the new version, PlatformIO's registry rejects it with a database mismatch error:

```text
Manifest name='foo' <> DB name='Foo' and pkg_id is not passed
```

Because the registry backend permanently bound the original case to the package's internal ID, and the PlatformIO CLI exposes no flag or self-service mechanism to override it (`pio pkg publish --help` has no package-ID option), you are left with two difficult paths:

- **Delete and republish under the new name (not tested).** Maybe possible, but probably destructive to download metrics. (Fortunately, PlatformIO's registry handles lookups case-insensitively, meaning downstream `platformio.ini`/`lib_deps` referencing the old casing will still resolve successfully).

- **Ask the registry's support team to rename the entry server-side.** PlatformIO's support team can rename the existing package in place, preserving its history and download metrics, in response to a direct support request (as experienced firsthand during the publishing lifecycle of the [CS123x library](https://github.com/FMazz97/CS123x)).

---

## 5.5 The Generalizable Naming Rule

**Decide the published package's name — in every registry it will live in — before the first publish.**

To avoid cross-toolchain friction and registry traps, establish a coherent naming strategy across all environments before your first public release:

### 1. Local Development Directory & ESP-IDF Component

Keep the local component folder and the name in `idf_component.yml` strictly lowercase (e.g., `foo`). This adheres to ESP-IDF's filesystem constraints, `EXTRA_COMPONENT_DIRS` behavior, and ESP Component Registry requirements established in [Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md).

### 2. Online Repository & Arduino Library

Name the GitHub repository and `library.properties` using **pascalCase** or **CamelCase** (e.g., `Foo`). This aligns with the Arduino community standard where library names match their primary C++ facade class name (e.g., `#include <Foo.h>`; `classMethod();`, as explain on [Chapter 3 - Agnostic Core & HAL Architecture](03-agnostic-core-hal-architecture.md)).

### 3. PlatformIO Registry

PlatformIO allows flexibility in package naming and can resolve dependencies across casing styles without breaking. However, **recommending the same lowercase name as ESP-IDF** (`foo`) is best practice to prevent subtle cross-ecosystem naming conflicts and avoid the PlatformIO backend package rename trap.

---

## Beyond *Arduino* & *ESP-IDF*: Extending and Contributing

While this guide focused heavily on bridging Arduino, ESP-IDF and PlatformIO, the underlying architectural patterns (unified source layouts, explicit dependency management, and reciprocal registry exclusions) are broadly applicable to the wider embedded ecosystem.

Before diving into your own multi-ecosystem implementations, be sure to explore and contribute to the appendices for deeper technical insights:

- **[Appendix A: ESP-IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)** — A deep dive into how ESP-IDF resolves complex dependency graphs.
- **[Appendix B: Yield & Watchdog Conventions](../appendix/B-yield-watchdog.md)** — Guidelines on task yields and watchdog handling in long-running<!-- or multi-threaded environments-->.
- **[Appendix C: Naming Conventions](../appendix/C-naming-convention.md)** — A comprehensive recap class methods naming rules.

The embedded landscape is constantly evolving, and toolchains like Zephyr, new microcontrollers, or alternative build systems continue to emerge. If you are looking to extend this framework-agnostic approach to other platforms, refine the appendices, or share your own experiences, contributions and pull requests are always welcome.

Embedded development thrives on shared knowledge; feel free to open an issue, propose improvements, or adapt these patterns to help make multi-ecosystem library maintenance smoother for everyone.

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
