# Final Checklist

Every item below is a rule that came out of the previous chapters and of the development of CS123x, in the order in which they come up.

## Before writing code

- [ ] Fix the **final package name** for every registry before the first publish. ESP component: lowercase, 2–64 characters, `a-z`, `0-9`, `-`, `_`, no leading, trailing or consecutive separators (`compote` lowercases `--name` anyway). PlatformIO: the same lowercase name. Arduino library and GitHub repository: `PascalCase`. ([Ch. 1](01-ecosystem-constraints.md#registry-naming-rules), [Ch. 5](05-publishing.md#55-the-generalizable-naming-rule), [App. C](../appendices/C-naming-convention.md))
- [ ] Fix the naming scheme: `snake_case` in the core, `camelCase` methods in the Arduino facade, shared types and constants identical in both; prefix semantics `get_` / `set_` / `read_` / `write_`. ([App. C](../appendices/C-naming-convention.md))
- [ ] Use plain enums whose enumerators carry the class prefix; avoid `enum class`. ([App. C](../appendices/C-naming-convention.md))
- [ ] Hardware access goes through **compile-time macros** (HAL), not virtual interfaces or templates. ([Ch. 3](03-agnostic-core-hal-architecture.md))
- [ ] The core calls no framework API directly. Test `ARDUINO` before `ESP_PLATFORM`. ([Ch. 3](03-agnostic-core-hal-architecture.md))
- [ ] In headers shared with AVR, include the C headers (`<stdint.h>`, `<math.h>`): `<cstdint>` and `<cmath>` broke the AVR build in this project.

## Repository layout

- [ ] IDF manifest and `CMakeLists.txt` at the **repository root**; sources under `src/`, referenced by paths that only go down (never `..`). ([Ch. 2](02-repo-structure.md))
- [ ] Arduino: everything that must compile lives under `src/`; examples are `examples/<Name>/<Name>.ino`. ([Ch. 1](01-ecosystem-constraints.md), [Ch. 4](04-examples.md))
- [ ] The facade header and the core header differ only in case, so they never share a folder (for now: see the warning in the [Introduction](../README.md)). ([App. C](../appendices/C-naming-convention.md))
- [ ] Check what the package exclusion lists remove: excluding the Arduino HAL header breaks anyone building the IDF package with `ARDUINO` defined. ([Ch. 5](05-publishing.md))
- [ ] Each IDF example is a native project: `main/`, `main/idf_component.yml` (with `override_path`), and a `platformio.ini` with `src_dir = main` and `lib_deps`. ([Ch. 4](04-examples.md))

## Manifests

- [ ] `library.properties` (Arduino): `name`, `version`, `includes`.
- [ ] `library.json` (PlatformIO): `name` lowercase, `"frameworks": ["arduino", "espidf"]`, `"build": {"flags": ["-I src/core"]}` for headers in subfolders, example `name`s **without parentheses and spaces**.
- [ ] `idf_component.yml` (ESP): `version`, `description`, `url`/`repository`, `idf` constraint, `examples`, `files: exclude`.
- [ ] **One version number, identical in all three.**

## Dependencies under ESP-IDF

- [ ] Give each toolchain **one** authoritative resolution path: `idf_component.yml` for native `idf.py`, `lib_deps` for PlatformIO. Never `lib_deps` and `REQUIRES` together without a manifest or `EXTRA_COMPONENT_DIRS`: PlatformIO fetches the package, but nothing registers it as a component. ([Ch. 4](04-examples.md#43-platformio-component-manager-and-cmake-are-independent-dependency-resolution-systems))
- [ ] With `EXTRA_COMPONENT_DIRS`: the `list(APPEND …)` goes **before** `include(…project.cmake)`; the component is named after its folder. ([App. A](../appendices/A-esp-idf-dependency-resolution.md))
- [ ] `REQUIRES` is optional in `main`, which requires every component automatically, and needed in any other component. ([App. A](../appendices/A-esp-idf-dependency-resolution.md))
- [ ] `REQUIRES <name>` accepts the short name for Registry components; the long `<namespace>__<name>` is rejected (in tests). ([App. A](../appendices/A-esp-idf-dependency-resolution.md))
- [ ] Test the example **as a user gets it**: `idf.py create-project-from-example "<namespace>/<name>^<version>:<example>"`, then build.

## Cooperating with the scheduler

- [ ] No portable `yield()`: AVR does nothing, ESP8266 needs the real call, ESP32 (Arduino and IDF) needs `vTaskDelay(1)`, not `taskYIELD()` or Arduino's `yield()`, which are only reschedule requests. ([App. B](../appendices/B-yield-watchdog.md))
- [ ] Polling loops: stay tight during the normal conversion window, yield after it, stop yielding near the timeout.
- [ ] The guard before the timeout is **larger than one yield** (up to one tick period), so a yield can never cause a timeout.
- [ ] Very long averages at high rates throttle their outer loop too, at an interval well below the watchdog timeout.

## Testing during development

- [ ] The PlatformIO project directory is **outside** the library repository; `src_dir` may point inside (the example folder, or its `main/` for ESP-IDF). Library linked with `symlink://`. ([App. D](../appendices/D-testing-during-development.md))
- [ ] One PlatformIO environment per wiring or USB port; `pio run` (all environments) before every release.
- [ ] Native build: `idf.py -C <example> -B <output-folder> build`; check `git status` afterwards.
- [ ] Arduino IDE: copy the whole repository into `libraries/`, keep one copy.
- [ ] Never mix `idf.py` and PlatformIO builds in one folder without cleaning (`idf.py fullclean`). ([Ch. 4](04-examples.md))
- [ ] When framework behaviour matters, open its source with F12 (Go to Definition, works with PlatformIO) instead of trusting memory. ([App. D](../appendices/D-testing-during-development.md))

## Release, in order

1. [ ] Build every environment and flash at least one board.
2. [ ] Bump `version` in all three manifests.
3. [ ] Commit and **push** the bump. The tag snapshots the commit it points to.
4. [ ] `arduino-lint --library-manager submit`.
5. [ ] `pio pkg pack`, then list the archive (`tar -tzf`).
6. [ ] `compote component upload --name <name> --dry-run`, then list the archives in `dist/`. The dry run validates without creating a version and needs no token.
7. [ ] Tag and publish the GitHub Release.
8. [ ] Arduino: the first release goes through a Pull Request to the registry; updates are automatic after a new tag.
9. [ ] `pio pkg publish` (logged in with `pio account login`). `accepted` is not `published`: wait for the email and check the package page.
10. [ ] `compote component upload --name <name> --namespace <namespace>` (token in `IDF_COMPONENT_API_TOKEN`). Processing is asynchronous; check `upload-status`.
11. [ ] Install from each registry in a **clean project** and build.

See [Chapter 5](05-publishing.md) for the details of each registry.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 6 - Conclusions](06-conclusions.md) — [Introduction](../README.md)

</div>
<!-- markdownlint-enable MD033 -->