# Changelog

All notable changes to the x402 Reliability Spec will be documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.2.0] — 2026-08-29

### Added
- **Facilitator Discovery** — new section defining the 3-level algorithm for discovering the correct facilitator URL from a 402 response (`matchingOption.extra.facilitator` → `matchingOption.facilitator` → `paymentRequirements.facilitator`). Routing to the wrong facilitator is a silent failure; this section explains the distinction between a facilitator routing error and a service failure
- **Payment Readiness Verification** — new optional pre-flight tier between stages 1–4 and stages 5–7. Calls the facilitator's `/verify` endpoint with a signed EIP-3009 authorization to confirm a payment would succeed, without calling `/settle`. No USDC moves. Includes evidence record format, error codes (`FACILITATOR_NOT_REGISTERED`, `VERIFY_TIMEOUT`, `VERIFY_REJECTED`, `PAYMENT_TERMS_ERROR`), and replay risk assessment
- **`examples/payment-readiness-check.json`** — example evidence record for a passing payment readiness check
- **Conformance item 6** — implementations must discover the facilitator URL from the 402 response rather than assuming a fixed facilitator

### Changed
- Stage 3 evidence fields: added optional `facilitator_url` and `facilitator_is_custom` (recommended for all conforming implementations)
- Stage 5: added note to use the facilitator URL discovered in stage 3 rather than a hardcoded default
- Check Frequency Guidance: expanded table to include the payment readiness tier (15 min, near-zero cost)
- Definitions: added **Facilitator** entry
- Overview: added cross-reference to Payment Readiness Verification section

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
