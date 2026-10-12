# Chapter 5 — Publishing Across Ecosystems and Package Registry Mechanics

Publishing a framework-agnostic C++ library requires interacting with three independent package registries: the [Arduino Library Manager](https://docs.arduino.cc/libraries/), the [ESP Component Registry](https://components.espressif.com/), and the [PlatformIO Registry](https://registry.platformio.org/). Each ecosystem uses its own manifest format, indexing pipeline, and rule set for package immutability and naming.

Rather than treating publishing as a single monolithic step, this chapter breaks down the publication mechanics for each registry individually, covering metadata definition, file filtering, local pre-validation, release workflows, and lifecycle maintenance.

---

## 5.1 Arduino Library Manager

The Arduino Library Manager uses an automated, tag-driven indexing model. Rather than hosting binary archives or accepting direct uploads via CLI, the registry periodically scans registered Git repositories.

### 5.1.1 Manifest and Metadata

The single source of truth for Arduino library metadata is the `library.properties` file located at the repository root.

- **Version Indexing:** Arduino's indexer relies strictly on Git tags (e.g., `v1.0.0` or `1.0.0`). The `version` field in `library.properties` **must match the Git tag name exactly**. The indexer reads the manifest from the commit attached to the tag, not from the default branch. Updating `library.properties` on `main` after creating a release tag will not update the published release.

- **Dependency Declaration:** Dependencies resolution is supported, but remains library-centric. Refer to other Arduino libraries using the `depends` field in `library.properties` (see `depends` point in Arduino [library format](https://docs.arduino.cc/arduino-cli/library-specification/#library-metadata) section).

- **File Management:** not allow.

For more details, see [Arduino Library Specification](https://docs.arduino.cc/arduino-cli/library-specification/).

### 5.1.2 Local Pre-Verification

Before submitting a library to the Arduino Registry, validate the layout and metadata locally using `arduino-lint`:

```Bash
arduino-lint --library-manager submit
```

This command runs the exact same automated checks that the Arduino indexing bot performs, catching missing fields, incorrect directory structures, or invalid dependency declarations before initiating a review.

For more details, see [Arduino Lint](https://arduino.github.io/arduino-lint/latest/).

### 5.1.3 Initial Publication

Initial inclusion requires a Pull Request against [arduino/library-registry](https://github.com/arduino/library-registry), adding the library repository URL to the registry list. The submission is then validated automatically through the same checks performed by `arduino-lint`.

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

An ESP-IDF component's metadata is defined in `idf_component.yml` inside the component directory

- **Version Indexing:** Versioning is driven explicitly by the version: field inside `idf_component.yml` following Semantic Versioning (SemVer).

- **Dependency Declaration:** External component dependencies are listed in the `dependencies:` section:

  ```yaml
  dependencies:
    <autor>/<component>: "^1.0.0"
  ```

- **File Management:** To ensure the published component package remains clean of foreign toolchain files, use the `files.exclude` key in `idf_component.yml`:

  ```yaml
    files:
        exclude:
            - "platformio.ini"
            - "examples/*/*.ino"
            - "examples/*/platformio.ini"
            - ".github/"
            - ".mignore"
  ```

<!-- parti precedenti da controllare se riprendere qualcosa 
### Component metadata is taken from the published package version, not from the current repository state

Once a version is published to the Component Registry, the corresponding `idf_component.yml` and associated metadata are immutable from the consumer's perspective. Updating the manifest on the default branch has no effect on already-published versions; a new version must be released for dependency or metadata changes to become visible.

### Dependency management is component-based

Components declare their dependencies through mechanisms such as `REQUIRES`, `PRIV_REQUIRES`, or `idf_component.yml`, allowing ESP-IDF to construct an explicit dependency graph at build time.
-->

For more details, see [idf_component.yml Manifest File](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/manifest_file.html).

### 5.2.2 Local Pre-Verification

Before uploading to the Espressif Component Registry, validate the manifest and simulate the archive creation using `compote` (Espressif's Component Manager CLI):

```bash
compote component upload --name <component_name> --dry-run
```

This command parses `idf_component.yml`, checks SemVer compliance, verifies exclusion rules, and confirms that the archive package builds correctly without actually uploading it to the server.

### 5.2.3 Initial Publication

Publishing is performed directly via the CLI:
<!-- dopo aver creato un toke su esp component registry ed essersi registrati con tale toke tramite:
```bash
$env:IDF_COMPONENT_API_TOKEN="<token>"
```
-->

```bash
compote component upload --name <component_name> --namespace <your_user_name>
```

For more details, see [Compote CLI](https://docs.espressif.com/projects/idf-component-manager/en/latest/reference/compote_cli.html).

### 5.2.4 Updates

Published versions (`namespace/name^version`) on the ESP Component Registry are permanent and immutable. Once a version is uploaded, Espressif's backend prevents re-uploading or modifying that exact version string. To publish bug fixes or changes, you must increment `version:` in `idf_component.yml` and run again:

```bash
compote component upload --name <component_name> --namespace <your_user_name>
```

---

## PlatformIO Registry — self-service via CLI, immutable once published

### Package metadata is taken from the published package version, not from the current repository state

Once a version is published to PlatformIO Registry, the corresponding manifest `library.json` and associated metadata are immutable from the consumer's perspective. Updating the repository's default branch has no effect on already-published versions; a new version must be released for changes to become visible. Consequently, early naming decisions are permanent, making it best practice to adopt the strictest convention — such as ESP-IDF's lowercase style — from day one.

### Dependency management is package-based

Dependencies are declared in `library.json` and resolved through the PlatformIO Registry, allowing PlatformIO to automatically fetch and install the required packages.

### Publishing pkg

```bash
pio pkg publish
```

No review step — the package is live as soon as the command succeeds. This also means the immutability constraint has no safety net: **a given `name@version` combination can never be republished**, even if deleted. Verify the manifest's `version` field is correct before running the command, not after.

---

## The case-sensitivity trap: it follows the package name, not just the source folder

Chapter 1 established that ESP-IDF's native dependency resolvers are case-sensitive. What's easy to miss is that **this doesn't stop being true once a package is registered** — the name recorded at publish time becomes another case-sensitive identity that every subsequent `REQUIRES`/`idf.py add-dependency` reference has to match exactly, just like the local folder name did before publishing existed at all.

This library was first published to both PlatformIO and the ESP Component Registry as `CS123x` (uppercase), matching the Arduino Library Manager entry already live. It later needed to be `cs123x` (lowercase) for consistent resolution under ESP-IDF's build system, once native ESP-IDF support was added. Renaming `library.json`'s `name` field and republishing produced:

```text
Manifest name='cs123x' <> DB name='CS123x' and pkg_id is not passed
```

The registry backend had recorded the original case as the package's identity — the manifest and the database no longer agreed, and no CLI flag exists to say "this is the same package under a new name" (`pio pkg publish --help` exposes no package-ID option). Two paths exist once this happens:

- **Delete and republish under the new name.** Available, but destructive: existing download statistics are lost, and any already-published `platformio.ini`/`lib_deps` referencing the old name breaks immediately for anyone using it.
- **Ask the registry's support team to rename the entry server-side.** This is what resolved it in practice — PlatformIO's support renamed the existing package in place, preserving its history, in response to a direct support request. No public self-service rename mechanism was found; asking is the only tested path that keeps the package's history intact.

The ESP Component Registry side never hit this problem, for a simple reason worth generalizing: it was the *first* publication of that component under any name — there was nothing to rename, only something to name correctly from the start.

## The generalizable rule

**Decide the published package's name — in every registry it will live in — before the first publish, not after.** A rename mid-lifecycle isn't just an inconvenience; on registries with immutable versioning, it's either destructive (delete and lose history) or dependent on a support team's willingness to intervene manually, since the tooling itself provides no self-service path for it. The lowercase-everywhere convention this guide has used throughout Chapters 1–3 isn't only about satisfying `EXTRA_COMPONENT_DIRS`'s local-folder constraint — it's the same choice that, made early, avoids this exact registry-level trap entirely.
