---
rfc: 0007
title: Swarm Real-Time Contract
status: draft
track: scheduling
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-06-04
updated: 2026-06-04
implementation:
  - axonos-swarm neural_ptp module (NeuralPtpKalman)
  - axonos-swarm swarm module (SwarmScheduler)
  - axonos-swarm fault module (SwarmFaultDetector)
references:
  - AxonOS RFC-0004 — Dual-Core Real-Time Contract.
  - AxonOS RFC-0001 — EDF Scheduler with Biological Deadlines.
  - AxonOS RFC-0003 — Validation Status Framework (L1/L2/L3).
  - IEEE 1588 — Precision Time Protocol (PTP).
  - Welch, G. & Bishop, G. — An Introduction to the Kalman Filter.
  - Cristian, F. (1989). Probabilistic Clock Synchronization.
---

# RFC-0007: Swarm Real-Time Contract

## Summary

RFC-0004 bounds the real-time behaviour of a single AxonOS node. This RFC defines the contract that governs a *group* of such nodes acting as one coordinated mesh. The contract is stated as a **seven-clause swarm real-time contract** — SC0 through SC6 — covering clock-offset uncertainty (SC0), preservation of each node's local pipeline budget (SC1), globally synchronised epoch release (SC2), bounded co-availability of intent outputs (SC3), detection of misbehaving peers (SC4), degradation to local-only operation (SC5), and probabilistic delivery bounding (SC6). Each clause has a falsification criterion and an explicit evidence level under RFC-0003. The contract is implemented in the `axonos-swarm` crate (`#![no_std]`, no allocator on the real-time path). It is an **engineering contract, not a clinical-certification claim**, and several clauses are explicitly analytical or research-level today.

## Motivation

A single bounded node does not make a bounded mesh. RFC-0004 establishes that one node's pipeline has a worst-case response time, a fault-containment story, and a graceful-degradation rule. As soon as two or more nodes are expected to behave as a single coordinated BCI surface, a new class of properties appears that no single-node contract can express:

- the nodes do not share a clock, only noisy estimates of each other's clocks;
- each node must still meet its *local* deadline while also honouring a *shared* schedule;
- the useful output is not one node's intent but the *joint* availability of several nodes' intents within a usable window;
- a peer can fail silently, drift, or emit inconsistent state, and the mesh must notice;
- the mesh must degrade to something safe rather than to undefined behaviour when peers are lost.

Without a written contract, each of these becomes a bespoke investigation per bug, and "distributed" degenerates from a feature into a liability. Following the discipline of RFC-0004, this RFC chooses, for each cross-node property the mesh cares about, **a single explicit guarantee with a falsification criterion**. A property without a falsification criterion is a hope; a property with one can be measured, tested, and audited. Where a guarantee is not yet measured, this RFC says so rather than overstating it (RFC-0003).

## Guide-level explanation

`axonos-swarm` is deliberately independent of `axonos-sdk` and `axonos-consent`: it can be reviewed as a small timing-and-fault-coordination crate without the full application stack. It provides three pieces:

- **`NeuralPtpKalman`** (module `neural_ptp`) — a Kalman-filtered estimator of the local clock's offset from a reference, designed for noisy, wireless, PTP-style measurements. It produces both an offset estimate and an uncertainty (variance), which is what makes a *bounded* synchronisation claim possible rather than a point estimate.

- **`SwarmScheduler`** (module `swarm`) — computes the local release time for a globally synchronised **4 ms swarm epoch**. Each node runs its own pipeline locally, but releases its epoch against the shared global timeline derived from the PTP estimate, so that nodes are working on the same epoch at (approximately) the same time.

- **`SwarmFaultDetector`** (module `fault`) — tracks peers for silence, degradation, desynchronisation, and Byzantine-like inconsistency, and reports their health so the mesh can react.

The contract below states what each node promises the mesh, and what the mesh promises the system. It builds directly on RFC-0004: SC1 is RFC-0004's local contract, *inherited unchanged*; SC0 and SC2–SC6 are the additional obligations that appear only because there is more than one node.

The crate follows the same constraints as the rest of AxonOS: no allocator on the real-time path; no hidden background coordination state; no unbounded retry loops in fault-sensitive paths; no claim above its evidence level; and no dependence on application-layer trust for timing correctness.

## Reference-level explanation

The mesh comprises *N* AxonOS nodes, each internally compliant with RFC-0004. Nodes exchange timing and health information over an unreliable, variable-latency link. There is no shared physical clock and no central coordinator; each node maintains its own estimate of the global timeline via `NeuralPtpKalman`. The global epoch period is **4 ms**.

Notation: `σ_offset` is the standard deviation of a node's clock-offset estimate as reported by the Kalman filter; `W` is the co-availability window computed by `SwarmScheduler`; "epoch" means one 4 ms swarm period.

### SC0 — Bounded clock-offset uncertainty

**Guarantee.** Each node's clock-offset estimate, as produced by `NeuralPtpKalman`, keeps its uncertainty within a bounded 3σ envelope under the filter's modelled noise assumptions; the estimate carries that uncertainty explicitly rather than presenting a bare point value.

