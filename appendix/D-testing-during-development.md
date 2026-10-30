# Appendix D — Testing During Development with PlatformIO

Before anything is published, a library has to run on every board it supports, ideally from a single example edited in one place. PlatformIO fits this well: one `platformio.ini`, one environment per board, and one `pio run` that builds them all, as long as the project is laid out correctly. The workflow is the same for Arduino and ESP-IDF: only a few settings change. Three more tools complete the picture: VS Code to read framework sources, `idf.py` for native ESP-IDF builds out of tree, and the Arduino IDE for what its users get.

## The stable layout: the project lives *outside* the library folder

```text
workspace/
├── <library>/                      ← the library repository
│   ├── src/
│   └── examples/
│       ├── <arduino_example>/
│       └── <idf_example>/
├── bench-arduino/platformio.ini    ← PlatformIO projects: only platformio.ini lives here
└── bench-espidf/platformio.ini
```

A `platformio.ini` has a single `src_dir`, so each framework needs its own project. The common skeleton:

```ini
[platformio]
; Take the sources straight from the library's own example: one source of truth.
; Arduino: <example> folder.  ESP-IDF: the example's main/ folder.
src_dir = <path-to-library>/examples/<example>

[env]
framework = arduino                  ; or espidf
monitor_speed = 115200
; Link (not copy) the library: edits are picked up by the next build.
lib_deps = symlink://<path-to-library>
```

- **`src_dir`** makes the example the test project: an Arduino `.ino` is converted to `.cpp` at build time, and an ESP-IDF example uses its `main/` folder, as in [Chapter 4](../chapters/04-examples.md). It still points *into* the library repository, but the **project directory** (`platformio.ini` and `.pio/`) does not.
- **`symlink://`** links the library folder, so a change in `src/` is used by the next build without reinstalling anything (`file://` would copy a snapshot).
- **Paths:** on Windows, `${sysenv.USERPROFILE}/…` keeps the file portable between machines; on Linux/macOS use `${sysenv.HOME}/…` or a relative path (not tested here).

---

## One environment per board

The same sources are built for every board; only the per-environment settings change, and the framework adds a few differences:

| | `framework = arduino` | `framework = espidf` |
| :--- | :--- | :--- |
| **Boards** | AVR, ESP8266, ESP32 family | ESP32 family only |
| **`src_dir`** | the example folder | the example's `main/` folder |
| **Platform** | one per architecture | pinned, e.g. `espressif32@6.4.0` (IDF 5.1.1): an unpinned platform update changes the IDF under your tests |
| **Native USB console** | `-D ARDUINO_USB_CDC_ON_BOOT=1 -D ARDUINO_USB_MODE=1`, plus `monitor_dtr`/`monitor_rts` | `board_build.sdkconfig_options` (`CONFIG_ESP_CONSOLE_USB_CDC=y`, `CONFIG_ESP_CONSOLE_UART_NONE=y`) |
| **Serial monitor** | `monitor_speed` | `monitor_speed`, plus `monitor_filters = direct` for readable colored logs |
| **Dependencies** | `lib_deps` | `lib_deps` and the example's `main/idf_component.yml` coexist, with some redundancy (see [Chapter 4](../chapters/04-examples.md)) |

**Arduino** (added to the skeleton above):

```ini
[env:nodemcu8266]
platform = espressif8266
board = nodemcuv2
build_flags = -D DATA_PIN=2 -D CLOCK_PIN=14   ; GPIO numbers: silkscreen D4 = GPIO2, D5 = GPIO14

[env:esp32s3-native-usb]                      ; same board as the next one, other USB port
platform = espressif32
board = esp32-s3-devkitc-1
build_flags = -D ARDUINO_USB_CDC_ON_BOOT=1 -D ARDUINO_USB_MODE=1
monitor_dtr = 1                               ; needed to see output after (re)connecting USB
monitor_rts = 1

[env:esp32s3-uart]                            ; USB-to-UART bridge port: no extra flags
platform = espressif32
board = esp32-s3-devkitc-1
```

**ESP-IDF** (the skeleton with `framework = espidf` and `src_dir` ending in `/main`; environments with no special settings, like the UART one above, only need their `board` line):

