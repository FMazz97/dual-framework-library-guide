# Appendix F — Version Check: Keeping the Manifests Aligned

The three manifests (`library.properties`, `library.json`, `idf_component.yml`) must carry the same version, and nothing keeps them aligned except the maintainer (see the [Final Checklist](../chapters/07-final-checklist.md)). This appendix describes a small script that catches the usual mistake, a forgotten bump, before it leaves your machine. The files are in the template repository [DualBlinkCpp](https://github.com/FMazz97/DualBlinkCpp).

> [!IMPORTANT]
> This is a **check, not a guarantee**. It reduces the chance of a mistake; it cannot make a bad release impossible (see [Limits](#limits)).

---

## What it checks

- The three manifests carry the **same, valid semantic version** (`X.Y.Z`, optionally `-prerelease`).
- **Branch push:** that version is strictly **greater than the highest existing tag**.
- **Tag push:** the tag name equals the version (a leading `v` is ignored) and is greater than every other tag.
- With **no tags** there is no limit. Unusual version strings (characters outside `a-z`, `0-9`, `.`, `-`, or over 20 characters, the limits in PlatformIO's documentation) only raise a warning.

---

## How it works

The point is **what is read**: not the working tree, but the manifests **as stored in the commit being pushed** (`git show <commit>:<file>`). A tag snapshots a commit, so if you tag misaligned manifests and then fix the files, a check on the working tree would pass while the tag still points at the bad commit. This also mirrors the Arduino indexer, which reads the version from `library.properties` in the tagged commit.

Two layers run the same script:

1. **A local `pre-push` hook, the strong point.** It aborts the push of any branch or tag that breaks a rule, so nothing reaches GitHub or the registries.
2. **A GitHub Actions workflow, the safety net.** It reports the problem if the hook was not active, but it can only react after the push. An optional job also deletes a tag (and its Release) that fails the check.

---

## Setup

Copy `scripts/check_versions.py`, `.githooks/pre-push` and `.github/workflows/check-versions.yml` into the repository, then:

> [!IMPORTANT]
> **Enable the hook, once per clone.** Git ignores `.githooks/` until told otherwise, and the setting is stored locally:
>
> ```bash
> git config core.hooksPath .githooks
> ```
>
> If a push does not print a `pre-push: checking …` line, the hook is not active.

Keep the hook on LF line endings (`.githooks/* text eol=lf` in `.gitattributes`) or its first line breaks on Windows, and set the executable bit for Linux and macOS users: `git update-index --chmod=+x .githooks/pre-push`.

**Release flow:** bump all three manifests in a single commit and push; create the tag with the same version; then **bump to the next version immediately**, because until then the version equals the last tag and every push is rejected.

---

## Assumptions

- The manifests are at the repository root, each with a **top-level** version key (versions of dependencies are not read). Other names or paths: edit `MANIFESTS` in the script.
- Versions are semantic; tags are compared with semver precedence and non-semver tags are ignored.
- Python 3 is available, and there is a single version line (no hotfixes on older versions).

---

## Limits

- **The hook is per-clone**, can be skipped with `git push --no-verify`, and skips the check, with a message, if no working Python is found.
- **GitHub cannot block a push**, only report it. A required status check on `main` blocks merges, but only through pull requests.
- **Nothing protects `main` by default.** Branch protection is a setting of each GitHub repository, not part of the files: without it, a merge made on the website skips the hook and a failing check does not stop it.
- **A Release created on the GitHub website never passes through the hook.** Only the workflow sees it, afterwards. The deletion job, verified in a test repository, removes the tag within moments, but a registry may have looked at it first.
- **Deleting a tag does not unpublish a version**: the Arduino Library Manager indexes hourly, and removing a release requires an issue. After a bad tag, fix it and publish the **next** version. A number whose tag was deleted can also be reused: this is not covered.
- **Only the tip of each pushed branch is checked**, and only versions are compared, not the other fields.
- **Old branches are rejected** until updated from the main branch, and re-running the workflow of a commit already released fails by design.

---

<!-- markdownlint-disable MD033 -->
<div align="center">

[⬅️ Appendix E - Minimal Example](E-minimal-example.md) — [Introduction](../README.md)

</div>
<!-- markdownlint-enable MD033 -->
