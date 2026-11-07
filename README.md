# Dual-Framework Library Guide

**Native libraries for Arduino & ESP-IDF**: a practical guide to architecture, repository layout, and publishing for ESP32 libraries that run natively on both ecosystems.

**Who it's for:** developers who want to write libraries that work natively on both Arduino and ESP-IDF, starting from either ecosystem.

## Table of Contents

- **Chapters:**
  - [Chapter 1 - Ecosystem Constraints](chapters/01-ecosystem-constraints.md)
  - [Chapter 2 - Repository Structure](chapters/02-repo-structure.md)
  - [Chapter 3 - Agnostic Core & HAL Architecture](chapters/03-agnostic-core-hal-architecture.md)
  - [Chapter 4 - Example Conventions](chapters/04-examples.md)
  - [Chapter 5 - Publishing](chapters/05-publishing.md)
  - [Chapter 6 - Conclusions](chapters/06-conclusions.md)
  - [Final Checklist](chapters/07-final-checklist.md)

- **Appendices:**
  - [Appendix A - IDF Dependency Resolution](appendices/A-esp-idf-dependency-resolution.md)
  - [Appendix B - Yield & Watchdog](appendices/B-yield-watchdog.md)
  - [Appendix C - Naming Convention](appendices/C-naming-convention.md)
  - [Appendix D - Testing During Development](appendices/D-testing-during-development.md)

- **Licenses:**
  - [LICENSE](LICENSE)
  - [LICENSE-CODE](LICENSE-CODE)

---

## Why build a library that's native to both Arduino & ESP-IDF

Before diving into the technical constraints, it's worth answering a more fundamental question: **why accept the complexity of supporting both Arduino and ESP-IDF natively at all? Why not simply choose one ecosystem, or rely on Arduino-as-a-component as a compatibility layer?**

> **Spoiler:** the final solution is far less complicated than this guide might suggest. Most of the discussion that follows is not about complexity, but about explaining the constraints, trade-offs, and reasoning that led to each architectural choice.

---

## 🚪 Arduino is still the front door

**Arduino is, and will remain for a long time, the primary entry point into embedded development.**

Its greatest strength is not raw technical capability, but **accessibility.** Projects can be initialized in seconds, the API is intentionally approachable, and compiling and flashing firmware is often reduced to a single click. Beginners, hobbyists, educators, and even experienced developers building quick prototypes.
This reduction in friction is not a secondary advantage: it is the very reason the platform became dominant and continues to attract the largest community in the embedded world.

**Ignoring Arduino means ignoring where most users begin.**

---

## ⚙️ ESP-IDF is becoming the destination

At the same time, ESP-IDF is increasingly establishing itself as a reference platform for ESP32 development, not only for greenfield professional projects, but also for projects that originally grew within the Arduino ecosystem itself.

