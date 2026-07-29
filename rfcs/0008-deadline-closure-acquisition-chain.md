---
rfc: 0008
title: Deadline Closure for the Acquisition Chain
status: draft
track: kernel
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-07-29
updated: 2026-07-29
implementation:
  - axonos-hal — TimingBudget::close — v0.1.1, partially conformant (see §9)
  - axonos-supervisor — deadline-miss observability — v0.1.0
  - Promotion to active pending L3 oscilloscope validation per RFC-0003
references:
  - RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels (IETF, 1997)
  - RFC-0003 — Validation Status Framework (AxonOS)
  - RFC-0004 — Dual-Core Real-Time Contract (AxonOS)
  - C. L. Liu and J. W. Layland, "Scheduling Algorithms for Multiprogramming in a Hard-Real-Time Environment", JACM 20(1), 1973
  - M. Joseph and P. Pandya, "Finding Response Times in a Real-Time System", The Computer Journal 29(5), 1986
  - N. Audsley et al., "Applying New Scheduling Theory to Static Priority Pre-emptive Scheduling", Software Engineering Journal 8(5), 1993
  - L. Sha, R. Rajkumar and J. Lehoczky, "Priority Inheritance Protocols: An Approach to Real-Time Synchronization", IEEE Trans. Computers 39(9), 1990
  - R. Wilhelm et al., "The Worst-Case Execution-Time Problem — Overview of Methods and Survey of Tools", ACM TECS 7(3), 2008
  - Texas Instruments, ADS1299 datasheet (SBAS499), low-noise 8-channel 24-bit ΔΣ AFE
---

# RFC-0008: Deadline Closure for the Acquisition Chain

## Summary

This RFC specifies the condition under which an AxonOS acquisition chain may be
admitted at a given sample rate, the terms that condition is composed of, and
the obligations an implementation carries when it declares any of those terms to
be zero.

The condition is a response-time test, not a utilisation test:

> **R = J + B + C + I ≤ D**

for release jitter *J*, blocking *B*, chain execution time *C*, and interference
*I*, against the deadline *D*, which for the acquisition chain is one sample
period. An implementation MUST refuse to enter a configuration for which the
test does not hold, and MUST declare every term — including the terms it sets to
zero — together with the method by which each was obtained.

The RFC exists because a published worst-case response time is a claim about a
*model*, and a model whose assumptions are not written down is not falsifiable.
The reference implementation, `axonos-hal` v0.1.1, currently sets *B = I = 0*
without discharging that assumption; §9 records this as a known deviation rather
than presenting the implementation as conformant.

## Motivation

AxonOS publishes a worst-case response time of 972 µs and jitter of 2.1 µs σ,
6.5 µs at the 99.9th percentile. Figures of this kind ordinarily live in
documentation, where nothing checks them and where the day a pipeline stage is
added they quietly stop being true.

Making them checkable requires three things that no amount of measurement
supplies on its own:

1. **A stated system model.** A response time is meaningless without saying what
   may preempt the chain, what it may block on, and how the deadline relates to
   the arrival it is measured from.
2. **A refusal, not a warning.** A configuration that does not close must be
   impossible to enter, not merely reported. Otherwise the guarantee is a
   convention that survives exactly as long as everyone remembers it.
3. **An explicit account of the terms set to zero.** Every real-time analysis
   omits terms. An analysis that does not say *which* is indistinguishable from
   one that forgot them.

This is also the layer at which power management becomes a correctness problem
rather than a comfort one: a frequency change or a wait-state change alters *C*,
so a budget closed at one operating point says nothing about another (§6.4).

## Guide-level explanation

An acquisition chain converts one simultaneous sample of all channels into an
authorised action. It is released once per sample period and must finish before
the next release, or the sample it was working on is stale and the one arriving
has nowhere to go.

Four things consume the period:

| Term | What it is | Who owns it |
|:--|:--|:--|
| *J* | how late the release can be relative to the ideal periodic instant | the interrupt path and the converter's `DRDY` timing |
| *B* | time spent waiting for a resource held by lower-priority work | shared buses, mutexes, DMA channels |
| *C* | the chain's own execution, summed across its stages | the pipeline |
| *I* | execution stolen by higher-priority work | ISRs, radio stacks, DMA contention |

The chain closes when their sum fits the period. The reference implementation
takes the sample rate and the stage table, computes the sum, and returns either a
`TimingBudget` — a value that only exists if the test passed — or a refusal
naming the term that overran. Because the device's configuration entry point
consumes a `TimingBudget` *by value*, a chain that does not close cannot be
started: the proof travels with the data instead of living in a comment.

## Reference-level explanation

### 1. System model

**M1.** Samples arrive periodically with period *T = 1/f_s*. The converter is the
timing master; the host does not choose when a sample exists.

**M2.** The acquisition chain is a single logical task composed of *n* stages
executed in sequence, with no scheduling decision between them:
*C = Σ_{i=1..n} C_i*.

**M3.** The chain has an implicit deadline: *D = T*. A sample whose processing is
not complete when the next sample arrives has missed.

**M4.** Deadlines are measured from the **ideal** periodic instant, not from the
actual release. This is the conservative choice: it charges release jitter to the
budget rather than to the observer.

**M5.** Stage execution times are constants for a given operating point. An
operating point is the tuple (core frequency, memory wait states, cache
configuration, compiler and its options).

Assumptions M2 and M5 are the load-bearing ones. M2 fails if a stage can be
preempted by another stage or by unrelated application work; M5 fails silently
across a frequency change, which is why §6.4 exists.

### 2. The closure condition

For the chain to be admissible at rate *f_s*:

```
R = J + B + C + I  ≤  D = T = 1 / f_s                                    (1)
```

Under M2 this is both necessary and sufficient for the single-task case: a task
with no higher-priority interference, no blocking and no jitter is schedulable
against an implicit deadline exactly when *C ≤ T* (Liu & Layland 1973), and the
three additional terms are additive delays on the same timeline (Joseph & Pandya
1986; Audsley et al. 1993).

Where interference is present and periodic, *I* is not a constant and (1) MUST be
replaced by the standard fixed-point response-time recurrence over the
higher-priority set *hp*:

```
R^(k+1) = C + B + Σ_{j ∈ hp} ⌈(R^(k) + J_j) / T_j⌉ · C_j                 (2)
```

iterated from *R^(0) = C + B* until convergence or until *R > D*. An
implementation that declares *I = 0* asserts *hp = ∅* for the chain, which is a
statement about the whole system and not about the chain alone.

### 3. Jitter

Release jitter enters (1) additively. It also enters the *signal* independently
of the deadline, and the two are separate obligations that a single number is
often mistakenly used to discharge.

For a sinusoidal component at frequency *f* sampled with timing jitter of
standard deviation *σ*, the jitter-limited signal-to-noise ratio is

```
SNR_jitter = −20 · log₁₀(2π f σ)                                          (3)
```

At the canonical σ = 2.1 µs this gives 65.6 dB at 40 Hz and 57.6 dB at 100 Hz;
at the P99.9 figure of 6.5 µs, 55.7 dB and 47.8 dB respectively. The ADS1299 at
250 SPS and gain 24 delivers on the order of 21 effective bits, and the
electrode-side noise floor of a scalp recording is well above that. Sampling
jitter is therefore **not** the limiting term for signal fidelity in the EEG band
under this configuration, and the jitter budget is justified by the deadline test
rather than by spectral requirements.

This finding is configuration-specific and MUST be re-derived for any deployment
that raises *f_s* or widens the band of interest: (3) degrades linearly in *f*.

### 4. Worst-case execution time

**W1.** An implementation MUST state, for each stage, how *C_i* was obtained:
static analysis, measurement, or assertion.

