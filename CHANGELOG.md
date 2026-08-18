# Changelog

All notable changes to the x402 Reliability Spec will be documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.1.1] — 2026-08-18

### Added
- GitHub Actions CI: validates all `examples/*.json` against `schema/evidence-record.json` on every PR
- Issue templates: ambiguity report, edge case discussion
- `CODE_OF_CONDUCT.md`

### Changed
- Stage 3: added reference link to the Coinbase x402 protocol specification

---

## [0.1.0] — 2026-08-18

### Added
- Initial working draft of the 7-stage x402 Reliability Specification
- JSON Schema for stage results (`schema/check-result.json`)
- JSON Schema for evidence records (`schema/evidence-record.json`)
- Example records: passing all stages, failing stage 2, failing stage 5
- `CONTRIBUTING.md` with contribution workflow
- `README.md` with overview and repo structure
