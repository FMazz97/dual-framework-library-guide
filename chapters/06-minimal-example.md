# Chapter 6 — A Minimal Example

This chapter puts the guide together in the smallest library that follows its rules: a `DualBlinkCpp` class that blinks one pin without blocking, through a compile-time HAL shared by Arduino and ESP-IDF. The buildable version lives in the [DualBlinkCpp - v1.1.1](https://github.com/FMazz97/DualBlinkCpp) repository, and the code below is a copy at the same tag, without the Doxygen comments (see them for complete details). The `Cpp` suffix keeps the plain `DualBlink` name free for a future variant.

> **Tested with:** Arduino IDE 2.3.x, PlatformIO (`framework = arduino` and `framework = espidf`, `espressif32@6.4.0` with IDF 5.1.1) and native ESP-IDF v6.1, on an Arduino Nano and on an ESP32 (`esp32dev`) board.

## 6.1 Repository tree

The repository is named `DualBlinkCpp` on GitHub (the Arduino library name, `PascalCase`) and cloned into a lowercase folder, which becomes the component name (see [Chapter 2](02-repo-structure.md#22-local-folder-name-vs-github-repository-name)):

```bash
git clone https://github.com/FMazz97/DualBlinkCpp.git dual_blink_cpp
```

```text
dual_blink_cpp/                      ← repository root = component root
├── CMakeLists.txt                   ← ESP-IDF
├── idf_component.yml                ← ESP-IDF
├── library.properties               ← Arduino
├── library.json                     ← PlatformIO
├── src/
│   ├── DualBlinkCpp.h               ← Arduino facade
│   ├── core/                        ← framework-agnostic core
│   │   ├── dual_blink_cpp.h
│   │   └── dual_blink_cpp.cpp
│   └── hal/                         ← Hardware Abstraction Layer
│       ├── dual_blink_cpp_hal_arduino.h
│       └── dual_blink_cpp_hal_espidf.h
└── examples/
    ├── BasicExample/                ← Arduino sketch
    └── basic_example/               ← ESP-IDF project
```

> [!NOTE]
> The repository also contains the files that check that the versions declared in the manifests are aligned. They are not part of the library and are left out of the tree above: see [Appendix E - Check Manifest Versions](../appendices/E-manifest-version-check.md).

---

## 6.2 The five source files

### 6.2.1 `src/hal/dual_blink_cpp_hal_arduino.h`

```cpp
#ifndef DUAL_BLINK_CPP_HAL_ARDUINO_H
#define DUAL_BLINK_CPP_HAL_ARDUINO_H

#include <Arduino.h>

#define DUAL_BLINK_CPP_PIN_OUTPUT()       pinMode(_pin, OUTPUT)
#define DUAL_BLINK_CPP_PIN_WRITE(level)   digitalWrite(_pin, level)
#define DUAL_BLINK_CPP_PIN_READ()         digitalRead(_pin)
#define DUAL_BLINK_CPP_MILLIS()           (static_cast<uint32_t>(millis()))

#endif /* DUAL_BLINK_CPP_HAL_ARDUINO_H */
```

### 6.2.2 `src/hal/dual_blink_cpp_hal_espidf.h`

```cpp
#ifndef DUAL_BLINK_CPP_HAL_ESPIDF_H
#define DUAL_BLINK_CPP_HAL_ESPIDF_H

#include "driver/gpio.h"
#include "esp_timer.h"

#define DUAL_BLINK_CPP_PIN_OUTPUT()       gpio_set_direction(static_cast<gpio_num_t>(_pin), GPIO_MODE_INPUT_OUTPUT)
#define DUAL_BLINK_CPP_PIN_WRITE(level)   gpio_set_level(static_cast<gpio_num_t>(_pin), level)
#define DUAL_BLINK_CPP_PIN_READ()         gpio_get_level(static_cast<gpio_num_t>(_pin))
#define DUAL_BLINK_CPP_MILLIS()           (static_cast<uint32_t>(esp_timer_get_time() / 1000))

#endif /* DUAL_BLINK_CPP_HAL_ESPIDF_H */
```

The two HALs expose the same three macros, which expand directly inside the core methods and rely on its `_pin` member ([Chapter 3](03-agnostic-core-hal-architecture.md)). `millis()` on Arduino and `esp_timer_get_time()` on ESP-IDF are the equivalent clocks: the blink never blocks.

### 6.2.3 `src/core/dual_blink_cpp.h`

```cpp
#ifndef DUAL_BLINK_CPP_CORE_H
#define DUAL_BLINK_CPP_CORE_H

#include <stdint.h>

#if defined(ARDUINO)
  #include "../hal/dual_blink_cpp_hal_arduino.h"
#elif defined(ESP_PLATFORM)
  #include "../hal/dual_blink_cpp_hal_espidf.h"
#else
  #error "DualBlinkCpp core: unsupported platform (expected ARDUINO or ESP_PLATFORM)"
#endif

class dual_blink_cpp {
  public:
    explicit dual_blink_cpp(int pin, uint32_t interval_ms = 500);

    bool begin();
    bool update();

    bool set_level(bool level);
    bool get_level() const;

    void set_interval_ms(uint32_t interval_ms);
    uint32_t get_interval_ms() const;

  private:
    bool read_level();
    void write_level(bool level);

    int      _pin;
    uint32_t _interval_ms;
    uint32_t _last_toggle_ms;
    bool     _level;
};

#endif /* DUAL_BLINK_CPP_CORE_H */
```

`ARDUINO` is tested first because Arduino-ESP32 defines both macros. The methods follow the prefix scheme of [Appendix C](../appendices/C-naming-convention.md): `set_level()`/`get_level()` and `set_interval_ms()`/`get_interval_ms()` are pairs, and the getters are `const` and never touch the hardware. `begin()` and `update()` are domain vocabulary, outside the scheme.

### 6.2.4 `src/core/dual_blink_cpp.cpp`

```cpp
#include "dual_blink_cpp.h"

dual_blink_cpp::dual_blink_cpp(int pin, uint32_t interval_ms)
    : _pin(pin), _interval_ms(interval_ms), _last_toggle_ms(0), _level(false) {}

bool dual_blink_cpp::begin() {
    DUAL_BLINK_CPP_PIN_OUTPUT();
    bool confirmed = set_level(false);
    _last_toggle_ms = DUAL_BLINK_CPP_MILLIS();
    return confirmed;
}

bool dual_blink_cpp::update() {
    uint32_t now = DUAL_BLINK_CPP_MILLIS();
    if (now - _last_toggle_ms >= _interval_ms) {
        _last_toggle_ms = now;
        return set_level(!_level);
    }
    return true;
}

bool dual_blink_cpp::set_level(bool level) {
    write_level(level);
    _level = read_level();
    return _level == level;
}

bool dual_blink_cpp::get_level() const {
    return _level;
}

void dual_blink_cpp::set_interval_ms(uint32_t interval_ms) {
    _interval_ms = interval_ms;
}

uint32_t dual_blink_cpp::get_interval_ms() const {
    return _interval_ms;
}

bool dual_blink_cpp::read_level() {
    return DUAL_BLINK_CPP_PIN_READ() != 0;
}

void dual_blink_cpp::write_level(bool level) {
    DUAL_BLINK_CPP_PIN_WRITE(level);
}
```

The core never calls an Arduino or ESP-IDF API directly: only the `DUAL_BLINK_CPP_*` macros. The unsigned subtraction `now - _last_toggle_ms` stays correct when the millisecond counter wraps around.

### 6.2.5 `src/DualBlinkCpp.h`

```cpp
#ifndef DUAL_BLINK_CPP_H
#define DUAL_BLINK_CPP_H

#include "core/dual_blink_cpp.h"

class DualBlinkCpp {
  private:
    dual_blink_cpp _core;

  public:
    DualBlinkCpp(int pin, uint32_t intervalMs = 500) : _core(pin, intervalMs) {}

    bool begin()                        { _core.begin(); }
    bool update()                       { _core.update(); }
    bool setLevel(bool level)           { _core.set_level(level); }
    bool getLevel() const               { return _core.get_level(); }
    void setIntervalMs(uint32_t ms)     { _core.set_interval_ms(ms); }
    uint32_t getIntervalMs() const      { return _core.get_interval_ms(); }
};

#endif /* DUAL_BLINK_CPP_H */
```

> [!TIP]
> The facade and the core headers use **different include guards** (`DUAL_BLINK_CPP_H` and `DUAL_BLINK_CPP_CORE_H`). The facade includes the core, so with the same guard the core header would be silently skipped.

---

## 6.3 The manifests

### 6.3.1 `library.properties` (Arduino)

```properties
name=DualBlinkCpp
version=1.1.1
author=FMazz97
maintainer=FMazz97
sentence=Non-blocking blink, native to Arduino and ESP-IDF.
paragraph=Minimal example of a C++ library with a compile-time HAL shared by both frameworks.
category=Other
url=https://github.com/FMazz97/DualBlinkCpp
architectures=*
includes=DualBlinkCpp.h
```

### 6.3.2 `library.json` (PlatformIO)

```json
{
    "name": "dual_blink_cpp",
    "version": "1.1.1",
    "description": "Minimal example of a C++ library with a compile-time HAL shared by both frameworks, implementing a non-blocking blink, native to Arduino and ESP-IDF",
    "keywords": ["blink", "hal", "example"],
    "homepage": "https://github.com/FMazz97/DualBlinkCpp",
    "repository": {
        "type": "git",
        "url": "https://github.com/FMazz97/DualBlinkCpp.git"
    },
    "authors": [{"name": "FMazz97", "url": "https://github.com/FMazz97", "maintainer": true}],
    "license": "MIT",
    "frameworks": ["arduino", "espidf"],
    "platforms": "*",
    "build": {
        "flags": [
            "-I src/core"
        ]
    },
    "export": {
        "exclude": [
            "CMakeLists.txt",
            "idf_component.yml",
            "library.properties",
            "examples/*/main/idf_component.yml",
            ".gitattributes",
            ".githooks/*",
            ".github/*",
            "scripts/*"
        ]
    },
    "examples": [
        {
            "name": "Arduino_BasicExample",
            "base": "examples/BasicExample",
            "files": ["BasicExample.ino", "platformio.ini"]
        },
        {
            "name": "ESP-IDF_basic_example",
            "base": "examples/basic_example",
            "files": ["main/main.cpp", "main/CMakeLists.txt", "CMakeLists.txt", "platformio.ini"]
        }
    ]
}
```

The `"build"` flag exposes `src/core/` to PlatformIO, as explained in [Chapter 2](02-repo-structure.md#213-platformio-source--include-path).

### 6.3.3 `idf_component.yml` (ESP-IDF)

```yaml
version: "1.1.1"
description: Minimal example of a C++ library with a compile-time HAL shared by both frameworks, implementing a non-blocking blink, native to Arduino and ESP-IDF
url: https://github.com/FMazz97/DualBlinkCpp
repository: https://github.com/FMazz97/DualBlinkCpp.git
documentation: "https://github.com/FMazz97/DualBlinkCpp#readme"
issues: "https://github.com/FMazz97/DualBlinkCpp/issues"
maintainers:
  - "FMazz97"

dependencies:
  idf: ">=5.0"

examples:
- path: ./examples/basic_example

files:
  exclude:
    - "library.properties"
    - "library.json"
    - "src/DualBlinkCpp.h"                      # Arduino's facade *
    - "src/hal/dual_blink_cpp_hal_arduino.h"    # Arduino's HAL *
    - ".gitattributes"
    - ".githooks/*"
    - ".github/*"
    - "scripts/*"
```

The entries marked with `*` are optional: leave them out of the exclusions to support Arduino as an ESP-IDF component ([Chapter 5](05-publishing.md#521-manifest-and-metadata)). Add the `examples` key when the repository has examples.

### 6.3.4 `CMakeLists.txt` (ESP-IDF)

```cmake
# Select dependencies based on ESP-IDF version
if(${IDF_VERSION_MAJOR} STREQUAL 5 AND ${IDF_VERSION_MINOR} LESS 3)
    # Use driver component as esp_driver_gpio is not available before 5.3
    set(req driver esp_timer)
else()
    set(req esp_driver_gpio esp_timer)
endif()

idf_component_register(
    SRCS
        "src/core/dual_blink_cpp.cpp"
    INCLUDE_DIRS
        "src/core"
    REQUIRES
        ${req}
)
```

> [!NOTE]
> The components are declared in `REQUIRES`, not `PRIV_REQUIRES`: the public header `dual_blink_cpp.h` includes the ESP-IDF HAL, which includes `driver/gpio.h` and `esp_timer.h`, so consumers of the component need them too.

---

## 6.4 Using the library

**Arduino sketch** (`examples/BasicExample/BasicExample.ino`):

```cpp
#include <Arduino.h>

#include <DualBlinkCpp.h>

// Test pin configuration (adjust according to your hardware setup)
#ifndef BLINK_PIN
#define BLINK_PIN 4
#endif

DualBlinkCpp led(BLINK_PIN, 500);

void setup() {
  led.begin();
}

void loop() {
  led.update();
}
```

**ESP-IDF** (`examples/basic_example/main/main.cpp`):

```cpp
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "dual_blink_cpp.h"

// Test pin configuration (adjust according to your hardware setup)
#ifndef BLINK_PIN
#define BLINK_PIN 4
#endif

extern "C" void app_main(void) {
    dual_blink_cpp led(BLINK_PIN, 500);
    led.begin();

    while (true) {
        led.update();
        vTaskDelay(1);  // lets the IDLE task run, see Appendix B at https://github.com/FMazz97/dual-framework-library-guide
    }
}
```

On ESP-IDF a polling loop must hand control back to the scheduler, with `vTaskDelay(1)` and not `taskYIELD()` ([Appendix B](../appendices/B-yield-interrupts.md)). The example layouts, the `platformio.ini` and the dependency declarations are described in [Chapter 4](04-examples.md).

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 5 - Publishing](05-publishing.md) — [Introduction](../README.md) — [Chapter 7 - Conclusions ➡️](07-conclusions.md)

</div>
<!-- markdownlint-enable MD033 -->
