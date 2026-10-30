# Testing During Development with PlatformIO

Before anything is published, the library has to run on every supported board, ideally from a single test sketch edited in one place. PlatformIO fits this well: one `platformio.ini`, one environment per board, and a single `pio run` that builds them all. How the project is laid out on disk decides whether this is reliable or not.

## D.1 What each tool is for

| Goal | Tool | Section |
| :--- | :--- | :--- |
| One sketch on many Arduino boards (AVR, ESP8266, ESP32) | PlatformIO, `framework = arduino` | D.2 |
| The ESP32 family on native ESP-IDF, one environment per board | PlatformIO, `framework = espidf` | D.3 |
| Read the framework's own source behind a call | VS Code "Go to Definition" | D.4 |
| Native ESP-IDF exactly as its users run it, without touching the repository | `idf.py` with `-C`/`-B` | D.5 |
| What Arduino IDE users get | Arduino IDE, repository in `libraries/` | D.6 |

## D.2.1 The stable layout: the project lives *outside* the library folder

```text
workspace/
├── cs123x/                    ← the library repository
│   ├── src/
│   └── examples/
│       └── TestExhaustive/    ← the test sketch
└── bench/                     ← the PlatformIO project: only platformio.ini lives here
    └── platformio.ini
```

```ini
[platformio]
; Take the project's sketch straight from the library's own example:
; one source of truth, nothing to copy.
src_dir = ${sysenv.USERPROFILE}/Documents/VSCode/cs123x/examples/TestExhaustive

[env]
framework = arduino
monitor_speed = 115200
; Link (not copy) the library folder: edits are picked up by the next build.
lib_deps = symlink://${sysenv.USERPROFILE}/Documents/VSCode/cs123x
```

- **`src_dir`** tells PlatformIO where the project's own sources are. Pointing it at the example folder makes the example the test sketch. PlatformIO converts the `.ino` into a `.cpp` at build time.
- **`lib_deps = symlink://…`** links the library folder instead of copying it, so a change in `src/` is used by the next build without reinstalling anything. (`file://` would copy a snapshot.)
- `${sysenv.USERPROFILE}` is a Windows variable. On Linux/macOS use `${sysenv.HOME}` or a relative path (not tested here).

Note what is, and is not, inside the library: `src_dir` still points *into* the library repository, but the **project directory** (the folder holding `platformio.ini` and the `.pio/` build tree) does not. That distinction is what D.4 is about.

## D.2.2 One environment per board

The same sketch is built for every board; only the per-environment flags change:

```ini
[env:arduino-mega2560]
platform = atmelavr
board = megaatmega2560

[env:arduino-nano328]
platform = atmelavr
board = nanoatmega328            ; clones with the old bootloader (see `pio boards nano`)

[env:nodemcu8266]
platform = espressif8266
board = nodemcuv2
build_flags = -D DOUT_PIN=2 -D SCLK_PIN=14   ; GPIO numbers: silkscreen D4 = GPIO2, D5 = GPIO14

[env:esp32c3supermini]
platform = espressif32
board = esp32-c3-devkitm-1
build_flags =
    -D ARDUINO_USB_CDC_ON_BOOT=1             ; Serial over the chip's native USB
    -D ARDUINO_USB_MODE=1
monitor_dtr = 1                              ; needed to see output after (re)connecting USB
monitor_rts = 1

[env:esp32s3native-usb]                      ; same board as the next one, other USB port
platform = espressif32
board = esp32-s3-devkitc-1
build_flags =
    -D ARDUINO_USB_CDC_ON_BOOT=1
    -D ARDUINO_USB_MODE=1
monitor_dtr = 1
monitor_rts = 1

[env:esp32s3uart]                            ; USB-to-UART bridge port: no extra flags
platform = espressif32
board = esp32-s3-devkitc-1

[env:cyd]
platform = espressif32
board = esp32dev
build_flags = -D DOUT_PIN=27 -D SCLK_PIN=22
; Same board with a CS1238 module, wired differently:
;build_flags = -D CS_TYPE=CS123X_TYPE_CS1238 -D DOUT_PIN=22 -D SCLK_PIN=27
```