A concrete signal is [ESPHome's migration toward ESP-IDF](https://esphome.io/guides/esp32_arduino_to_idf): a project born and grown within the Arduino ecosystem that chose to move its engine onto native ESP-IDF.

There is also a technical reality that is often underestimated:

<!-- markdownlint-disable MD033 -->
<p align="center">
  <i><b>when you write Arduino code for an ESP32 board, you're already running on top of ESP-IDF.</b></i>
</p>
<!-- markdownlint-enable MD033 -->

[`arduino-esp32`](https://docs.espressif.com/projects/arduino-esp32/en/latest/index.html) is itself a layer built on top of ESP-IDF as a component, not an independent alternative. The question isn't "Arduino or IDF?" but rather:

***Do you want to interact directly with the platform's engine, or through an additional translation layer?***

---

## 🔄 Compatibility layers solve portability, not dependency

The [Arduino as an ESP-IDF component](https://docs.espressif.com/projects/arduino-esp32/en/latest/esp-idf_component.html) compatibility layer lets you run an existing Arduino sketch inside an IDF project with minimal effort, preserving compatibility with a vast amount of existing code. It's a valid trade-off for reusing code without rewriting it, but it isn't free: application code still goes through *Arduino APIs* such as `digitalWrite` and `pinMode`, which are then translated into **ESP-IDF** calls.

The per-call overhead is often modest. The real price is architectural: **a library tied to the Arduino core forces that entire framework into every native IDF project that uses it.**

---

## 🧩 A HAL addresses the problem at the source

Designing with a Hardware Abstraction Layer (HAL) from the start solves the problem at its root.

Rather than choosing one ecosystem and translating toward the other, the application layer remains completely independent of both. This also removes the after-the-fact work described in Espressif's guide on [Porting a library to an ESP-IDF component](https://developer.espressif.com/blog/2025/10/porting-external-library-as-component/): there is nothing to port, because the library is native to both ecosystems from the start. Protocol handling, algorithms, state management, calculations, and business logic stay **framework-agnostic**. Only the lowest layer, the one that actually touches the hardware resources, changes according to the selected framework.

**Most importantly**, that choice is resolved at **compile time**, not at runtime (see [Chapter 3](chapters/03-agnostic-core-hal-architecture.md)). The result is a single codebase where each framework keeps its native implementation path, with no runtime switches or translation layers: *native performance on both sides, with application logic written and maintained once.*

---

## 📚 The purpose of this repository

This guide documents the architecture, constraints, and practical decisions required to make that approach work in a real-world library. It covers everything from the non-negotiable requirements of each ecosystem, to repository organization, build integration, and publication across all major distribution channels:

- [Arduino Library Manager](https://docs.arduino.cc/libraries/)
- [ESP Component Registry](https://components.espressif.com/)
- [PlatformIO Registry](https://registry.platformio.org/)

**Addressing this architectural choice right from the start helps avoid the need to adapt or refactor the library at a later stage.**

> [!NOTE]
> **Reference implementation:** this guide comes from the practical experience of developing my [CS123x](https://github.com/FMazz97/CS123x) library. It follows almost every pattern, layout, and configuration described in these chapters. A few differences remain, as the guide evolved beyond the original implementation through additional research and experimentation across both frameworks.
>
> **Tested with:** ESP-IDF v6.1 (native), PlatformIO `espressif32@6.4.0` (IDF 5.1.1), Arduino IDE 2.3.x.
> Registry/tool behavior observed in September 2026; it may change.
<!-- markdownlint-disable MD028 -->
<!-- markdownlint-enable MD028 -->
> [!WARNING]
> **The guide is based on a C++ class shared by both frameworks.** The previously mentioned [CS123x](https://github.com/FMazz97/CS123x) library, was born for Arduino and then adapted to ESP-IDF quickly, by adding only the HAL and the Arduino facade. This approach sits halfway between using the [Arduino-ESP32](https://docs.espressif.com/projects/arduino-esp32/en/latest/index.html) component — which brings the whole framework along — and rewriting the core in C with ESP-IDF convention.
> In the future, the guide may be updated to follow the path of a C core for ESP-IDF — with error handling through `esp_err_t`, if possible — and a C++ facade class for Arduino.

---

## License

This guide is dual-licensed:

- **Text** (prose, tables, diagrams): [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/), see [LICENSE](LICENSE).
- **Code** (every fenced code block and snippet): [MIT License](https://mit-license.org/), see [LICENSE-CODE](LICENSE-CODE).

[![CC BY 4.0](https://licensebuttons.net/l/by/4.0/88x31.png)](https://creativecommons.org/licenses/by/4.0/)

<!-- markdownlint-disable MD034 -->
If you reuse or adapt the text, give credit as: *"Dual-Framework Library Guide" by FMazz97, https://github.com/FMazz97/arduino-espidf-dual-library-guide*, and indicate if changes were made. Code snippets can be reused under the MIT terms, keeping the copyright notice.
<!-- markdownlint-enable MD034 -->

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[Chapter 1 - Ecosystem Constraints ➡️](chapters/01-ecosystem-constraints.md)

</div>
<!-- markdownlint-enable MD033 -->
