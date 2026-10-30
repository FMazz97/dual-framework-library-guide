# Appendix E — Final Checklist

Every item below is a rule that came out of the previous chapters. The last section lists what this guide did **not** verify; keep it honest.

## E.1 Before writing code

- [ ] Fix the **final package name** for every registry before the first publish: lowercase, 2–64 characters, `a-z`, `0-9`, `-`, `_`, no leading, trailing or consecutive separators. `compote` lowercases `--name` anyway. (Ch. 5, App. A, App. C)
- [ ] Fix the naming scheme: `snake_case` in the core, `camelCase` in the Arduino facade, shared types and constants identical in both; prefix semantics `get_` / `set_` / `read_` / `is_` / `calibrate_`. (App. C)
- [ ] Hardware access goes through **compile-time macros** (HAL), not virtual interfaces or templates. (Ch. 3)
- [ ] The core calls no framework API directly. Test `ARDUINO` before `ESP_PLATFORM`. (Ch. 3)
- [ ] In headers shared with AVR, include the C headers (`<stdint.h>`, `<math.h>`): `<cstdint>` and `<cmath>` broke the AVR build in this project.

## E.2 Repository layout

- [ ] IDF manifest and `CMakeLists.txt` at the **repository root**; sources under `src/`, referenced by paths that only go down (never `..`). (Ch. 2)
- [ ] Arduino: everything that must compile lives under `src/`; examples are `examples/<Name>/<Name>.ino`. (Ch. 1)
- [ ] The facade header and the core header differ only in case, so they never share a folder. (App. C)
- [ ] Check what the package exclusion lists remove: excluding the Arduino HAL header breaks anyone building the IDF package with `ARDUINO` defined. (Ch. 5)
- [ ] Each IDF example is a native project: `main/`, `main/idf_component.yml` (with `override_path`), and a `platformio.ini` with `src_dir = main`. (Ch. 4)

## E.3 Manifests

- [ ] `library.properties` (Arduino): `name`, `version`, `includes`.
- [ ] `library.json` (PlatformIO): `name` lowercase, `"frameworks": ["arduino", "espidf"]`, example `name`s **without parentheses**.
- [ ] `idf_component.yml` (ESP): `version`, `description`, `url`/`repository`, `idf` constraint, `examples`, `files: exclude`.
- [ ] **One version number, identical in all three.**

## E.4 Dependencies under ESP-IDF

- [ ] Use **one** resolution path: `idf_component.yml` (works under native IDF and PlatformIO), or `lib_deps` with **no** `REQUIRES`. Never `lib_deps` and `REQUIRES` together without a manifest. (App. A)
- [ ] With `EXTRA_COMPONENT_DIRS`: the `list(APPEND …)` goes **before** `include(…project.cmake)`; the folder name is the component name, case preserved.
- [ ] `REQUIRES <name>` accepts the short name for Registry components; the long `<namespace>__<name>` is rejected. (App. A)
- [ ] Test the example **as a user gets it**: `idf.py create-project-from-example "<namespace>/<name>^<version>:<example>"`, then build. (App. A)

## E.5 Cooperating with the scheduler

- [ ] No portable `yield()`: AVR does nothing, ESP8266 needs the real call, ESP32 (Arduino and IDF) needs `vTaskDelay(1)`, not `taskYIELD()`. (App. B)
- [ ] Polling loops: stay tight during the normal conversion window, yield after it, stop yielding near the timeout.
- [ ] The guard before the timeout is **larger than one yield**; very long averages at high rates throttle their outer loop too.
- [ ] Timeouts include a margin of at least one tick period.

## E.6 Testing during development

- [ ] The PlatformIO project directory is **outside** the library repository; `src_dir` may point inside. Library linked with `symlink://`. (App. D)
- [ ] One PlatformIO environment per wiring or USB port; `pio run` (all environments) before every release.
- [ ] Native build: `idf.py -C <example> -B <output-folder> build`; check `git status` afterwards.
- [ ] Arduino IDE: copy the whole repository into `libraries/`, keep one copy.
- [ ] Never mix `idf.py` and PlatformIO builds in one folder without cleaning (`idf.py fullclean`). (Ch. 4)
- [ ] When framework behaviour matters, open its source with F12 (Go to Definition) instead of trusting memory. (App. D)
- [ ] **Never paste an API token into a log, report or issue.** If it happens, revoke it.

## E.7 Release, in order

1. [ ] Bump `version` in all three manifests.
2. [ ] Commit and **push** the bump. The tag snapshots the commit it points to.
3. [ ] `arduino-lint --library-manager submit`.
4. [ ] `pio pkg pack`, then list the archive (`tar -tzf`).
5. [ ] `compote component upload --name <name> --dry-run`, then list the archives in `dist/`. The dry run uploads for validation and does not consume the version.
6. [ ] Build every environment and flash at least one board.
7. [ ] Tag and publish the GitHub Release.
8. [ ] `pio pkg publish`. `accepted` is not `published`: wait for the email and check the package page.
9. [ ] `compote component upload --name <name>`. Processing is asynchronous; check `upload-status`.
10. [ ] Arduino: first release goes through a Pull Request to the registry, updates are automatic after a new tag.
11. [ ] Install from each registry in a **clean project** and build.

## E.8 Known limits

- The PIO Home → Examples panel ignores `examples[].name` for libraries and lists files one by one; the web Registry shows the names. (Ch. 5)
- A PlatformIO package cannot be renamed from the CLI; it takes a support request. (Ch. 5)

## E.9 Not verified in this guide

- Arduino tag with a `v` prefix: only tags without it were tested.
- Version immutability on both Registries: stated from the documentation, never tested.
- `lib_deps` references to the old name after the PlatformIO rename.
- How the build system accepts `<name>` in `REQUIRES` for a component registered as `<namespace>__<name>`.
- Which file (`build/`, `sdkconfig`, `dependencies.lock`, `managed_components/`) causes the `idf.py` → PlatformIO conflict.
- Why the Arduino-ESP32 build never tripped the Task Watchdog (tick rate and core of `loop()` not compared).
- `sdkconfig` options for native USB console on the C3.
