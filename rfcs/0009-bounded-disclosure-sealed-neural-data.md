---
rfc: 0009
title: Bounded Disclosure of Sealed Neural Data
status: draft
track: security
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-07-29
updated: 2026-08-01
implementation:
  - axonos-vault — Sealed / Reducer / Grant / release — v0.1.1, partially conformant (see §10)
  - axonos-consent — withdrawal semantics this RFC mirrors — current release
  - Promotion to active pending resolution of the uncharged-channel question (§7)
references:
  - RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels (IETF, 1997)
  - RFC-0003 — Validation Status Framework (AxonOS)
  - RFC-0005 — Capability-Based Application Manifest (AxonOS)
  - C. E. Shannon, "A Mathematical Theory of Communication", Bell System Technical Journal, 1948
  - T. M. Cover and J. A. Thomas, "Elements of Information Theory", 2nd ed., Wiley, 2006
  - D. E. Denning, "Cryptography and Data Security", Addison-Wesley, 1982 — statistical database inference and query auditing
  - N. R. Adam and J. C. Worthmann, "Security-Control Methods for Statistical Databases: A Comparative Study", ACM Computing Surveys 21(4), 1989
  - I. Dinur and K. Nissim, "Revealing Information while Preserving Privacy", PODS 2003 — reconstruction from aggregate queries
  - C. Dwork, "Differential Privacy", ICALP 2006
  - J. M. Rabaey et al., on-device processing constraints for implantable and wearable neural interfaces
---

# RFC-0009: Bounded Disclosure of Sealed Neural Data

## Summary

This RFC specifies the boundary across which data derived from raw neural
recordings may leave an AxonOS device, and the accounting that governs it.

Raw samples are held in a **sealed** container that exposes no accessor. The only
way to learn anything from them is to run a **reducer** inside the boundary,
whose fixed-size output becomes a **disclosure** only after being charged, in
bits, against a **grant** naming a purpose and a budget — and only after being
recorded.

The central claim is information-theoretic and is proved in §4: for a grant of
budget β bits, the mutual information between the sealed window and everything
the application learns through disclosures is at most β. The bound holds against
a computationally unbounded adversary, because it is a counting argument about
channel width rather than an assumption about hardness.

The RFC is equally explicit about what the bound does **not** give. It constrains
*how much* leaves, not *which* bits; it adds no noise and therefore provides no
per-record indistinguishability; and it is defeated by an issuer that grants
without limit. §7 enumerates channels that carry information *without* being
charged, including one the reference implementation currently leaves open.

## Motivation

Every other component of AxonOS supports one promise: what a brain produced stays
with the person who produced it. The acquisition layer refuses to invent or lose
a sample; the consent layer decides who may act; the protocol carries what was
agreed. **None of them prevents an application from reading the samples.**

The obvious remedy — permit only aggregate queries — is known to be insufficient
and has been known for four decades. A permission system that answers yes or no
per request is defeated by asking many times: the statistical-database literature
(Denning 1982; Adam & Worthmann 1989) documents trackers and inference attacks
against exactly this design, and Dinur & Nissim (2003) showed that a database can
be reconstructed from a sufficient number of noisy aggregate answers. Eight
channels at 250 SPS and 24 bits is 48 kbit/s; an application permitted to ask for
"the mean of the last window" a thousand times a second has been handed the
signal back, one honest answer at a time.

The response specified here is not to make each answer safer, but to make the
total finite and accounted. A boundary that does not bound the aggregate is
decoration, however carefully each individual query is reviewed.

## Guide-level explanation

Four objects and one rule.

- **Sealed window** — the most recent *m* frames. It has no accessor, cannot be
  cloned, copied, or formatted, and is destroyed by `purge`, never revealed.
- **Reducer** — code that runs *inside* the boundary, sees frames one at a time,
  and may keep only its own compile-time-bounded state.
- **Grant** — a purpose, a budget in bits, an expiry, and the ability to be
  withdrawn. Withdrawal is terminal.
- **Disclosure** — a fixed-size result together with the terms it left under.

The rule: **a reduction becomes a disclosure only after it has been charged and
recorded.** If it cannot be charged, it is refused. If it cannot be recorded, it
is refused — an unrecorded disclosure is worse than a refused one, because the
record is the only artifact that outlives the data.

## Reference-level explanation

### 1. Definitions

Let *W* be the sealed window: *m* frames × *c* channels × *b* bits, so
*H(W) ≤ m·c·b* bits. For the canonical configuration *m* = 250, *c* = 8,
*b* = 24, giving 48 000 bits.

A **reduction** is a vector *r ∈ ℤ^{n}* with *n ≤ n_max*, produced by a
deterministic function of *W* and the reducer's own state. Its **cost** is

