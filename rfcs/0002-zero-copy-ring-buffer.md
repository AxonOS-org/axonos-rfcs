---
rfc: 0002
title: Zero-Copy Ring Buffer for Signal Path
status: active
track: memory
authors:
  - Denis Yermakou <axonosorg@gmail.com>
created: 2026-04-25
updated: 2026-04-25
implementation:
  - axonos-kernel signal path module (in development)
references:
  - Lamport, L. (1977). Concurrent reading and writing. CACM 20(11).
  - Vyukov, D. (2010). Single-Producer/Single-Consumer Queue. 1024cores.
  - AxonOS Article #16 — Zero-Copy Cognition.
  - AxonOS Article #19 — Zero-Latency Cognition (Lock-free IPC).
  - The Rustonomicon, Chapter 8 — Atomics. https://doc.rust-lang.org/nomicon/atomics.html
---

# RFC-0002: Zero-Copy Ring Buffer for Signal Path

## Summary

The AxonOS signal path is a **single-producer single-consumer (SPSC) zero-copy ring buffer**, statically allocated at link time, sized as a power of two for bitmask indexing, with double-buffered DMA writes from the ADC and slot borrowing by the classifier pipeline. All cross-thread synchronisation uses release-acquire memory ordering on a per-slot sequence number. No allocator on the hot path. No copy on the hot path.

This RFC documents the internal kernel implementation. It is **not** a standalone IPC protocol; it is the description of how AxonOS moves samples internally between the ADC interrupt and the pipeline task. External applications never see this surface — they receive typed `IntentObservation` events through the SDK boundary specified in RFC-0005.

## Motivation

Every memory copy in the signal path is a structural cost that cannot be tuned away. On Cortex-M4F at 168 MHz, a 32-byte memcpy costs ≈ 16 cycles plus cache effects. The signal pipeline runs at 250 SPS × 8 channels = 2000 samples/sec; even one copy per sample-batch costs measurable percentage of the available cycles. More importantly, copies introduce timing variance — cache state at the moment of copy is the dominant variable.

The pipeline must complete within the 4 ms epoch with zero deadline misses (RFC-0001). The way to achieve this is not to optimise copies but to eliminate them.

A second motivation is auditability. A signal path that allocates dynamically requires reasoning about heap fragmentation, allocator latency under worst case, and out-of-memory behaviour. A signal path that does not allocate at all has none of these concerns. For IEC 62304 Class B/C lifecycle documentation, "no allocator on the hot path" is a far simpler safety argument than "allocator is bounded under these conditions."

## Guide-level explanation

Samples arrive from the ADS1299 ADC at 250 SPS via SPI DMA. The DMA controller writes directly into a fixed memory region in SRAM that is statically allocated and never moved or freed. The classifier pipeline reads from the same memory region without any intermediate copy.

The memory region is structured as a ring of `N` slots. The DMA fills slot `head`; the classifier consumes slot `tail`. Both indices are 64-bit monotonically increasing counters. The mapping from index to actual slot location is `index & (N - 1)` — a bitmask, requiring `N` to be a power of two.

Synchronisation is achieved via a per-slot sequence number. The producer (DMA completion handler) writes a slot's sequence number with `Release` ordering to publish that slot. The consumer (classifier task) reads the sequence number with `Acquire` ordering before reading the slot's payload. This pattern guarantees that any payload write that happened-before the sequence-number publish is visible to any thread that has observed the published sequence number — across cores, across cache hierarchies, on weak memory models.

There is no lock. There is no kernel mediation. The hot path is: DMA completion interrupt → sequence number store → classifier task wake → sequence number load → slot read. End to end, this measures sub-0.2 µs cross-core IPC delivery.

If the buffer is full when the producer wants to write, the producer signals overrun. AxonOS rejects buffer overrun as a fatal pipeline error rather than dropping samples — see "Backpressure" in the reference-level explanation below.

## Reference-level explanation

### Capacity and indexing

