# Chapter 5 — Publishing on All Three Registries, and the Case-Sensitivity Trap That Follows You Everywhere

Publishing to Arduino Library Manager, the PlatformIO Registry, and the ESP Component Registry are three independent processes, with three independent review models. But the same case-sensitivity constraint from Chapter 1 resurfaces at publish time in a way that isn't obvious until it's hit directly — and, in this case, it wasn't caught until after a first publication had already happened under the "wrong" case.

## Arduino Library Manager — automated, tag-driven, no manual review of content

Submission is a Pull Request against `arduino/library-registry`, adding the repository URL to a plain text list. A bot then runs the same checks `arduino-lint` performs locally — running it yourself first (`arduino-lint --library-manager submit`) catches structural issues before they cost a review round-trip. Two requirements are easy to overlook on a first attempt:

- **At least one tagged GitHub Release must already exist**, with `library.properties`'s `version` field matching. The indexer reads the manifest **from the tagged commit**, not from the latest commit on the default branch — a release created before a version bump was committed will silently index the *old* version, appearing to "not update" with no error at all.
- **After acceptance, updates are fully automatic**: publishing a new tagged Release with a bumped `version` is the entire update process. No second Pull Request is needed.

## PlatformIO Registry — self-service via CLI, immutable once published

```bash
pio pkg publish
```

No review step — the package is live as soon as the command succeeds. This also means the immutability constraint has no safety net: **a given `name@version` combination can never be republished**, even if deleted. Verify the manifest's `version` field is correct before running the command, not after.

## ESP Component Registry — self-service via `compote`, same immutability rule

```bash
compote component validate   # catches manifest issues before an upload attempt
compote component upload --name <component-name>
```

Functionally the same shape as PlatformIO's flow — CLI-driven, no human review, and the same "once published, that exact name+version is permanent" rule applies here too.

## The case-sensitivity trap: it follows the package name, not just the source folder

Chapter 1 established that ESP-IDF's native dependency resolvers are case-sensitive. What's easy to miss is that **this doesn't stop being true once a package is registered** — the name recorded at publish time becomes another case-sensitive identity that every subsequent `REQUIRES`/`idf.py add-dependency` reference has to match exactly, just like the local folder name did before publishing existed at all.

This library was first published to both PlatformIO and the ESP Component Registry as `CS123x` (uppercase), matching the Arduino Library Manager entry already live. It later needed to be `cs123x` (lowercase) for consistent resolution under ESP-IDF's build system, once native ESP-IDF support was added. Renaming `library.json`'s `name` field and republishing produced:

```text
Manifest name='cs123x' <> DB name='CS123x' and pkg_id is not passed
```

The registry backend had recorded the original case as the package's identity — the manifest and the database no longer agreed, and no CLI flag exists to say "this is the same package under a new name" (`pio pkg publish --help` exposes no package-ID option). Two paths exist once this happens:

- **Delete and republish under the new name.** Available, but destructive: existing download statistics are lost, and any already-published `platformio.ini`/`lib_deps` referencing the old name breaks immediately for anyone using it.
- **Ask the registry's support team to rename the entry server-side.** This is what resolved it in practice — PlatformIO's support renamed the existing package in place, preserving its history, in response to a direct support request. No public self-service rename mechanism was found; asking is the only tested path that keeps the package's history intact.

The ESP Component Registry side never hit this problem, for a simple reason worth generalizing: it was the *first* publication of that component under any name — there was nothing to rename, only something to name correctly from the start.

## The generalizable rule

**Decide the published package's name — in every registry it will live in — before the first publish, not after.** A rename mid-lifecycle isn't just an inconvenience; on registries with immutable versioning, it's either destructive (delete and lose history) or dependent on a support team's willingness to intervene manually, since the tooling itself provides no self-service path for it. The lowercase-everywhere convention this guide has used throughout Chapters 1–3 isn't only about satisfying `EXTRA_COMPONENT_DIRS`'s local-folder constraint — it's the same choice that, made early, avoids this exact registry-level trap entirely.