```
cost(r) = n · w        where w is the fixed scalar width in bits            (1)
```

A **grant** *g = (id, ρ, β, e)* carries purpose ρ, budget β bits, expiry *e*, and
mutable state (spent bits, withdrawn flag).

A **disclosure transcript** *D = (d_1, …, d_k)* is the ordered sequence of
reductions released under a grant.

### 2. The release predicate

A reduction *r* is releasable under grant *g* at time *t* iff **all** hold:

```
¬withdrawn(g)  ∧  t ≤ e(g)  ∧  ρ(request) = ρ(g)
              ∧  n(r) ≥ 1   ∧  support(r) ≥ 1
              ∧  spent(g) + cost(r) ≤ β(g)
              ∧  recordable()                                              (2)
```

The conjuncts MUST be evaluated in the order given, and the refusal MUST name the
**first** conjunct that failed. This is not cosmetic: a withdrawn grant that
reports `budget exhausted` produces an audit trail that misdescribes what
happened, and the audit trail is the artifact this whole construction exists to
produce.

### 3. Cost is computed, never declared

**C1.** `cost(r)` MUST be computed by the enforcing component from the actual
content of *r*, per (1).

**C2.** An implementation MUST NOT accept a cost supplied by the requester, and
MUST NOT allow a reducer to declare its own price. A reducer that can price
itself prices itself at zero.

**C3.** A reduction wider than *n_max* MUST be refused, not truncated. Silently
dropping part of an answer yields a number that means something other than what
its author intended, and the reader has no way to detect the difference.

### 4. The bound

> **Theorem 1 (disclosure bound).** Let *D* be the transcript released under a
> grant of budget β. Then *I(W ; D) ≤ β*.

*Proof.* Each *d_j* is *n_j* scalars of *w* bits, and the release predicate (2)
enforces *Σ_j n_j·w ≤ β*. The transcript is therefore representable as a binary
string of length at most β, so *H(D) ≤ β*. Mutual information is bounded by the
entropy of either argument, *I(W ; D) ≤ H(D)*, hence *I(W ; D) ≤ β*. ∎

Two properties of this bound are worth stating because they are unusual.

**It is unconditional.** No computational assumption appears in the proof. An
adversary with unlimited time and memory is bounded by the same β, because the
constraint is the width of the channel and not the difficulty of inverting a
function.

**It is agnostic to the reducer set.** No reducer, however cleverly constructed,
can carry more than the bits it occupies. Adding reducers therefore does not
weaken Theorem 1 — which is what makes the reducer set an extensibility point
rather than a review burden.

> **Theorem 2 (composition).** For grants *g_1 … g_p* with budgets *β_1 … β_p*,
> the transcript across all of them satisfies *I(W ; D) ≤ Σ_i β_i*.

*Proof.* Concatenation; *H* of a concatenation is at most the sum of the parts. ∎

Theorem 2 is the reason §6 places a normative obligation on the **issuer**. The
enforcing component cannot bound what the issuer grants without limit, and a
system that re-issues an exhausted grant has a budget of infinity written in
instalments.

### 5. What the bound does not give

**L1 — It does not choose which bits.** β bits may be the *most* informative β
bits available. A single well-chosen 32-bit correlation score against a specific
hypothesis may be the only quantity an adversary wanted. Constraining *which*
questions may be asked is the job of the purpose field and the reducer set, and
those are policy, not arithmetic.

**L2 — It is not differential privacy.** No noise is added. Theorem 1 bounds the
*total* information about the window; differential privacy bounds what is learned
about any *individual record* regardless of total volume (Dwork 2006). Neither
implies the other. An implementation MAY compose the two by adding a
noise-injecting reducer; this RFC does not require it and MUST NOT be cited as
providing it.

**L3 — It bounds a window, not a lifetime.** *W* is the most recent *m* frames.
Disclosures across successive windows are disclosures about different data, and
Theorem 1 applies to each separately. Bounding what a session as a whole reveals
requires lifetime accounting (N6), which is a policy layer above this one.

**L4 — Tightness is not claimed.** β is an upper bound on mutual information, not
an estimate of it. Typical disclosures carry far less; the bound exists to make
the worst case finite, not to predict the average.

### 6. Normative requirements

**N1.** Raw samples MUST NOT be reachable through any public interface of the
sealed container. In a language with a type system capable of expressing it, the
absence MUST be structural — no accessor, no clone, no copy, no formatting
implementation — rather than enforced by convention or review.

**N2.** All disclosure MUST pass through the release predicate (2). An
implementation MUST NOT provide a second path.

