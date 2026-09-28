# x402 Reliability Specification

**Version:** 0.3  
**Status:** Working Draft  
**Date:** 2026-09-28

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

A check can also fail for reasons that have nothing to do with the service — the checker's own wallet is empty, its budget is spent, it cannot build a payment the service accepts. These are **checker-side errors**. They are recorded, but they never count against the service. See [Checker-Side Errors](#checker-side-errors).

**Why 7 stages and not fewer?** Each stage catches a distinct failure mode. A service can pass availability and still have malformed payment terms (stage 3). It can accept payment and still return an empty body (stage 6). Collapsing stages would hide which part of the x402 contract broke — making it harder to debug and impossible to compare failures across services.

---

## Definitions

**Endpoint** — A URL that implements the x402 payment protocol.

**Check** — A single run of the verification pipeline against an endpoint at a point in time.

**Stage** — One of the 7 verification steps. Each stage has a pass/fail result and an optional evidence payload.

**Evidence record** — The structured output of a check: the endpoint, timestamp, stage results, and overall outcome.

**Synthetic check** — A check initiated by a monitoring tool (not an end user) using a dedicated test wallet.

**Facilitator** — The service that verifies and settles x402 payments on behalf of the resource server. The resource server chooses its facilitator; clients (including checkers) never call it during a payment. Some services publish their facilitator URL in the 402 response. See [Facilitator Discovery](#facilitator-discovery).

**Checker-side error** — A check that could not be completed for a reason attributable to the checker, not the service. See [Checker-Side Errors](#checker-side-errors).

**Test input** — The request body or query a checker sends to a service that needs input. See [Test Input](#test-input).

---

## x402 Protocol Versions

x402 has two wire formats in use. Conforming checkers MUST support both.

| | x402 V1 | x402 V2 |
|---|---|---|
| Payment terms (402 response) | JSON body: `{ x402Version: 1, accepts: [...] }` | `PAYMENT-REQUIRED` response header, base64-encoded JSON `PaymentRequired` (`{ x402Version: 2, resource, accepts: [...] }`). The body may be empty. |
| Price field | `maxAmountRequired` — atomic units of the asset (see note) | `amount` — always atomic units of the asset |
| Network identifier | Short name, e.g. `base` | CAIP-2, e.g. `eip155:8453` |
| Payment header (client → server) | `X-PAYMENT` | `PAYMENT-SIGNATURE`, base64 JSON `PaymentPayload` whose `accepted` field echoes the chosen payment requirement exactly as received |
| Settlement receipt (server → client) | `X-PAYMENT-RESPONSE` | `PAYMENT-RESPONSE`, base64 JSON `SettlementResponse` (`success`, `transaction`, `network`, `payer`, `errorReason`) |

**Reading payment terms.** Checkers read terms from the 402 body first, then from the `PAYMENT-REQUIRED` header. The declared `x402Version` decides the version; terms found only in `PAYMENT-REQUIRED` without a declared version are V2. Checkers MAY also accept the non-standard `X-PAYMENT-REQUIRED` header that some gateways send.

**Header encoding.** V2 headers are base64-encoded JSON. Checkers SHOULD also accept plain JSON in these headers, which some servers send.

**Networks.** Short names and CAIP-2 identifiers that refer to the same chain are equivalent (`base` ≡ `eip155:8453`, `base-sepolia` ≡ `eip155:84532`). Record the identifier exactly as the service sent it.

**Price units.** V2 `amount` is atomic units. V1 `maxAmountRequired` is atomic units per the protocol, but some V1 gateways send a decimal amount (`"0.001"`). Checkers SHOULD treat an integer value ≥ 1 as atomic and any other numeric value as a decimal amount of the asset, and MUST record which interpretation was used (`price_atomic_units`).

Protocol reference: [x402 specification](https://github.com/coinbase/x402) (Coinbase / x402 Foundation).

---

## The 7 Stages

**Why the lightweight/full split?** Stages 1–4 require no payment and can run every few minutes at near-zero cost. Stages 5–7 spend real money — even at $0.001 per call, running full verification every minute across many services is expensive. The split lets implementations run lightweight checks frequently for uptime detection, and full verification less often for end-to-end confidence. Both produce valid evidence records.

### Stage 1: Availability

**What it checks:** The endpoint returns any HTTP response within the timeout window.

**Method:** HTTP GET (or POST, if specified) to the endpoint URL. For services that take input, send the [test input](#test-input).

**Pass condition:** HTTP response received within timeout. Any status code counts — including 402, 200, 500.

**Fail condition:** Connection refused, DNS failure, TLS error, or timeout.

**Safety:** A checker makes requests to URLs supplied by other people. It MUST refuse to connect to private, loopback, link-local or otherwise non-public addresses (checking the address actually connected to, not only the one first resolved), and SHOULD refuse redirects to them. A refused target fails stage 1 with `BLOCKED_ADDRESS`.

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

**Method:** HTTP GET or POST to the endpoint URL, with no payment header. Some services only require payment on one method; checkers SHOULD use the method the service documents and MAY retry with the other method before failing. Record the method that produced the 402 (`method`).

**Pass condition:** Response status is `402 Payment Required`. A V2 402 with an empty body is valid — the terms are in the `PAYMENT-REQUIRED` header.

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

**What it checks:** The 402 response carries well-formed payment terms per the x402 protocol, in either version.

**Method:** Read the terms from the 402 response of stage 2 as described in [x402 Protocol Versions](#x402-protocol-versions), then select the payment option the checker will use (a network and asset it supports).

**x402 protocol reference:** Field definitions follow the [x402 protocol specification](https://github.com/coinbase/x402). This spec targets x402 as of the date in the header above; if the x402 spec has changed, open an issue.

**Pass condition:** Terms are found and parse as JSON, `accepts` holds at least one payment option, and the selected option has the required fields.

**Required fields (selected option):** `scheme`, `network`, price (`amount` in V2, `maxAmountRequired` in V1), `asset`, `payTo`.

**Recommended fields:** `maxTimeoutSeconds`, `extra` (for EVM `exact` payments: the token's EIP-712 `name` and `version`), and in V1 `resource`, `description`, `mimeType`. A missing recommended field SHOULD be recorded but does not fail the stage — real services omit them and still accept payment.

**Fail condition:** No terms found, terms are not JSON, `accepts` is empty, no option on a network the checker supports (`UNSUPPORTED_NETWORK`), or the selected option lacks a required field (`MISSING_FIELDS`).

**Facilitator URL:** During stage 3, implementations SHOULD record the facilitator URL if the service publishes one (see [Facilitator Discovery](#facilitator-discovery)). It is used only by payment readiness verification — not by stage 5.

**Payment required:** No

**Evidence fields:**
```json
{
  "stage": 3,
  "name": "payment_terms",
  "passed": true,
  "x402_version": 2,
  "terms_source": "payment-required",
  "scheme": "exact",
  "network": "eip155:8453",
  "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "facilitator_url": "https://api.bankr.bot/facilitator",
  "error": null
}
```

`x402_version` — `1` or `2`.  
`terms_source` — where the terms were read from: `body`, `payment-required` or `x-payment-required`.  
`facilitator_url` — the facilitator URL the service published, or `null` if it published none.  
`facilitator_is_custom` — deprecated in v0.3 (there is no default facilitator any more); if present, `true` means the service published its own facilitator.

---

### Stage 4: Price Validity

**What it checks:** The price specified in payment terms is within an acceptable range for the service type.

**Method:** Read the price of the selected option (`amount` in V2, `maxAmountRequired` in V1) and convert it to the asset's decimal amount as described in [Price units](#x402-protocol-versions).

**Pass condition:** Price is a valid positive number and falls within a reasonable range for the declared service category. Upper bound is implementation-defined but must be documented.

**Fail condition:** Price is zero (`ZERO_PRICE`), negative or non-numeric (`INVALID_PRICE_FORMAT`), or exceeds the defined ceiling (`PRICE_EXCEEDS_MAXIMUM`).

**Input-dependent prices:** Some services quote a price based on the request — and quote `0` for an empty or placeholder input. A zero price seen with no [test input](#test-input) (`input_source: "none"`) may say more about the checker's request than about the service; see [Test Input](#test-input).

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
  "price_atomic_units": true,
  "currency": "USDC",
  "error": null
}
```

---

### Stage 5: Payment Processing

**What it checks:** The endpoint successfully processes a real on-chain payment.

**Method:** Sign a payment for the selected option with the synthetic check wallet and repeat the request (same method, same test input) with the payment header for the protocol version: `X-PAYMENT` (V1) or `PAYMENT-SIGNATURE` (V2). The payment mechanism (scheme, network, asset) is determined entirely by the payment terms — this stage does not assume a specific chain or token.

**Facilitator:** The checker does not contact any facilitator in stage 5. The service forwards the payment to its own facilitator to verify and settle it. (v0.2 said to route stage 5 through the discovered facilitator; that was wrong.)

**Pass condition:** The endpoint accepts the payment and returns a 2xx response. Some endpoints may return 3xx codes — implementations may treat these as passing if documented.

**Fail condition:** Endpoint returns 402 again (payment rejected), another 4xx, 5xx (server error after payment), or no response. Record the status as `response_status`.

**Checker-side failures:** If the checker cannot produce a payment at all — its wallet is empty or unconfigured, its budget is spent, it does not support the payment scheme — stage 5 records `passed: false` with `fault: "checker"` and the check's outcome is `checker_error`. See [Checker-Side Errors](#checker-side-errors).

**Settlement receipt:** After the paid request, read the settlement receipt header (`PAYMENT-RESPONSE` in V2, `X-PAYMENT-RESPONSE` in V1) — on every outcome, including failures, so "paid but not delivered" can be proven. Record:

- `settlement_status` — `confirmed` (receipt says success and names a transaction), `failed` (receipt says the settlement failed), or `unconfirmed` (no readable receipt, or success without a transaction).
- `tx_hash` — only from the receipt. Implementations MUST NOT invent or guess it; with no receipt it is `null`.
- `receipt_header` — the header the receipt came from, or `null`.

A missing receipt does not fail stage 5 by itself — many services do not send one — but the evidence must say `unconfirmed`.

**Failure severity:** Stage 5 failures are **hard failures** — a payment authorization was sent and may have been settled; the funds are not recoverable by the checker. The settlement receipt, when present, shows whether money actually moved. This is categorically different from stages 1–4 failures, where no payment is made. Implementations must surface this distinction in evidence records and monitoring alerts.

**Payment required:** Yes — a real payment is made using the synthetic check wallet.

**Evidence fields:**
```json
{
  "stage": 5,
  "name": "payment_processing",
  "passed": true,
  "response_status": 200,
  "payment_header": "PAYMENT-SIGNATURE",
  "settlement_status": "confirmed",
  "tx_hash": "0xabc...",
  "receipt_header": "PAYMENT-RESPONSE",
  "network": "eip155:8453",
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

**Fail condition:** Validation errors (`SCHEMA_VALIDATION_FAILED`), or the body cannot be parsed as the declared content type (`INVALID_JSON` for JSON).

**Note:** Stage 7 is optional when no schema is available. Implementations should record `schema_source: null` and `passed: null` (not `false`) when no schema exists — this is not a failure, it is an absence of evidence. **Exception:** when the service declares or returns JSON (`application/json`) and the body is not valid JSON, stage 7 fails with `INVALID_JSON` even without a schema — a JSON service that returns non-JSON is broken regardless of schema.

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

The facilitator is the service that verifies and settles x402 payments for a resource server. In the x402 protocol the facilitator is the **server's** choice and is opaque to clients: a client pays the resource server, and the server talks to its facilitator. Some services nonetheless publish their facilitator URL in the 402 response, which lets a checker run [payment readiness verification](#payment-readiness-verification).

### Discovery algorithm

Check the following locations in the 402 response, in order, and use the first valid HTTPS URL found (both `facilitator` and `facilitatorUrl` keys):

1. `selectedOption.extra.facilitator`
2. `selectedOption.facilitator` (top level of the selected payment option)
3. `paymentRequired.facilitator` (root of the payment terms object)

If no facilitator URL is found, the facilitator is **unknown**. Record `facilitator_url: null`. Do **not** fall back to a default facilitator.

### Why there is no default (changed in v0.3)

v0.2 told checkers to fall back to `https://x402.org/facilitator`. That produces wrong results: a service that uses another facilitator gets verified against one that has never heard of it and appears "not ready" when nothing is wrong. Payment readiness for a service that does not publish its facilitator is `unavailable`, not a failure.

### Routing errors

A published facilitator that answers HTTP 500 "No facilitator registered for scheme … and network …" is misconfigured for this service. That is a real problem for agents paying the service, and it is distinct from a payment rejection; record it as `FACILITATOR_NOT_REGISTERED`.

---

## Payment Readiness Verification

Payment readiness verification is an optional pre-flight check that confirms a payment *would* succeed, without spending USDC. It calls the facilitator's `/verify` endpoint with a signed EIP-3009 authorization — the same authorization that would be submitted in a real payment — but does not call `/settle`. No funds move.

This check sits between the lightweight check (stages 1–4) and full verification (stages 5–7). It catches a class of failures that stage 4 misses but that don't warrant spending USDC to confirm: facilitator routing failures, wallet authorization rejections, and TTL mismatches.

### When to use it

- As a middle tier between lightweight checks and full verification
- To detect facilitator routing failures before spending USDC
- As a pre-flight before a scheduled full verification run
- Recommended cadence: **15 minutes** (adds ~95 ms per service at this frequency)

Payment readiness is only possible when the service publishes its facilitator (see [Facilitator Discovery](#facilitator-discovery)) and that facilitator accepts `/verify` requests from the checker. Otherwise the result is `unavailable` — never a failure.

### What it validates

- The facilitator URL is published in the 402 response
- The facilitator accepts the wallet's signed authorization (`isValid: true`)
- The EIP-3009 signature is well-formed and within the TTL window

### What it does NOT validate

- That the service will deliver the resource after payment (stage 6)
- That on-chain settlement will succeed
- Schema conformance (stage 7)

### Replay risk

The signed EIP-3009 authorization has a short TTL (`maxTimeoutSeconds` from payment terms — typically 60 s). The `/verify` call goes to the same facilitator the service already trusts, so the facilitator holds the signed authorization during any real payment flow anyway. The window where a facilitator could call `/settle` before TTL expiry is narrow and is not meaningfully different from the risk in a real paid check. Because it is possible, checkers MUST only sign readiness authorizations for amounts they would be willing to pay in a full check, and SHOULD count them against the same spending limits.

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
| `facilitator_url` | string or null | Facilitator URL used for /verify; `null` when the service publishes none (`unavailable`) |
| `facilitator_is_custom` | boolean | Deprecated in v0.3 (there is no default facilitator). Optional; `true` when present |
| `facilitator_responded` | boolean | Whether the facilitator returned any response |
| `is_valid` | boolean or null | Facilitator's verdict. `null` if the facilitator did not respond |
| `authorization_ttl_seconds` | integer | TTL of the signed EIP-3009 authorization |
| `status` | string | `"ready"`, `"not_ready"`, `"unavailable"`, or `"error"` |

**Statuses:**

- `ready` — the facilitator accepted the authorization (`isValid: true`).
- `not_ready` — a real agent's payment would fail for a service-side reason (see error codes).
- `unavailable` — the check could not run: the service publishes no facilitator, or the facilitator requires credentials the checker does not have. Not a failure. Checkers SHOULD re-try `unavailable` services rarely (e.g. daily).
- `error` — a [checker-side error](#checker-side-errors). Not counted against the service.

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
| Code | Status | Meaning |
|------|--------|---------|
| `FACILITATOR_NOT_PUBLISHED` | `unavailable` | The service publishes no facilitator URL |
| `FACILITATOR_AUTH_REQUIRED` | `unavailable` | The facilitator answered 401/403 — it only serves its own customers |
| `FACILITATOR_NOT_REGISTERED` | `not_ready` | The published facilitator answered 500 "No facilitator registered" for this scheme and network — the service is routed to a facilitator that cannot settle for it |
| `FACILITATOR_UNREACHABLE` / `VERIFY_TIMEOUT` | `not_ready` | The published facilitator did not answer — real payments would fail too |
| `FACILITATOR_ERROR` | `not_ready` | The facilitator answered 5xx without a verdict |
| `VERIFY_REJECTED` | `not_ready` | `isValid: false` with a service-side `invalidReason`: `invalid_exact_evm_payload_recipient_mismatch`, `invalid_network`, `invalid_payment_requirements`, `invalid_scheme`, `unsupported_scheme` |
| `VERIFY_REJECTED_CHECKER_SIDE` | `error` | `isValid: false` for any other reason (signature, amount, timing, balance) — the checker may have built the authorization wrong |
| `VERIFY_REQUEST_REJECTED` | `error` | The facilitator answered another 4xx without a verdict — it did not accept how the checker built the request |
| `PAYMENT_TERMS_ERROR` | `not_ready` | Could not parse payment terms — stage 3 would also fail |

`FACILITATOR_NOT_REGISTERED` and `VERIFY_REJECTED` should be displayed distinctly: the first is a routing problem, the second a payment rejection.

---

## Checker-Side Errors

A reliability record is only useful if its failures are the service's failures. When a check cannot be completed because of the checker, blaming the service is worse than recording nothing: it produces false incidents, false alerts to builders, and lower reliability scores for services that did nothing wrong.

**A checker-side error is any of:**

| Code | Meaning |
|------|---------|
| `WALLET_NOT_CONFIGURED` | The checker has no payment wallet |
| `INSUFFICIENT_BALANCE` | The checker's wallet cannot cover the price |
| `BALANCE_READ_FAILED` | The checker could not read its own balance |
| `SPEND_CAP_EXCEEDED` | The checker's own budget (daily, monthly or per-service) is spent. Implementations MAY use more specific codes (`DAILY_SPEND_CAP_EXCEEDED`, `MONTHLY_SPEND_CAP_EXCEEDED`) |
| `SPEND_RESERVATION_FAILED` | The checker could not reserve budget for the payment |
| `PAYMENT_TIMEOUT` | The checker timed out producing the payment |
| `UNSUPPORTED_PAYMENT_METHOD` | The checker does not support the payment method the service requires (e.g. a transfer method it cannot sign) |
| `CHECKER_ERROR` | Any other internal failure of the checker |

**Rules:**

1. The stage where it happened records `passed: false`, `fault: "checker"`, and the code in `error_code`. Later stages are skipped (`passed: null`).
2. The evidence record's `outcome` is `checker_error` and `overall_passed` is `false`.
3. Checker-side errors MUST NOT be counted in a service's failure rate, uptime or reliability score, MUST NOT open incidents against the service, and MUST NOT alert the service's owner. They MAY be shown as "not checked".
4. Consumers MUST read `outcome` when present, and MUST NOT treat `overall_passed: false` alone as a service failure.

---

## Test Input

Many x402 services take input — a query, a URL, a prompt. A checker that sends an empty or made-up body can get an error, a different price (often `0`), or a response that says nothing about what real agents receive.

**Known-good input.** Checkers SHOULD send an input known to be valid for the service, from these sources in order:

1. `owner_provided` — the service owner gave the checker an input.
2. `service_example` — the service published an example, e.g. in its discovery listing (such as an input example in a Bazaar listing) or its schema.
3. `none` — no known-good input; the checker sent an empty body or no body.

Record the source in the evidence record as `input_source`.

**Interpretation.** For a service that takes input, failures at stages 4–7 with `input_source: "none"` may reflect the checker's request rather than the service. Checkers SHOULD NOT open incidents or alert owners on that basis alone.

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
| `outcome` | string | `passed`, `failed` (the service failed a stage), `partial` (stages 1–4 only, all passed), or `checker_error` (see [Checker-Side Errors](#checker-side-errors)). Conforming v0.3 checkers MUST set it. |
| `check_type` | string | `lightweight` (stages 1–4) or `full` (stages 1–7) |
| `x402_version` | integer | `1` or `2`, when terms were read |
| `input_source` | string | `owner_provided`, `service_example` or `none` — see [Test Input](#test-input) |

**Stage result fields (all stages, optional):**

| Field | Type | Description |
|-------|------|-------------|
| `error_code` | string | Machine-readable code for a failed stage (see [Error Codes](#error-codes)) |
| `fault` | string | `service` or `checker` for a failed stage; `null` otherwise |

---

## Error Codes

Stage results SHOULD carry a machine-readable `error_code` alongside the human-readable `error`. The codes below are used by the [test vectors](#reference-test-vectors); implementations MAY add their own.

| Stage | Codes |
|-------|-------|
| 1 Availability | `UNREACHABLE`, `TIMEOUT`, `INVALID_URL`, `BLOCKED_ADDRESS` |
| 2 402 Response | `UNEXPECTED_STATUS` |
| 3 Payment Terms | `INVALID_PAYMENT_TERMS`, `UNSUPPORTED_NETWORK`, `MISSING_FIELDS` |
| 4 Price Validity | `INVALID_PRICE_FORMAT`, `ZERO_PRICE`, `PRICE_EXCEEDS_MAXIMUM` |
| 5 Payment Processing | `UNEXPECTED_STATUS`, `NO_RESPONSE`, `PAYMENT_SIGNING_FAILED`, plus the [checker-side codes](#checker-side-errors) |
| 6 Delivery | `EMPTY_BODY`, `CONTENT_TYPE_MISMATCH` |
| 7 Schema Validity | `INVALID_JSON`, `SCHEMA_VALIDATION_FAILED` |

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

Test vectors let implementations self-certify without a live x402 endpoint. A test vector is a mock x402 service — what it answers without payment and with payment — plus the evidence record a conforming checker must produce from it.

Test vectors live in `test-vectors/`. Each vector is a directory containing:

- `vector.json`
  - `description` — what the vector tests
  - `checker` — checker settings for this vector: `max_price_usdc`, `schema` (JSON Schema for stage 7, or `null`), `wallet_balance_usdc` (the balance the checker's wallet must report; checkers simulate it)
  - `unpaid` — the response to a request without payment: `status`, `headers`, `body`
  - `paid` — the response to a request carrying a payment header, or `null` if the checker must not pay
- `expected.json` — the expected evidence record

**How to compare.** Run the checker against a local server that answers as the vector says. Compare the produced record with `expected.json` on: `outcome`, `overall_passed`, `highest_stage_passed`, `x402_version`, and for each stage `stage`, `name`, `passed`, `error_code`, `fault`, plus stage 5's `settlement_status` and `tx_hash` when present. Ignore timestamps, latencies, `endpoint`, `checker_id` and free-text `error`. In `vector.json`, a header value written as a JSON object means "this JSON, base64-encoded"; a body written as an object or array means "this JSON as text"; `{{ENDPOINT}}` stands for the mock service's URL and `{{CHECKER_WALLET}}` for the checker's wallet address. Full format: [`test-vectors/README.md`](./test-vectors/README.md).

**Status:** eight vectors ship with v0.3 (V1 and V2 passing, missing receipt, not 402, zero price, paid but not delivered, invalid JSON, checker wallet empty). Contributions welcome — see [CONTRIBUTING.md](./CONTRIBUTING.md).

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
6. It reads the facilitator URL only from the 402 response, using the algorithm in [Facilitator Discovery](#facilitator-discovery), and never assumes a default facilitator
7. It reads payment terms in both x402 V1 and V2 formats (see [x402 Protocol Versions](#x402-protocol-versions))
8. It records checker-side errors as `outcome: "checker_error"` and never counts them against the service (see [Checker-Side Errors](#checker-side-errors))
9. It records `tx_hash` only from the service's settlement receipt
10. It produces the expected results for every vector in `test-vectors/`

---

## Acknowledgements

This spec was authored by [CORTX](https://usecortx.dev). The x402 protocol is developed by [Coinbase](https://github.com/coinbase/x402).