- **Pins and chip type are macros.** The sketch reads optional `DOUT_PIN`, `SCLK_PIN` and `CS_TYPE` and falls back to defaults, so boards that need nothing special (AVR, `nodemcu32`) have no flags at all.
- **Pass GPIO numbers, not silkscreen labels.** NodeMCU's `D4` is GPIO2, not 4.
- **The native-USB flags and `monitor_dtr`/`monitor_rts`** were required on the S3 and C3 boards to get Serial output. The underlying reason was not investigated here.
- **One physical board can need two environments** (S3: native USB vs UART bridge).

## D.2.3 Daily workflow

```bash
pio run                                  # build every environment: the regression check before a release
pio run -e esp32s3native-usb             # build one board
pio run -e esp32s3native-usb -t upload   # build and flash it
pio device monitor -e esp32s3native-usb  # serial monitor with that environment's settings
```

If more than one board is connected, set `upload_port` and `monitor_port` per environment.

## D.2.4 The unstable layout: the project lives *inside* the library folder

Keeping `platformio.ini` at the root of the library repository looks convenient, because everything is in one place and a clone is enough to run the tests:

```ini
; UNSTABLE — the project directory is the library repository root
[platformio]
src_dir = examples/TestExhaustive
include_dir = src                     ; make the library headers visible

[env]
build_src_filter =
    +<*>                              ; every file in src_dir (.ino files become .ino.cpp)
    +<../../src/*.cpp>                ; library sources, two levels up from src_dir
```

**What was observed:**

- A build stops partway because PlatformIO re-runs its own initialization while the build is in progress.
- In a multi-environment build (`pio run`), one environment fails with errors such as:

  ```log
  xtensa-lx106-elf-ar: .pio\build\nodemcu8266\FrameworkArduino\Crypto.cpp.o: No such file or directory
  *** [.pio\build\nodemcu8266\libFrameworkArduino.a] Error 1
  ```
  
  In practice it was the ESP8266 environment, after the AVR ones.
- Building the failing environment alone sometimes succeeds, more often it fails midway: the behaviour is not deterministic.

**What fixed it:** moving the project directory out of the library folder (D.1), with nothing else changed. The library stayed where it was and the same example was used as `src_dir`.

