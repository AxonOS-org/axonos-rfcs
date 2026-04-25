---
rfc: 0003
title: Validation Status Framework (L1/L2/L3)
status: active
track: process
authors:
  - Denis Yermakou <axonosorg@gmail.com>
created: 2026-04-25
updated: 2026-04-25
implementation:
  - axonos.org technology page
  - axonos-rfcs (this repository, all RFCs)
references:
  - IEC 62304:2006/A1:2015 — Medical device software lifecycle processes
  - FDA (2021). Implanted Brain-Computer Interface Devices for Patients with Paralysis or Amputation.
  - aiT WCET analyzer documentation, AbsInt GmbH.
---

# RFC-0003: Validation Status Framework (L1/L2/L3)

## Summary

Every published performance, latency, or correctness claim about AxonOS is classified into one of three validation levels: **L1 (instruction-count derived)**, **L2 (runtime measured)**, or **L3 (independent oscilloscope-validated)**. Each level has explicit evidence requirements and an audit checklist. Claims may not be classified above the level of evidence that exists for them at the time of publication. Pending claims must be marked pending — never silently elevated.

This RFC is the project-wide standard for how AxonOS describes what it has actually built versus what it has designed but not yet measured.

## Motivation

A safety-critical kernel project has a structural temptation: every architectural decision is satisfying to write about long before the corresponding hardware measurement is run. The temptation is to write claims in present tense ("AxonOS achieves 2.1 µs jitter") regardless of whether the measurement supporting that claim has been performed on shipped hardware, on a simulator, or only on paper.

This temptation is not unique to AxonOS — it is endemic in the entire deep-tech and BCI space. The cost of giving in to it is substantial:

- **For investors**, claims that turn out to be aspirational rather than measured destroy the founder's credibility for every future claim.
- **For clinical partners**, claims that conflate "designed for" with "measured at" make the project unreviewable for IEC 62304 lifecycle documentation.
- **For regulators**, claims that the FDA cannot map back to a specific evidence file in the Q-Sub package result in additional information requests and delays.
- **For engineers evaluating AxonOS**, claims that don't classify themselves are simply not useful — there is no way to know whether to trust a number without knowing how it was obtained.

The defence against the temptation is not "be honest" — that is a hope, not a process. The defence is a **standard taxonomy of claims**, applied uniformly to every public surface, with audit checklists that make it inexpensive to verify whether a given claim is correctly classified.

This RFC defines that taxonomy.

## Guide-level explanation

A claim about AxonOS performance, latency, correctness, or behaviour is classified by **how it was obtained**:

- **Level 1 — Instruction-count derived.** The number is computed from the assembly output of the implementation, summed against the published instruction-timing reference for the target CPU. This is conservative: it assumes worst-case pipeline behaviour, no benefit from cache, no benefit from branch prediction. It does not require the code to have been run on hardware. It does require the code to have been compiled for the target.

- **Level 2 — Runtime measured.** The number is observed by executing the implementation on the reference hardware over a stated interval, with a stated input distribution, and a stated measurement instrument (typically the on-chip cycle counter). This validates that the L1 estimate holds under the actual cache, branch prediction, and bus contention conditions of the target platform.

- **Level 3 — Independent oscilloscope-validated.** The number is observed by an external instrument (logic analyzer, oscilloscope, current probe) that is not part of the system under test. This eliminates the possibility that a software-side bug in the measurement instrumentation produces a misleading number. Level 3 evidence is what regulatory submissions ultimately rely on.

A claim is classified at **the level of the evidence that supports it at the time of publication**. A claim that is currently L1 may be elevated to L2 once the hardware measurement is performed. A claim that is currently L2 may be elevated to L3 once the oscilloscope-validated measurement is performed. Claims may not be silently elevated — every elevation must be a documented evidence acquisition with a date, an instrument identification, and a sample size.

A claim that is **not yet measured at any level** is classified as **pending** with an explicit target date. Pending claims may not be presented as if they were measured.

## Reference-level explanation

### Level 1 — Instruction-count derived

**Definition.** The claim is computed from the assembly representation of the implementation, summed against the cycle-timing reference for the target instruction set architecture, assuming worst-case pipeline conditions.

**Evidence requirements.**

- The compiler version, optimisation level, and target triple must be stated. Example: `rustc 1.75.0, --opt-level=3, --target=thumbv7em-none-eabihf`.
- The cycle-count reference document must be cited. Example: ARM, *Cortex-M4 Technical Reference Manual* r0p1.
- The arithmetic must be reproducible — given the same source, compiler, and target, an independent reader must arrive at the same number.
- The assumption set must be stated. Example: "Assumes worst-case pipeline flush on every branch; no instruction-cache hit benefit."

**Audit checklist.**

- [ ] Compiler version, flags, and target triple stated
- [ ] Cycle-timing reference document cited
- [ ] Sum of cycle counts for the path matches the claimed total
- [ ] Worst-case assumptions explicitly listed
- [ ] No L2 or L3 evidence is silently incorporated

**What L1 does not validate.**