**N3.** Cost MUST be computed per §3.

**N4.** A disclosure MUST be recorded before it is returned. If the record cannot
be written, the disclosure MUST be refused.

**N5.** An implementation MUST NOT issue a grant whose budget cannot be spent
within its own recording capacity. Where the audit log holds *L* entries and the
minimum chargeable disclosure is *w* bits, a grant with β > L·w advertises a
ceiling it will not honour, and the effective ceiling MUST equal the advertised
one.

**N6.** The issuer MUST maintain accounting across grants for a subject, per
Theorem 2. An enforcing component that meets N1–N5 while its issuer grants
without limit satisfies the letter of this RFC and defeats its purpose.

**N7.** Withdrawal MUST be terminal and MUST take precedence over every other
refusal reason. This mirrors consent withdrawal exactly, because it is the same
property: an authority that can restore itself is not an authority.

**N8.** Every channel that carries information across the boundary without being
charged MUST be enumerated and bounded in the implementation's documentation
(§7). An unenumerated side channel is a defect regardless of its rate.

### 7. Uncharged channels

Theorem 1 bounds the transcript. It does not bound information carried by facts
*about* the transcript, and an honest specification must enumerate them.

**U1 — Refusal reasons.** Whether a release succeeds, and why it failed, is
information the requester obtains without charge. Where refusal depends only on
grant state — withdrawn, expired, wrong purpose, budget — this reveals only what
the requester already knows, and the leak is nil.

**U2 — Content-dependent refusal.** Where refusal depends on the *window*, the
requester learns about the window for free. The predicate (2) contains two such
conjuncts: `n(r) ≥ 1` and `support(r) ≥ 1` fail exactly when the window is empty.
A requester may therefore poll for one bit — "is the device recording?" — at no
cost and without limit.

This is a real, if narrow, channel: it carries session liveness, and it is
uncharged and unbounded in rate. An implementation MUST do one of:

  (a) make the refusal content-independent by charging the attempt; or
  (b) charge attempts against the budget whether or not they succeed; or
  (c) rate-limit and count attempts so the channel's bandwidth is bounded and
      visible in the record.

The reference implementation currently does none of these and is non-conformant
against N8 for this reason (§10).

**U3 — Timing.** Release latency may depend on window content through cache
behaviour. Under the fixed-window, allocation-free construction this RFC assumes,
the dependence is small but not provably zero, and an implementation MUST NOT
claim it is absent without measurement.

**U4 — Existence of the grant.** That a grant exists, and its purpose, is known
to the requester by construction and is not a leak about *W*.

### 8. Adversary model

The adversary is an application that has been granted a legitimate capability
under RFC-0005 and behaves within the interface — the honest-but-curious model.
It may issue arbitrary reduction requests in arbitrary order at arbitrary rate,
may collude with other applications holding other grants, and is computationally
unbounded.

Explicitly **out of scope**: an adversary with kernel privilege, physical memory
access, or the ability to substitute the vault implementation. This boundary is
an *architectural* control between application and kernel, not a defence against
a compromised kernel, and MUST NOT be described as one. Defences at that level
belong to secure boot and attestation, which are not yet specified.

### 9. Conformance vectors

An implementation claiming conformance MUST reproduce:

| Scenario | Required outcome |
|:--|:--|
| β = 3200, w = 32, unlimited requests over a full window | exactly ⌊β/w⌋ = 100 successful releases, then refusal naming the budget — subject to N5 |
| withdrawn grant, request also expired, wrong purpose, over budget | refusal names **withdrawal**, per N7 |
| expiry *e*, request at *t = e* | released — the expiry bound is inclusive |
| expiry *e*, request at *t = e+1* | refused, naming expiry |
| reduction of 0 scalars | refused; MUST NOT be released as a zero-cost disclosure |
| reduction over an empty window | refused (see U2 for the channel this opens) |
| record capacity exhausted, budget remaining | refused, naming the record — never released unrecorded |

## Drawbacks

Charging by scalar count (1) is crude: it prices a 32-bit count of bad frames
identically to a 32-bit correlation against a hypothesis, though the second may
be enormously more revealing. A semantically weighted cost would be better and is
not currently specifiable — assigning a defensible weight to a reducer's output
requires knowing what the recipient can do with it, which is not knowable at the
boundary. The crude measure is chosen because it is unforgeable and computable;
this is a deliberate trade of precision for soundness.

The fixed window also means a patient adversary can wait: β bits per window,
across many windows, is a large total. N6 exists precisely because this RFC's
own mechanism does not address it.

## Rationale and alternatives

**Counting queries instead of bits.** Query counting is the design the statistical
database literature already broke: an unbounded-width answer defeats a bounded
count. Bits is the only measure that composes.

