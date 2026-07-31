---
rfc: 0008
title: Deadline Closure for the Acquisition Chain
status: draft
track: kernel
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-07-29
updated: 2026-08-01
implementation:
  - axonos-hal — TimingBudget::close — v0.2.0 adopts the published figures; v0.1.1 deviations recorded in §9
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
for this model. Under M2 the exact test is (1). A ceiling is a *robustness
margin against W2* — against the possibility that *C* is underestimated — and
MUST be documented as such rather than presented as a scheduling result.

**W4.** The ceiling published for this system is **U_max = 0.25** (RFC-0001,
which records the admitted task set at *U* = 0.174 with 0.076 of headroom
remaining). An implementation that applies a different ceiling is applying a
different policy and MUST say so, with its basis. A ceiling chosen for
plausibility rather than derived from a published margin is a number with no
authority, and it will be discovered by the first configuration it wrongly
admits.

### 4a. The published figures already show that *B* and *I* are not zero

RFC-0001 publishes two numbers measured on the same reference platform:

| Quantity | Value | Source |
|:--|--:|:--|
| Σ *C_i* over the admitted task set | 694.2 µs | RFC-0001 §"Pipeline task set as currently admitted" |
| End-to-end WCRT, L2 measured | 972.0 µs | RFC-0001, 12-hour run, 10.8 M epochs |

The difference is **277.8 µs — 28.6 % of the measured worst case.** Release
jitter at the published P99.9 figure accounts for 6.5 µs of it. The remaining
**271.3 µs is blocking, interference, and scheduling overhead**: terms that are
present in the measurement and absent from every admission test written so far.

This is stronger than the observation of §9 that *B* = *I* = 0 is undischarged.
The assumption is **contradicted by the project's own published measurement**,
and the size of the contradiction is a quarter of the budget.

Two obligations follow.

**O1.** An implementation MUST NOT present a measured *response time* as if it
were an execution time *C*. Adding jitter to a figure that already contains it
double-counts one term while continuing to omit two others; the arithmetic
happens to be conservative, and the model is nonetheless wrong. Only the
decomposition makes the error visible, which is why N3 requires it.

**O2.** Until *B* and *I* are measured separately, an implementation MUST use
the measured end-to-end WCRT as *R* directly, and MUST label it as such — not
decompose it into stages it did not measure.

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
published task set (Σ *C_i* = 694.2 µs, RFC-0001) against the published ceiling
*U_max* = 0.25, with the measured end-to-end WCRT *R* = 972 µs:

| *f_s* | *T* | *U* = Σ*C_i*/*T* | *R* ≤ *T* | Required outcome |
|:--|--:|--:|:--|:--|
| 250 SPS | 4000 µs | 0.174 | yes | **admitted** |
| 500 SPS | 2000 µs | 0.347 | yes | **refused** — utilisation ceiling |
| 1000 SPS | 1000 µs | 0.694 | yes | **refused** — utilisation ceiling |
| 2000 SPS | 500 µs | 1.388 | **no** | **refused** — deadline |

The 500 SPS row is the discriminating case. The deadline test passes there —
972 µs fits inside 2000 µs — and the configuration is nonetheless inadmissible
under the published ceiling. An implementation that admits 500 SPS has either
adopted a different ceiling without saying so, or has no ceiling at all.

The 2000 SPS row is the only one refused by the deadline itself; an
implementation that reports the other two as missed deadlines has conflated the
two refusal reasons, which produces an audit trail that misdescribes the
system.

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
- **The four-task execution times of RFC-0001** — L2. Measured on STM32F407
  over a 12-hour run of 10.8 M epochs; the coverage argument required by W2 is
  not published and no safety factor is stated.
- **Any finer decomposition than those four tasks** — **not validated at any
  level, and none is published.** See D1.
- **The claim *B* = *I* = 0** — **refuted**, not merely unvalidated. The
  project's own published figures differ by 277.8 µs between Σ *C_i* and the
  measured end-to-end WCRT (§4a), of which at most 6.5 µs is jitter.
- **Jitter-limited SNR (3)** — L1 as arithmetic; the σ figures it consumes are
  L2, pending L3 oscilloscope validation per RFC-0003.

D1 and D2 were closed in `axonos-hal` 0.2.0, which adopted the published task
set and the published ceiling; the crate now refuses 500 SPS, as RFC-0001's
policy requires.

N4 was closed in 0.3.0, which added operating points and made a frequency
transition a re-admission rather than a setting. The requirement mattered more
than its position in the list suggested: it is what makes power management a
correctness problem in this system, and a device that lowers its clock without
re-closing its budget fails as a missed sample rather than as an error.

D3 stands, and its resolution is a decision rather than a fix: the L2 figures
are measured on STM32F407 while the platform's stated direction is a Cortex-M33
part. An L3 campaign on the newer silicon would validate the *future* platform
while the published numbers describe the *current* one. Until that is settled in
writing — which figures the campaign discharges, and which remain F407
measurements — this RFC MUST NOT be promoted from draft to active.

## Conformance status of the reference implementation

`axonos-hal` v0.3.0 measured against this RFC is **conformant**. D3 remains open and is a decision rather than a defect. The
three deviations below were found by reconciling v0.1.1 against RFC-0001 rather
than by testing it against itself; D1 and D2 are closed, D3 is open and is a
decision rather than a defect. The remaining historical text is kept because a
conformance table that erases its own failures is not evidence of anything, in
ways that were found by reconciling it against RFC-0001 rather than by testing
it against itself.

| Requirement | Status |
|:--|:--|
| N1 refuse on failure | conformant |
| N2 name the term and both sides | conformant |
| N3 declare all terms including zeros | conformant as of 0.2.0 — `close()` takes blocking and interference as a mandatory argument, including when zero |
| N4 re-close on operating-point change | conformant as of 0.3.0 — `AdmittedPoint::transition_to` re-closes the budget at the destination before the device may arrive, and refuses with the receiver unchanged. Execution is scaled from the reference; a measured point overrides the model, and `is_measured()` makes the difference legible |
| N5 aggregate equals stage sum | conformant as arithmetic, but see D1 |
| N6 non-forgeable proof | conformant — `configure` consumes `TimingBudget` by value |
| O1 response time not presented as execution time | conformant as of 0.2.0 — the constant is named `CANONICAL_WCRT_MEASURED_NS` and jitter is no longer added twice |
| O2 use the measured WCRT undecomposed | conformant as of 0.2.0 — the four published tasks replaced the invented seven-way split |
| W4 published ceiling | conformant as of 0.2.0 — the ceiling is RFC-0001's 0.25, and 500 SPS is refused at 0.347 |
| F1 deadline miss counted | partial — the supervisor counts acquisition faults; the chain does not report its own overrun |
| §7 conformance vectors | conformant — all four rows, including the discriminating 500 SPS refusal |

**D1 — a fabricated decomposition.** The crate carries a seven-entry stage
table summing to 972 000 ns, documented as *"measured on the reference
hardware"*. No such per-stage measurement is published anywhere in this
project. RFC-0001 publishes a **four-task** set totalling 694.2 µs and,
separately, an end-to-end WCRT of 972 µs. The seven-way split is an invention
that reproduces the correct total, and presenting it as measurement is exactly
the class of claim RFC-0003 exists to forbid. It also commits the O1 error:
972 µs is a *response* time, and the crate adds jitter to it a second time.

**D2 — a ceiling with no published basis.** The crate applies *U_max* = 0.80.
The published ceiling is 0.25. The consequence is not theoretical: at 500 SPS
the crate reports 48.9 % utilisation and **admits** the configuration, while
the published policy refuses it at 0.347. A configuration the project's own
RFC forbids is currently reachable through the reference implementation.

**D3 — the fixture question.** RFC-0001 and RFC-0002 schedule L3 validation on
an **STM32H573** fixture, while every L2 figure they publish was measured on
**STM32F407**. Validating a Cortex-M33 part does not validate numbers measured
on a Cortex-M4F part. Either the reference platform is moving — in which case
the published figures are legacy and must be labelled so — or the fixture is
the wrong board. This RFC does not resolve the question; it records that it is
open, because an L3 campaign that measures the wrong silicon would consume the
schedule and produce no evidence for the claims it was meant to discharge.

All three are corrected in `axonos-hal` 0.2.0, which adopts the published task
set, the published ceiling, and the measured WCRT undecomposed.

## References

See front matter.