**Mechanism.** The Kalman filter maintains offset and drift state with an associated covariance. `predict(dt)` propagates state and inflates covariance over elapsed time; `update(measurement)` folds in a new PTP-style measurement and contracts covariance. The reported variance is the basis for every downstream "bounded" claim.

**Falsification.** Observed clock offset exceeds the filter's reported 3σ envelope materially more often than 3σ statistics permit, over a sustained measurement run on the target link.

**Evidence level.** L1 (analytical model + unit/convergence tests). Hardware validation pending — no L3 oscilloscope/GPIO trace is claimed.

### SC1 — Local pipeline budget preserved

**Guarantee.** Participation in the mesh does not cause any node to violate its own RFC-0004 local pipeline WCRT budget. The local real-time contract is preserved unchanged.

**Mechanism.** Swarm coordination runs as bounded, statically sized work; it does not introduce an allocator, an unbounded loop, or a blocking dependency on a peer into a node's local hot path. A node computes its release time and health locally and never waits indefinitely on the mesh to meet a local deadline.

**Falsification.** Enabling swarm participation pushes a node's measured local response time beyond its RFC-0004 budget on the single-node fixture.

**Evidence level.** Inherited from RFC-0004's single-node model (L1 by inheritance). Swarm-on/off differential measurement is a pending L2 task.

### SC2 — Synchronised epoch release

**Guarantee.** Nodes release their pipeline epochs against a *shared global epoch* derived from the synchronised timeline, rather than against unrelated local clocks.

**Mechanism.** `SwarmScheduler` combines the node's local time with the `NeuralPtpKalman` offset estimate to compute the local release instant of the next 4 ms global epoch boundary. All nodes computing against the same global timeline therefore release the same epoch within their synchronisation uncertainty.

**Falsification.** With a known reference offset injected, `SwarmScheduler` releases an epoch whose computed global-epoch phase deviates from the expected boundary by more than the synchronisation uncertainty.

**Evidence level.** L1 — implemented and unit-checked in `SwarmScheduler`.

### SC3 — Bounded co-availability window

**Guarantee.** The intent outputs of synchronised nodes for a given epoch become co-available within a bounded window `W`, and `W` is computed conservatively from the current synchronisation uncertainty rather than assumed.

**Mechanism.** `SwarmScheduler::co_availability_window_us()` returns the window (in microseconds) within which co-released epochs are expected to be jointly available, as a function of the offset uncertainty. The window widens as uncertainty grows and is reported as `None` when it cannot be established, so a consumer never receives a fabricated bound.

**Falsification.** Across a run, the fraction of epochs whose peer outputs fall within the reported `W` is materially lower than the window's stated confidence.

**Evidence level.** L1 — implemented as a conservative window calculation. Empirical window-coverage measurement is a pending L2 task.

### SC4 — Peer fault detection

**Guarantee.** Peers that are silent, degraded, desynchronised, or emitting inconsistent (Byzantine-like) state are detected and surfaced as health reports.

**Mechanism.** `SwarmFaultDetector` tracks per-peer liveness and behaviour using fixed-size state and bounded logic — no unbounded retry, no heap on the path. It classifies a peer along silence, degradation, desynchronisation, and inconsistency axes and exposes the result for the mesh (and, per the roadmap, eventually the consent layer) to act on.

**Falsification.** A peer driven into a defined fault mode (forced silence, injected drift beyond threshold, or contradictory reports) is *not* flagged by `SwarmFaultDetector` within its specified detection horizon.

**Evidence level.** L1 — implemented and unit-checked in `SwarmFaultDetector`.

### SC5 — Local-only fallback

**Guarantee.** A mesh that has degraded — too few healthy peers, or synchronisation lost — falls back to safe local-only operation rather than to undefined coordinated behaviour. A node that cannot trust the mesh continues to honour its own RFC-0004 contract alone.

**Mechanism.** Fallback is an architectural rule: mesh coordination is advisory over a node that is independently real-time-correct. When `SwarmFaultDetector` reports the mesh as unusable, the node stops depending on co-availability and runs as a single RFC-0004 node.

**Falsification.** Under induced mesh degradation, a node enters an undefined or unsafe coordinated state instead of clean local-only operation.

**Evidence level.** L1 (architectural rule). The concrete integration policy — exact thresholds and the consent-layer handshake — is pending and is an open question below.

### SC6 — Probabilistically bounded cross-node delivery

**Guarantee.** Cross-node delivery of coordination/intent information is characterised by a *probabilistic* bound (a delivery-within-deadline probability), not a hard real-time delivery guarantee.

**Mechanism.** The transport is unreliable and variable-latency; this clause deliberately makes a weaker, honest claim than SC2/SC3's per-node timing. It is framed in the tradition of probabilistic clock synchronisation and probabilistic real-time analysis.

**Falsification.** A stated delivery-within-deadline probability is not met over a sufficiently long measurement on the target link.

**Evidence level.** Research target. **Not** yet a hard runtime claim and not yet measured; listed here so it is not silently assumed by a reader.

### Summary table