**W2.** A measured *C_i* is a **lower bound** on the true worst case. The
measurement observed some paths; the worst case may lie on a path not observed
(Wilhelm et al. 2008). An implementation deriving *C_i* by measurement MUST state
the coverage argument and the safety factor applied, or explicitly state that
none was applied.

**W3.** A utilisation ceiling below 100 % is **not** a schedulability criterion
for this model. Under M2 the exact test is (1). A ceiling such as the reference
implementation's 80 % is a *robustness margin against W2* — against the
possibility that *C* is underestimated — and MUST be documented as such rather
than presented as a scheduling result.

### 5. Failure semantics

**F1.** A deadline miss MUST be detectable by the implementation and MUST be
surfaced with a count. A miss that is absorbed silently is indistinguishable from
correct operation in every log and every metric.

**F2.** A missed deadline MUST NOT cause a sample to be fabricated, duplicated,
or silently substituted. The sample is either delivered or reported lost, per
RFC-0003 evidence discipline and the acquisition contract.

**F3.** Sustained deadline misses MUST reach the posture authority so that the
right to act can be withdrawn. Deadline correctness and signal trustworthiness
are separate properties; a chain that meets its deadline while processing an
unusable signal is not a chain that may act.

### 6. Normative requirements

**N1.** An implementation MUST evaluate (1) — or (2) where *hp ≠ ∅* — before a
chain is admitted, and MUST refuse admission when the test fails.

**N2.** The refusal MUST name the term that overran and both sides of the
comparison. `deadline missed` without figures is not a diagnostic.

**N3.** An implementation MUST declare all four terms of (1), **including terms
set to zero**, together with the derivation of each. A declared zero is a claim
about the system and is subject to conformance review; an undeclared zero is a
defect.

**N4.** The budget MUST be re-evaluated on any change of operating point as
defined in M5. An implementation that permits dynamic frequency or wait-state
changes MUST close the budget at every reachable operating point, or MUST refuse
transitions to operating points at which it does not close.

