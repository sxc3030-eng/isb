# ISB — Idea → System Builder

> Turn a plain-language project brief into a structured system specification — in one pass.

![Status](https://img.shields.io/badge/status-design%20phase-orange)
![Spec](https://img.shields.io/badge/spec-2026--04--22-blue)
![Method](https://img.shields.io/badge/method-AI--paired-purple)

ISB is a single-page tool that takes a paragraph of plain-language project intent and produces a structured system specification: components, data model, API surface, infrastructure decisions, and acceptance criteria — ready to feed an implementation team or an AI coding agent.

## Why

Most "vibe coding" failures start with a fuzzy brief. Engineers and AI agents alike improvise structure under pressure, accumulating decisions that don't compose. ISB front-loads the structure pass so the build is deterministic from the first line of code.

## Status

**Design phase.** The full system spec lives at [`docs/superpowers/specs/2026-04-22-isb-design.md`](docs/superpowers/specs/2026-04-22-isb-design.md). Implementation is the next sprint.

## Pipeline

1. **Brief intake** — free-form text, ≤500 words
2. **Component extraction** — frontend / backend / data / infra slots
3. **Decision triangulation** — stack, hosting, auth model, persistence, scale ceiling
4. **Acceptance criteria** — testable invariants per component
5. **Output** — single Markdown spec + JSON manifest

## Method

Architecture-first, AI-paired. Designed in a **single April 2026 sprint** with **Claude (Opus 4.6)** as paired implementation and audit partner. Each commit cross-audited: code review, dependency check, spec-quality pass on the brief-to-system pipeline.

## Related

Part of the same "design-before-code" workflow used to build:
- [pechepro](https://github.com/sxc3030-eng/pechepro) — desktop fishing app, 311 tests, shipped from a single design doc
- [FORGE](https://github.com/sxc3030-eng/FORGE) — 3D architecture navigator, 1500+ tests
