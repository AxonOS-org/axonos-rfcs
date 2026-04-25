---
rfc: 0001
title: EDF Scheduler with Biological Deadlines
status: active
track: scheduling
authors:
  - Denis Yermakou <axonosorg@gmail.com>
created: 2026-04-25
updated: 2026-04-25
implementation:
  - axonos-kernel scheduler module (in development)
references:
  - Liu, C. L. & Layland, J. W. (1973). Scheduling algorithms for multiprogramming in a hard-real-time environment. JACM 20(1), 46–61.
  - Buttazzo, G. (2011). Hard Real-Time Computing Systems (3rd ed.), Springer.
  - AxonOS Article #4 — The Zero-Jitter Mind. https://medium.com/@AxonOS/title-the-zero-jitter-mind-why-axonos-requires-a-hard-real-time-microkernel-subtitle-when-a-fdf364e0e6f7
  - AxonOS Article #18 — The Attention Scheduler. https://medium.com/@AxonOS/title-axonos-article-18-the-attention-scheduler-managing-cognitive-concurrency-at-the-bare-f00e14b6d8b7
---

# RFC-0001: EDF Scheduler with Biological Deadlines

## Summary

AxonOS adopts an **earliest-deadline-first (EDF) preemptive scheduler** with task deadlines derived from the biological signal pipeline structure rather than from software convenience. Schedulability is verified at admission time using the Liu–Layland EDF utilization bound, with conservative budget inflation to absorb measured cache and bus contention. Rate-monotonic scheduling and round-robin alternatives are explicitly rejected.

## Motivation

A brain-computer interface kernel has a property that distinguishes it from general-purpose real-time systems: the deadlines are not software-derived, they are **biological**. The motor cortex produces beta-band desynchronisation events on a 4 ms epoch. The classifier must complete and the resulting intent must be delivered before the next epoch closes — otherwise the intent is dropped, the user perceives unreliability, and (in closed-loop stimulation contexts) the safety envelope is violated.

A scheduler that meets average-case latency targets but misses worst-case deadlines is unfit for this purpose. The cost of a missed deadline is not "slow response"; it is "the user lost agency for that epoch." Repeated over a session, this is the difference between a usable BCI and a research demonstration.

The scheduling problem is therefore: **how do we admit a task set such that no task ever misses its deadline, even under worst-case interrupt and bus contention?**

## Guide-level explanation

The kernel admits a task set only if it can prove, at admission time, that no task will miss its deadline under worst-case execution. The proof is the Liu–Layland EDF schedulability test. If the test fails, the task set is rejected and an error is returned to the caller. There is no "best effort" mode, no "soft real-time" fallback, no priority inheritance escape hatch.

When multiple admissible tasks are runnable, the scheduler always runs the task whose absolute deadline is earliest. This is the EDF discipline. Compared to the rate-monotonic alternative, EDF achieves higher utilization (up to 100% in the ideal case, vs ~69% for rate-monotonic) and has the property that any periodic task set with utilization ≤ 1 is schedulable.

Each task in AxonOS declares:

- A **period** `T` derived from the upstream signal pipeline (typically 4 ms for the primary classifier task)
- A **worst-case execution time** `C` measured on the reference hardware with a documented validation level (see RFC-0003)
- An **absolute deadline** equal to the next period boundary (the deadline = period assumption)

The scheduler validates `∑(Cᵢ / Tᵢ) ≤ 1 − headroom` and admits the set, or rejects it. In the current AxonOS configuration, `headroom = 0.75`, meaning the system runs at ≤ 25% utilization in steady state. The 75% reserve is for measured worst-case interrupt bursts, cache misses, bus contention, and future pipeline extensions.

## Reference-level explanation

### Task model

A task `τᵢ` is a tuple `(Cᵢ, Tᵢ, Dᵢ)` where:

- `Cᵢ` ∈ ℝ⁺ — worst-case execution time, in microseconds
- `Tᵢ` ∈ ℝ⁺ — period, in microseconds, with `Tᵢ ≥ Cᵢ`
- `Dᵢ` ∈ ℝ⁺ — relative deadline, with `Dᵢ ≤ Tᵢ`

AxonOS adopts the **deadline-equal-to-period** model: `Dᵢ = Tᵢ` for all tasks. This is the standard hard-real-time configuration.

### Schedulability test

For a periodic task set `Γ = {τ₁, ..., τₙ}` with `Dᵢ = Tᵢ`, EDF is schedulable on a single core if and only if:

```
U = ∑(Cᵢ / Tᵢ) ≤ 1
```

