# Changelog

All notable changes to the x402 Reliability Spec will be documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.3.0] — 2026-09-28

Brings the spec in line with x402 as deployed today, from a month of running the reference implementation against live services.

### Added
- **x402 Protocol Versions** — conforming checkers must read both V1 (terms in the 402 body, `X-PAYMENT`, `X-PAYMENT-RESPONSE`) and V2 (base64 `PAYMENT-REQUIRED` / `PAYMENT-SIGNATURE` / `PAYMENT-RESPONSE` headers, atomic `amount`, CAIP-2 networks). A v0.2-conforming checker failed every V2 service at stage 3.
- **Checker-Side Errors** — a check that fails because of the checker (empty or unconfigured wallet, budget spent, unsupported payment method, checker crash) records `outcome: "checker_error"` and `fault: "checker"`, and must never count against the service, open incidents or alert its owner.
- **Test Input** — checkers should send a known-good input (`owner_provided`, `service_example`) and record `input_source`; failures seen with `none` should not raise incidents on their own.
- **Settlement receipt** in stage 5: `settlement_status` (`confirmed` / `failed` / `unconfirmed`), `tx_hash` taken only from the receipt (never invented), `receipt_header`. Read on every outcome so "paid but not delivered" is provable.
- **Error Codes** — a standard `error_code` per stage, and `fault` on failed stages.
- **Reference test vectors** (`test-vectors/`, 8 vectors) with a documented format and comparison rules. CI validates every expected record.
- Evidence record fields: `outcome`, `check_type`, `x402_version`, `input_source`. Stage fields: `error_code`, `fault`, `method`, `x402_version`, `terms_source`, `facilitator_url`, `price_atomic_units`, `payment_header`, `settlement_status`, `receipt_header`.
- Payment readiness: status `unavailable` (facilitator not published, or it requires credentials); codes `FACILITATOR_NOT_PUBLISHED`, `FACILITATOR_AUTH_REQUIRED`, `FACILITATOR_UNREACHABLE`, `FACILITATOR_ERROR`, `VERIFY_REJECTED_CHECKER_SIDE`, `VERIFY_REQUEST_REJECTED`; which `invalidReason` values are service-side; readiness authorizations count against the checker's spending limits.
- Stage 1 safety: checkers must refuse private and internal addresses (`BLOCKED_ADDRESS`).
- Conformance items 7–10 (both protocol versions, checker-side errors, receipt-only `tx_hash`, test vectors).

### Changed
- **Facilitator Discovery no longer falls back to `https://x402.org/facilitator`.** x402 keeps the facilitator a server-side choice; guessing one made healthy services look "not ready". No published facilitator → readiness `unavailable`. `facilitator_is_custom` is deprecated.
- **Stage 5 does not route through a facilitator.** The checker pays the service; the service talks to its own facilitator. (v0.2 said otherwise.)
- Stage 3 required fields are now `scheme`, `network`, price, `asset`, `payTo`; `resource`, `description`, `mimeType`, `maxTimeoutSeconds`, `extra` are recommended. Real services omit them and still accept payment.
- Stage 4 reads V2 `amount` as atomic units, and V1 `maxAmountRequired` as atomic unless it is not an integer ≥ 1.
- Stage 7: a JSON service that returns non-JSON fails (`INVALID_JSON`) even without a schema.

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
