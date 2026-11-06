# Chapter 5 — Publishing

Publishing a framework-agnostic library means interacting with three independent package registries: the [Arduino Library Manager](https://docs.arduino.cc/libraries/), the [ESP Component Registry](https://components.espressif.com/), and the [PlatformIO Registry](https://registry.platformio.org/). Each ecosystem uses its own manifest format, indexing pipeline, and rules for package immutability and naming.

Rather than treating publishing as a single step, this chapter covers each registry individually: how its manifest handles metadata and file filtering (as introduced in [Chapter 4 - Example Conventions](04-examples.md)), local pre-verification, initial publication, and updates. It closes with the naming rules that keep the three registries consistent.

---

## 5.1 Arduino Library Manager

The Arduino Library Manager uses an automated, tag-driven indexing model. Rather than hosting binary archives or accepting direct uploads via CLI, the registry periodically scans registered Git repositories.

### 5.1.1 Manifest and Metadata

The single source of truth for Arduino library metadata is `library.properties`, located at the repository root. The indexer reads the manifest from the commit attached to the **tag**, not from the default branch: updating `library.properties` on `main` after creating a release tag will not update the published release.

- **Library Name:** name comes from the `name` field in `library.properties` manifest;
- **Version Indexing:** Arduino's indexer relies strictly on Git tags (e.g. `1.0.0`). The `version` field in `library.properties` **must match the Git tag name exactly**, following Semantic Versioning (SemVer)
- **Dependency Declaration:** dependency resolution is supported, but remains library-centric. Refer to other Arduino libraries using the `depends` field (see the **depends** point in the [library format](https://docs.arduino.cc/arduino-cli/library-specification/#library-metadata) section of the Arduino documentation).
- **File Management:** the Arduino Library Manager does not provide manifest-level file filtering. The published package always reflects the contents of the tagged release, making repository structure the primary mechanism for controlling what is distributed.

For more details, see the official [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/) documentation.

### 5.1.2 Pre-Verification

Before submitting a library, validate the layout and metadata locally using:

```bash
arduino-lint --library-manager submit
```

This command runs Arduino Lint with the extra checks that Library Manager requires for a submission, catching missing fields, incorrect directory structures, or invalid dependency declarations before the review starts. ESP-IDF files are simply ignored.

For more details, see the official [Arduino Lint](https://arduino.github.io/arduino-lint/latest/) documentation.

### 5.1.3 Initial Publication

Initial inclusion requires a Pull Request against the [arduino/library-registry](https://github.com/arduino/library-registry) GitHub repository, adding the library repository URL to the registry list. The submission is then validated automatically by the same tool (Arduino Lint) so a clean local run is a good sign, though not a guarantee, that the automated check will pass.

Before submitting, **at least one tagged version must already exist, and its version must match the `version` field declared in `library.properties`.** Arduino accepts either a release or a tag, but creating the tag together with its corresponding GitHub Release is the recommended practice.

### 5.1.4 Updates

After acceptance, updates are fully automatic. To publish a new release:

1. Bump the `version` field in `library.properties`.
2. Validate the update locally using: `arduino-lint --library-manager update`
3. Create and publish a matching Git tag and the corresponding GitHub Release.

The Arduino indexer detects the new release and updates the library. No additional Pull Requests are required.

---

## 5.2 ESP Component Registry

The ESP Component Registry is Espressif's official package manager for ESP-IDF components. It operates under a self-service, push-based model.

### 5.2.1 Manifest and Metadata

The single source of truth for ESP-IDF component metadata is `idf_component.yml`. The registry always indexes the metadata associated with the published version, not the current state of the repository.

- **Package Name:** component name come through the `--name` field specified in the CLI upload command, rather than the `name` field in the `idf_component.yml` manifest.
- **Version Indexing:** versioning is driven explicitly by the `version:` field inside `idf_component.yml`, following Semantic Versioning (SemVer).
- **Dependency Declaration:** external component dependencies are listed in the `dependencies:` section:

  ```yaml
  dependencies:
    <namespace>/<component>: "^<version>"
  ```

- **File Management & Example Inclusion:** to keep the published package free of foreign toolchain files and to select explicitly which examples to package, combine `examples` with the `files.exclude` key in `idf_component.yml`:

  ```yaml
  examples:
    - path: ./examples/basic_example

  files:
    exclude:
      - "library.properties"          # Arduino's manifest
      - "library.json"                # PlatformIO's manifest
      - "src/Foo.h"                   # Arduino's facade *
      - "src/hal/foo_hal_arduino.h"   # Arduino's HAL *
      - "examples/*/*.ino"            # Arduino's examples
      - "examples/*/platformio.ini"   # PlatformIO's examples configuration
      - "assets/**"                   # other documentation
      - "docs/**"                     # other documentation
      # add any other file that is irrelevant for an ESP-IDF component
  ```

> [!TIP]
> **`*`** If you want to support `Arduino as an ESP-IDF component`, leave them out of the exclusions so the Arduino facade and HAL stay in the package: the overhead is minimal.

For more details, see the official [How to Package and Upload Examples](https://docs.espressif.com/projects/idf-component-manager/en/latest/publish/how_to_examples.html) and [Component Files](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/component_files.html) documentation.

### 5.2.2 Pre-Verification

Before uploading to the ESP Component Registry, validate the manifest and simulate the archive creation using `compote`, the Component Manager CLI:

```bash
compote component upload --name <component_name> --dry-run
```

This command validates the component against the registry (it checks `idf_component.yml` compliance, verifies the exclusion rules, and confirms that the archive builds correctly) without creating a version, and it does not require the API token.

### 5.2.3 Initial Publication

Publishing is performed directly via the CLI:

```bash
compote component upload --name <component_name> --namespace <your_user_name>
```

The archive is uploaded and then processed server-side; the version appears once processing completes.

> [!NOTE]
> **Token authentication:**
> The upload command requires an ESP Component Registry API token. Set the `IDF_COMPONENT_API_TOKEN` environment variable (`export IDF_COMPONENT_API_TOKEN=<token>` on Linux/macOS, `$env:IDF_COMPONENT_API_TOKEN="<token>"` in PowerShell) before running `compote component upload`.

For more details, see the IDF Component Manager official documentation on [Publish components](https://docs.espressif.com/projects/idf-component-manager/en/latest/publish/index.html) and [Compote CLI](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/compote_cli.html).

### 5.2.4 Updates

To publish bug fixes or changes, bump `version:` in `idf_component.yml` and upload the component again, using the same command and the token authentication as the initial publication:

```bash
compote component upload --name <component_name> --namespace <your_user_name>
```

Each published version of a component (`<namespace>/<component_name>`) is permanent and immutable: once a version is uploaded, Espressif's backend prevents re-uploading or modifying that exact version string.

---

## 5.3 PlatformIO Registry

The PlatformIO Registry is a self-service, CLI-driven package registry supporting Arduino, ESP-IDF, and other embedded frameworks.

### 5.3.1 Manifest and Metadata

PlatformIO package metadata is declared in `library.json` at the root of the project. The registry always indexes the metadata associated with the published version, not the current state of the repository. Declare both `"arduino"` and `"espidf"` in the `"frameworks"` field.

- **Library Name:** name comes from the `"name"` string inside `library.json`;
- **Version Indexing:** versioning is controlled by the `"version"` string inside `library.json`, following Semantic Versioning (SemVer).
- **Dependency Declaration:** package dependencies are declared inside the `"dependencies"` object:

  ```json
  "dependencies": {
    "<owner>/<name>": "^<version>"
  }
  ```

- **Header folders:** PlatformIO by default adds `src/` and `include/` to the include path, not its subfolders, so the core's public headers in `src/core/` are declared with:

  ```json
  "build": {
    "flags": ["-I src/core"]
  }
  ```

  With `framework = espidf` this matters even more: the header an ESP-IDF example includes is the core one in `src/core/`, not one directly in `src/`, and a library linked through `lib_deps` is built by PlatformIO's library builder, which never reads the library's `CMakeLists.txt`, so the `INCLUDE_DIRS "src/core"` of its `idf_component_register()` is not seen (see [2.1.3 PlatformIO Source & Include Path](02-repo-structure.md#213-platformio-source--include-path)).

- **File Management & Example Inclusion:** to omit the ESP-IDF component files along with `assets/`, `docs/`, and other folders containing non-essential contents, and to declare explicitly which example projects (and which of their files) are packaged and under what name, combine the `"export"` exclusion block with the `"examples"` configuration in `library.json`.
  - **Example names:** use only letters, digits, `-` and `_` in the `"examples"` → `"name"` field. Parentheses make the manifest invalid: `pio pkg pack` and `pio pkg publish` both report it, so the pre-verification step catches it before publishing (other special characters were not tested). For ESP-IDF examples, whitespace also breaks the build: the registry derives the download directory from the name, and a path containing spaces is not supported by the ESP-IDF build system (see the [limitations](https://docs.platformio.org/en/latest/frameworks/espidf.html#limitations) of PlatformIO's ESP-IDF integration).
  - **Arduino examples:** PlatformIO does not require the `.ino` filename to match its parent folder, but keep the convention from [Chapter 4](04-examples.md#41-arduino), since the same files are consumed by the Arduino IDE.

  ```json
  "examples": [
    {
      "name": "Arduino_BasicExample",
      "base": "examples/BasicExample",
      "files": ["BasicExample.ino", "platformio.ini"]
    },
    {
      "name": "ESP-IDF_basic_example",
      "base": "examples/basic_example",
      "files": [
        "main/main.cpp",
        "main/CMakeLists.txt",
        "CMakeLists.txt",
        "platformio.ini",
        "README.md"
      ]
    }
  ],
  "export": {
    "exclude": [
      "assets",
      "docs",
      "CMakeLists.txt",
      "idf_component.yml",
      "library.properties",
      "examples/*/main/idf_component.yml"
    ]
  }
  ```

> [!IMPORTANT]
> **PIO Home example rendering:**
> When examples are declared as structured multi-file entries in `library.json` (`name`, `base`, `files`), the **PIO Home** panel in VS Code does not group their files correctly: instead of a single entry labeled with the example's `name`, it flattens the array and lists each source file as a separate item (`main`, `CMakeLists`, etc.). This is a known limitation of the desktop interface, reported in [PlatformIO Issue #12672](https://github.com/platformio/platformio-home/issues/12672). The metadata and structure are correct, and the examples render properly on the online PlatformIO Registry.

For more details, see the official PlatformIO [library.json](https://docs.platformio.org/en/latest/manifests/library-json/index.html) documentation.

### 5.3.2 Pre-Verification

To verify which files will be included in the published package, generate a local package archive using the PlatformIO Core CLI:

```bash
pio pkg pack
```

This creates a `.tar.gz` archive in the project root and validates the `library.json` manifest. Inspecting its contents lets you confirm that the `export.exclude` rules worked and that no unwanted build files or manifests were included.

### 5.3.3 Initial Publication

To publish the package to the PlatformIO Registry, execute:

```bash
pio pkg publish
```

The package is processed server-side and the result is reported by email.

> [!NOTE]
> **Account authentication:**
> The publish command requires an authenticated PlatformIO account. Make sure you are logged in locally (`pio account login`) before running `pio pkg publish`.

For more details, see the official PlatformIO [Package Management CLI](https://docs.platformio.org/en/latest/core/userguide/pkg/index.html) documentation.

### 5.3.4 Updates

To publish bug fixes or changes, bump `"version"` in `library.json` and publish the package again, with the same account authentication as the initial publication:

```bash
pio pkg publish
```

Each published version of a package (`<owner>/<name>`) is permanent and immutable: once a version is uploaded, the PlatformIO backend prevents re-uploading or modifying that exact version string.

---

## 5.4 The case-sensitivity trap: PlatformIO's immutable package binding

PlatformIO doesn't constrain the case of a library published on its registry. A package can be named `Foo` or `foo` and builds the same way: PlatformIO resolves package names case-insensitively, and the folder it installs takes the registry name, in either case. The ESP-IDF rule that a component is named after its folder doesn't come into play either: a library linked through `lib_deps` is built by PlatformIO's library builder and never becomes an ESP-IDF component (see [2.1.3 PlatformIO Source & Include Path](02-repo-structure.md#213-platformio-source--include-path)).

[Chapter 1 - Ecosystem Constraints](01-ecosystem-constraints.md#registry-naming-rules) showed that Registry-compatible ESP-IDF components require lowercase names; PlatformIO has no such requirement. What it has is a binding: its registry backend ties a package's name to its internal ID, so **the name and casing recorded at the moment of your very first publish stay bound to the package's `pkg_id` and can't be changed from the CLI.**

This typically bites when a cross-framework library is first published following Arduino conventions (e.g. uppercase characters, as in `Foo`) and native ESP-IDF support is added later. The natural move is to align the `"name"` field in `library.json` to lowercase (`foo`), matching the ESP Component Registry. Publishing the new version then fails with a database mismatch error:

```text
Manifest name='foo' <> DB name='Foo' and pkg_id is not passed
```

Because the registry permanently bound the original casing to the package ID, and the PlatformIO CLI exposes no flag or self-service mechanism to override it (`pio pkg publish --help` has no package-ID option), two difficult paths remain:

- **Delete and republish under the new name** (not tested). Likely destructive to download metrics. Downstream `platformio.ini`/`lib_deps` entries referencing the old casing should keep resolving, since the registry handles lookups case-insensitively.

- **Ask the registry's support team to rename the entry server-side.** PlatformIO's support can rename the existing package in place, preserving its history and download metrics, in response to a direct support request. This is the path followed for the [CS123x library](https://github.com/FMazz97/CS123x).

Since PlatformIO accepts either case, the conclusion is not that the name must be lowercase there, but that it must be chosen once: using the same lowercase name as the ESP Component Registry from the first publish avoids both this trap and any mismatch between the two ecosystems.

---

## 5.5 The Generalizable Naming Rule

Package renaming is not handled consistently across ecosystems: the Arduino Library Manager provides a dedicated *Library Name Change* procedure, while PlatformIO, as the previous section showed, requires a server-side intervention. The safest approach is therefore to avoid renaming altogether by choosing a coherent naming strategy before the first public release.

**Decide the published package's name — in every registry it will live in — before the first publish.**

In practice:

### 5.5.1 Local Development Directory & ESP-IDF Component

Keep the local component folder (and any `name:` you declare in the manifest) lowercase, using only `a-z`, `0-9`, `_` and `-` (e.g. `foo`). ESP-IDF itself doesn't enforce this locally: the constraint comes from publication, where the Registry enforces lowercase names (see: [1.2.2 Publication imposes the constraints](01-ecosystem-constraints.md#122-publication-imposes-the-constraints)). Naming the component this way from day one avoids renames later.

### 5.5.2 Online Repository & Arduino Library

Name the GitHub repository and `library.properties` using **PascalCase** (e.g. `Foo`). This aligns with the Arduino community standard where library names match their primary C++ facade class name (see [Case styles](../appendix/C-naming-convention.md#case-styles-snake_case-for-the-core-directly-used-by-esp-idf-camelcase-for-the-arduino-facade) section in [Appendix C](../appendix/C-naming-convention.md)).

### 5.5.3 PlatformIO Registry

PlatformIO allows flexibility in package naming and can resolve dependencies across casing styles without breaking. However, **using the same lowercase name as ESP-IDF** (`foo`) is best practice: it prevents subtle cross-ecosystem naming conflicts and avoids the PlatformIO package rename trap.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 4 - Example Conventions](04-examples.md) — [Introduction](../README.md) — [Chapter 6 - Conclusions ➡️](06-conclusions.md)

</div>
<!-- markdownlint-enable MD033 -->
