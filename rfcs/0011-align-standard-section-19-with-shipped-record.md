---
rfc: 0011
title: Align Standard Section 19 with the shipped intent-observation record
status: draft
track: api
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-08-26
updated: 2026-08-26
implementation:
  - https://github.com/AxonOS-org/axonos-sdk/blob/main/src/intent.rs
  - https://github.com/AxonOS-org/axonos-conformance
supersedes:
references:
  - RFC-0006, Intent Wire Format ABI
  - RFC-0002, Zero-Copy Ring Buffer
  - STANDARD.md Sections 10, 19, 26, 27, 28
  - GOVERNANCE.md Sections 3 through 5
---

# RFC-0011: Align Standard Section 19 with the shipped intent-observation record

## Summary

`STANDARD.md` Section 19 specifies a 28-byte intent-observation record. Every
implementation, every conformance vector and every one of the six independent
codecs implements a **32-byte** record with a different layout, specified in
RFC-0006. The two records share no field at any offset — not one agrees on offset, width or existence.

This RFC proposes that the Standard be corrected to describe the record that is
actually shipped, and records what that costs: two fields present in the
Standard and absent from the implementation are lost, and the ring-buffer slot
of Section 10.2 loses the field its consistency discipline is anchored to.

Under Section 26 this is a **breaking change** and carries a major-version
increment. Under Section 27 it requires the full amendment process including the
ninety-day notice. Under Section 28 it removes wire-format fields, which the
stability commitment forbids within a major-version line, which is precisely why
it cannot be done as a patch and is filed here instead.

## Motivation

### What diverges

| | Section 19 (v1.1.0) | RFC-0006 / `axonos-sdk` / conformance |
|:--|:--|:--|
| Record size | 28 bytes | **32 bytes**, asserted at compile time |
| offset 0 | `kind` (1) | `timestamp_us` (8) |
| offset 1 | `flags` (1) | — |
| offset 2 | `confidence` (2), **Q1.15** | — |
| offset 4 | `timestamp` (8) | — |
| offset 8 | — | `kind_tag` (2) |
| offset 10 | — | `quality_raw` (2), fraction of `u16::MAX` |
| offset 12 | `sequence` (8) | `payload` (4) |
| offset 16 | — | `session_id` (8) |
| offset 20 | `source_node` (4) | — |
| offset 24 | `reserved` (4) | `attestation` (8) |

### Why it matters more than a documentation defect

`axonos-conformance/CHALLENGE.md` invites an outside engineer to implement the
record from the published text and be graded byte-for-byte by CI. An engineer
who takes "the published text" to mean the document named *the Standard* writes
a 28-byte record, fails every vector, and the failure looks like theirs.

The project's central claim is that a stranger can implement from the
specification alone. That claim cannot hold while there are two specifications
and the more authoritative-sounding one is the one nobody implements.

There is also an inversion of status. RFC-0006 is `draft` and governs everything
built; the released Standard `v1.1.0` — the one an outsider would reach for —
says something else.

## Guide-level explanation

The proposal is to make Section 19 say what the kernel emits and the SDK
consumes, and to be explicit about the two fields that disappear in doing so.

**`sequence` (Section 19, offset 12) is removed.** It served two purposes. The transport
one — anchoring the torn-read check of Section 10.4 — is preserved, and moves
into the ring-buffer slot where it belongs. Its protocol purpose,
replay and drop detection, is **not** preserved and is not replaced here.
RFC-0006 already records that gap as an open question with three candidate
answers and none chosen. Removing the field from the Standard does not answer
it; it makes the gap visible in both documents instead of one.

**`source_node` (Section 19, offset 20) is removed.** Multi-node attribution has
no field in the shipped record. Until a revision restores it or RFC-0007 places
it in the mesh layer, an implementation cannot attribute an observation to an
acquisition node, and the Standard should say so rather than specify a field
nobody writes.

**`confidence` changes encoding.** Section 19 specifies Q1.15, a signed format
over [-1, 1). The value is a classifier posterior — a probability, with no negative values —
so the sign bit is both semantically invalid and a wasted bit of precision. The shipped encoding divides by `u16::MAX`, giving
[0, 1] with 1.0 exactly representable. Note that this is **not** textbook Q0.16,
which divides by 2^16 and cannot represent 1.0; the SDK's doc comment calls it
Q0.16 and should be corrected along with this RFC.

## Reference-level explanation

### Section 19, replacement text

The record **MUST** be exactly 32 bytes, little-endian, 8-byte aligned, with
fields at exactly these offsets:

| Offset | Size | Field | Notes |
|---:|---:|:--|:--|
| 0 | 8 | `timestamp_us` | monotonic clock, microseconds, at emission |
| 8 | 2 | `kind_tag` | capability discriminant; unassigned values decode to `Unknown` and **MUST NOT** be acted on |
| 10 | 2 | `quality_raw` | classifier posterior as `v / 65535`, range [0, 1] inclusive |
| 12 | 4 | `payload` | byte 0 carries the kind-dependent enum; bytes 1–3 **MUST** be zero and **MUST NOT** be interpreted |
| 16 | 8 | `session_id` | opaque session token; no structure is defined and none may be inferred |
| 24 | 8 | `attestation` | high 8 bytes of the HMAC-SHA256 over bytes 0–23 |

A software development kit receiving a record that is not exactly 32 bytes, or
in which `payload[1..=3]` is non-zero, or whose `kind_tag` corresponds to a
capability the application did not declare, **MUST** reject the record and
terminate the connection with the appropriate error of Section 20.

### Section 10.2, replacement text