- That the cache will behave under runtime contention as assumed
- That bus arbitration will not extend the path
- That interrupt service will not extend the path
- That the compiler will produce identical output in future compilations

**Where L1 is appropriate.**

L1 is the right evidence level for early architectural claims, scheduler admission tests, and theoretical worst-case analysis. It is not appropriate for claims that hardware behaviour validates the analysis — those require L2 or higher.

### Level 2 — Runtime measured

**Definition.** The claim is observed by executing the implementation on the reference hardware platform, with measurement performed by an on-chip instrument (typically the cycle counter or a hardware timer), over a stated interval and under a stated input distribution.

**Evidence requirements.**

- The reference hardware (board, CPU SKU, clock speed, peripherals) must be identified.
- The measurement instrument must be stated. Example: "DWT cycle counter, 168 MHz tick rate, sampled at every pipeline epoch."
- The measurement interval must be stated. Example: "12 continuous hours, 10.8 M samples."
- The input distribution must be stated. Example: "BCI Competition IV Dataset 2a replayed via ADS1299 test-signal mux."
- The summary statistic must be computed honestly: maximum (worst-case), 99.9th percentile, median, mean, standard deviation. Headline numbers should typically use maximum or 99.9th percentile, not average.
- The raw measurement data must be retained or made available. For closed-source future deployments where this is not possible, the measurement protocol must be reproducible by an independent party with the same hardware.

**Audit checklist.**

- [ ] Hardware platform identified (board, CPU, clock speed)
- [ ] Measurement instrument stated
- [ ] Sample size stated
- [ ] Interval stated
- [ ] Input distribution stated
- [ ] Summary statistic chosen honestly (maximum or P99.9 for hard-real-time claims)
- [ ] Raw data retained or methodology reproducible

**What L2 does not validate.**

- That an external observer would measure the same number — software-side measurement instrumentation can have bugs.
- That the result holds across different units of the same hardware (silicon variation).
- That the result holds across operating temperature range.
- That the result holds in the presence of unmodelled radio interference or EMI.

**Where L2 is appropriate.**

L2 is the right evidence level for engineering claims to peer engineers, for benchmark publication, and for early-stage investor evidence. It is not yet sufficient for regulatory submission.

### Level 3 — Independent oscilloscope-validated

**Definition.** The claim is observed by an instrument that is independent of the system under test. The instrument is not running the same code as the implementation. The measurement does not depend on any software-side timing reference.

**Evidence requirements.**

- The instrument (logic analyzer, oscilloscope, network analyzer, current probe) must be identified by make and model. Example: "Saleae Logic Pro 16 at 100 MHz sample rate, 10 ns resolution."
- The signal being measured must be physically observable. For latency claims, the typical pattern is GPIO pin toggling: producer toggles pin HIGH at start of path, consumer toggles pin LOW at end; the pulse width is the path latency.
- The instrument calibration must be current. For high-precision claims, the calibration certificate is part of the evidence file.
- Multiple measurements over multiple hardware units are recommended to bound silicon variation.
- The raw waveform capture or logic-analyzer trace must be retained.

**Audit checklist.**

- [ ] Instrument make, model, and resolution stated
- [ ] Signal physically observable (oscilloscope photo or logic-analyzer trace)
- [ ] Instrument calibration current
- [ ] Multiple samples, multiple units where possible
- [ ] Raw trace retained

**What L3 validates.**

- The actual end-to-end timing as observed externally
- The accuracy of L1 and L2 estimates
- The absence of software-side measurement bugs
- The stability of the result across silicon variation

**Where L3 is required.**

- Hard-real-time claims used in regulatory submissions
- Headline performance claims to clinical partners
- Investor due-diligence packages
- Any claim that contradicts Level 2 measurements taken with the same instrumentation

### Pending state

A claim that is not yet evidenced at any level is **pending** and must be tagged with:

- The intended evidence level (typically L3 if the claim has any clinical or regulatory implication).
- A target date for evidence acquisition.
- The blocker, if any (e.g., "requires H573 evaluation board fixture, in procurement").

Pending claims may be discussed in design documents, RFCs, and architectural articles. They may not be quoted as if they were measured. The phrase "GPIO-measured" is only appropriate when L3 evidence exists; until then the equivalent text reads "L1 instruction-count derived (L3 GPIO validation pending Q2 2026)" or similar.

### Application of the framework

This framework applies to:

- The axonos.org technology page
- Every RFC in this repository
- Every architectural article published on Medium or dev.to
- Every benchmark report (the canonical AxonOS Article #12)
- Every investor or clinical pitch deck
- Every regulatory submission

It does not apply to:

- Forward-looking statements clearly labelled as roadmap or plan
- Explicitly aspirational design documents (which should still distinguish "designed for" from "measured at")
- Documentation of external standards or third-party claims (which carry their own evidence)

### Mistake recovery

If a published claim is later discovered to have been classified above its actual evidence level, the recovery process is:

1. Edit the original publication with a dated correction note.
2. Reduce the claim's classification level to match the actual evidence.
3. If an investor, clinical partner, or regulator was the audience, send a written correction to that audience.
4. Open an issue in this repository titled "Corrigendum: <publication>" with the original claim, the corrected claim, and the cause.

This process protects the credibility of the L1/L2/L3 framework itself. A framework that tolerates silent corrections has no value.

## Drawbacks

Maintaining the framework adds documentation overhead. Every claim must carry its level. Every elevation must carry an evidence file. This is not free.

The framework can come across as defensive or self-deprecating in marketing contexts. Engineers who are accustomed to deep-tech projects that conflate L1 and L3 claims may initially read AxonOS's careful classification as "less impressive" than competitors' undifferentiated claims. The defence is engagement: the careful classification is the differentiator, not a weakness.

The framework requires the project to acquire L3 instrumentation (logic analyzer, oscilloscope, current probe) before the corresponding claims can be elevated. This has a cost in equipment and time.

## Rationale and alternatives

### Alternative 1 — informal "tested vs not tested" labelling

A binary "this is measured / this is theoretical" distinction. Simpler.

**Rejected because** it does not distinguish runtime measurement from independent observation. A software-side cycle counter measurement can be wrong in ways that an oscilloscope cannot. The L2/L3 distinction is exactly the distinction that matters for regulatory submission.

### Alternative 2 — adopt FDA evidence levels directly

The FDA has its own evidence taxonomy for medical device claims (bench testing, animal study, clinical study).

**Rejected because** AxonOS is software; FDA's evidence levels are defined for medical interventions, not software performance characteristics. The L1/L2/L3 framework is software-specific and aligns with what IEC 62304 lifecycle documentation expects.

### Alternative 3 — IEEE 1012 Verification and Validation classification

A formal V&V classification with five levels of independence.

**Rejected because** the IEEE 1012 levels are designed for V&V of full systems against requirements, not for individual performance claims. The simpler L1/L2/L3 taxonomy is more useful for the "how was this number obtained" question that this framework addresses.

### Alternative 4 — no formal classification

Just write claims and let readers assess.

**Rejected because** this is the status quo in deep-tech and is exactly the practice that destroys founder credibility when claims turn out to be aspirational. The point of the framework is to make the classification cheap and explicit so it actually gets done.

## Prior art

- **aiT / OTAWA (AbsInt GmbH and University of Toulouse)** — academic and commercial WCET analysis tools. The L1 instruction-count approach is what these tools systematically automate.
- **The Linux real-time community's `cyclictest` benchmark** — a canonical L2 measurement tool. Its results are typically cited without further independent validation, which is acceptable for the Linux RT community's purposes but illustrates the L2 ceiling.
- **NIST SP 800-90B** — the entropy-source validation standard. Notable as an example of a regime that requires both software-side and hardware-side evidence for the same claim.
- **The DARPA HACMS programme** — used formally verified components for safety-critical systems. The "verified" status under HACMS corresponds approximately to a "Level 4" beyond L3 — formal proof. AxonOS does not yet pursue Level 4 but reserves the option.

## Unresolved questions

- **Should there be a Level 4 (formally verified)?** Tools like Kani, Prusti, and CBMC could verify properties of the Rust source. This RFC does not adopt Level 4 because no current AxonOS claim is formally verified, but a future RFC may. The framework is extensible.
- **How should L3 evidence be archived for long-term preservation?** Logic analyzer traces are large binary files. The current plan is to retain raw files locally with SHA-256 manifest, mirroring the manifest publicly. A more durable scheme may be appropriate for regulatory submission.
- **How should this framework apply to claims about third-party components AxonOS depends on?** For example, a claim about ADS1299 noise performance derives from Texas Instruments' datasheet. This is not L1, L2, or L3 in our framework — it is "vendor-claimed." A future revision may add a "V" tier for vendor-claimed evidence.

## Future possibilities

- An automated checker that grep's the project's documentation for performance claims and verifies each carries an L1/L2/L3 tag.
- Integration with the regulatory submission package generator (out of scope today, in scope for the FDA Q-Sub workstream).
- A public dashboard showing every published L2 measurement with its sample size, instrument, and date.

## Validation evidence level

This RFC does not make performance claims; it defines the classification for them. The framework itself is validated by being applied uniformly across the project.

## References

1. IEC 62304:2006 + A1:2015. *Medical device software — Software life cycle processes*. International Electrotechnical Commission.
2. FDA (2021). *Implanted Brain-Computer Interface (BCI) Devices for Patients with Paralysis or Amputation — Non-Clinical Testing and Clinical Considerations*. Draft Guidance.
3. AbsInt GmbH. *aiT WCET Analyzer — Technical Documentation*. https://www.absint.com/ait/
4. Wilhelm, R. et al. (2008). *The worst-case execution-time problem—overview of methods and survey of tools*. ACM TECS 7(3).
5. NIST SP 800-90B (2018). *Recommendation for the Entropy Sources Used for Random Bit Generation*.

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). It is the project-wide validation standard and may be cited by other AxonOS documents.*