**N5.** The published aggregate figure (the chain's *C*) MUST equal the sum of
its declared stages, and this equality MUST be enforced automatically rather than
maintained by hand.

**N6.** A configuration proof MUST be non-forgeable: the interface that starts a
chain MUST accept only a value that could not have been constructed had the test
failed. Passing a raw sample rate and checking it inside the device is
insufficient, because it places the obligation on every future caller.

### 7. Conformance vectors

An implementation claiming conformance MUST reproduce the following for the
canonical stage table (*C* = 972 µs) with *J* = 6.5 µs, *B* = *I* = 0, and a
robustness ceiling of 80 %:

| *f_s* | Required outcome | Reported figures |
|:--|:--|:--|
| 250 SPS | admitted | R = 978.5 µs, D = 4000 µs, 24.46 % |
| 500 SPS | admitted | R = 978.5 µs, D = 2000 µs, 48.93 % |
| 1000 SPS | **refused** — margin | R = 978.5 µs, D = 1000 µs, 97.85 % > 80 % |
| 2000 SPS | **refused** — deadline | R = 978.5 µs > D = 500 µs |

The 1000 SPS row is the discriminating case: an implementation that admits it has
implemented a deadline test without a robustness policy, and one that reports it
as a missed deadline has conflated the two refusal reasons.

## Drawbacks

The model is deliberately simple, and simplicity has a price. Treating the chain
as one task (M2) forbids the natural optimisation of running a long stage at
lower priority so that a short one can meet a tighter deadline; a system that
wants that must move to a multi-task analysis and this RFC does not cover it.
Requiring re-closure at every operating point (N4) makes aggressive dynamic
frequency scaling awkward, which is a real cost on a battery-powered device and
is the subject of a future RFC.

## Rationale and alternatives

**A utilisation bound instead of a response-time test.** The Liu–Layland bound
*U ≤ n(2^{1/n} − 1)* is sufficient but not necessary, and for *n = 1* it
degenerates to *U ≤ 1*, which (1) already states exactly. Using a utilisation
bound here would trade an exact test for a conservative one and gain nothing.

**Measuring end-to-end rather than summing stages.** An end-to-end measurement is
a single number that cannot be attributed. When it moves — and it moves whenever
anything changes — the question is always *which stage*, and a single number
cannot answer it. Summing declared stages costs nothing at runtime and makes
regression attributable.

**Leaving jitter out of the budget.** Some treatments compare jitter against the
deadline separately. This is wrong under M4: an interrupt that arrives 6.5 µs
late has spent 6.5 µs of the same period the chain is spending.

## Prior art

Response-time analysis in the form used here is due to Joseph & Pandya (1986)
and was extended to release jitter and blocking by Audsley et al. (1993);
blocking bounds under priority inheritance are from Sha, Rajkumar & Lehoczky
(1990). The WCET-measurement caveat in W2 follows the survey of Wilhelm et al.
(2008), whose central point — that measurement bounds observed paths, not all
paths — is the reason W2 is normative rather than advisory. AUTOSAR and ARINC 653
both require declared timing budgets per partition; neither requires the *zero*
terms to be declared, which is the specific obligation N3 adds.

## Unresolved questions

- How should *B* be bounded for the shared SPI path when DMA is in use? A
  measurement-derived bound is available; a protocol-derived one is preferable
  and is not yet specified.
- Whether *I* should be declared per interrupt source or as a single aggregate.
  Per-source is more useful and more onerous.
- Whether a chain admitted at one operating point should be *suspended* or
  *degraded* on a transition to an operating point where it does not close.

## Future possibilities

A machine-readable budget declaration — the stage table, the four terms and
their derivations as a schema'd artifact — would let the conformance suite check
a foreign implementation's arithmetic without reading its source, and would let
the supervisor read the margin at runtime rather than having it compiled in.

## Validation evidence level

Per RFC-0003, the claims in this RFC stand at the following levels:

- **The closure arithmetic and its refusal behaviour** — L1. Enforced by
  compile-time-evaluable code in `axonos-hal` and covered by unit tests,
  including the four conformance vectors of §7.
- **Aggregate equals the sum of stages (N5)** — L1. Asserted by a test that
  fails the build when the published figure and the stage table disagree.
- **The stage execution times themselves** — L2 at best. They are measured
  figures on the reference hardware; the coverage argument required by W2 has
  not been published, and no safety factor has been applied.
- **The claim *B* = *I* = 0** — **not validated at any level.** It is an
  assumption of the reference implementation, not a result (§9).
- **Jitter-limited SNR (3)** — L1 as arithmetic; the σ figures it consumes are
  L2, pending L3 oscilloscope validation per RFC-0003.

This RFC MUST NOT be promoted from draft to active while the *B* = *I* = 0 claim
remains undischarged.

## Conformance status of the reference implementation

`axonos-hal` v0.1.1 implements (1) with *B* = *I* = 0 and does not declare those
terms. Measured against this RFC it is **partially conformant**:

| Requirement | Status |
|:--|:--|
| N1 refuse on failure | conformant |
| N2 name the term and both sides | conformant |
| N3 declare all terms including zeros | **not conformant** — *B* and *I* are absent from the API, not merely zero |
| N4 re-close on operating-point change | **not conformant** — no operating-point concept exists |
| N5 aggregate equals stage sum | conformant, test-enforced |
| N6 non-forgeable proof | conformant — `configure` consumes `TimingBudget` by value |
| F1 deadline miss counted | partially — the supervisor counts acquisition faults; the chain does not yet report its own overrun |
| §7 conformance vectors | conformant, all four |

The gap in N3 is the substantive one. On hardware where the acquisition path
shares a bus with DMA, or where any interrupt may preempt the chain, *B* and *I*
are not zero, and a budget that omits them is optimistic by an unstated margin.
Closing this requires an API change and is the first item for `axonos-hal` 0.2.

## References

See front matter.
