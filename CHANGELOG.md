# Changelog

All notable changes to the Concordex SDK specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning per `sdk-spec.md` §11.

## [Unreleased]

## [0.5.0] — 2026-05-30

First spec-gated release. Establishes the four-SDK lockstep contract.

### Added
- Canonical surface defined in `sdk-spec.md` (§§1–11).
- JSON Schemas for the two endpoints currently in scope:
  `/v1/agent-stream/event` and `/v1/cb/check`.
- Contract test corpus: `golden-envelopes.json`, `signature-vectors.json`,
  `error-mapping.json`.
- Webhook signature verification helper specification (§9).
- Naming map for Python / TypeScript / C# / Java (§8).
- Coordinated release workflow (`promote.yml`).

### Changed
- Python SDK supersedes its unilateral `0.1.0` line; new starting point
  is the lockstep `0.5.0` across all four implementations.

### Migration notes (for the existing Python SDK consumers)
- `EmitResult` gains optional fields (`frame_id`, `subject_id`,
  `outcome`, `triage_decision`, `tags_fired`, `scored_by_canons`,
  `soul_version`, `ledger_index`, `follow_my_data`) populated in
  synchronous mode. Async mode (`X-Concordex-Async: true`) preserves
  the previous minimal envelope.
- No breaking changes to method signatures.