```rust
// Compile-time power-of-two enforcement
const fn assert_power_of_two<const N: usize>() {
    assert!(N.is_power_of_two(), "ring capacity must be power of two");
    assert!(N >= 16, "ring capacity must be ≥ 16 slots");
}

const RING_CAPACITY: usize = 64;          // power of two, currently 64
const RING_MASK: usize = RING_CAPACITY - 1;
```

Capacity is fixed at compile time. The `RING_CAPACITY` chosen for the AxonOS reference platform is 64 slots — this gives 256 ms of buffering at 250 SPS, well in excess of the 4 ms epoch but small enough to fit comfortably in the L1 data cache on Cortex-M4F.

Slot index mapping is the bitmask:

```rust
fn slot_for(index: u64) -> usize {
    (index as usize) & RING_MASK
}
```

64-bit indices guarantee that no index wraparound is observable in the lifetime of the device — at 250 SPS, the index would take 2.3 × 10⁹ years to wrap.

### Slot layout and alignment

Each slot contains:

- A sequence number (`AtomicU64`)
- The sample payload (8 channels × 4 bytes per channel = 32 bytes)
- Padding to align the slot to a cache line

```rust
#[repr(C, align(64))]
struct Slot {
    seq: AtomicU64,           // 8 bytes
    payload: [u32; 8],        // 32 bytes — 8 channels of 24-bit ADC sign-extended to u32
    _padding: [u8; 24],       // align to 64-byte cache line
}

const _: () = assert!(core::mem::size_of::<Slot>() == 64);
```

64-byte alignment is the standard cache line size on Cortex-A53. On Cortex-M4F, cache lines are 32 bytes — the Slot is then exactly two lines, which is also acceptable for false-sharing avoidance.

### Sequence number protocol

The sequence number on slot `i` carries the producer-side index that last published this slot, plus an offset to distinguish epochs. Specifically:

- **FREE** (slot available for producer): `seq == i` for the next epoch
- **PUBLISHED** (slot ready for consumer): `seq == i + 1`
- **CONSUMED** (slot released for next epoch): `seq == i + RING_CAPACITY`

This three-state encoding is the classical Vyukov SPSC pattern. A consumer reading `seq == i + 1` knows the slot is ready; after consumption, it writes `seq == i + RING_CAPACITY` so the producer can use it again in the next ring traversal.

### Producer (DMA completion handler)

```rust
fn produce(payload: [u32; 8]) -> Result<(), Overrun> {
    let index = HEAD.load(Ordering::Relaxed);
    let slot = &SLOTS[slot_for(index)];

    let expected_seq = index;
    let observed_seq = slot.seq.load(Ordering::Acquire);

    if observed_seq != expected_seq {
        // Slot still owned by consumer - buffer is full
        return Err(Overrun);
    }

    // SAFETY: we have unique access to this slot; consumer has not yet
    //         seen seq == index + 1 so cannot be reading payload.
    unsafe {
        core::ptr::write(slot.payload.as_ptr() as *mut [u32; 8], payload);
    }

    // Publish: any thread that observes seq == index + 1 will also
    //          observe the payload write above.
    slot.seq.store(index + 1, Ordering::Release);

    HEAD.store(index + 1, Ordering::Relaxed);
    Ok(())
}
```

### Consumer (classifier task)

```rust
fn consume() -> Option<[u32; 8]> {
    let index = TAIL.load(Ordering::Relaxed);
    let slot = &SLOTS[slot_for(index)];

    let observed_seq = slot.seq.load(Ordering::Acquire);

    if observed_seq != index + 1 {
        // No payload published yet
        return None;
    }

    // SAFETY: producer has published; we are sole reader
    let payload = unsafe { core::ptr::read(slot.payload.as_ptr()) };

    // Release slot for next epoch
    slot.seq.store(index + RING_CAPACITY as u64, Ordering::Release);

    TAIL.store(index + 1, Ordering::Relaxed);
    Some(payload)
}
```

### Memory ordering — formal argument

The release-acquire pair on `slot.seq` establishes a synchronises-with relationship: every memory operation that happens-before the producer's `store(index + 1, Release)` becomes visible to any thread that observes that value via `load(Acquire)`. This is the [C++11 / Rust release-acquire semantics](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering).

