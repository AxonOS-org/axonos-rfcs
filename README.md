<div align="center">

# AxonOS RFCs

### Engineering design documents for the AxonOS real-time BCI kernel

The AxonOS RFC ("Request for Comments") process is how substantial architectural decisions are designed, reviewed, and recorded. Each RFC captures the reasoning, alternatives, formal model, and acceptance criteria for one design decision in the AxonOS kernel.

[![RFCs](https://img.shields.io/badge/RFCs-5-blueviolet?style=for-the-badge)](rfcs/)
[![License](https://img.shields.io/badge/license-CC--BY--SA--4.0-green?style=for-the-badge)](LICENSE)
[![Status](https://img.shields.io/badge/process-active-success?style=for-the-badge)](#process)
[![Discussions](https://img.shields.io/badge/discussions-open-informational?style=for-the-badge&logo=github)](../../discussions)

[![Website](https://img.shields.io/badge/axonos.org-000000?style=for-the-badge&logoColor=white)](https://axonos.org)
[![Medium](https://img.shields.io/badge/Medium-02b875?style=for-the-badge&logo=medium&logoColor=white)](https://medium.com/@AxonOS)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/axonos)
[![dev.to](https://img.shields.io/badge/dev.to-0A0A0A?style=for-the-badge&logo=devdotto&logoColor=white)](https://dev.to/axonosorg)
[![Email](https://img.shields.io/badge/axonosorg%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:axonosorg@gmail.com)

</div>

---

## Why an RFC process

A real-time kernel for safety-critical brain-computer interface systems cannot be built by accreting code commits. The decisions that will matter most — scheduler choice, memory ordering, isolation model, validation framework — must be defensible in writing, reviewable by independent engineers, and citable when the regulatory submission asks "why this approach."

This repository is the public record of those decisions.

The format is adapted from the [Rust RFC process](https://github.com/rust-lang/rfcs) and the [IETF RFC tradition](https://www.rfc-editor.org/rfc-index/), with adjustments for the safety-critical-systems context.

---

## Index of RFCs

| # | Title | Status | Track | Updated |
|:---:|:---|:---:|:---:|:---:|
| [0001](rfcs/0001-edf-scheduler-biological-deadlines.md) | EDF Scheduler with Biological Deadlines | ![status-active](https://img.shields.io/badge/-active-success) | scheduling | 2026-04-25 |
| [0002](rfcs/0002-zero-copy-ring-buffer.md) | Zero-Copy Ring Buffer for Signal Path | ![status-active](https://img.shields.io/badge/-active-success) | memory | 2026-04-25 |
| [0003](rfcs/0003-validation-status-framework.md) | Validation Status Framework (L1/L2/L3) | ![status-active](https://img.shields.io/badge/-active-success) | process | 2026-04-25 |
| [0004](rfcs/0004-dual-core-real-time-contract.md) | Dual-Core Real-Time Contract | ![status-active](https://img.shields.io/badge/-active-success) | scheduling | 2026-04-25 |
| [0005](rfcs/0005-capability-based-app-manifest.md) | Capability-Based Application Manifest | ![status-active](https://img.shields.io/badge/-active-success) | security | 2026-04-25 |

### Status legend

- ![draft](https://img.shields.io/badge/-draft-lightgrey) — being drafted, not yet ready for review
- ![review](https://img.shields.io/badge/-review-yellow) — ready for review, comments open in [Discussions](../../discussions)
- ![active](https://img.shields.io/badge/-active-success) — accepted, implementation underway or complete
- ![final](https://img.shields.io/badge/-final-blue) — implementation complete and stable
- ![superseded](https://img.shields.io/badge/-superseded-orange) — replaced by a newer RFC; RFC remains visible for history
- ![withdrawn](https://img.shields.io/badge/-withdrawn-red) — withdrawn before acceptance

### Track legend

- **scheduling** — real-time scheduling, deadline analysis, timing contracts
- **memory** — memory ordering, allocation discipline, cache model
- **security** — isolation, attestation, capability model, consent enforcement
- **process** — engineering process, validation methodology, evidence standards
- **hardware** — reference hardware, certification requirements, BOM constraints
- **api** — application-facing interfaces, SDK design

---

## Process

The lifecycle of an RFC is:

```
   draft ──────► review ──────► active ──────► final
                                   │
                                   └────► superseded (by future RFC)
                  │
                  └─► withdrawn (before acceptance)
```

### How to propose an RFC

1. Open a [discussion](../../discussions) in the **RFC ideas** category to gauge interest. State the problem, not the solution.
2. If the problem is in scope, fork this repository and copy `templates/0000-template.md` to `rfcs/NNNN-short-title.md` where `NNNN` is the next free number.
3. Fill in every section. The "Reference-level explanation" section must be detailed enough that an independent engineer can implement against it.
4. Open a pull request titled `RFC-NNNN: <title>`.
5. Substantive feedback is collected for a minimum of 14 calendar days before merge.
6. Acceptance criteria: at minimum, no unaddressed objection from a maintainer, and the proposing author has revised in response to all written feedback.
7. On merge, status becomes `active` and an implementation tracking issue is opened in the relevant code repository.

### What belongs in an RFC

- Architecture or interface decisions that will be hard to reverse later
- New cross-cutting concerns (a new state, a new safety property, a new IPC class)
- Changes to validation, evidence, or compliance methodology
- Changes that affect the regulatory submission narrative

### What does not belong in an RFC

- Bug fixes — file an issue
- Refactors that preserve external behaviour — open a pull request
- Performance optimisations that do not change interfaces — open a pull request
- Day-to-day engineering choices — make them in code review

If unsure, ask in [Discussions](../../discussions) before opening an RFC.

---

## Anti-bikeshedding clause

RFCs are accepted on technical merit and engineering rigour. The following do **not** count as objections:

- Stylistic preference unless it affects readability for a non-author engineer
- "I would have done it differently" without a stated reason that the alternative is materially better
- Concerns about implementation effort — these belong in the implementation issue, not the RFC

The following **do** count as objections:

- A specific failure mode the RFC does not address
- A specific safety property the RFC violates
- A specific regulatory standard the RFC contradicts (with citation)
- A specific better alternative with a stated trade-off analysis

---

## Reading order for new readers

If you are an engineer evaluating AxonOS:

1. Start with [RFC-0003 (Validation Status Framework)](rfcs/0003-validation-status-framework.md). It explains how AxonOS distinguishes claims by evidence level. Every other RFC's claims classify themselves under this framework.
2. Then read [RFC-0001 (EDF Scheduler)](rfcs/0001-edf-scheduler-biological-deadlines.md) for the core scheduling argument.
3. Then [RFC-0004 (Dual-Core Contract)](rfcs/0004-dual-core-real-time-contract.md) for the partition model.
4. The remaining RFCs can be read in any order.

If you are reviewing for clinical or regulatory engagement:

1. RFC-0003 (Validation Status Framework) — the evidence taxonomy
2. RFC-0004 (Dual-Core Contract) — the partition and fault containment model
3. RFC-0005 (Capability-Based Manifest) — the application isolation model

If you are an investor evaluating engineering rigour:

1. RFC-0003 (Validation Status Framework) — the founder's posture toward overclaiming
2. Browse the index — the existence of an RFC process is itself the signal

---

## License

The contents of this repository — including all RFC documents — are licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License (CC-BY-SA-4.0)](LICENSE).

You may freely:

- Quote, paraphrase, and cite the RFCs
- Translate them into other languages
- Use them as the basis for your own RFCs (with attribution and the same license)

The `axonos-consent` Rust source code, referenced from several RFCs as the canonical implementation, is dual-licensed Apache-2.0 OR MIT in its [own repository](https://github.com/AxonOS-org/axonos-consent) — not under CC-BY-SA-4.0. Different artifacts, different licenses.

---

## Contact

Denis Yermakou, founder.
**axonosorg@gmail.com**

For substantive RFC discussion, please use [GitHub Discussions](../../discussions) rather than email — public technical record is the point.

---

<div align="center">
<sub>© 2026 Denis Yermakou · AxonOS · The RFC process is how the kernel earns the right to be called real-time.</sub>
</div>