**Working hypothesis (not verified in PlatformIO's source):** the build writes its own files (the `.pio/` tree, the converted `.ino.cpp`) into the same folder tree PlatformIO scans for library and project sources. It notices those changes mid-build and re-initializes.

**A second argument against this layout:** the `build_src_filter` above hard-codes the repository layout (two levels up, `src/*.cpp`). It needs manual updates whenever that layout changes, whereas with `lib_deps` the library resolves its own sources.

## D.3 PlatformIO with the ESP-IDF framework

The idea carries over from D.2: one environment per board, the project directory outside the library repository (D.2.1), the example folder as `src_dir`, the library linked with `symlink://`. Only the ESP32 family appears here, because ESP-IDF has no AVR or ESP8266 targets.

```ini
[platformio]
src_dir = ${sysenv.USERPROFILE}/Documents/VSCode/cs123x/examples/test_gaussian

[env]
framework = espidf
platform = espressif32@6.4.0              ; IDF 5.1.1 / CMake 3.16.4
lib_deps = symlink://${sysenv.USERPROFILE}/Documents/VSCode/cs123x
monitor_speed = 115200
monitor_filters = direct                  ; keeps ESP-IDF's colored log output readable

[env:nodemcu32]
board = nodemcu-32s

[env:esp32s3native-usb]                   ; native USB console, set through sdkconfig
board = esp32-s3-devkitc-1
board_build.sdkconfig_options =
    CONFIG_ESP_CONSOLE_USB_CDC=y
    CONFIG_ESP_CONSOLE_UART_NONE=y

[env:esp32s3uart]                         ; USB-to-UART bridge port
board = esp32-s3-devkitc-1

[env:cyd]
board = esp32dev
build_flags = -D DOUT_PIN=27 -D SCLK_PIN=22
```

- **Pin the platform.** `espressif32@6.4.0` fixes the ESP-IDF version (5.1.1). Without a pin, a platform update changes the IDF under your tests.
- **Pins and chip type** use the same macros as in D.2.2, and plain numbers work (`-D DOUT_PIN=27`) because the constructor takes the pin as an integer.
- **The USB console** is selected in `sdkconfig`: the Arduino flags of D.2.2 (`ARDUINO_USB_CDC_ON_BOOT`, …) don't exist here.
- **Dependencies.** The example's own `main/idf_component.yml` and the `lib_deps` entry are both present and coexist without conflict (Chapter 4, Appendix A). `REQUIRES` is not affected by the environment.
- Commands are the same as in D.2.3: `pio run`, `pio run -e <env> -t upload`, `pio device monitor -e <env>`.

## D.4 Reading the framework source: "Go to Definition"

A practical advantage of PlatformIO is that `platformio.ini` tells VS Code which framework, platform and board you are working with. Selecting an environment in the status bar rebuilds the IntelliSense configuration (PlatformIO prints `Updating metadata for the vscode IDE…`), so "Go to Definition" (F12) on any call opens the **real source of that environment's framework**, under `~/.platformio/packages/`.

This is how the behaviour in Appendix B was checked, instead of relying on documentation or memory: what `yield()` does in the Arduino-ESP32 core, how the ESP8266 core implements its interrupt lock (`esp8266::InterruptLock`), and what `vTaskDelay()` and `taskYIELD()` do in FreeRTOS under ESP-IDF.

- The same call opens **different code in different environments**: switch the environment, press F12 again.
- Some symbols open only a **declaration** in a header, and VS Code reports "Function definition not found" (this happened with `yield()`). The definition is then in a source file IntelliSense didn't index: search the framework's folder for it.

## D.5 Native ESP-IDF, out of tree

To test the example the way an ESP-IDF user would, without leaving build files in the repository:

```powershell
idf.py -C "<path-to-library>\examples\test_exhaustive" -B <output-folder> build
```

- `-C` selects the project directory (the example) and `-B` the build directory. Run the command from any folder: the example folder gets no `build/` tree.
- `-B` must be repeated on every command that uses the build: `idf.py -C … -B … -p <PORT> flash monitor`.
- Run it from an ESP-IDF terminal, with the environment exported.
- `-B` moves the build directory only. `dependencies.lock` still ends up next to the project (seen in earlier logs), and `sdkconfig` and `managed_components/` may too. Check `git status` after the first build and ignore them in `.gitignore`.

## D.6 Arduino IDE

The Arduino IDE only looks in its own `libraries` folder, so the only way to test there is to **copy or move the whole repository** into `Documents\Arduino\libraries\`. The folder name didn't matter in testing: the library compiled with its folder renamed to lowercase.

- Keep **one copy**, not two: you will end up editing one and testing the other.
- Restart the IDE if the library doesn't appear in the menu.

## D.7 Rules of thumb

- Keep the **project directory** (`platformio.ini`, `.pio/`) outside the library repository. `src_dir` may point inside it.
- Link the library with `symlink://`, not a copy.
- Run `pio run` (all environments) before every release, then flash and read the output of each board you have at hand.
- Use one environment per wiring or per USB port, not per board model.
- Use PlatformIO for many boards, `idf.py -C/-B` for the native build, and the Arduino IDE for what its users see.
- When behaviour of a framework matters, open its source with F12 instead of trusting memory.
