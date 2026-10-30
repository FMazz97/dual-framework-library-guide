# Appendix D — Testing During Development with PlatformIO

Before anything is published, a library has to run on every board it supports, ideally from a single test sketch edited in one place. PlatformIO fits this well: one `platformio.ini`, one environment per board, and a single `pio run` that builds them all. How the project is laid out on disk decides whether this is reliable or not. The workflow is the same for the Arduino and ESP-IDF frameworks: only a handful of settings change, and they are collected in one place below.

## What each tool is for

| Goal | Tool |
| :--- | :--- |
| One example on many boards, built with one command | PlatformIO, one environment per board (`framework = arduino` or `espidf`) |
| Read the framework's own source behind a call | VS Code "Go to Definition" |
| Native ESP-IDF exactly as its users run it, without touching the repository | `idf.py` with `-C`/`-B` |
| What Arduino IDE users get | Arduino IDE, repository in `libraries/` |

---

## The stable layout: the project lives *outside* the library folder

```text
workspace/
├── <library>/                 ← the library repository
│   ├── src/
│   └── examples/
│       ├── <arduino_example>/ ← the Arduino test sketch
│       └── <idf_example>/     ← the ESP-IDF test example
├── bench-arduino/             ← a PlatformIO project: only platformio.ini lives here
│   └── platformio.ini
└── bench-espidf/
    └── platformio.ini
```

A `platformio.ini` has a single `src_dir`, so an Arduino sketch and an ESP-IDF example each need their own project. Everything below applies to both:

```ini
[platformio]
; Take the project's sources straight from the library's own example:
; one source of truth, nothing to copy.
src_dir = <path-to-library>/examples/<example>

[env]
framework = arduino                  ; or espidf
monitor_speed = 115200
; Link (not copy) the library folder: edits are picked up by the next build.
lib_deps = symlink://<path-to-library>
```

- **`src_dir`** tells PlatformIO where the project's own sources are. Pointing it at the example folder makes the example the test project, and for an Arduino sketch PlatformIO converts the `.ino` into a `.cpp` at build time.
- **`lib_deps = symlink://…`** links the library folder instead of copying it, so a change in `src/` is used by the next build without reinstalling anything. (`file://` would copy a snapshot.)
- **Paths:** on Windows, `${sysenv.USERPROFILE}/…` keeps the file portable between machines; on Linux/macOS use `${sysenv.HOME}/…` or a relative path (not tested here).

Note what is, and is not, inside the library: `src_dir` still points *into* the library repository, but the **project directory** (the folder holding `platformio.ini` and the `.pio/` build tree) does not.

---

## One environment per board

The same sources are built for every board; only the per-environment settings change. The framework adds just a few differences:

| | `framework = arduino` | `framework = espidf` |
| :--- | :--- | :--- |
| **Boards** | AVR, ESP8266, ESP32 family | ESP32 family only |
| **Platform** | one per architecture | pinned, e.g. `espressif32@6.4.0` (IDF 5.1.1): without a pin, a platform update changes the IDF under your tests |
| **Native USB console** | `build_flags` (`-D ARDUINO_USB_CDC_ON_BOOT=1`, `-D ARDUINO_USB_MODE=1`) plus `monitor_dtr`/`monitor_rts` | `board_build.sdkconfig_options` (`CONFIG_ESP_CONSOLE_USB_CDC=y`, `CONFIG_ESP_CONSOLE_UART_NONE=y`) |
| **Serial monitor** | `monitor_speed` | `monitor_speed`, plus `monitor_filters = direct` to keep ESP-IDF's colored log output readable |
| **Dependencies** | `lib_deps` | `lib_deps` and the example's own `main/idf_component.yml` coexist, with some redundancy (see [Chapter 4](../chapters/04-examples.md)) |

**Arduino** (`bench-arduino/platformio.ini`, in addition to the common part above):

```ini
[env:mega2560]
platform = atmelavr
board = megaatmega2560

[env:nano328]
platform = atmelavr
board = nanoatmega328                     ; clones with the old bootloader: see `pio boards nano`

[env:nodemcu8266]
platform = espressif8266
board = nodemcuv2
build_flags = -D DATA_PIN=2 -D CLOCK_PIN=14   ; GPIO numbers: silkscreen D4 = GPIO2, D5 = GPIO14

[env:esp32dev]
platform = espressif32
board = esp32dev
build_flags = -D DATA_PIN=27 -D CLOCK_PIN=22

[env:esp32s3-native-usb]                  ; same board as the next one, other USB port
platform = espressif32
board = esp32-s3-devkitc-1
build_flags =
    -D ARDUINO_USB_CDC_ON_BOOT=1          ; Serial over the chip's native USB
    -D ARDUINO_USB_MODE=1
monitor_dtr = 1                           ; needed to see output after (re)connecting USB
monitor_rts = 1

[env:esp32s3-uart]                        ; USB-to-UART bridge port: no extra flags
platform = espressif32
board = esp32-s3-devkitc-1
```

**ESP-IDF** (`bench-espidf/platformio.ini`): the common part changes `framework`, pins the platform and adds the monitor filter, and the environments only need a board:

```ini
[env]
framework = espidf
platform = espressif32@6.4.0              ; IDF 5.1.1
monitor_filters = direct

[env:esp32dev]
board = esp32dev
build_flags = -D DATA_PIN=27 -D CLOCK_PIN=22

[env:esp32s3-native-usb]                  ; native USB console, set through sdkconfig
board = esp32-s3-devkitc-1
board_build.sdkconfig_options =
    CONFIG_ESP_CONSOLE_USB_CDC=y
    CONFIG_ESP_CONSOLE_UART_NONE=y

[env:esp32s3-uart]                        ; USB-to-UART bridge port
board = esp32-s3-devkitc-1
```