The 32-byte record leaves no room in a 32-byte slot for the trailing sequence
copy, and carries no sequence field to copy. The slot therefore changes.

Each slot **MUST** be exactly **64 bytes**:

| Offset | Size | Content |
|---:|---:|:--|
| 0 | 4 | slot sequence, owned by the ring buffer |
| 4 | 4 | reserved, **MUST** be zero |
| 8 | 32 | the intent-observation record of Section 19 |
| 40 | 4 | trailing copy of the slot sequence |
| 44 | 20 | reserved, **MUST** be zero |

The slot sequence is **ring-buffer metadata, not a record field**. This is the
substantive design change and it is an improvement independent of the size
question: the wire record becomes transport-agnostic, and the consistency
discipline stops depending on a field whose meaning belongs to the protocol.

One slot then occupies one 64-byte line on the application core, which is the
side that has a data cache. The architecture is asymmetric and the argument has
to respect that: the signal-processing core is a Cortex-M4F with no data cache,
writing into shared static memory, so no cache reasoning applies to the producer
at all.

What the alignment buys is on the consumer. Where the shared region is cacheable
and the application core must invalidate before reading, a 64-byte slot makes
the invalidation granularity and the slot granularity the same. Under the
previous 32-byte layout, invalidating one line discarded two slots, one of which
the producer might have been mid-write on, producing a torn neighbour that the
consistency check catches at the cost of a retry.

This benefit is **analytical and unmeasured**, and it is conditional: if the
shared region is mapped non-cacheable on the application core, as such regions
often are on heterogeneous parts, it does not arise. The cost is not
conditional. Slot memory doubles, from 32N to 64N bytes for a ring of N slots.

### Section 10.4, replacement text

The discipline is unchanged in shape and re-anchored to the slot sequence. The
producer writes the slot sequence at offset 0, writes the record, issues a
memory barrier, then writes the trailing copy at offset 40. The consumer reads
the trailing copy, then the record, then the leading sequence; if the two
sequences disagree the consumer was racing the producer and **MUST** discard and
retry.

## Drawbacks

Slot memory doubles. On a part where the ring is sized in kilobytes this is a
real cost and should be measured rather than assumed negligible.

Removing `source_node` leaves multi-node deployments with no attribution field
in the record, at a point where `axonos-swarm` exists and may need one.

Removing `sequence` leaves replay detection unspecified in the Standard as well
as in RFC-0006. Two documents now record the same gap. That is honest and it is
still a gap.

## Rationale and alternatives

**Align the implementation to the Standard instead.** This would require
rewriting the SDK, RFC-0006, six conformance codecs and the frozen vectors, to
adopt a record with a signed probability field and no attestation. Rejected: the
Standard's record has no attestation at all, and removing cryptographic
attestation from the wire to satisfy a document is the wrong direction.

**Keep both and declare them different boundaries.** Rejected on the text:
Section 19 and RFC-0006 both describe the record crossing from the kernel to the
software development kit. They are the same object.

**Slot of 40 bytes** (4 + 32 + 4). Rejected: 40 does not divide 64, so slots
straddle cache lines, which is the property Section 10.2 exists to guarantee.

**Slot of 32 bytes with the sequence held in a parallel array.** Preserves slot
density at a cost of 4N additional bytes rather than 32N, which is materially
cheaper. The consumer touches two regions per read, but the consumer is the
application core and carries no real-time obligation, so that cost falls where
there is budget for it.

This alternative is **not rejected**. It is the cheaper option and the
comparison between the two turns on the same measurement as unresolved question
3. It is recorded here so that the choice is made on a number rather than on
whichever option was drafted first.

## Unresolved questions

1. **Replay detection.** Inherited from RFC-0006 question 4, unchanged and still
   unanswered: 24 bits in `payload[1..=3]`, a counter in a v2 record, or
   detection at the transport layer. Decision before this RFC is accepted.
2. **Multi-node attribution.** Whether `source_node` returns in a future record,
   moves to the mesh layer under RFC-0007, or is dropped with the consequence
   stated. Decision before acceptance.
3. **Which slot layout to adopt.** The 64-byte slot costs 32N additional bytes
   and matches invalidation granularity on the application core; the parallel
   array costs 4N and does not. Whether the shared region is even cacheable on
   the application core decides whether the first buys anything. A measurement,
   not an opinion, and it should be taken before acceptance.

## Validation evidence level

This RFC proposes normative text; it makes no quantitative claim of its own. The
divergence it describes is verifiable by inspection: `STANDARD.md` Section 19,
`axonos-sdk/src/intent.rs` compile-time assertions, and the vectors in
`axonos-conformance`. The false-sharing argument for 64-byte slots is
**analytical** and untested on hardware; it is stated as reasoning, not as a
measured improvement, and Unresolved question 3 is the measurement that would
settle it.

## Process note

Under `STANDARD.md` Section 26 this is a breaking change: it changes the wire
format. It therefore carries a **major-version increment to v2.0.0**, and under
Section 27 requires the fourteen-day review and the ninety-day breaking-change
notice. Under Section 28 it removes wire-format fields, which no amendment
within the v1 line may do.

The fastest route would have been to edit Section 19 directly. That route was
available and was not taken, because the Standard's governance is the property
that makes the Standard worth building against, and a maintainer who bypasses it
when it is inconvenient has demonstrated that it can be bypassed.

An erratum has been filed against Section 19 in the interim, warning implementers
that the section does not describe any shipped implementation and directing them
to RFC-0006 until this amendment concludes. An erratum warns; it does not change
a requirement, and it does not substitute for this process.
