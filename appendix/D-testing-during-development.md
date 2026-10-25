# Testing During Development with PlatformIO

Before anything is published, the library has to run on every supported board, ideally from a single test sketch edited in one place. PlatformIO fits this well: one `platformio.ini`, one environment per board, and a single `pio run` that builds them all. How the project is laid out on disk decides whether this is reliable or not.

## D.1 The stable layout: the project lives *outside* the library folder

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

## D.2 One environment per board

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

## D.3 Daily workflow

```bash
pio run                                  # build every environment: the regression check before a release
pio run -e esp32s3native-usb             # build one board
pio run -e esp32s3native-usb -t upload   # build and flash it
pio device monitor -e esp32s3native-usb  # serial monitor with that environment's settings
```

If more than one board is connected, set `upload_port` and `monitor_port` per environment.

## D.4 The unstable layout: the project lives *inside* the library folder

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

## D.5 Rules of thumb

- Keep the **project directory** (`platformio.ini` + `.pio/`) separate from the library repository. `src_dir` may point inside the library.
- Link the library with `symlink://`, not a copy.
- Run `pio run` (all environments) before every release, then flash and read the output of each board you have at hand.
- Use one environment per wiring or per USB port, not per board model.
