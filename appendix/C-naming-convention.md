# The naming convention (`get_`/`read_`/`set_`) — and why it only emerged by iterating

<!--aggiungere convenzione snake_case/camelCase-->

While refining the public API of the [CS123x](https://github.com/FMazz97/CS123x) library, a recurring pattern emerged: several method names were making promises their implementations didn't actually keep. What initially looked like a library-specific cleanup eventually turned into a more general naming convention that applies to many hardware drivers and communication libraries.

## The rule

Once made explicit:

- **`get_*`** — reads **already-held state**. Zero hardware communication, always `const`, cannot fail, cannot block.

- **`set_*`** — mutates **already-held state**, or writes it to hardware while explicitly reporting success or failure through its return value and documented behavior.

- **`read_*`** — communicates with the underlying device. Can **block**, can **time out**, and may **return sentinel error values**.

- **write_*** — communicates with the underlying device to modify its state. Can **block**, can **time out**, and should **report success or failure** through its return value and documented behavior.

---

## Where the convention came from

Two real-world examples illustrate where this convention was being violated before it was consciously defined:

- `getValue()`/`getUnits()` **looked** like pure state accessors — same `get_` prefix as `getOffset()`/`getScale()` sitting right next to them — but both internally called `read_average()`, with all the blocking and failure modes that implies. Renamed to `readNetCounts()`/`readNetUnits()`.

- `setTempCalibration(float)` (the single-argument overload) silently read a live value from the chip to derive a calibration point — a `set_` name hiding a `read_` operation, overloaded against a *second*, genuinely pure `setTempCalibration(float, int32_t)` that only wrote already-known values to memory. Splitting them into `calibrateTemp()` (hardware-reading) and `setTempCalibration()` (pure) removed the ambiguity that the shared name — and the overload resolution needed to tell them apart — had been carrying.

The lesson this generalizes into is simple: **a naming convention discovered by auditing a real API for places where names and implementations disagree is often more reliable than one designed purely in the abstract.**
Both renames above were only visible once every method could be listed side by side and asked the same question — "does this touch hardware, yes or no?" — a question that's easy to get right on a small API and easy to drift on as one grows, unless the convention is written down and treated as load-bearing rather than cosmetic once it's found. It's also, not incidentally, the kind of change that's worth doing as a deliberate breaking version bump rather than deferring indefinitely — a naming inconsistency left in a public API only gets more expensive to fix as more code depends on the wrong name.

---

## Table of Contents

- [Introduction](../README.md)
- [Chapter 1 - Ecosystem Constraints](../chapters/01-ecosystem-constraints.md)
- [Chapter 2 - Repository Structure](../chapters/02-repo-structure.md)
- [Chapter 3 - Agnostic Core & HAL Architecture](../chapters/03-agnostic-core-hal-architecture.md)
- [Chapter 4 - Example Conventions](../chapters/04-examples.md)
- [Chapter 5 - Publishing](../chapters/05-publishing.md)
- [Appendix A - IDF Dependency Resolution](A-esp-idf-dependency-resolution.md)
- [Appendix B - Yield & Watchdog](B-yield-watchdog.md)
- [Appendix C - Naming Convention](C-naming-convention.md)