```ini
[env]
platform = espressif32@6.4.0
monitor_filters = direct

[env:esp32s3-native-usb]                      ; native USB console, set through sdkconfig
board = esp32-s3-devkitc-1
board_build.sdkconfig_options =
    CONFIG_ESP_CONSOLE_USB_CDC=y
    CONFIG_ESP_CONSOLE_UART_NONE=y

```

- **Pins and variants are macros.** The test project reads optional macros (here `DATA_PIN` and `CLOCK_PIN`) and falls back to defaults, so boards that need nothing special have no flags at all. Pass GPIO numbers, not silkscreen labels.
- **The native-USB settings** were required on the S3 and C3 boards to get serial output; the reason was not investigated here.
- **One physical board can need two environments** (native USB vs UART bridge): use one environment per wiring or USB port, not per board model.

The commands are the same for both frameworks:

```bash
pio run                                  # build every environment: the regression check before a release
pio run -e esp32s3-native-usb -t upload  # build and flash one board
pio device monitor -e esp32s3-native-usb # serial monitor with that environment's settings
```

With more than one board connected, set `upload_port` and `monitor_port` per environment.

---

## The unstable layout: the project lives *inside* the library folder

> [!WARNING]
> Keeping `platformio.ini` at the root of the library repository looks convenient, since a clone is enough to run the tests, but it produced non-deterministic build failures.

```ini
; UNSTABLE — the project directory is the library repository root
[platformio]
src_dir = examples/<example>
include_dir = src                     ; make the library headers visible

[env]
build_src_filter =
    +<*>                              ; every file in src_dir
    +<../../src/*.cpp>                ; library sources, two levels up from src_dir
```

**Observed:** while the build advances, the new files it creates in the directory appear to relaunch PlatformIO's own reconfiguration, which makes the build unstable: it stops partway or fails non-deterministically. Moving the project directory out of the library folder, with nothing else changed, fixed it.

---

## Reading the framework source: "Go to Definition"

`platformio.ini` tells VS Code which framework, platform and board you work with. Selecting an environment in the status bar rebuilds the IntelliSense configuration, so "Go to Definition" (F12) opens the **real source of that environment's framework**, under `~/.platformio/packages/`. This is how the behaviour in [Appendix B](B-yield-watchdog.md) was checked, instead of relying on documentation or memory.

- The same call opens **different code in different environments**: switch the environment, press F12 again.
- Some symbols open only a **declaration** ("Function definition not found", as with `yield()`): the definition is in a source file IntelliSense didn't index, so search the framework's folder for it.

---

## Native ESP-IDF, out of tree

To build an ESP-IDF example the way its users do, without leaving build files in the repository:

```powershell
idf.py -C "<path-to-library>\examples\<idf_example>" -B <output-folder> build
```

`-C` selects the project directory and `-B` the build directory, so the example folder gets no `build/` tree. Run it from an ESP-IDF terminal, and repeat `-B` on every command that uses the build (`idf.py -C … -B … -p <PORT> flash monitor`).

> [!TIP]
> `-B` moves the build directory only. `dependencies.lock` still ends up next to the project, and `sdkconfig` and `managed_components/` may too. Check `git status` after the first build and ignore them in `.gitignore`.

---

## Arduino IDE

The Arduino IDE only looks in its own `libraries` folder, so the only way to test there is to **copy or move the whole repository** into `Documents\Arduino\libraries\`. The folder name doesn't matter, since a library's identity comes from its public headers (see [Chapter 1](../chapters/01-ecosystem-constraints.md)). Keep **one copy**, not two, and restart the IDE if the library doesn't appear in the menu.

---

## Rules of thumb

- Keep the **project directory** (`platformio.ini`, `.pio/`) outside the library repository; `src_dir` may point inside it.
- Link the library with `symlink://`, not a copy.
- Run `pio run` (all environments) before every release, then flash and read the output of each board you have at hand.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Appendix C - Naming Convention](C-naming-convention.md) — [Introduction](../README.md)

</div>
<!-- markdownlint-enable MD033 -->
