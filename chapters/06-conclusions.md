# Chapter 6 — Conclusions

The guide started from a question: why accept the complexity of supporting both Arduino and ESP-IDF natively, instead of choosing one ecosystem or relying on a compatibility layer? This chapter collects what the answer turned out to be, what remains open, and where to go from here.

---

## 6.1 What the guide settled

Five decisions, each following from the previous one:

1. **Constraints first** ([Chapter 1](01-ecosystem-constraints.md)). Arduino compiles everything under `src/`, ESP-IDF needs explicit, self-contained components, and registry naming rules are enforced at publication. Everything else comes from satisfying them together.
2. **One repository, one component boundary** ([Chapter 2](02-repo-structure.md)). The repository root is the component root and `src/` is the single source of truth: no separate branches, no framework-specific distributions.
3. **A framework-agnostic core** ([Chapter 3](03-agnostic-core-hal-architecture.md)). Protocol logic is written once, hardware access is selected at compile time through a HAL, and the Arduino facade only forwards.
4. **Examples that build with both toolchains** ([Chapter 4](04-examples.md)). One project layout for `idf.py` and PlatformIO, with one dependency declaration per toolchain.
5. **Publication as a design concern** ([Chapter 5](05-publishing.md)). Three registries with different manifests, filters, and immutability rules, and a package name decided before the first publish.

The [Final Checklist](07-final-checklist.md) condenses them, in the order in which they come up.

---

## 6.2 What remains open

The guide rests on a single library and a single design (see the warning at the end of the [Introduction](../README.md#-the-purpose-of-this-repository)). Where it reports an observation instead of a documented fact, it says so in the text: this section collects what was not verified and the limits found along the way, and each point is a good starting point for a contribution.

### Not verified in this guide

- **Premises observed, not measured**
  - That a C API is the dominant design choice for ESP-IDF components: an observation, and the premise that could reshape the architecture of [Chapter 3](03-agnostic-core-hal-architecture.md).
- **Not tested**
  - Arduino tag with a `v` prefix: only tags without it were tested.
  - Version immutability on both Registries: stated from the documentation, never tested.
- **Observed, not explained**
  - How the build system accepts `<name>` in `REQUIRES` for a component registered as `<namespace>__<name>` ([Appendix A](../appendix/A-esp-idf-dependency-resolution.md)).
  - Why the Arduino-ESP32 build never tripped the Task Watchdog: tick rate and core of `loop()` were not compared ([Appendix B](../appendix/B-yield-watchdog.md)).
  - The hypothesis on the unstable PlatformIO layout ([Appendix D](../appendix/D-testing-during-development.md)): the files created while building appear to relaunch PlatformIO's reconfiguration, not verified in its source.

### Known limits

- The PIO Home → Examples panel ignores `examples[].name` for libraries and lists files one by one; the web Registry shows the names ([5.3.1](05-publishing.md#531-manifest-and-metadata)).
- A PlatformIO package cannot be renamed from the CLI; it takes a support request ([5.4](05-publishing.md#54-the-case-sensitivity-trap-platformios-immutable-package-binding)).

---

## 6.3 Beyond Arduino & ESP-IDF: Extending and Contributing

While this guide focused on bridging Arduino, ESP-IDF, and PlatformIO, the underlying patterns (unified source layouts, explicit dependency management, and reciprocal registry exclusions) are likely applicable to the wider embedded ecosystem. The appendices go deeper on the technical details:

- **[Appendix A: IDF Dependency Resolution](../appendix/A-esp-idf-dependency-resolution.md)**: how ESP-IDF discovers components, and what the Component Manager adds to the build system.
- **[Appendix B: Yield & Watchdog](../appendix/B-yield-watchdog.md)**: how long blocking operations cooperate with the scheduler across AVR, ESP8266, and FreeRTOS-based ESP32 targets.
- **[Appendix C: Naming Convention](../appendix/C-naming-convention.md)**: what a method name promises, and how case styles map across the core and the Arduino facade.
- **[Appendix D: Testing During Development](../appendix/D-testing-during-development.md)**: one example tested on many boards with PlatformIO, and the layouts that keep it reliable.

The same approach should carry over to other families and frameworks, such as STM32 with its Arduino and STM32Cube frameworks, where a new HAL would be the only framework-specific addition, or Zephyr (not tested here). The embedded landscape keeps evolving, with new microcontrollers and alternative build systems continuing to emerge. If you want to extend this framework-agnostic approach to other platforms, refine the appendices, or share your own experience, issues and pull requests are always welcome: embedded development thrives on shared knowledge.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Chapter 5 - Publishing](05-publishing.md) — [Introduction](../README.md) — [Final Checklist ➡️](07-final-checklist.md)

</div>
<!-- markdownlint-enable MD033 -->