- **Pins and variants are macros.** The test project reads optional macros (here `DATA_PIN` and `CLOCK_PIN`) and falls back to defaults, so boards that need nothing special have no flags at all. Plain numbers work as long as the library takes pins as integers.
- **Pass GPIO numbers, not silkscreen labels.** NodeMCU's `D4` is GPIO2, not 4.
- **The native-USB settings** were required on the S3 and C3 boards to get serial output. The underlying reason was not investigated here.
- **One physical board can need two environments** (S3: native USB vs UART bridge): use one environment per wiring or per USB port, not per board model.

### Daily workflow

The commands are the same for both frameworks:

```bash
pio run                                  # build every environment: the regression check before a release
pio run -e esp32s3-native-usb            # build one board
pio run -e esp32s3-native-usb -t upload  # build and flash it
pio device monitor -e esp32s3-native-usb # serial monitor with that environment's settings
```

If more than one board is connected, set `upload_port` and `monitor_port` per environment.

---

## The unstable layout: the project lives *inside* the library folder

> [!WARNING]
> Keeping `platformio.ini` at the root of the library repository looks convenient, because a clone is enough to run the tests, but it produced non-deterministic build failures.

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

**What was observed:**

- A build stops partway because PlatformIO re-runs its own initialization while the build is in progress.
- In a multi-environment build (`pio run`), one environment fails with errors such as the following (in practice the ESP8266 one, after the AVR ones):

  ```log
  xtensa-lx106-elf-ar: .pio\build\nodemcu8266\FrameworkArduino\Crypto.cpp.o: No such file or directory
  *** [.pio\build\nodemcu8266\libFrameworkArduino.a] Error 1
  ```

- Building the failing environment alone sometimes succeeds, more often it fails midway: the behaviour is not deterministic.

**What fixed it:** moving the project directory out of the library folder, with nothing else changed. The library stayed where it was and the same example was used as `src_dir`.

> [!NOTE]
> **Working hypothesis (not verified in PlatformIO's source):** the build writes its own files (the `.pio/` tree, the converted `.ino.cpp`) into the same folder tree PlatformIO scans for library and project sources, notices those changes mid-build, and re-initializes.

**A second argument against this layout:** the `build_src_filter` above hard-codes the repository layout (two levels up, `src/*.cpp`) and needs manual updates whenever it changes, whereas with `lib_deps` the library resolves its own sources.

---

## Reading the framework source: "Go to Definition"

A practical advantage of PlatformIO is that `platformio.ini` tells VS Code which framework, platform and board you are working with. Selecting an environment in the status bar rebuilds the IntelliSense configuration (PlatformIO prints `Updating metadata for the vscode IDE…`), so "Go to Definition" (F12) on any call opens the **real source of that environment's framework**, under `~/.platformio/packages/`.

This is how the behaviour in [Appendix B](B-yield-watchdog.md) was checked, instead of relying on documentation or memory: what `yield()` does in the Arduino-ESP32 core, and what `vTaskDelay()` and `taskYIELD()` do in FreeRTOS under ESP-IDF.

- The same call opens **different code in different environments**: switch the environment, press F12 again.
- Some symbols open only a **declaration** in a header, and VS Code reports "Function definition not found" (this happened with `yield()`). The definition is then in a source file IntelliSense didn't index: search the framework's folder for it.

---

## Native ESP-IDF, out of tree

To test an ESP-IDF example the way an ESP-IDF user would, without leaving build files in the repository:

```powershell
idf.py -C "<path-to-library>\examples\<idf_example>" -B <output-folder> build
```

- `-C` selects the project directory (the example) and `-B` the build directory. Run the command from any folder: the example folder gets no `build/` tree.
- `-B` must be repeated on every command that uses the build: `idf.py -C … -B … -p <PORT> flash monitor`.
- Run it from an ESP-IDF terminal, with the environment exported.

> [!TIP]
> `-B` moves the build directory only. `dependencies.lock` still ends up next to the project, and `sdkconfig` and `managed_components/` may too. Check `git status` after the first build and ignore them in `.gitignore`.

---

## Arduino IDE

The Arduino IDE only looks in its own `libraries` folder, so the only way to test there is to **copy or move the whole repository** into `Documents\Arduino\libraries\`. The folder name doesn't matter, since a library's identity comes from its public headers (see [Chapter 1](../chapters/01-ecosystem-constraints.md)).

- Keep **one copy**, not two: you will end up editing one and testing the other.
- Restart the IDE if the library doesn't appear in the menu.

---

## Rules of thumb

- Keep the **project directory** (`platformio.ini`, `.pio/`) outside the library repository. `src_dir` may point inside it.
- Link the library with `symlink://`, not a copy.
- Run `pio run` (all environments) before every release, then flash and read the output of each board you have at hand.
- Use one environment per wiring or per USB port, not per board model.
- Use PlatformIO for many boards, `idf.py -C/-B` for the native build, and the Arduino IDE for what its users see.
- When the behaviour of a framework matters, open its source with F12 instead of trusting memory.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Appendix C - Naming Convention](C-naming-convention.md) — [Introduction](../README.md)

</div>
<!-- markdownlint-enable MD033 -->