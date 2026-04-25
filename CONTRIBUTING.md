# Contributing to AxonOS RFCs

This repository is the public engineering record for the AxonOS real-time BCI kernel. Contributions are welcome — and they are governed by the RFC process described in [README.md](README.md).

## Quick decision tree

**Found a typo, broken link, or factual error in an existing RFC?**
→ Open a pull request directly. Small fixes do not require an issue first.

**Have a question about an RFC?**
→ Open a [discussion](../../discussions) in the relevant RFC's category.

**Want to propose a new RFC?**
→ See "How to propose an RFC" below.

**Want to implement an active RFC?**
→ Open an issue in the relevant code repository (e.g., `axonos-consent`, `axonos-sdk`) referencing the RFC number.

**Found a bug in the AxonOS implementation?**
→ This repository is for design documents, not implementation. File the bug in the relevant code repository.

## How to propose an RFC

### Before you write

1. Browse the existing [RFC index](README.md#index-of-rfcs) to confirm the topic is not already covered.
2. Open a [discussion](../../discussions) in the **RFC ideas** category. State the **problem** in 2–3 paragraphs:
   - What is the current behaviour or limitation?
   - Who feels that limitation (engineer, application author, clinical partner, regulator)?
   - What is the cost of doing nothing?
3. Wait at least 72 hours for response. If no maintainer or contributor pushes back, proceed to drafting. If there is pushback, address it before drafting.

### Drafting

1. Fork this repository.
2. Copy `templates/0000-template.md` to `rfcs/NNNN-short-title.md`. `NNNN` is the next free number — check the index.
3. Fill in **every** section of the template. Sections that don't apply should say so explicitly ("This RFC introduces no new failure modes — see RFC-0004 for the existing fault containment model"); they should not be left blank.
4. Validation evidence level — be honest. If a claim is currently L1, classify it L1. If it's pending L3, say "pending L3".
5. References — academic papers, prior art, standards documents. Citations are how the RFC earns its keep.

### Submission

1. Open a pull request titled `RFC-NNNN: <title>`.
2. The PR description should include:
   - One-paragraph summary
   - The motivating problem
   - Any breaking changes or RFCs this supersedes
   - Implementation tracking issue (if implementation is already in flight)
3. Maintainer will tag the PR with `RFC: under review` and announce it in [Discussions](../../discussions).

### Review

- Substantive feedback is collected for a **minimum of 14 calendar days** before merge.
- Feedback should be specific (cite a section, suggest concrete edits). Feedback like "I don't like this" without a stated reason is not actionable.
- The proposing author is expected to revise in response to all written feedback. They are not expected to accept every suggestion — disagreement with a documented rationale is fine.
- Maintainer makes the final accept/reject call after the review window closes.

### After acceptance

1. The RFC's `status:` field is updated to `active` on merge.
2. An implementation tracking issue is opened in the relevant code repository.
3. The author is invited to either implement directly or hand off the implementation to a contributor.

## What does not belong here

- Implementation bugs — file in the relevant code repository
- Refactors that preserve external behaviour — pull request in the code repository
- Day-to-day engineering choices — code review

If unsure, ask in [Discussions](../../discussions).

## Anti-bikeshedding clause (repeated for emphasis)

RFCs are accepted on technical merit. The following do **not** count as objections:

- Stylistic preference unless it affects readability for a non-author engineer
- "I would have done it differently" without a stated reason that the alternative is materially better
- Concerns about implementation effort — these belong in the implementation issue, not the RFC

The following **do** count as objections:

- A specific failure mode the RFC does not address
- A specific safety property the RFC violates
- A specific regulatory standard the RFC contradicts (with citation)
- A specific better alternative with a stated trade-off analysis

## Code of conduct

Be civil. Engineering is a written-record profession; assume your comments will be read by the author, by future contributors, and by clinical partners auditing the project's design history. Write accordingly.

Personal attacks, harassment, and bad-faith argumentation will result in repository moderation up to and including ban. The maintainer's judgement is final.

## License

By contributing, you agree that your contributions will be licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License (CC-BY-SA-4.0)](LICENSE), the same license as the rest of this repository.

For substantive contributions (anything more than a typo fix), please add yourself to the `authors:` list in the RFC's frontmatter.

## Contact

For matters that don't fit the public-record process: **axonosorg@gmail.com**.
