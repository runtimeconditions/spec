# Runtime Conditions Specification

Runtime Conditions is currently seeking adoption by an established parent
project. The repositories in this organization are split for hands-on usability,
review, demos, and implementation feedback. They are not intended to present
Runtime Conditions as a standalone foundation or competing project.

Start here: https://runtimeconditions.github.io/

## Purpose

This repository contains the Runtime Conditions Profile specification drafts,
whitepaper material, implementation guides, historical design notes, and draft
examples that support the public guide.

Runtime Conditions Profiles are portable, machine-readable declarations of the
external runtime integrations required by one workload. Profiles describe
demand. Generators emit them from code and package metadata. Adapters and
platforms decide how to fulfill them.

## Contents

- `docs/sixth-draft.md` - current core Runtime Conditions Profile draft.
- `docs/runtime-conditions-whitepaper-draft.md` - whitepaper-oriented narrative.
- `docs/guides/` - implementation guidance for extensions, package artifacts,
  SDK metadata, and generator discovery.
- `docs/history/` and `docs/core/` - earlier design material and supporting
  notes.
- `examples/` - incomplete draft examples used by the docs. These are kept here
  until they are ready to become a supported package surface.

## Related Repositories

- `runtimeconditions.github.io` - public reader guide.
- `extensions` - first-party extension definitions and declaration packages.
- `go-rc-profiler`, `java-rc-profiler`, `python-rc-profiler` - language
  profilers and generators.
- `rc-demos` - runnable demo apps and adapter assets.
