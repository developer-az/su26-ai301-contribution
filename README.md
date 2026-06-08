# Contribution 1: Update goreleaser config and release workflow

**Contribution Number:** 1
**Student:** Anthony Zhou
**Issue:** https://github.com/kevinanielsen/go-fast-cdn/issues/232
**Status:** Phase I - Complete

---

## Why I Chose This Issue

I picked this issue because I want more hands-on experience with real CI and release pipelines, not just running builds locally. Updating the GoReleaser config and fixing a failing release workflow feels like a practical way to learn how Go projects are actually shipped in production.

It also lines up with my goal of getting better at debugging problems that only show up in CI. By working through why the release job fails while local and Docker builds pass, I’m hoping to deepen my understanding of how environment differences and configuration choices affect a project’s reliability.

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