This is the [Liu–Layland EDF bound](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling#Schedulability_test).

AxonOS uses a more conservative admission test:

```
U ≤ U_max  where  U_max = 0.25
```

The reserve `1 − U_max = 0.75` absorbs:

- Cache miss penalty under contention (measured ≤ 200 µs worst case at the L1/L2 boundary on Cortex-M4F)
- Interrupt service time for the highest-priority interrupt, including Cortex-M context save (~12 cycles)
- Bus contention with the DMA controller during ADC reads (worst case ~40 cycles per word)
- Future pipeline stage additions without re-architecting

The 0.25 admission ceiling is a project-wide invariant. Loosening it requires a new RFC.

### Pipeline task set as currently admitted

The AxonOS classifier pipeline as admitted on the STM32F407 reference platform:

| Task | Cᵢ (µs) | Tᵢ (µs) | Cᵢ/Tᵢ |
|:---|---:|---:|---:|
| Signal pipeline (Kalman → FIR → Notch → Artifact → CSP → LDA) | 640.2 | 4000 | 0.160 |
| Consent service | 12.0 | 4000 | 0.003 |
| Attestation (HMAC-SHA256 over event) | 18.0 | 4000 | 0.005 |
| Network egress (BLE intent publish) | 24.0 | 4000 | 0.006 |
| Background diagnostics | 100.0 | 1000000 | 0.0001 |
| **Total utilization** | | | **0.174** |

Total utilization 0.174 — well below the `U_max = 0.25` ceiling, with 0.076 of headroom remaining.

### Algorithm — preemption

On each scheduler tick (or task release):

```
fn schedule() {
    let now = monotonic_clock();
    let runnable = ready_queue.iter()
        .filter(|t| t.is_ready(now))
        .collect();

    let next = runnable.iter()
        .min_by_key(|t| t.absolute_deadline)
        .copied();

    match (next, current) {
        (Some(n), Some(c)) if n.absolute_deadline < c.absolute_deadline => {
            preempt(c, n);
        }
        (Some(n), None) => dispatch(n),
        _ => {} // no change
    }
}
```

Preemption granularity on Cortex-M4F: 12 cycles for context save (PUSH {r4–r11, lr}, MSR PSP, MSR CONTROL), 12 cycles for restore. At 168 MHz this is 71.4 ns each direction — negligible relative to the 640 µs pipeline WCET.

### Why deadline = period

A more general task model permits `D < T`, allowing tasks that must complete well before the next period for jitter-suppression reasons. AxonOS deliberately does not use this. Instead:

- The signal pipeline task is given the full 4 ms epoch as its deadline.
- Jitter is controlled by the `U ≤ 0.25` admission ceiling and by the zero-copy ring buffer (RFC-0002), not by tightening individual task deadlines.
- This keeps the schedulability test simple and the analysis transparent for FDA Q-Sub documentation.

If a future task class requires `D < T`, that motivates a successor RFC, not an in-place modification of this one.

### Memory ordering for shared state

The scheduler's ready queue is accessed from the scheduler tick interrupt handler and from the systick-driven scheduling pass. All access to the ready queue uses `core::sync::atomic` with `Ordering::Acquire` on read and `Ordering::Release` on write. No mutex, no spinlock. The ready queue is sized at compile time.

### Failure modes

- **Schedulability test failure at admission** — caller receives `Error::NotSchedulable(utilization_after_admission)`. No partial admission.
- **Runtime deadline miss** — would indicate a flaw in WCET estimation. AxonOS treats this as a panic in the current implementation. Production deployments would log to a non-volatile fault buffer and enter a safe-idle state per RFC-0004.
- **Tick loss** — the systick interrupt is given the highest priority on the Cortex-M4F vector table, with the consent-withdrawal interrupt as the only higher-priority handler. Tick loss would indicate either a hardware fault or a bug in the interrupt-priority configuration.

## Drawbacks

EDF requires sorted insertion into the ready queue, which is O(log n) for n tasks rather than the O(1) of priority-bucket-based rate-monotonic. For the AxonOS task count (typically n ≤ 16), this is negligible — but it is not free.

EDF is harder to reason about under overload than rate-monotonic. If a task set ever exceeds U = 1 at runtime (impossible by admission control, but possible if the worst-case execution time was estimated incorrectly), EDF degrades catastrophically: every task starts missing its deadline. Rate-monotonic, by contrast, degrades gracefully — only the lowest-priority tasks miss. The defence against this in AxonOS is conservative WCET estimation and the 0.25 admission ceiling.

## Rationale and alternatives

### Alternative 1 — Rate-monotonic (RMS)

Higher-frequency tasks get higher priority; admission test is `U ≤ n(2^(1/n) − 1) ≈ 0.69`. Simpler analysis, graceful overload.

**Rejected because:** The `0.69` ceiling forces over-provisioning of CPU resources. AxonOS already commits to `U_max = 0.25` for safety; halving this further to fit RMS gives no benefit and constrains future pipeline expansion. EDF's higher utilization ceiling is a real engineering advantage when the admission test is conservative.

### Alternative 2 — Round-robin or fair-queueing

Generic best-effort scheduling. **Rejected immediately** — provides no deadline guarantees, no schedulability test, no admission control.

### Alternative 3 — Linux PREEMPT_RT

Tuned mainline Linux kernel with preemption enabled. Industry practice for non-safety-critical real-time.

**Rejected because:** Even with PREEMPT_RT, observed worst-case latencies on ARM are 83–160 µs (per the [ProteanOS analysis](https://proteanos.com/), 2026) compared to AxonOS's 8.2 µs. The kernel surface area also vastly exceeds what is reviewable for IEC 62304 Class B/C lifecycle documentation. Linux is the wrong starting point for a regulated medical-device kernel.

### Alternative 4 — FreeRTOS with priority preemption

Industry standard for embedded real-time. Bare metal, well-tested, small footprint.

**Rejected because:** Priority preemption rather than deadline preemption gives less precise control over the deadline-miss behaviour the project most cares about. The application-facing API and capability model AxonOS requires (RFC-0005) does not exist in FreeRTOS — building it on top duplicates substantial kernel state. AxonOS shares the bare-metal philosophy but not the scheduler.

### Alternative 5 — Time-Triggered Architecture (TTA)

Strict cyclic schedule, no runtime preemption. Highest determinism.

**Rejected because:** TTA forces all tasks into a single static schedule that cannot accommodate runtime application loading (capability-manifested user apps per RFC-0005). The flexibility cost is too high. EDF gives near-TTA determinism with runtime task admission.

## Prior art

- **Linux SCHED_DEADLINE** (since kernel 3.14, 2014) — Linux mainline implementation of EDF with Constant Bandwidth Server. Useful prior art for the runtime semantics, but the surrounding kernel is not appropriate for AxonOS's safety context.
- **VxWorks 653 / ARINC 653** — partition-based real-time used in avionics. The partition model influenced RFC-0004; the scheduler within each partition is RMS or fixed-cyclic, not EDF.
- **Liu & Layland (1973)** — the seminal paper establishing EDF and RMS schedulability bounds. Fifty years of subsequent literature has refined the conditions under which EDF is preferable.
- **Buttazzo (2011), *Hard Real-Time Computing Systems*** — the canonical textbook treatment of EDF for safety-critical applications.

## Unresolved questions

- **Sporadic task admission** — the current admission test handles strictly periodic tasks. Sporadic tasks (with a minimum inter-arrival time but variable arrival pattern) require a refinement of the schedulability test using sporadic-server theory. Out of scope for this RFC; a future RFC will address this if user-loaded applications require it.
- **Multiprocessor EDF** — this RFC covers single-core EDF only. The dual-core model is the subject of RFC-0004.
- **Energy-aware scheduling** — extending EDF with DVFS for power optimisation is interesting but premature. The current pipeline runs comfortably within the energy budget.

## Future possibilities

- Formal verification of the admission test using Kani or Prusti (the test is small enough to be feasible).
- Online WCET refinement — feeding measured execution time into a refinement of the admission test, allowing tasks that were initially rejected to be re-admitted as their actual WCET becomes known.

## Validation evidence level

- **Level 1 (instruction-count derived)** — The 640.2 µs pipeline WCET is instruction-count derived from `rustc 1.75 --opt-level=3` for `thumbv7em-none-eabihf`, summed against the [Cortex-M4F instruction timing reference](https://developer.arm.com/documentation/ddi0439/b/Programmers-Model/Instruction-set-summary).
- **Level 2 (runtime measured)** — The 972 µs end-to-end WCRT and the 2.1 µs σ jitter are measured on STM32F407 over a 12-hour continuous run with BCI Competition IV Dataset 2a replayed via the ADS1299 test-signal mux. Sample population: 10.8 M epochs. Zero deadline misses.
- **Level 3 (independent oscilloscope-validated)** — **Pending**, scheduled Phase 1 gate Q2 2026 on STM32H573 fixture.

## References

1. Liu, C. L. & Layland, J. W. (1973). Scheduling algorithms for multiprogramming in a hard-real-time environment. *Journal of the ACM*, 20(1), 46–61.
2. Buttazzo, G. (2011). *Hard Real-Time Computing Systems: Predictable Scheduling Algorithms and Applications* (3rd ed.). Springer.
3. AxonOS Article #4 — *The Zero-Jitter Mind*. Medium, 2026.
4. AxonOS Article #18 — *The Attention Scheduler*. Medium, 2026.
5. ARM. *Cortex-M4 Technical Reference Manual*, Revision r0p1.
6. Madden, M. M. (2020). *Challenges Using Linux as a Real-Time Operating System*. NASA TM-2020–220568.

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). The AxonOS scheduler implementation referenced from this RFC is licensed Apache-2.0 OR MIT in [axonos-kernel](https://github.com/AxonOS-org).*
