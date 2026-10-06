# Why build a library that's native to both Arduino & ESP-IDF ecosystems

Before diving into the technical constraints, it's worth answering a more fundamental question: **why accept the complexity of supporting both Arduino and ESP-IDF natively at all?** Why not simply choose one ecosystem, or rely on Arduino-as-a-component as a compatibility layer?

>**Spoiler:** the final solution is far less complicated than this guide might suggest. Most of the discussion that follows is not about complexity, but about explaining the constraints, trade-offs, and reasoning that led to each architectural choice.

---

## 🚪 Arduino is still the front door

**Arduino remains, and will remain for a long time, the primary entry point into embedded development.**

**Its greatest strength is not raw technical capability, but accessibility.** Projects can be initialized in seconds, the API is intentionally approachable, and compiling and flashing firmware is often reduced to a single click. For beginners, hobbyists, educators, and even experienced developers building quick prototypes, **this reduction in friction is not a secondary advantage**: it is the very reason the platform became dominant and continues to attract the largest community in the embedded world.

**Ignoring Arduino means ignoring where most users begin.**

---

## ⚙️ ESP-IDF is becoming the destination

At the same time, ESP-IDF is increasingly establishing itself as a reference platform ESP32 development, not only for greenfield professional projects, but also for projects that originally grew within the Arduino ecosystem itself.

A concrete signal is [ESPHome's migration toward ESP-IDF](https://esphome.io/guides/esp32_arduino_to_idf) where a project born and grown within the Arduino ecosystem chose to move its engine onto native IDF.

There is also a technical reality that is often underestimated:
<!-- markdownlint-disable MD033 -->
<p align="center">
  <i><b>when you write for Arduino code for an ESP32 board, you're already running on top of ESP-IDF.</b></i>
</p>
<!-- markdownlint-disable MD033 -->

The `arduino-esp32` is itself a layer built on top of ESP-IDF, not an independent alternative. The question isn't "Arduino or IDF?" but rather:
***Do you want to interact directly with the platform's engine, or through an additional translation layer?***

---

## 🔄 Compatibility layers solve portability

The [Arduino as an ESP-IDF component](https://docs.espressif.com/projects/arduino-esp32/en/latest/esp-idf_component.html) compatibility layer isn't free. It lets you run an existing Arduino sketch inside an IDF project with minimal effort and preserves compatibility with a vast amount of existing code, but it doesn't eliminate the double layer of abstraction — it just relocates it.

Application code still goes through the same *Arduino APIs* such as `digitalWrite`, `pinMode`, etc., which are then translated into **ESP-IDF** calls. It's a valid trade-off for reusing existing code without rewriting it, but it's not a zero-cost solution: the back-and-forth path between the two layers remains, with its corresponding overhead and architectural constraints.

---

## 🧩 A HAL addresses the problem at the source

Designing with a Hardware Abstraction Layer (HAL) from the start solves the problem at its root. Rather than choosing one ecosystem and translating toward the other, the application layer remains completely independent of both. Protocol handling, algorithms, state management, calculations, and business logic are implemented once and remain **framework-agnostic**. Only the lowest layer, the one that actually touches the hardware resources, changes according to the selected framework.

**Most importantly**, that choice is resolved at **compile time**, not at runtime.

**The result is a single codebase that preserves each framework's native implementation path** while avoiding runtime switches, translation layers, and duplicated application logic. In practice, you get the native performance of each ecosystem while *writing and maintaining your application logic only once.*

---

## 📚 The purpose of this repository

This guide documents the architecture, constraints, and practical decisions required to make that approach work in a real-world library.

It covers everything from the non-negotiable requirements of each ecosystem, to repository organization, build integration, and publication across all major distribution channels:

* Arduino Library Manager
* ESP Component Registry
* PlatformIO Registry

More importantly, the conclusions presented here are not theoretical deductions from documentation alone. Every design decision and every claim has been validated empirically on real hardware and through the development of the **CS123x** library:
👉 [https://github.com/FMazz97/CS123x](https://github.com/FMazz97/CS123x)

---

## Table of Contents

* [Introduction](README.md)
* [Chapter 1 - Ecosystem Constraints](chapters/01-ecosystem-constraints.md)
* [Chapter 2 - Repository structure](chapters/02-repo-structure.md)
* [Chapter 3 - Agnostic Core & HAL Architecture](chapters/03-agnostic-core_hal-architecture.md)
* [Chapter 4 - Example conventions](chapters/04-examples.md)
* [Chapter 5 - Publishing](chapters/05-publishing.md)
* [Chapter 6 - Bonus: Yield & Watchdog - Naming Convention](chapters/06-bonus_yield-watchdog_naming-convention.md)
