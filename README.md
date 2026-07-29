# Open Source Contributions — go-fast-cdn

**Student:** Anthony Zhou  
**Repository:** https://github.com/kevinanielsen/go-fast-cdn  
**Fork:** https://github.com/developer-az/go-fast-cdn

| # | Issue | Status |
|---|-------|--------|
| 1 | [#232 — Update goreleaser config and release workflow](https://github.com/kevinanielsen/go-fast-cdn/issues/232) | **Complete** |
| 2 | [#232 follow-up — Replace deprecated `archives.builds` with `ids`](https://github.com/kevinanielsen/go-fast-cdn/issues/232) | **Submitted — Awaiting Review** ([#248](https://github.com/kevinanielsen/go-fast-cdn/pull/248)) |

---

# Contribution 1: Update goreleaser config and release workflow

**Contribution Number:** 1  
**Issue:** https://github.com/kevinanielsen/go-fast-cdn/issues/232  
**Status:** Complete  
**Issue comment:** https://github.com/kevinanielsen/go-fast-cdn/issues/232#issuecomment-4645216364

---

## Why I Chose This Issue

I picked this issue because I want more hands-on experience with real CI and release pipelines, not just running builds locally. Updating the GoReleaser config and fixing a failing release workflow feels like a practical way to learn how Go projects are actually shipped in production.

It also lines up with my goal of getting better at debugging problems that only show up in CI. By working through why the release job fails while local and Docker builds pass, I'm hoping to deepen my understanding of how environment differences and configuration choices affect a project's reliability.

---

## Understanding the Issue

### Problem Description

The release workflow for this project is out of date and unreliable. The GoReleaser config uses an old version format and deprecated options, and the release job fails with Go build errors even though local and Docker builds succeed.

### Expected Behavior

The release workflow should run successfully in GitHub Actions using a current GoReleaser configuration. It should build the project without warnings or Go build errors, producing the expected release artifacts.

### Current Behavior (at time of investigation)

When the release workflow runs, GoReleaser reports that only version 2 configuration files are supported and that `archives.format` is deprecated. The workflow then fails with CGO-related Go build errors that do not appear when running the build locally or via the Dockerfile.

### Affected Components

This primarily affects the GoReleaser configuration file (`.goreleaser.yaml`) and the GitHub Actions release workflow, as well as the Go build configuration used during that workflow.

---

## Reproduction Process

### Environment Setup

1. Forked the repository at https://github.com/developer-az/go-fast-cdn
2. Cloned the fork locally: `git clone https://github.com/developer-az/go-fast-cdn`
3. Installed Go (>=1.21.3), Docker, pnpm, and Node.js 20 as required by the project
4. Confirmed local build succeeds with `make build` and Docker build succeeds with `docker-compose up -d`

### Steps to Reproduce

1. Fork or clone the upstream repo at the state before PR #234 (commit `d3398fd`)
2. Trigger the release workflow manually via GitHub Actions (`workflow_dispatch`) or push a tag
3. Observe the GoReleaser step fail with: `WARN only version: 2 configuration files are supported, yours is version: 0`
4. Observe the deprecation warning: `DEPRECATED: archives.format should not be used anymore`
5. Watch the build fail with CGO-related Go build errors that don't occur locally or in the Dockerfile

### Reproduction Evidence

- **Upstream failing run:** https://github.com/kevinanielsen/go-fast-cdn/actions/runs/22641464756/job/65618351522
- **My fork (used for investigation):** https://github.com/developer-az/go-fast-cdn
- **My findings:** The `.goreleaser.yaml` was missing `version: 2` at the top, causing GoReleaser v2 to reject it. The `archives` section used the deprecated `format` key instead of `formats`. The release workflow used `goreleaser/goreleaser-action@v6` without Docker-based cross-compilation, which lacks the CGO toolchains needed to build for Linux/macOS/Windows with CGO enabled (required for WEBP support via `github.com/chai2010/webp`).

---

## Solution Approach

### Analysis

There are two distinct root causes:

1. **Outdated GoReleaser config format:** The `.goreleaser.yaml` lacked `version: 2` at the top, causing GoReleaser v2 to reject the configuration with a version mismatch warning and fall back to broken defaults. Additionally, the `archives` entries used the deprecated `format` key (singular) which was removed in v2 in favor of `formats` (plural).

2. **Missing CGO cross-compilation toolchains in CI:** The release workflow was calling the standard `goreleaser/goreleaser-action` GitHub Action, which runs GoReleaser in the plain GitHub Actions runner environment. This environment does not have the cross-compilation toolchains (e.g., `aarch64-linux-gnu-gcc`, `arm-linux-gnueabihf-gcc`, `o64-clang`, `x86_64-w64-mingw32-gcc`) needed to build CGO-enabled binaries for multiple platforms. Local builds and Docker builds succeed because they either don't cross-compile or use a Docker image that includes these tools.

### Proposed Solution

- Add `version: 2` to the top of `.goreleaser.yaml` to satisfy GoReleaser v2's configuration format requirement
- Replace all `format:` keys with `formats:` in the `archives` section of `.goreleaser.yaml`
- Replace the `goreleaser/goreleaser-action` step in the release workflow with a `docker run` call using the `goreleaser/goreleaser-cross:v1.27.0` image, which bundles all required CGO cross-compilation toolchains
- Fix the tag push step in the workflow to include git user config and `git push origin <tag>` so manual dispatch tagging works correctly

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The release workflow fails because (a) GoReleaser v2 rejects the old `version: 0` config format and the deprecated `format` key in archives, and (b) the GitHub Actions runner lacks CGO cross-compilation toolchains needed to produce multi-platform binaries with WEBP support.

**Match:** The project already uses `goreleaser-cross` referenced in its `Makefile` and `README.md` for local releases. The fix aligns CI with how releases are meant to be run locally. The `goreleaser/goreleaser-cross` Docker image is the canonical solution for CGO cross-compilation in GoReleaser workflows.

**Plan:**
1. In `.goreleaser.yaml`: Add `version: 2` as the first line
2. In `.goreleaser.yaml`: Change every `format: zip` under `archives` to `formats: zip` (7 occurrences)
3. In `.github/workflows/release.yml`: Remove the `goreleaser/goreleaser-action@v6` step
4. In `.github/workflows/release.yml`: Add a `docker/setup-buildx-action@v3` step
5. In `.github/workflows/release.yml`: Add a `docker run` step using `goreleaser/goreleaser-cross:v1.27.0` that mounts the workspace and passes `GITHUB_TOKEN`
6. In `.github/workflows/release.yml`: Fix the tag creation step to add git user config and push the tag to origin

**Implement:** https://github.com/developer-az/go-fast-cdn/tree/main

**Review:**
- Follows project's `CONTRIBUTING.md` guidelines
- No new dependencies introduced
- Workflow changes are minimal and targeted
- Config changes match GoReleaser v2 documentation

**Evaluate:** Verify by triggering the release workflow and confirming GoReleaser completes without version warnings, CGO build errors, or failed artifact generation.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: Run `goreleaser release --snapshot --clean` via `goreleaser-cross` Docker image — **passed**, snapshot artifacts produced for all 6 platforms
- [x] Test case 2: Run `pnpm --dir=./ui build` + `go test ./...` to confirm the project still builds and tests — **passed** on Linux CI; local Windows run blocked by disk space during CGO compile
- [x] Test case 3: Run `goreleaser check` — **partial**: original `version: 0` and `archives.format` issues resolved on `main`; one remaining deprecation (`archives.builds`) identified for Contribution 2

### Integration Tests

- [x] Confirm upstream release workflow succeeds on `main` — **passed**: [upstream run 22668518856](https://github.com/kevinanielsen/go-fast-cdn/actions/runs/22668518856)
- [x] Confirm all expected release artifacts (binaries + archives for all 6 platforms) — **confirmed**: linux-amd64, linux-arm64, linux-armv7, darwin-amd64, darwin-arm64, windows-amd64

### Manual Testing

Ran `goreleaser release --snapshot --clean` locally using the `goreleaser/goreleaser-cross:v1.27.0` Docker image against upstream `main`. GoReleaser completed without the original version or `archives.format` deprecation warnings and produced snapshot artifacts for all configured platforms.

---

## Implementation Notes

### Progress Summary

Investigated the failing CI run and independently identified both root causes: the missing `version: 2` declaration and the lack of CGO cross-compilation toolchains in the standard goreleaser GitHub Action. Validated the fix on upstream `main` after PR #234 and PR #237 merged. Confirmed the fork is in sync with upstream.

### Outcome

The primary issues described in #232 were fixed upstream before I opened a separate PR:

| Change | Upstream PR | Status on `main` |
|--------|-------------|------------------|
| `version: 2` + goreleaser-cross in CI | [#234](https://github.com/kevinanielsen/go-fast-cdn/pull/234) | Merged |
| `format` → `formats` in archives | [#237](https://github.com/kevinanielsen/go-fast-cdn/pull/237) | Merged |
| Release workflow green on `main` | [run 22668518856](https://github.com/kevinanielsen/go-fast-cdn/actions/runs/22668518856) | Verified |

My analysis matched the maintainer's solution. I commented on the issue to claim it and documented reproduction steps, root-cause analysis, and local validation. One remaining GoReleaser deprecation (`archives.builds` → `archives.ids`) is tracked as Contribution 2.

### Key Upstream Commits Referenced

- [`20a5bcb`](https://github.com/kevinanielsen/go-fast-cdn/commit/20a5bcb) — CI: Fix release workflow (goreleaser-cross Docker image, tag push fix)
- [`9b7b9fb`](https://github.com/kevinanielsen/go-fast-cdn/commit/9b7b9fb) — Change `format` to `formats` in goreleaser config

---

## Pull Request

**PR Link:** N/A — equivalent fix merged upstream via [#234](https://github.com/kevinanielsen/go-fast-cdn/pull/234) and [#237](https://github.com/kevinanielsen/go-fast-cdn/pull/237) before a separate PR was submitted under my account.

**Maintainer Feedback:** No direct reply to my issue comment yet. Release workflow success on `main` confirms the fix is in production.

**Status:** Complete (investigation, validation, and issue claim documented; upstream fix verified)

---

## Learnings & Reflections

### Technical Skills Gained

- How GoReleaser v2 config versioning works and why deprecated fields break CI silently
- Why CGO cross-compilation requires platform-specific toolchains that plain GitHub Actions runners lack
- How to use `goreleaser-cross` Docker for reproducible multi-platform releases
- How to compare local, Docker, and CI build environments to isolate environment-specific failures

### Challenges Overcome

- Discovering that upstream had already merged the fix while I was still documenting my approach — learned to check open PRs and `main` branch state early
- Validating releases on Windows required Docker path mounting and dealing with local disk constraints during CGO builds

### What I'd Do Differently Next Time

- Comment on the issue and open a draft PR earlier, before spending time re-implementing a fix that may already be in flight
- Run `goreleaser check` first — it surfaces deprecations like `archives.builds` that snapshot releases still tolerate

---

## Resources Used

- [GoReleaser v2 migration guide](https://goreleaser.com/deprecations/)
- [goreleaser-cross Docker image docs](https://github.com/goreleaser/goreleaser-cross)
- [GitHub Actions - actions/checkout](https://github.com/actions/checkout)
- [kevinanielsen/go-fast-cdn issue #232](https://github.com/kevinanielsen/go-fast-cdn/issues/232)
- [Upstream fix PR #234](https://github.com/kevinanielsen/go-fast-cdn/pull/234)
- [Upstream fix PR #237](https://github.com/kevinanielsen/go-fast-cdn/pull/237)

---
---

# Contribution 2: Replace deprecated `archives.builds` in GoReleaser config

**Contribution Number:** 2  
**Issue:** https://github.com/kevinanielsen/go-fast-cdn/issues/232 (follow-up)  
**Status:** Submitted — Awaiting Review  
**Planned branch:** `fix/goreleaser-archives-ids` (pushed)

---

## Why I'm Continuing on #232

Contribution 1 resolved the original release workflow failures (`version: 0`, `archives.format`, CGO build errors). After validating upstream `main`, `goreleaser check` still reports:

```
DEPRECATED: archives.builds should not be used anymore
configuration is valid, but uses deprecated properties
```

GoReleaser v2.8+ renamed `archives.builds` to `archives.ids`. Snapshot releases work, but `goreleaser check` fails until this is updated. This is a small, non-duplicate follow-up where I can submit my own PR.

---

## Understanding the Issue

### Problem Description

Each archive entry in `.goreleaser.yaml` uses the deprecated `builds:` key to reference which build target to package. GoReleaser now expects `ids:` instead.

### Expected Behavior

`goreleaser check` passes with no deprecation warnings or errors.

### Current Behavior

`goreleaser check` exits with an error citing deprecated `archives.builds`. Snapshot releases still succeed.

### Affected Components

- `.goreleaser.yaml` — 6 archive entries under `archives:` (lines ~110–163)

---

## Proposed Solution

Rename `builds:` to `ids:` inside each `archives:` block (do **not** change the top-level `builds:` section that defines compile targets):

```yaml
# Before
archives:
  - id: linux-amd64
    builds:
      - linux-amd64

# After
archives:
  - id: linux-amd64
    ids:
      - linux-amd64
```

Reference: [GoReleaser deprecations — archives.builds](https://goreleaser.com/deprecations/#archivesbuilds)

---

## Implementation Plan

1. ~~Comment on [#232](https://github.com/kevinanielsen/go-fast-cdn/issues/232) with a progress update and intent to open a follow-up PR~~
2. ~~Create branch `fix/goreleaser-archives-ids` from upstream `main`~~
3. ~~Replace `builds:` with `ids:` in all 6 archive entries in `.goreleaser.yaml`~~
4. Run `goreleaser check` via `goreleaser/goreleaser-cross:v1.27.0` Docker image
5. Run `goreleaser release --snapshot --clean` to confirm all 6 platform artifacts still build
6. ~~Open draft PR to upstream with `Fixes #232` in the description~~

### Code Changes

- **File modified:** `.goreleaser.yaml`
- **Change:** Renamed `builds:` → `ids:` in all 6 archive entries (6 insertions, 6 deletions)
- **Branch:** https://github.com/developer-az/go-fast-cdn/tree/fix/goreleaser-archives-ids
- **Commit:** `c5a1336` — fix: replace deprecated archives.builds with ids in goreleaser config

---

## Testing Strategy

- [ ] `goreleaser check` — passes with no deprecation warnings
- [ ] `goreleaser release --snapshot --clean` — all 6 platform zips produced
- [ ] `go test ./...` — existing tests still pass

---

## Pull Request

**PR Link:** https://github.com/kevinanielsen/go-fast-cdn/pull/248

**Status:** Open — awaiting maintainer review

**Suggested follow-up comment on [#232](https://github.com/kevinanielsen/go-fast-cdn/issues/232):**

> Update: The original release workflow issues are fixed on `main` (#234/#237). I opened PR #248 to address the remaining GoReleaser deprecation (`archives.builds` → `archives.ids`) so `goreleaser check` passes cleanly.

---

## Resources Used

- [GoReleaser deprecations — archives.builds](https://goreleaser.com/deprecations/#archivesbuilds)
- [kevinanielsen/go-fast-cdn issue #232](https://github.com/kevinanielsen/go-fast-cdn/issues/232)
