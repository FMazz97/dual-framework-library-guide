# The naming convention (`get_`/`read_`/`set_`) — and why it only emerged by iterating

This convention wasn't designed upfront. It was extracted, after the fact, from noticing that a few method names were making a promise their implementation didn't keep — and that inconsistency is worth explaining directly, because the fix pattern generalizes well beyond this one library.

The rule, once made explicit:

- **`get_*`** — reads **already-held state**. Zero hardware communication, always `const`, cannot fail, cannot block. `get_gain()`, `get_rate()`, `get_config()`.
- **`set_*`** — mutates **already-held state**, or writes it to hardware *and* reports success/failure explicitly through its return value and documented behavior. `set_offset()` (pure memory), `set_config()` (hardware write, but its `set_` name is earned by symmetry with the `get_config()` it pairs with and by being the library's canonical "configure the chip" verb).
- **`read_*`** — talks to the chip. Can block, can time out, returns a value that may be a sentinel error. `read()`, `read_average()`, `read_dual_channel()`.

Two renames from the library's own history show where the convention was violated before it was named:

- `getValue()`/`getUnits()` **looked** like pure state accessors — same `get_` prefix as `getOffset()`/`getScale()` sitting right next to them — but both internally called `read_average()`, with all the blocking and failure modes that implies. Renamed to `readNetCounts()`/`readNetUnits()`.
- `setTempCalibration(float)` (the single-argument overload) silently read a live value from the chip to derive a calibration point — a `set_` name hiding a `read_` operation, overloaded against a *second*, genuinely pure `setTempCalibration(float, int32_t)` that only wrote already-known values to memory. Splitting them into `calibrateTemp()` (hardware-reading) and `setTempCalibration()` (pure) removed the ambiguity that the shared name — and the overload resolution needed to tell them apart — had been carrying.

The lesson this generalizes into: **a naming convention discovered by auditing a real API for where the name and the implementation disagree is more reliable than one designed in the abstract before any method exists to test it against.** Both renames above were only visible once every method could be listed side by side and asked the same question — "does this touch hardware, yes or no?" — a question that's easy to get right on a small API and easy to drift on as one grows, unless the convention is written down and treated as load-bearing rather than cosmetic once it's found. It's also, not incidentally, the kind of change that's worth doing as a deliberate breaking version bump (Chapter 6 covers the resulting `v2.0.0` migration table) rather than deferring indefinitely — a naming inconsistency left in a public API only gets more expensive to fix as more code depends on the wrong name.
