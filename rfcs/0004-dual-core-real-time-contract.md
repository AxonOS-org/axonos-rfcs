---
rfc: 0004
title: Dual-Core Real-Time Contract
status: active
track: scheduling
authors:
  - Denis Yermakou <axonosorg@gmail.com>
created: 2026-04-25
updated: 2026-04-25
implementation:
  - axonos-kernel partition module (in development)
references:
  - ARINC 653 — Avionics Application Software Standard Interface
  - Burns, A. & Wellings, A. (2009). Real-Time Systems and Programming Languages (4th ed.).
  - AxonOS Article #35 — The Dual-Core Real-Time Contract.
---

# RFC-0004: Dual-Core Real-Time Contract

## Summary

The AxonOS reference platform runs on a heterogeneous dual-core topology: a Cortex-M4F DSP core handles signal processing under hard real-time constraints; a Cortex-A53 application core handles session state, network I/O, and WASM-sandboxed application servicing under soft real-time constraints. The cores communicate via shared-SRAM IPC. This RFC defines the **six-clause real-time contract** that governs the partition: DC1 (DSP WCET bound), DC2 (IPC latency bound), DC3 (wake-up determinism), DC4 (fault containment), DC5 (graceful degradation), DC6 (attestation propagation). Every clause has a falsification criterion. Violation of any clause is a kernel-level fault.

## Motivation

A single-core real-time system has one place where the worst-case execution must be bounded. A dual-core heterogeneous system has many: each core's local execution, the inter-core communication path, the synchronisation of state between cores, and the failure behaviour when one core misbehaves. Without a written contract, the dual-core property degenerates from a feature into a liability — every cross-core bug becomes a bespoke investigation.

The contract written below is the result of choosing, for each cross-core property the kernel cares about, **a single explicit guarantee with a falsification criterion**. A property without a falsification criterion is a hope, not an engineering commitment. A property with a falsification criterion can be measured, tested, and audited.

## Guide-level explanation

The dual-core partition divides AxonOS responsibilities by criticality:

- **Cortex-M4F DSP core (168 MHz)** — runs the signal pipeline. Hard real-time. Subject to RFC-0001's EDF schedulability discipline. Owns the consent state machine and the stimulation interlock (RFC-0005). Statically allocated, no allocator on the hot path.

- **Cortex-A53 application core (1.2 GHz)** — runs session state, network I/O, the WASM application sandbox. Soft real-time. Has a heap (used for application-side data only). Linux-style mutable state for session management.

The cores communicate via two shared-SRAM rings: one DSP→A53 (intent events flowing up to applications) and one A53→DSP (control events flowing down to consent and stimulation). Both rings use the protocol of RFC-0002.

The contract states what each core promises to the other and to the system as a whole:

- **DC1** — The DSP guarantees the signal pipeline completes within its budget every epoch.
- **DC2** — The IPC delivers messages with bounded latency.
- **DC3** — The A53 wakes deterministically when the DSP signals.
- **DC4** — A failure on either core does not corrupt the other core's safety-critical state.
- **DC5** — When one core stops responding, the other transitions to a defined safe state.
- **DC6** — Cryptographic attestation is preserved across the IPC boundary.

These clauses together define what "dual-core real-time" means in AxonOS. A future deployment, hardware variant, or implementation must satisfy all six or document explicit deviations.

## Reference-level explanation

### DC1 — DSP WCET bound

**Statement.** The DSP signal pipeline (defined by RFC-0001 admission) completes within its scheduled deadline every epoch. The current admitted task set has total utilization U = 0.174 (RFC-0001 § Pipeline task set), well below the U_max = 0.25 admission ceiling.

**Falsification criterion.** Any single epoch where the pipeline does not produce a published intent event within the period boundary (4 ms for the current configuration). The DSP cycle counter is sampled at the start of each epoch and at intent-publish; the difference is logged.

**Current evidence.** Measured WCRT 972 µs over 12 hours / 10.8 M epochs, zero deadline misses (Level 2). Independent oscilloscope validation pending Q2 2026 (Level 3).

### DC2 — IPC latency bound

**Statement.** A message published to either inter-core ring is observed by the consuming core within 0.2 µs.

