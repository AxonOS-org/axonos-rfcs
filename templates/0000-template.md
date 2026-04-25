---
rfc: 0000
title: <Short descriptive title>
status: draft
track: <scheduling | memory | security | process | hardware | api>
authors:
  - Denis Yermakou <axonosorg@gmail.com>
created: YYYY-MM-DD
updated: YYYY-MM-DD
implementation:
  - <link to issue or PR in the implementing repository, or "n/a">
supersedes: <RFC number, or omit>
superseded-by: <RFC number, or omit>
references:
  - <URL or formal reference>
---

# RFC-0000: <Title>

## Summary

One paragraph. State the proposal in plain terms. A reader should know after this paragraph whether they need to read the rest.

## Motivation

Why does this design decision need to be made? What is the problem in the absence of this RFC? Who feels that problem (engineer, application author, clinical partner, regulator)?

## Guide-level explanation

Explain the proposal as if teaching it to a kernel engineer who is new to AxonOS. Use the project's existing terminology where possible. Where new terminology is introduced, define it precisely.

This section should be readable without specialist background in the formal model. The next section is for the formal model.

## Reference-level explanation

The detailed technical specification. This section must be detailed enough that an independent engineer could implement against it. Include:

- Formal model (state machine, equations, invariants)
- Wire formats or memory layouts where relevant
- Algorithms in pseudocode or actual Rust
- Memory ordering requirements
- Worst-case timing or space bounds
- Error and fault behaviour

If equations are involved, state every variable and its domain.

## Drawbacks

What are the real costs of this proposal? Implementation cost, runtime cost, complexity cost, learning cost.

## Rationale and alternatives

What other designs were considered? Why were they rejected? What would happen if we did nothing?

For each alternative listed, state the specific trade-off that decided against it.

## Prior art

Where has this approach been tried before? Cite Linux, Rust, real-time systems literature, BCI literature, formal methods work — wherever the prior art is.

If this is genuinely novel, say so explicitly and explain why no prior art applies.

## Unresolved questions

What is left to figure out before this RFC can move from `active` to `final`? What is explicitly out of scope and deferred to a future RFC?

## Future possibilities

Sketches of what this RFC enables in the future, without committing to those future RFCs being accepted.

## Validation evidence level

State the validation level (per RFC-0003) at which the claims in this RFC currently stand.

- **Level 1 (instruction-count derived)** — what numbers are derived from instruction counts on published reference hardware
- **Level 2 (runtime measured)** — what numbers are derived from execution on real hardware over a stated interval
- **Level 3 (independent oscilloscope-validated)** — what numbers are validated by external instrument, status

If a claim is currently at Level 1 with Level 3 pending, say so explicitly. Do not classify pending claims as if they were validated.

## References

1. ...
2. ...

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). Implementations referenced from this RFC may be licensed differently — see the implementation repository for terms.*