In particular, the unsafe payload write happens-before the `seq.store`, and the consumer's payload read happens-after the `seq.load`. Therefore the consumer is guaranteed to observe the producer's payload exactly as written.

This is the **only** synchronisation mechanism on the hot path. There is no lock, no mutex, no spinlock, no priority inheritance. The pattern is correct on:

- x86_64 (TSO — release-acquire is implicit but compiler ordering is still enforced)
- ARMv7-M and ARMv8-M (weak memory model — `dmb ish` instruction inserted by `Release` and `Acquire` semantics)
- ARMv8-A (weak memory model — `dmb ishst` and `dmb ishld` inserted)

### Backpressure policy — fail-fast, do not drop

When the producer observes the slot is still owned by the consumer (observed sequence ≠ expected), the producer **does not advance the tail** to drop the oldest sample. Instead, the producer returns an `Overrun` error which the DMA completion handler escalates to a pipeline fault.

This is a deliberate departure from generic SPSC queues that adopt drop-oldest semantics. In AxonOS the rationale is:

- Drop-oldest in an SPSC queue requires the producer to mutate state owned by the consumer (the tail). This breaks the SPSC invariant and reintroduces synchronisation overhead.
- A signal-path overrun in a real-time pipeline indicates a scheduling failure (the consumer task is not running fast enough). Silently dropping samples masks the failure. Fail-fast surfaces it.
- For closed-loop stimulation contexts, dropped samples are a safety hazard. The consent and stimulation paths require complete sample histories.

If a deployment context truly requires drop-oldest semantics, a separate ring buffer with multi-producer or sequenced-write semantics would be specified in a future RFC, not as a mode of this one.

### Static allocation

The ring is allocated as `static mut`:

```rust
static mut SLOTS: [Slot; RING_CAPACITY] = [Slot::FREE_INIT; RING_CAPACITY];
static HEAD: AtomicU64 = AtomicU64::new(0);
static TAIL: AtomicU64 = AtomicU64::new(0);
```

Initialisation runs at link time (BSS zero-fill is sufficient because the FREE state has `seq == 0` for all slots, which matches `index == 0` at startup). No runtime constructor.

The `static mut` is wrapped in a `&'static` accessor that performs the atomic operations described above. Direct mutable access to `SLOTS` is `unsafe` and used only inside the produce/consume functions documented in this RFC.

### What this is not

This RFC documents the internal signal path between the ADC interrupt handler and the pipeline task. It is **not**:

- A general-purpose IPC protocol for inter-application communication
- A wire format for any external protocol
- A standalone library intended for use outside AxonOS
- A replacement for or alternative to any third-party protocol AxonOS implements

External protocol surfaces are documented in the implementing crates (such as `axonos-consent`) and in their respective specifications, not in AxonOS RFCs.

## Drawbacks

The fail-fast policy means a slow consumer crashes the pipeline rather than degrading gracefully. This is intentional but has a cost: the bar for consumer execution time is hard. The 0.25 admission ceiling (RFC-0001) is the project-wide safeguard against this.

Static allocation at compile time means the buffer size cannot be reconfigured at runtime. Different deployment contexts (higher channel counts, different sampling rates) require recompilation.

Power-of-two capacity constraint means buffer sizes go in factors of 2 (32, 64, 128, ...). For most deployments 64 is correct; for unusual cases this is a constraint to be aware of.

## Rationale and alternatives

### Alternative 1 — kernel-mediated IPC (mailbox, message passing)

A kernel call boundary on every sample. Linux-style. Generic and well-understood.

**Rejected because** the kernel call boundary on Cortex-M is ~30–50 cycles each direction. At 2000 samples/second this is 4–7% of the cycle budget for a no-op kernel call. AxonOS recovers this by eliminating the boundary entirely.

### Alternative 2 — copying queue (bounded MPSC)

Producer copies into queue, consumer copies out. Standard concurrent-queue pattern.

**Rejected because** the copy is the structural cost we are trying to eliminate. Two copies per sample (in + out) instead of zero.