**Falsification criterion.** Any IPC delivery measurement, GPIO-instrumented end-to-end, exceeding 0.2 µs in normal operation. ("Normal operation" excludes the ~10 µs interrupt re-dispatch path on the consuming core; the 0.2 µs bound is for the IPC mechanism itself, not the responding task's wake-up time.)

**Mechanism.** The IPC follows the protocol of RFC-0002. A producer's `seq.store(index + 1, Release)` becomes visible to the consumer's `seq.load(Acquire)` within the cache coherence latency of the hardware platform. On the reference platform, the M4F and A53 share an L2 cache; observed cross-core sync time is approximately 80 ns plus the memory-system latency.

**Current evidence.** Sub-0.2 µs measured during 12-hour pipeline run (Level 2). Independent oscilloscope validation pending Q2 2026 (Level 3).

### DC3 — Wake-up determinism

**Statement.** When the DSP completes an epoch and publishes an intent, the A53 application servicing task is dispatched to handle that intent within 50 µs.

**Falsification criterion.** Any A53 dispatch delay exceeding 50 µs as measured from DSP intent-publish (DC2 boundary) to A53 application-task entry.

**Mechanism.** The DSP publishes via the IPC ring, then triggers an inter-processor interrupt (IPI). The A53 IPI handler is registered as a high-priority interrupt that pre-empts whatever the A53 was doing (subject only to other higher-priority interrupts being already in service). The handler dispatches to the application servicing task, which reads the IPC ring per RFC-0002.

**Current evidence.** Measured 281 µs total A53 pipeline (Article #12 § A53 pipeline), within which the dispatch component is < 50 µs (Level 2). Independent validation pending.

### DC4 — Fault containment

**Statement.** A failure on the A53 (panic, hang, segfault, OOM) does not corrupt the DSP's signal pipeline state, the consent state machine, or the stimulation interlock.

**Falsification criterion.** A test in which the A53 is deliberately faulted (e.g., kernel panic via debug interface) and the DSP is then observed to either (a) fail to produce a correct intent event from a known-good input, (b) fail a consent state transition that should be valid, or (c) violate the stimulation interlock invariants.

**Mechanism.**

- DSP-owned state is in DSP-local SRAM. The A53 has read-only access to this region (mapped via the bus matrix's permission settings) and cannot write to it.
- The consent state machine is stored in DSP-local SRAM with a CRC-32 over the live state. Any A53-side memory bug that corrupted DSP SRAM (which it cannot, by hardware) would also need to update the CRC, which it doesn't have the code for.
- The stimulation interlock is a Secure World partition (TrustZone-S on the H573 target). The A53 has no access to Secure World resources by hardware design.

**Current evidence.** Reasoning argument plus bus-matrix configuration verification (Level 1 for the configuration). Hardware fault-injection testing planned for Phase 2 (Level 2 / Level 3).

### DC5 — Graceful degradation

**Statement.** When the A53 stops responding (no IPC messages observed for 10 ms), the DSP transitions the system to a defined safe-idle state: continues to acquire signal samples (so the consent state machine remains coherent), but ceases publication of intent events to applications until the A53 is observed responsive again.

**Falsification criterion.** A test in which the A53 is held in reset for ≥ 10 ms during normal pipeline operation, and the DSP is observed to either (a) panic, (b) continue publishing intent events into the now-dead IPC ring (overrun would not be observable to anyone), or (c) violate any DC1 or DC2 commitment in the meantime.

**Mechanism.** The DSP monitors the consumer-side of the DSP→A53 ring for liveness. When the consumer's `tail` does not advance within the staleness threshold (10 ms = 2.5× the epoch period), the DSP suspends intent-event publication and emits a "consumer-stalled" diagnostic event into a separate diagnostic ring. The signal pipeline itself continues; only the user-facing event publication stops.

When the A53 becomes responsive again, the DSP detects the resumed `tail` advancement and resumes intent publication. There is no resync handshake — the consent state machine and signal pipeline state were continuous on the DSP side throughout.

**Current evidence.** Mechanism designed (Level 1). Runtime fault-injection testing planned for Phase 2.

### DC6 — Attestation propagation

**Statement.** Every IntentObservation event published by the DSP carries a truncated HMAC-SHA256 attestation tag computed using a per-session key resident in the ATECC608B secure element. The tag is propagated unchanged across the IPC boundary; the A53 must verify it before delivering the event to user-space applications. Events without a valid tag, or with tags that do not match the per-session key, are rejected at the application boundary.

**Falsification criterion.** Any event that reaches a user-space application with an invalid or absent attestation tag.

**Mechanism.**

- At session start, the DSP requests a fresh HMAC key from the ATECC608B. The key never leaves the secure element; the DSP receives a key handle.
- For each intent event, the DSP submits the event payload to the secure element, which returns a 16-byte (truncated) HMAC-SHA256 tag.
- The tag is appended to the event before publication on the IPC ring.
- The A53's application-servicing task verifies the tag on every event before delivering it to user-space.
- If the tag is missing or fails verification, the event is dropped and a `Error::AttestationFailed` diagnostic is emitted. This is treated as a serious anomaly (potential memory corruption between DSP write and A53 read).

**Current evidence.** HMAC-SHA256 mechanism specified (Level 1). Runtime measurement of attestation overhead pending Phase 1 H573 fixture (Level 2/3).

### Mapping to the swarm contract

A future RFC on swarm real-time will extend this contract to multi-node deployments. The dual-core contract is the single-node primitive that the swarm contract composes — see AxonOS Article #36 for the swarm-level analysis.

### Summary table

| Clause | What | Bound | Evidence Level |
|:---|:---|:---|:---:|
| DC1 | DSP WCET bound | 4 ms epoch, U ≤ 0.25 | L2 |
| DC2 | IPC latency bound | ≤ 0.2 µs | L2 |
| DC3 | A53 wake-up determinism | ≤ 50 µs dispatch | L2 |
| DC4 | Fault containment | A53 fault → DSP unaffected | L1 (config), L2/L3 pending |
| DC5 | Graceful degradation | A53 stall → DSP safe-idle | L1 (design), L2 pending |
| DC6 | Attestation propagation | HMAC-SHA256 over every event | L1 (design), L2/L3 pending |

## Drawbacks

The contract is verbose. Six clauses × falsification criteria × evidence files is more documentation than a single-core kernel needs. The cost is paid once at design time and amortised over every subsequent deployment.

The asymmetric core assignment (DSP hard real-time, A53 soft real-time) leaves the A53's full performance under-utilised. A more aggressive design would push more pipeline stages onto the A53. The current asymmetry is deliberate — the A53's complexity (heap, threads, network stack) is exactly what disqualifies it from the consent-bearing path.

## Rationale and alternatives

### Alternative 1 — single-core (Cortex-A53 only) with PREEMPT_RT

Push everything onto the A53 with mainline Linux RT.

**Rejected because** the A53 with Linux cannot meet DC1's 4 ms epoch with bounded WCET and zero deadline misses. PREEMPT_RT's worst-case latency is in the 80–160 µs range (Madden 2020); add the kernel's open-ended interrupt processing and the bound is unprovable.

### Alternative 2 — single-core (Cortex-M4F only) with no application core

Drop the A53 entirely; run everything bare-metal.

**Rejected because** the application surface AxonOS exposes (WASM-sandboxed apps, network I/O, session management) does not fit comfortably on the M4F's 192 KB SRAM. The asymmetric design separates "things that need an MMU and a heap" from "things that need bounded WCET" — both are real needs.

### Alternative 3 — two A53 cores with partitioned scheduling

Use two A53 cores and partition statically. Avoid the heterogeneous-ISA complexity.

**Rejected because** the M4F's deterministic instruction timing is what enables the L1-derivable WCET claims. A53's out-of-order pipeline and deeper memory hierarchy resist the same analysis. Heterogeneous is a feature here, not a bug.

### Alternative 4 — formal-process partition (ARINC 653) on a single SoC

Use ARINC 653 partitioning on a single processor with strict time-windowed scheduling.

**Rejected because** ARINC 653 is excellent prior art for partitioning but does not fit the AxonOS workload — the signal pipeline is event-driven (epoch boundary) rather than cyclic-window-driven. The dual-core approach gives ARINC 653-style isolation with event-driven semantics within each partition.

## Prior art

- **ARINC 653** — avionics partitioning standard. Influences DC4 and DC5 but doesn't fit the event-driven workload.
- **AUTOSAR Adaptive** — automotive multi-core RT. Solves similar problems for ADAS workloads.
- **Burns & Wellings (2009)** — *Real-Time Systems and Programming Languages*, the canonical textbook treatment.
- **NVIDIA Drive AGX** — heterogeneous safety-critical compute. The pattern of "hard-real-time microcontroller + soft-real-time application processor on shared memory" is established in automotive autonomy.
- **Tesla FSD compute** — uses a similar safety-critical microcontroller plus application processor partitioning.

## Unresolved questions

- **Is 50 µs the right DC3 bound, or should it be tighter?** The current bound is set conservatively to leave headroom for A53 cache misses on first dispatch. Tightening would require characterising the A53 cache behaviour more precisely.
- **What is the right policy for repeated A53 stalls?** A single 10 ms stall triggers safe-idle and resumes. Repeated stalls (e.g., 5 stalls in 1 minute) might warrant a more aggressive response — perhaps full session termination. This is a deployment-policy question more than a kernel question. Deferred.
- **Does DC4 hold under intentional rather than accidental A53 fault?** Hostile A53 firmware that exploits a not-yet-discovered hardware issue is a different threat model. The DC4 argument as stated covers accidental faults; deliberate adversarial faults require additional analysis.

## Future possibilities

- A formally verified bus-matrix configuration check that confirms DC4 at every kernel boot.
- Online liveness monitoring of A53 with telemetry for the deployment operator.
- A swarm-level extension of this contract for multi-node deployments (subject of a future RFC).

## Validation evidence level

Per-clause evidence level is in the summary table above. The contract as a whole is at **L1 (design)** for fault-containment and graceful-degradation claims; **L2 (runtime measured)** for DC1, DC2, DC3 timing claims; pending L3 oscilloscope validation for all timing claims, scheduled Q2 2026.

## References

1. Burns, A. & Wellings, A. (2009). *Real-Time Systems and Programming Languages* (4th ed.). Addison-Wesley.
2. ARINC 653 — *Avionics Application Software Standard Interface, Part 1: Required Services*. ARINC, 2015.
3. AUTOSAR. *Adaptive Platform Specification*. https://www.autosar.org/
4. AxonOS Article #35 — *The Dual-Core Real-Time Contract*. Medium, 2026.
5. AxonOS Article #36 — *Swarm Real-Time*. Medium, 2026.
6. Madden, M. M. (2020). *Challenges Using Linux as a Real-Time Operating System*. NASA TM-2020–220568.

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). Implementation in [axonos-kernel](https://github.com/AxonOS-org) under Apache-2.0 OR MIT.*
