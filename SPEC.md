# x402 Reliability Specification

**Version:** 0.2  
**Status:** Working Draft  
**Date:** 2026-08-29

---

## Overview

This document defines the x402 Reliability Specification: a 7-stage pipeline for verifying that an x402 payment endpoint is reliably operational.

An x402 service is **reliable** when it:

1. Is reachable
2. Returns a valid 402 response on unpaid requests
3. Publishes well-formed payment terms
4. Offers a price within an acceptable range
5. Successfully processes a payment
6. Delivers the promised resource after payment
7. Returns a response conforming to its advertised schema

A service must pass all 7 stages to be considered fully verified. Partial verification (stages 1–4 only) is valid and useful when a full payment check is not warranted. An optional **payment readiness verification** tier sits between stages 1–4 and stages 5–7 — see [Payment Readiness Verification](#payment-readiness-verification).

**Why 7 stages and not fewer?** Each stage catches a distinct failure mode. A service can pass availability and still have malformed payment terms (stage 3). It can accept payment and still return an empty body (stage 6). Collapsing stages would hide which part of the x402 contract broke — making it harder to debug and impossible to compare failures across services.

---

## Definitions

**Endpoint** — A URL that implements the x402 payment protocol.

**Check** — A single run of the verification pipeline against an endpoint at a point in time.

**Stage** — One of the 7 verification steps. Each stage has a pass/fail result and an optional evidence payload.

**Evidence record** — The structured output of a check: the endpoint, timestamp, stage results, and overall outcome.

**Synthetic check** — A check initiated by a monitoring tool (not an end user) using a dedicated test wallet.

**Facilitator** — The service that verifies and settles x402 payments. Each service specifies its own facilitator URL in the 402 response — this is not always the default (`https://x402.org/facilitator`). See [Facilitator Discovery](#facilitator-discovery).

---

## The 7 Stages

**Why the lightweight/full split?** Stages 1–4 require no payment and can run every few minutes at near-zero cost. Stages 5–7 spend real money — even at $0.001 per call, running full verification every minute across many services is expensive. The split lets implementations run lightweight checks frequently for uptime detection, and full verification less often for end-to-end confidence. Both produce valid evidence records.

### Stage 1: Availability

**What it checks:** The endpoint returns any HTTP response within the timeout window.

**Method:** HTTP GET (or POST, if specified) to the endpoint URL.

**Pass condition:** HTTP response received within timeout. Any status code counts — including 402, 200, 500.

**Fail condition:** Connection refused, DNS failure, TLS error, or timeout.

**Payment required:** No

**Rationale:** Stage 1 accepts any HTTP response — including 500 — because the goal here is only to confirm the endpoint is reachable. Whether it responds correctly is tested in stage 2 onward. Conflating reachability with correctness makes failures harder to diagnose.

**Evidence fields:**
```json
{
  "stage": 1,
  "name": "availability",
  "passed": true,
  "latency_ms": 234,
  "error": null
}
```

---

### Stage 2: 402 Response

**What it checks:** The endpoint returns HTTP 402 when a request is made without payment.

**Method:** HTTP GET or POST to the endpoint URL, with no payment header.

**Pass condition:** Response status is `402 Payment Required`.

**Fail condition:** Any other status code, including 200 (service should not respond freely), 401, 403, 500.

**Failure severity:** Stages 1–4 failures are **soft failures** — no payment has been made. The service is misbehaving, but no funds are at risk.

**Payment required:** No

**Evidence fields:**
```json
{
  "stage": 2,
  "name": "402_response",
  "passed": true,
  "status_code": 402,
  "error": null
}
```

---

### Stage 3: Payment Terms

**What it checks:** The 402 response body contains well-formed payment terms per the x402 protocol.

**Method:** Parse the body of the 402 response from stage 2.

**x402 protocol reference:** Field definitions follow the [x402 protocol specification](https://github.com/coinbase/x402) (Coinbase). This spec targets x402 as of the date in the header above; if the x402 spec has changed, open an issue.

**Pass condition:** The response body is valid JSON and contains the required x402 payment fields: `accepts`, `error`.

**Required fields:**
- `accepts` — array of one or more accepted payment methods
- Each payment method must include: `scheme`, `network`, `maxAmountRequired`, `resource`, `description`, `mimeType`, `payTo`, `maxTimeoutSeconds`, `asset`, `extra`

**Fail condition:** Body is not JSON, missing required fields, or fields are malformed.

**Facilitator URL:** During stage 3, implementations should discover and record the facilitator URL using the algorithm defined in [Facilitator Discovery](#facilitator-discovery). This URL is used in payment readiness verification and in stage 5.

**Payment required:** No

**Evidence fields:**
```json
{
  "stage": 3,
  "name": "payment_terms",
  "passed": true,
  "scheme": "exact",
  "network": "base",
  "asset": "USDC",
  "facilitator_url": "https://api.bankr.bot/facilitator",
  "facilitator_is_custom": true,
  "error": null
}
```

`facilitator_url` — the discovered facilitator URL (may be the default or a service-specific one).  
`facilitator_is_custom` — `true` if the service specified its own facilitator in the 402 response; `false` if the default was used. Both fields are optional but recommended.

---

### Stage 4: Price Validity

**What it checks:** The price specified in payment terms is within an acceptable range for the service type.

**Method:** Parse `maxAmountRequired` from the payment terms.

**Pass condition:** Price is a valid non-negative number and falls within a reasonable range for the declared service category. Upper bound is implementation-defined but must be documented.

**Fail condition:** Price is zero, negative, non-numeric, or exceeds the defined ceiling for the category.

**Note:** Implementations should document their price validity ceiling. The spec does not mandate a specific ceiling — what is reasonable varies by service type.

**Rationale:** Price validity is deliberately left implementation-defined because x402 services span a wide range — a weather API at $0.001 and a compute service at $0.50 are both legitimate. A hard ceiling in the spec would either block valid high-value services or allow runaway costs in monitoring wallets. The ceiling belongs in the tool, not the standard.

**Payment required:** No

**Evidence fields:**
```json
{
  "stage": 4,
  "name": "price_validity",
  "passed": true,
  "price_usdc": 0.001,
  "currency": "USDC",
  "error": null
}
```

---

### Stage 5: Payment Processing

**What it checks:** The endpoint successfully processes a real on-chain payment.

**Method:** Submit a valid payment (using a synthetic check wallet) per the payment terms returned in stage 3. This includes constructing and signing the payment header per the scheme and network specified in stage 3, then making the authenticated request. The payment mechanism (scheme, network, asset) is determined entirely by the payment terms — this stage does not assume a specific chain or token.

**Facilitator:** Use the facilitator URL discovered in stage 3 via the [Facilitator Discovery](#facilitator-discovery) algorithm. Do not assume `https://x402.org/facilitator` — routing to the wrong facilitator produces an HTTP 500, not a clear payment failure.

**Pass condition:** The endpoint accepts the payment and returns a non-402 response. Response status 200–299 is a clear pass. Some endpoints may return other 2xx or even 3xx codes — implementations may treat these as passing if documented.

**Fail condition:** Endpoint returns 402 again (payment rejected), 4xx (payment invalid), 5xx (server error after payment), or no response.

**Failure severity:** Stage 5 failures are **hard failures** — a payment may have been submitted and the funds are not recoverable by the checker. This is categorically different from stages 1–4 failures, where no payment is made. Implementations must surface this distinction in evidence records and monitoring alerts.

**Payment required:** Yes — a real payment is made using the synthetic check wallet.

**Evidence fields:**
```json
{
  "stage": 5,
  "name": "payment_processing",
  "passed": true,
  "tx_hash": "0xabc...",
  "network": "base",
  "response_status": 200,
  "error": null
}
```

---

### Stage 6: Delivery

**What it checks:** The endpoint delivers the promised resource in the response body after payment.

**Method:** Inspect the response body from stage 5.

**Pass condition:** Response body is non-empty. Where `mimeType` is specified in payment terms, the response `Content-Type` header must match (or be a valid subtype).

**Fail condition:** Response body is empty, or `Content-Type` does not match the declared `mimeType`.

**Failure severity:** Stage 6 failures are **hard failures** — payment was accepted but the promised resource was not delivered. This is the most consequential failure mode in the pipeline: money was spent and nothing was received.

**Payment required:** Yes (same payment as stage 5)

**Evidence fields:**
```json
{
  "stage": 6,
  "name": "delivery",
  "passed": true,
  "content_type": "application/json",
  "body_bytes": 842,
  "error": null
}
```

---

### Stage 7: Schema Validity

**What it checks:** The delivered response conforms to the endpoint's advertised schema.

**Method:** Validate the response body from stage 5 against a schema. Schema source options (in priority order):
1. A schema URL published by the service (e.g. in `extra.schema_url` in payment terms)
2. A schema registered with a known registry
3. A community-contributed schema for this endpoint

**Pass condition:** Response body validates successfully against the schema.

**Fail condition:** Validation errors, schema not found, or body cannot be parsed as the declared content type.

**Note:** Stage 7 is optional when no schema is available. Implementations should record `schema_source: null` and `passed: null` (not `false`) when no schema exists — this is not a failure, it is an absence of evidence.

**Rationale:** `null` vs `false` is a meaningful distinction throughout this spec. `false` means "we checked and it failed." `null` means "we could not check." Recording `false` when no schema exists would make a service look non-conformant for something it has no control over. Consumers of evidence records must treat `null` as "unknown" and `false` as "verified failure."

**Payment required:** Yes (same payment as stage 5)

**Evidence fields:**
```json
{
  "stage": 7,
  "name": "schema_validity",
  "passed": true,
  "schema_source": "service_published",
  "schema_url": "https://api.example.com/schema.json",
  "validation_errors": [],
  "error": null
}
```

---

## Facilitator Discovery

The facilitator is the service that verifies and settles x402 payments. The x402 protocol allows each service to specify its own facilitator URL directly in the 402 response — this is not always `https://x402.org/facilitator`. Routing to the wrong facilitator is a silent failure: the facilitator returns HTTP 500 ("No facilitator registered for scheme: exact and network: base"), which looks like a service error but is actually a routing error. These are not the same thing, and implementations must distinguish them.

### Discovery algorithm

Check the following locations in the 402 response, in order, and use the first valid HTTPS URL found:

1. `matchingOption.extra.facilitator`
2. `matchingOption.facilitator` (top level of the accepted payment option)
3. `paymentRequirements.facilitator` (root of the payment terms object)

If no facilitator URL is found at any level, fall back to the default: `https://x402.org/facilitator`.

### Implementation note

Record the discovered facilitator URL in the stage 3 evidence as `facilitator_url`. Set `facilitator_is_custom: true` when a service-specific URL was found, `false` when the default was used. This lets evidence consumers identify which facilitator handled (or would handle) each check without re-parsing the 402 response.

### Why this matters

In practice, x402 services use different facilitators — x402.org is not a registry for all x402 services. A conforming implementation that assumes x402.org is universal will produce incorrect results for any service using a custom facilitator. The 402 response is the authoritative source; always read it.

---

## Payment Readiness Verification

Payment readiness verification is an optional pre-flight check that confirms a payment *would* succeed, without spending USDC. It calls the facilitator's `/verify` endpoint with a signed EIP-3009 authorization — the same authorization that would be submitted in a real payment — but does not call `/settle`. No funds move.

This check sits between the lightweight check (stages 1–4) and full verification (stages 5–7). It catches a class of failures that stage 4 misses but that don't warrant spending USDC to confirm: facilitator routing failures, wallet authorization rejections, and TTL mismatches.

### When to use it

- As a middle tier between lightweight checks and full verification
- To detect facilitator routing failures before spending USDC
- As a pre-flight before a scheduled full verification run
- Recommended cadence: **15 minutes** (adds ~95 ms per service at this frequency)

### What it validates

- The facilitator URL is discoverable from the 402 response
- The facilitator accepts the wallet's signed authorization (`isValid: true`)
- The EIP-3009 signature is well-formed and within the TTL window

### What it does NOT validate

- That the service will deliver the resource after payment (stage 6)
- That on-chain settlement will succeed
- Schema conformance (stage 7)

### Replay risk

The signed EIP-3009 authorization has a short TTL (`maxTimeoutSeconds` from payment terms — typically 60 s). The `/verify` call goes to the same facilitator the service already trusts, so the facilitator holds the signed authorization during any real payment flow anyway. The window where a facilitator could call `/settle` before TTL expiry is narrow and is not meaningfully different from the risk in a real paid check.

### Evidence record

Payment readiness verification produces its own evidence record, separate from stage-based records.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `check_type` | string | Always `"payment_readiness"` |
| `spec_version` | string | Version of this spec |
| `endpoint` | string | The endpoint URL checked |
| `checked_at` | string | ISO 8601 timestamp |
| `network` | string | Blockchain network |
| `facilitator_url` | string | Facilitator URL used for /verify |
| `facilitator_is_custom` | boolean | Whether a service-specified facilitator was used |
| `facilitator_responded` | boolean | Whether the facilitator returned any response |
| `is_valid` | boolean or null | Facilitator's verdict. `null` if the facilitator did not respond |
| `authorization_ttl_seconds` | integer | TTL of the signed EIP-3009 authorization |
| `status` | string | `"ready"`, `"not_ready"`, or `"error"` |

**Example — passing:**
```json
{
  "check_type": "payment_readiness",
  "spec_version": "0.2",
  "endpoint": "https://api.bankr.bot/v1/search",
  "checked_at": "2026-08-28T12:00:00Z",
  "network": "base",
  "facilitator_url": "https://api.bankr.bot/facilitator",
  "facilitator_is_custom": true,
  "facilitator_responded": true,
  "is_valid": true,
  "invalid_reason": null,
  "authorization_ttl_seconds": 60,
  "latency_ms": 95,
  "status": "ready"
}
```

### Error codes

| Code | Meaning |
|------|---------|
| `FACILITATOR_NOT_REGISTERED` | Facilitator returned HTTP 500 "No facilitator registered" — wrong facilitator was routed |
| `VERIFY_TIMEOUT` | Facilitator did not respond within the timeout window |
| `VERIFY_REJECTED` | Facilitator responded `isValid: false` |
| `PAYMENT_TERMS_ERROR` | Could not parse payment terms — stage 3 would also fail |

`FACILITATOR_NOT_REGISTERED` is not a service failure — it is a facilitator routing failure. Implementations should display it distinctly from a payment rejection.

---

## Evidence Record Format

A complete check produces a single evidence record. See `schema/evidence-record.json` for the full JSON Schema.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `spec_version` | string | Version of this spec (e.g. `"0.2"`) |
| `endpoint` | string | The URL that was checked |
| `checked_at` | string | ISO 8601 timestamp |
| `network` | string | Blockchain network (e.g. `"base"`, `"ethereum"`) |
| `stages` | array | Results for each stage attempted |
| `overall_passed` | boolean | True only if all attempted stages passed |
| `highest_stage_passed` | integer | The highest stage number that passed (0 if none) |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `checker_id` | string | Identifier of the tool or service that ran the check |
| `check_wallet` | string | Public address of the synthetic check wallet |
| `notes` | string | Free-text notes from the checker |

---

## Partial Verification

A check may stop early — either by design (lightweight check) or due to failure.

- **Stages 1–4 only:** Valid as a lightweight check. No payment is made. Record `highest_stage_passed` accordingly.
- **Stage failure:** When a stage fails, subsequent stages that depend on it MUST NOT be run (e.g. if stage 5 fails, stages 6 and 7 cannot be attempted). Record the failed stage, and mark remaining stages as `"skipped"`.

---

## Check Frequency Guidance

This spec does not mandate check frequency. Implementers should consider:

| Check type | Stages | Recommended cadence | Cost |
|---|---|---|---|
| Lightweight | 1–4 | Every 1–15 min | Near-zero |
| Payment readiness | 1–4 + /verify | Every 15 min | Near-zero (no USDC spent) |
| Full verification | 1–7 | Every 1–24 hr | Real USDC per check |

The three tiers are complementary, not mutually exclusive. A typical implementation runs lightweight checks frequently for uptime detection, payment readiness checks at a moderate cadence to catch facilitator issues early, and full verification on a longer cycle for end-to-end confidence.

---

## Failure Severity Summary

Stages divide into two severity classes based on whether a payment has been attempted:

| Stages | Severity | What it means |
|--------|----------|---------------|
| 1–4 | **Soft failure** | No payment made. Service is misbehaving but no funds are at risk. |
| 5–7 | **Hard failure** | Payment was submitted or completed. Funds may not be recoverable. |

Implementations should surface this distinction in alerts and dashboards. A hard failure at stage 6 (payment accepted, no delivery) warrants immediate notification; a soft failure at stage 2 warrants a degraded status.

---

## Reference Test Vectors

Test vectors allow implementations to self-certify without a live x402 endpoint. A test vector is a mock HTTP exchange — a defined request and response — that a conforming runner must process into a specific evidence record.

Test vectors live in `test-vectors/` in this repository. Each vector is a directory containing:
- `request.json` — the mock HTTP request
- `response-stage-N.json` — the mock HTTP response for each stage
- `expected-evidence-record.json` — the evidence record a conforming runner must produce

**Status:** Test vectors are planned for v0.3. Contributions welcome — see [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## Versioning

This spec follows semantic versioning. The `spec_version` field in evidence records must reference the version used.

- **Patch** (0.1.x): Clarifications, wording fixes, no schema changes
- **Minor** (0.x): New optional fields, new stage metadata — backward compatible
- **Major** (x.0): Breaking changes to required fields or stage definitions

---

## Conformance

An implementation conforms to this spec if:

1. It runs stages in order (1 through 7)
2. It skips subsequent stages after a failure
3. It produces evidence records matching the JSON Schema in `schema/evidence-record.json`
4. It correctly records `passed: null` (not `false`) when a stage is skipped or unattempted
5. It does not modify or discard stage results to improve apparent reliability
6. It discovers the facilitator URL from the 402 response using the algorithm in [Facilitator Discovery](#facilitator-discovery) rather than assuming a fixed facilitator

---

## Acknowledgements

This spec was authored by [CORTX](https://usecortx.dev). The x402 protocol is developed by [Coinbase](https://github.com/coinbase/x402).
