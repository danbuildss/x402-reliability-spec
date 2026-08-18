# x402 Reliability Spec

An open specification for verifying the reliability of x402 payment services.

x402 is the HTTP-native protocol for AI agent micropayments. As agents depend on paid external services, a failed tool call is no longer just an API error — it's money spent, a workflow interrupted, a customer left waiting.

This spec defines what it means for an x402 service to be reliably operational: not just reachable, but paying, delivering, and conforming to schema.

---

## What this defines

The x402 Reliability Spec describes a **7-stage verification pipeline** that any tool can implement to produce a structured reliability evidence record for an x402 endpoint.

The stages progress from lightweight (no payment required) to full end-to-end verification (real payment):

| Stage | Name | Payment required |
|-------|------|-----------------|
| 1 | Availability | No |
| 2 | 402 Response | No |
| 3 | Payment Terms | No |
| 4 | Price Validity | No |
| 5 | Payment Processing | Yes |
| 6 | Delivery | Yes |
| 7 | Schema Validity | Yes |

Stages 1–4 are lightweight checks that can run frequently. Stages 5–7 require a real payment and should run less often.

---

## Repository structure

```
x402-reliability-spec/
  README.md          ← this file
  SPEC.md            ← full 7-stage specification
  schema/
    check-result.json      ← JSON Schema for a single stage result
    evidence-record.json   ← JSON Schema for a full evidence record
  examples/
    passing-all-stages.json
    failing-stage-5.json
    failing-stage-2.json
  CONTRIBUTING.md    ← how to propose changes
  CHANGELOG.md       ← version history
```

---

## Who this is for

- **x402 service builders** — validate your implementation against an independent standard
- **Agent developers** — know what reliability evidence to request before depending on a service
- **Tool builders** — implement the pipeline and produce conforming evidence records
- **Researchers and protocol contributors** — propose changes to how reliability is defined

---

## Reference implementation

[CORTX](https://usecortx.dev) is the reference implementation of this spec. CORTX runs synthetic checks against live x402 services and accumulates reliability history over time.

Conforming with this spec does not require CORTX. Any tool that produces evidence records matching the schemas in `schema/` is a valid implementation.

---

## Status

**v0.1 — working draft.** The spec is stable enough to implement against. Breaking changes will be versioned.

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

[Apache 2.0](./LICENSE)