**Per-reducer allowlists instead of a budget.** Necessary but not sufficient —
this is the *which* problem of L1, and it is orthogonal. The purpose field
implements a coarse form of it; the two mechanisms are complementary and neither
substitutes for the other.

**Differential privacy instead.** DP answers a different question (L2) and costs
utility that a clinical signal-quality readout cannot afford. It is compatible
with this design and may be layered as a reducer property; it is not a
replacement for a rate bound, because DP composes with *degrading* guarantees
across many queries rather than stopping.

**Encrypt and export, decide later.** Moves the boundary off the device, which is
the thing this architecture exists to avoid.

## Prior art

Query auditing and inference control in statistical databases (Denning 1982; Adam
& Worthmann 1989) established that aggregate-only interfaces leak through
repetition. Dinur & Nissim (2003) gave the reconstruction result that makes the
leak quantitative. Differential privacy (Dwork 2006) took the noise-based branch;
this RFC takes the accounting branch, which is older, weaker in what it promises,
and unconditional in what it delivers. The specific combination specified here —
an unconditional information ceiling, a purpose binding, terminal withdrawal, and
mandatory recording as a precondition of release — is not, to the authors'
knowledge, standardised elsewhere for physiological data.

## Unresolved questions

- Which remedy of U2 is correct? Charging failed attempts (b) is simplest but
  lets a requester exhaust a legitimate budget by malformed requests, which is a
  denial-of-service against the user rather than a leak.
- Should *w* be the scalar width, or the entropy of the scalar's declared range?
  A count bounded to [0, 250] carries under 8 bits, not 32; charging 32 is sound
  but wasteful, and charging 8 requires trusting a declared range (C2).
- How should lifetime accounting (N6) be represented so it survives power cycles
  without becoming a durable record of what the subject was asked about?

## Future possibilities

A declared-range cost model would let a reducer that returns a bounded count be
charged for what it actually carries, materially increasing the number of honest
readings a budget affords. It requires making the range part of the reducer's
type rather than its documentation, so that C2 still holds.

## Validation evidence level

Per RFC-0003:

- **Theorem 1 and Theorem 2** — L1 as mathematics; they are proofs, not
  measurements. Their *applicability* depends on N1–N4 holding in the
  implementation, which is where the evidence question actually sits.
- **Structural unreachability of raw samples (N1)** — L1 for the reference
  implementation: the sealed type has no accessor and no derived traits, checked
  mechanically. A compile-fail test suite would raise this to L1 with coverage;
  it does not yet exist.
- **Release predicate ordering and refusal naming** — L1, unit-tested including
  the precedence case of N7.
- **The budget bound in practice** — L1. A test drives 10 000 requests against a
  budget and observes exactly the predicted number of successes.
- **Uncharged channel U2** — enumerated **and** remediated in `axonos-vault`
  0.2.0: a content-dependent refusal now costs a scalar, which brings the
  channel under Theorem 1 with everything else. A thousand probes against a
  128-bit grant yield four answers and then a refusal.
- **Timing independence (U3)** — not measured at any level; the RFC accordingly
  makes no claim.

## Conformance status of the reference implementation

`axonos-vault` v0.1.1 measured against this RFC:

| Requirement | Status |
|:--|:--|
| N1 structural unreachability | conformant |
| N2 single path | conformant |
| N3 computed cost | conformant |
| N4 record before return | conformant |
| N5 budget ≤ recordable capacity | conformant as of 0.2.0 — `issue()` refuses a budget the log cannot record, and sums commitments across live grants because the capacity is shared |
| N6 issuer-side lifetime accounting | **not implemented** — no issuer layer exists yet |
| N7 terminal withdrawal, highest precedence | conformant |
| N8 uncharged channels enumerated and bounded | conformant as of 0.2.0 for U2; U3 (timing) remains enumerated and unmeasured, and the RFC makes no claim about it |
| §9 conformance vectors | conformant |

The N5 deviation was found by running the organs together in `axonos-stack`
rather than by testing the vault alone, which is itself evidence for the
integration layer. Both N5 and U2 were closed in `axonos-vault` 0.2.0.

**What still blocks promotion out of draft.** N6 — issuer-side accounting
across grants for one subject — is unimplemented, and no issuer layer exists to
implement it in. Theorem 2 says the bound composes over grants; without an
issuer that tracks the sum, an enforcing component can satisfy every other
requirement here while its issuer hands out budgets without limit. That is not
a gap in the mechanism, it is a missing layer above it, and this RFC stays in
draft until either the layer exists or the requirement is withdrawn with an
argument.

## References

See front matter.
