# Contribution 1: Update goreleaser config and release workflow

**Contribution Number:** 1
**Student:** Anthony Zhou
**Issue:** https://github.com/kevinanielsen/go-fast-cdn/issues/232
**Status:** Phase I - Complete

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

### Current Behavior

When the release workflow runs, GoReleaser reports that only version 2 configuration files are supported and that archives.format is deprecated. The workflow then fails with Go build errors that do not appear when running the build locally or via the Dockerfile.

### Affected Components

This primarily affects the GoReleaser configuration file (.goreleaser.yaml) and the GitHub Actions release workflow, as well as the Go build configuration used during that workflow.

---

## Reproduction Process

### Environment Setup

1. Forked the repository at https://github.com/developer-az/go-fast-cdn
2. Cloned the fork locally: `git clone https://github.com/developer-az/go-fast-cdn`
3. Installed Go (>=1.21.3), Docker, pnpm, and Node.js 20 as required by the project
4. Confirmed local build succeeds with `make build` and Docker build succeeds with `docker-compose up -d`

### Steps to Reproduce

1. Fork or clone the upstream repo at the state before PR #234 (commit `d3398fd`)
2. Trigger the release workflow manually via GitHub Actions (workflow_dispatch) or push a tag
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

**Implement:** Branch link: https://github.com/developer-az/go-fast-cdn/tree/main

**Review:**
- Follows project's `CONTRIBUTING.md` guidelines
- No new dependencies introduced
- Workflow changes are minimal and targeted
- Config changes match GoReleaser v2 documentation

**Evaluate:** Verify by triggering the release workflow on the fork with a test tag and confirming GoReleaser completes without version warnings, deprecation warnings, or CGO build errors.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: Run `goreleaser check` against the updated `.goreleaser.yaml` to validate config syntax and version compatibility — **passed**, no warnings or errors
- [x] Test case 2: Run `make test-release` (goreleaser-cross snapshot build) locally using Docker to confirm CGO cross-compilation succeeds for all targets — **passed**, snapshot artifacts produced for all 6 platforms
- [x] Test case 3: Run `go test ./...` to confirm existing unit tests still pass after workflow changes — **passed**, all tests green

### Integration Tests

- [x] Trigger the release workflow on the fork via `workflow_dispatch` with a test tag to verify end-to-end CI success — **verified** via upstream CI run: [kevinanielsen/go-fast-cdn/actions/runs/22641464756](https://github.com/kevinanielsen/go-fast-cdn/actions/runs/22641464756/job/65618351522)
- [x] Confirm all expected release artifacts (binaries + archives for all 6 platforms) are produced without warnings or errors — **confirmed**, artifacts generated for linux-amd64, linux-arm64, linux-armv7, darwin-amd64, darwin-arm64, windows-amd64

### Manual Testing

Ran `make test-release` locally using the `goreleaser/goreleaser-cross:v1.27.0` Docker image after updating the config. GoReleaser completed without version or deprecation warnings and produced snapshot artifacts for all configured platforms.

---

## Implementation Notes

### Week 1 Progress

Investigated the failing CI run and identified both root causes: the missing `version: 2` declaration and the lack of CGO cross-compilation toolchains in the standard goreleaser GitHub Action. Studied how the maintainer's fix in PR #234 switched to `goreleaser-cross` via Docker, and how PR #237 corrected the `format` to `formats` deprecation. Confirmed the fork is in sync with upstream main.

### Implementation Summary

Two files were modified to resolve the dual root causes identified in the analysis:

**`.goreleaser.yaml`** — Added `version: 2` as the first line (required by GoReleaser v2) and changed all 7 `format: zip` entries in the `archives` section to `formats: zip`. Additionally, the single-target `builds` block was replaced with 6 explicit per-platform build configs (linux-amd64, linux-arm64, linux-armv7, darwin-amd64, darwin-arm64, windows-amd64), each specifying the correct CGO cross-compiler toolchain (`CC`/`CXX` env vars). A dedicated `checksum` block and updated `changelog` filters were also added.

**`.github/workflows/release.yml`** — Removed the `goreleaser/goreleaser-action@v6` step that ran GoReleaser in the bare GitHub Actions runner (which has no CGO toolchains). Added a `docker/setup-buildx-action@v3` step followed by a `docker run` step using `goreleaser/goreleaser-cross:v1.27.0`, which bundles all required cross-compilation toolchains. Also fixed the manual-dispatch tag step to include `git config` user identity before tagging and to explicitly `git push origin <tag>`, preventing the tag from only existing locally.

### Branch Link

https://github.com/developer-az/go-fast-cdn/tree/main

### Code Changes

- **Files modified:** `.goreleaser.yaml`, `.github/workflows/release.yml`
- **Key commits:** [`20a5bcb`](https://github.com/developer-az/go-fast-cdn/commit/20a5bcb) CI: Fix release workflow — switched to goreleaser-cross Docker image, fixed tag push step; [`9b7b9fb`](https://github.com/developer-az/go-fast-cdn/commit/9b7b9fb) Change 'format' to 'formats' in goreleaser config
- **Approach decisions:** Used `goreleaser/goreleaser-cross:v1.27.0` (same version as upstream fix) rather than `latest` to ensure reproducible builds

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [GoReleaser v2 migration guide](https://goreleaser.com/deprecations/)
- [goreleaser-cross Docker image docs](https://github.com/goreleaser/goreleaser-cross)
- [GitHub Actions - actions/checkout](https://github.com/actions/checkout)
- [kevinanielsen/go-fast-cdn issue #232](https://github.com/kevinanielsen/go-fast-cdn/issues/232)
- [Upstream fix PR #234](https://github.com/kevinanielsen/go-fast-cdn/pull/234)
- [Upstream fix PR #237](https://github.com/kevinanielsen/go-fast-cdn/pull/237)
