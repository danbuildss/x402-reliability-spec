# Test vectors

Mock x402 services and the evidence records a conforming checker must produce from them. See [SPEC.md → Reference Test Vectors](../SPEC.md#reference-test-vectors).

| Vector | What it tests | Expected outcome |
|---|---|---|
| `v1-passing` | V1: terms in the body, `X-PAYMENT`, `X-PAYMENT-RESPONSE` | `passed` |
| `v2-passing` | V2: terms only in `PAYMENT-REQUIRED` (empty body), `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE` | `passed` |
| `v2-no-receipt` | Delivers without a settlement receipt; no schema | `passed`, settlement `unconfirmed`, `tx_hash: null`, stage 7 `null` |
| `v2-not-402` | Answers 200 without payment | `failed` at stage 2 |
| `v2-zero-price` | Quotes a price of 0 | `failed` at stage 4, `ZERO_PRICE` |
| `v2-paid-not-delivered` | Settles, then answers 500 | `failed` at stage 5, settlement `confirmed` |
| `v1-invalid-json` | JSON service returns non-JSON; no schema | `failed` at stage 7, `INVALID_JSON` |
| `checker-wallet-empty` | The checker's wallet is empty | `checker_error`, stage 5 `fault: "checker"` |

## Format

Each directory holds `vector.json` and `expected.json`.

`vector.json`:

```jsonc
{
  "description": "…",
  "checker": {
    "max_price_usdc": 0.1,         // the checker's price ceiling for stage 4
    "schema": { … } | null,        // JSON Schema for stage 7, or null for none
    "wallet_balance_usdc": 1000    // balance the checker's wallet must report
  },
  "unpaid": { "status": 402, "headers": { … }, "body": … },   // answer without a payment header
  "paid":   { "status": 200, "headers": { … }, "body": … } | null  // answer with one; null = the checker must not pay
}
```

Encoding rules for `headers` and `body`:

- A **header value that is a JSON object** means: serialize it as JSON and base64-encode it (how x402 V2 headers are sent). A string header value is sent as is.
- A **body that is an object or array** is sent as JSON text. A string body is sent as is (`""` = empty body).
- `{{ENDPOINT}}` is replaced with the mock service's URL, and `{{CHECKER_WALLET}}` with the checker's wallet address, before encoding.

The mock answers every request without an `X-PAYMENT` or `PAYMENT-SIGNATURE` header with `unpaid`, and every request with one with `paid`.

## Comparing results

Compare the produced record with `expected.json` on:

- `outcome`, `overall_passed`, `highest_stage_passed`, `x402_version`
- each stage's `stage`, `name`, `passed`, `error_code` and `fault` (a missing field equals `null`)
- stage 5's `settlement_status` and `tx_hash`, when `expected.json` has them

Ignore `endpoint`, `checked_at`, `checker_id`, latencies and the free-text `error`.

`expected.json` files are validated against `schema/evidence-record.json` in CI.