| Clause | Guarantee | Primary mechanism | Evidence today |
|---|---|---|---|
| SC0 | Bounded 3σ clock-offset uncertainty | `NeuralPtpKalman` covariance | L1; HW validation pending |
| SC1 | Local WCRT budget preserved | Bounded, non-blocking swarm work | Inherited (RFC-0004) |
| SC2 | Synchronised epoch release | `SwarmScheduler` global-epoch phase | L1 |
| SC3 | Bounded co-availability window | `co_availability_window_us()` | L1; coverage L2 pending |
| SC4 | Peer fault detection | `SwarmFaultDetector` | L1 |
| SC5 | Local-only fallback | Architectural advisory rule | L1; policy pending |
| SC6 | Probabilistic delivery bound | Probabilistic transport model | Research target |

## Drawbacks

- The contract is wider than its current evidence. SC0, SC3, and SC6 in particular make claims whose *empirical* (L2) and *independent* (L3) confirmation does not yet exist; publishing the contract before the traces invites the criticism that it is aspirational. The mitigation is the explicit per-clause evidence level and falsification criterion — the contract is written so that it can be disproven, not just believed.
- A seven-clause contract is a maintenance surface: each clause must stay consistent with the `axonos-swarm` code, RFC-0003's taxonomy, and RFC-0004's parent contract. Drift between any of these is itself a defect.
- SC5 and SC6 are policy/research clauses, not implemented runtime guarantees; a careless reader could over-read them. They are labelled accordingly.

## Rationale and alternatives

### Alternative 1 — a single hard real-time delivery guarantee (no SC6 probabilistic clause)

Claiming hard cross-node delivery would be simpler to state but false over a wireless, variable-latency link. SC6's probabilistic framing is chosen precisely so the contract does not overclaim the transport.

### Alternative 2 — central coordinator / master clock

A single master clock would simplify SC0–SC2 but introduce a single point of failure incompatible with SC5's degradation goal and with a mesh of independently real-time-correct nodes. The chosen model keeps each node independently RFC-0004-correct and treats coordination as advisory.

### Alternative 3 — consensus / Byzantine-fault-tolerant state-machine replication

Full BFT replication is explicitly out of scope (see Non-goals): it is heavier than a timing-coordination substrate needs and would not, by itself, deliver the bounded *timing* properties SC0–SC3 target. SC4 detects Byzantine-*like* inconsistency; it does not promise BFT agreement.

### Alternative 4 — point-estimate PTP without uncertainty

A bare offset estimate cannot support a *bounded* claim. Carrying covariance (SC0) is what lets SC2/SC3 state windows rather than hopes.

## Prior art

PTP (IEEE 1588) for networked clock synchronisation; Kalman filtering (Welch & Bishop) for state estimation under noise; Cristian's probabilistic clock synchronisation and the broader probabilistic real-time literature for SC6's framing; and AxonOS RFC-0004, whose six-clause, falsification-per-clause structure this RFC mirrors at the mesh level.

## Unresolved questions

- SC5: the concrete fallback policy — exact peer-count and synchronisation thresholds, and the handshake by which `SwarmFaultDetector` health drives the `axonos-consent` layer — is not yet specified.
- SC0/SC3: the target-link noise model and the resulting numeric envelopes/windows need to be fixed against a real fixture before the L1 claims can be promoted to L2.
- SC6: a specific delivery-within-deadline probability and the measurement methodology to validate it are open.
- Fixed-point variants for FPU-less targets (currently floating-point) — required before some embedded targets can run the estimator deterministically.

## Future possibilities

- Promote SC0, SC2, SC3 from L1 to L2 with raw PTP-convergence and co-availability trace fixtures on a development harness.
- L3 GPIO/oscilloscope validation of epoch-release phase on a multi-node hardware fixture.
- Wire `SwarmFaultDetector` health into consent state transitions (a degraded/Byzantine peer as an input to consent withdrawal), connecting this RFC to the consent reason-code registry.
- Deterministic fixed-point `NeuralPtpKalman` for `thumbv7em`/`thumbv8m` targets without an FPU.

## Validation evidence level

Under RFC-0003, `axonos-swarm` and this contract are **L1/L2-oriented**: the clauses are analytical models with deterministic code-level checks (L1), with empirical fixture measurement (L2) identified as the immediate milestone for SC0/SC2/SC3 and independent instrumentation (L3) not claimed for any clause. SC6 is below L1 — a stated research target. No clause should be read as a clinical-certification or medical-device claim.

## References

1. AxonOS RFC-0004 — *Dual-Core Real-Time Contract*.
2. AxonOS RFC-0001 — *EDF Scheduler with Biological Deadlines*.
3. AxonOS RFC-0003 — *Validation Status Framework (L1/L2/L3)*.
4. IEEE 1588 — *Precision Time Protocol*.
5. Welch, G. & Bishop, G. — *An Introduction to the Kalman Filter*.
6. Cristian, F. (1989). *Probabilistic Clock Synchronization*. Distributed Computing.

---

*This RFC formalises the SC0–SC6 contract stated in the `axonos-swarm` README. It is normative for the swarm real-time contract once finalised; until then its status is `draft`. Claims are classified strictly under RFC-0003 and are falsifiable by construction.*
