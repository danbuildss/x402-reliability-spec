# Contributing to the x402 Reliability Spec

Thank you for your interest in improving the spec.

This is an open standard. Anyone can propose changes. The goal is a definition that is clear, implementable, and reflects how x402 services actually behave in the wild.

---

## What belongs in this repo

- **SPEC.md** — the core 7-stage definition
- **schema/** — JSON Schemas for evidence records and stage results
- **examples/** — real-world check results showing pass and failure cases
- **CONTRIBUTING.md / CHANGELOG.md** — meta files

This repo defines the standard, not an implementation. Implementation code belongs elsewhere (e.g. a check runner package).

---

## How to propose a change

**Small clarifications** (wording, examples, obvious errors): open a PR directly.

**Substantive changes** (new stage, changed field semantics, breaking schema changes): open an issue first to discuss before writing the PR.

---

## Opening an issue

Use issues to:
- Report ambiguity or gaps in the spec
- Discuss edge cases you've encountered in real services
- Propose a new stage or field
- Ask whether a specific behavior passes or fails a given stage

Be specific. Share the actual service behavior you observed, the stage result you expected, and why the current spec is unclear or wrong.

---

## Opening a PR

1. Fork the repo
2. Create a branch: `fix/stage-3-clarification` or `feat/add-timeout-field`
3. Make your changes
4. Update `CHANGELOG.md` with a summary
5. Open the PR with a clear description of what changed and why

**PR checklist:**
- [ ] `SPEC.md` changes are reflected in the relevant JSON schema if applicable
- [ ] Examples are updated or added if the change affects output format
- [ ] `CHANGELOG.md` is updated

---

## Schema changes

If you change a required field or remove a field, that is a **breaking change** and requires a major version bump. Add a note in your PR.

If you add an optional field or clarify an existing one, that is a minor/patch change.

---

## What gets merged

The maintainer (currently [@danbuildss](https://github.com/danbuildss)) decides what merges. Decisions are guided by:

- Does this reflect real x402 service behavior?
- Is the spec clearer and more implementable after this change?
- Is it backward compatible (or is the breaking change justified)?

There is no committee or voting process. If you disagree with a decision, explain your reasoning in the issue or PR thread.

---

## Code of conduct

Be direct and specific. Disagreements about spec design are welcome — personal attacks are not.