### Alternative 3 — Heaplesss / bbqueue / similar third-party crate

[bbqueue](https://github.com/jamesmunns/bbqueue) is a well-known no-std SPSC byte-buffer queue. It would solve the same problem.

**Rejected because** AxonOS samples are typed records, not byte buffers. Using bbqueue would require serialise/deserialise on every sample, which is the copy we are trying to eliminate. The structural advantage is keeping the sample as a typed record at a fixed memory address from DMA write to classifier read.

### Alternative 4 — multi-producer support

MPSC or MPMC ring with CAS-based reservation.

**Rejected because** the only producer is the DMA completion interrupt handler, which by hardware is single-instance. There is no second producer. Adding MPSC support adds CAS overhead with zero benefit.

## Prior art

- **Vyukov SPSC queue** ([1024cores.net](https://www.1024cores.net/home/lock-free-algorithms/queues/unbounded-spsc-queue), 2010) — the canonical lock-free SPSC pattern. The sequence-number publish-and-consume protocol used here is the Vyukov pattern, adapted from unbounded to bounded with the `+ RING_CAPACITY` consumed-state encoding.
- **LMAX Disruptor** (LMAX Exchange, 2011) — popularised the sequence-number-based ring approach for high-throughput trading systems. AxonOS does not adopt the multi-producer claim sequence pattern (we do not need it) but the single-producer/single-consumer subset is essentially Disruptor's hot path.
- **DPDK rte_ring** — kernel-bypass networking ring with similar semantics. Validates the approach at scale.
- **Lamport (1977)** — *Concurrent reading and writing*, CACM. The original formal treatment of the producer-consumer synchronisation problem.

## Unresolved questions

- **Multi-channel sample bursting** — when the ADC reports multiple samples in a single DMA burst (e.g., 4 samples × 8 channels), the producer publishes them as a contiguous range. The current design publishes one slot at a time. A future refinement could publish a range with a single sequence-number write to amortise the publish cost, at the cost of a more complex sequence-number protocol. Deferred.
- **Cache coherence on heterogeneous topology** — the ring sits in shared SRAM accessible from both M4F and A53. Cache coherence is hardware-managed on the reference platform, but the formal ordering argument for the heterogeneous case warrants a dedicated review. To be addressed in RFC-0004.

## Future possibilities

- Loom-based testing of the SPSC protocol. Loom can exhaustively check all interleavings under the C++11 memory model. Adding the ring buffer as a Loom-tested module would convert the informal correctness argument into a checked one.
- Formal verification using Kani or Prusti.
- Extending to a dual-buffered classifier-output ring for the consent path.

## Validation evidence level

- **Level 1 (instruction-count derived)** — The IPC delivery path is instruction-count derived at ≤ 0.2 µs on Cortex-M4F at 168 MHz, comprising one acquire load, one release store, and the slot read.
- **Level 2 (runtime measured)** — The SPSC protocol is measured at sub-0.2 µs cross-task delivery as part of the 12-hour pipeline runtime measurement (RFC-0001 § Validation evidence level).
- **Level 3 (independent oscilloscope-validated)** — **Pending**. GPIO-instrumented IPC delivery measurement on STM32H573 fixture is part of the Phase 1 gate Q2 2026.

## References

1. Vyukov, D. *Single-Producer/Single-Consumer Queue.* 1024cores, 2010.
2. Lamport, L. (1977). Concurrent reading and writing. *Communications of the ACM*, 20(11), 806–811.
3. *The Rustonomicon* — Chapter 8: Atomics. https://doc.rust-lang.org/nomicon/atomics.html
4. ARM. *Application Note 321: ARM Cortex-M Programming Guide to Memory Barrier Instructions*.
5. AxonOS Article #16 — *Zero-Copy Cognition*. Medium, 2026.
6. AxonOS Article #19 — *Zero-Latency Cognition* (Lock-free IPC). Medium, 2026.

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). The AxonOS signal-path implementation referenced from this RFC is licensed Apache-2.0 OR MIT in [axonos-kernel](https://github.com/AxonOS-org).*
