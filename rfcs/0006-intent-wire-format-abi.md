---
rfc: 0006
title: Intent Wire Format ABI
status: draft
track: api
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-05-15
updated: 2026-08-11
implementation:
  - axonos-sdk 0.3.5 — IntentObservation (32-byte, repr(C, align(8))) — current release; this RFC is reconciled against it field by field
  - axonos-consent — current release
  - Promotion to active pending an instrumented evaluation-board fixture, which is not yet procured; no date is given because one would be invented L3 validation per RFC-0003
references:
  - RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels (IETF, 1997)
  - RFC 2104 — HMAC: Keyed-Hashing for Message Authentication (IETF, 1997)
  - RFC 5869 — HMAC-based Extract-and-Expand Key Derivation Function (HKDF) (IETF, 2010)
  - RFC 6234 — US Secure Hash Algorithms (IETF, 2011)
  - IEC 62304:2006/AMD1:2015 — Medical device software lifecycle
  - AxonOS Article #30 — The Developer Ecosystem
---

# RFC-0006: Intent Wire Format ABI

## Summary

This RFC specifies the binary wire format that defines the AxonOS kernel-to-application boundary: the layout of `IntentObservation` records, the capability bitfield, the truncated HMAC-SHA256 attestation tag, and the versioning rules under which the format may evolve while preserving compatibility for downstream implementations. The format is fixed at 32 bytes, 8-byte aligned, with three unused payload bytes as its only extension space. The RFC is published as **draft** and will be promoted to **active** once Phase 1 L3 oscilloscope validation (pending an instrumented evaluation-board fixture, which is not yet procured; no date is given because one would be invented per RFC-0003) confirms the wire format performs to specification on the reference hardware.

## Motivation

A real-time kernel that defines its application contract only through SDK source code creates soft vendor lock-in: alternative implementations cannot claim compatibility without copying that source. This is incompatible with the AxonOS engineering principle that no proprietary lock-in is established via the kernel — the application surface is intended to be a portable ABI.

Three concrete pressures force this RFC now:

1. **Independent replication of timing measurements** (RFC-0003) requires that researchers be able to construct conformant observers from a written specification, without access to the AxonOS source tree.
2. **Regulatory traceability** under IEC 62304 Class C requires that the architecture description identify the interfaces between software items in a versioned, revision-controlled form. An RFC satisfies this in a form auditable by certification bodies.
3. **Multi-implementation interoperability** is a precondition for the AxonOS Consent Protocol (specified in `axonos-consent`) to be implemented by parties other than the kernel author. A published wire format makes "AxonOS-compatible" a verifiable property of bytes on a wire rather than a claim about a code base.

## Guide-level explanation

An AxonOS application sees the world as a stream of typed events, each delivered as a fixed-size 32-byte record. The record contains:

- A monotonic timestamp scoped to the current session.
- A discriminant identifying the event kind (Navigation, WorkloadAdvisory, SessionQuality, ArtifactEvents).
- A payload byte interpreted according to the kind.
- A fixed-point confidence value in $[0, 1)$.
- An opaque session identifier, so an observation can be attributed to the
  session whose consent authorised it.
- An 8-byte truncated HMAC-SHA256 tag for integrity, with the full 32-byte tag available out-of-band for stronger verification.

The application receives observations through the SDK's `IntentStream`. The SDK validates the record against this RFC: any record with a reserved value, a reserved bit set, a timestamp exceeding the documented bound, or a failed HMAC truncation check is rejected before reaching application code. An application that wishes to implement its own decoder against the wire format can do so against this RFC alone.

Capabilities are encoded as a separate 32-bit bitfield delivered once at subscription handshake. The bit layout is fixed across versions; new capabilities consume the next unused bit and never reuse positions.

## Reference-level explanation

### Conventions

All multi-byte integer fields are little-endian. Field offsets are stated in bytes from the start of the record. The full record is aligned to an 8-byte boundary in memory and on the wire. The key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY are to be interpreted as in RFC 2119.

Bytes marked unused — `payload[1..=3]` in ABI version 1 — MUST be zero on emission and MUST NOT be interpreted on receipt. Future minor revisions of this RFC MAY assign meaning to them. There is no separate reserved field: every one of the 32 bytes is assigned.

### Record layout — IntentObservation

```text
Offset  Size  Type    Field
─────── ───── ──────  ─────────────────────────────
  0      8    u64     timestamp_us
  8      2    u16     kind_tag
 10      2    u16     quality_raw        (Q0.16 confidence)
 12      4    u8[4]   payload
 16      8    u64     session_id
 24      8    u8[8]   attestation        (truncated HMAC-SHA256)
─────── ───── ──────  ─────────────────────────────
Total  32 bytes, 8-byte aligned
```

This table is the layout `axonos-sdk` emits and asserts at compile time
(`size_of == 32`, `align_of == 8`). Field names are the ones the SDK uses, so a
reader can move between this RFC and `axonos_sdk::IntentObservation` without a
translation step.

### Field semantics

**`timestamp_us` (u64, offset 0).** Microseconds since session start, as defined by the `MonotonicTimestamp` type in `axonos-sdk`. Within a session, this value MUST NOT decrease. Values MUST NOT exceed $\mathrm{SESSION\_MAX\_REASONABLE\_US} = 2^{48}$ (defined in `axonos-sdk` — approximately 8.9 years at 1 µs resolution). Receivers MUST reject any record with a timestamp exceeding this bound. The bound prevents adversarial inputs from corrupting downstream arithmetic.

**`kind_tag` (u16, offset 8).** Discriminant for the observation kind:

| Value    | Kind       | `payload[0]` interpretation |
|:---------|:-----------|:----------------------------|
| `0x0001` | Direction  | `Direction` enum            |
| `0x0002` | Load       | `Load` enum                 |
| `0x0003` | Quality    | `Quality` enum              |
| others   | unassigned | —                           |

An unassigned `kind_tag` does **not** invalidate the record. The SDK decodes it
to `IntentKind::Unknown` and passes it through, so that a receiver built against
this revision keeps working when a later kernel emits a kind it has never heard
of. A receiver MUST NOT act on an observation it decodes as `Unknown`, and MUST
NOT infer a default kind from one.

**`payload` (u8[4], offset 12).** Kind-dependent. Byte `payload[0]` carries the
enum value selected by `kind_tag`; bytes `payload[1..=3]` are unused in ABI
version 1, MUST be zero on emission, and MUST NOT be interpreted on receipt.
They are the only extension space in the record.

For `Direction` (`0x0001`): `0` Up, `1` Right, `2` Down, `3` Left, `4` Neutral.
For `Load` (`0x0002`): `0` Low, `1` Moderate, `2` High.
For `Quality` (`0x0003`): `0` High, `1` Moderate, `2` Low, `3` NoSignal.

A value outside the range defined for its kind decodes to `IntentKind::Unknown`,
by the same rule and for the same reason as an unassigned `kind_tag`.

**`quality_raw` (u16, offset 10).** Classifier confidence as Q0.16 fixed-point.
The encoded value $v$ represents $v / 65535$ — the denominator is
`axonos_sdk::CONFIDENCE_DENOM`, which is `u16::MAX`, **not** $2^{16}$. The range
is therefore $[0,\, 1.0]$ inclusive, and $1.0$ **is** representable as
`0xFFFF`.

> **Corrected 2026-08-11.** Earlier revisions of this RFC specified a
> denominator of $65536$ and stated that $1.0$ was deliberately unrepresentable,
> "eliminating a class of overconfident-classifier defects". The shipped ABI
> does not do that. The claim is withdrawn rather than restated, because a
> safety property that the implementation does not hold is worse than no claim:
> a reader could have built a consumer that treats $1.0$ as impossible. Whether
> to forbid $1.0$ is a live design question, recorded under Open questions.

Comparisons and decision logic MUST use the raw `u16`. The SDK's `f32`
conversion is documented for display only and is not architecture-stable.

**`session_id` (u64, offset 16).** Opaque session identifier, assigned by the
kernel at session activation. A receiver MUST treat it as an opaque token: it
carries no structure this RFC defines, and nothing may be inferred from its
value, ordering or spacing. It exists so that an observation can be attributed
to the session whose consent authorised it, and so that records from two
sessions cannot be interleaved undetected.

**`attestation` (u8[8], offset 24).** The high 8 bytes of the HMAC-SHA256 tag
computed per § Attestation. Receivers MUST verify it against either (a) a
locally recomputed HMAC over the first 24 bytes of the record using the session
key, or (b) the full 32-byte tag delivered out-of-band via the consent channel.
The SDK performs (a) unless built with the `kernel-stub` feature, which disables
verification and is not a deployment configuration.

### Capability bitfield

The set of capabilities authorised for a subscription is encoded as a `u32` bitfield delivered at the subscription handshake. Per-event records do not carry the bitfield.

```text
Bit    Mask          Capability
─────  ────────────  ─────────────────────
  0    0x00000001    Navigation
  1    0x00000002    WorkloadAdvisory
  2    0x00000004    SessionQuality
  3    0x00000008    ArtifactEvents
 4-31  0xFFFFFFF0    reserved (MUST be zero)
```

Receivers MUST reject any handshake with reserved bits set. The bit layout is fixed across versions of this RFC: any new capability MUST consume the next unused bit, and existing bits MUST NOT be renumbered. The draft-to-active transition (per RFC process) MUST NOT renumber existing bits.

### Attestation

The attestation tag is HMAC-SHA256 (RFC 2104, RFC 6234) computed over the first
24 bytes of the `IntentObservation` record — that is, every field except
`attestation` itself. The 32-byte result is truncated to the high 8 bytes and
placed at offset 24. The remaining 24 bytes are available out-of-band via the
consent channel for stronger verification when the consumer has the performance
budget.

Because `attestation` occupies bytes 24..32, "the first 24 bytes" and "every
field except the tag" are the same span. Earlier revisions placed the tag at
offset 16 while still specifying the HMAC over the first 24 bytes, which put the
tag inside its own input; no implementation could satisfy both halves of that
sentence.

The truncation rationale: the wire record budget is fixed at 32 bytes (one cache line on Cortex-M4F, matching the SPSC slot size in RFC-0002). At 64-bit length, the truncated tag provides $2^{63}$ expected work for a single forgery, which is acceptable for event-level integrity on a 4 ms epoch period. Consumers requiring stronger per-event integrity SHOULD verify the full 256-bit tag periodically (recommended: at least once per `SessionQuality` event, i.e., at most 2 Hz).

The HMAC key is derived per-session via HKDF-SHA256 (RFC 5869) from a
device-unique secret stored in the ATECC608B protected slot 0, with a session
identifier generated by the kernel at session activation as the derivation
input.

The relationship between that derivation input and the 8-byte `session_id`
carried in the record is deliberately not asserted here: this RFC specifies the
wire record, and the kernel's key schedule is out of its scope. A reader MUST
NOT assume the record field is the derivation input. Pinning that relationship
is recorded under Open questions.

### Versioning rules

ABI versions follow Semantic Versioning. The ABI is currently in **draft** status. The version is announced at subscription handshake as three `u8` fields (major, minor, patch). Consumers MUST refuse to subscribe if the kernel's major version does not match the consumer's compiled-in expectation.

Permitted changes by version step:

| Change                              | Patch | Minor | Major |
|:------------------------------------|:-----:|:-----:|:-----:|
| Reorder existing fields             |   ✗   |   ✗   |   ✓   |
| Renumber capability bits            |   ✗   |   ✗   |   ✗   |
| Add new `kind_tag` value            |   ✗   |   ✓   |   ✓   |
| Add new `direction` value           |   ✗   |   ✓   |   ✓   |
| Assign reserved bit/byte            |   ✗   |   ✓   |   ✓   |
| Change `attestation` length             |   ✗   |   ✗   |   ✓   |
| Change HMAC algorithm               |   ✗   |   ✗   |   ✓   |
| Change endianness                   |   ✗   |   ✗   |   ✗   |
| Change record total size            |   ✗   |   ✗   |   ✗   |

The record total size and endianness are frozen even across major versions: a 32-byte, little-endian, 8-byte-aligned record is part of the AxonOS architectural identity per RFC-0001 and RFC-0002.

### Promotion to active

This RFC will be promoted from **draft** to **active** when, and only when, all of the following are true:

1. L3 oscilloscope-validated WCRT measurement is published per RFC-0003 (target pending an instrumented evaluation-board fixture, which is not yet procured; no date is given because one would be invented).
2. At least two independent implementations have demonstrated interoperability against the conformance test vectors maintained in this repository.
3. A six-month public review window has elapsed since publication of the draft.

Until promotion, the format is the working specification implemented by `axonos-sdk` and `axonos-consent`. Breaking changes during the draft period require a new minor or major version bump per § Versioning rules; the format does not regress silently.

## Drawbacks

**Fixed record size constrains payload growth.** The 32-byte cap reflects the SPSC slot size from RFC-0002 (one cache line on Cortex-M4F). Adding new capability payloads beyond a single `u8` discriminant requires either re-using reserved bytes or a major version bump. This is a deliberate constraint — payload growth beyond enum discriminants would force more sophisticated decoders into the safety-oriented path — but it is a constraint nonetheless.

**Truncated HMAC tolerates a 64-bit work factor.** Operators requiring full 256-bit per-event integrity must verify against the consent channel, adding bandwidth cost. The truncation is a documented trade-off, not a security defect, but it is a trade-off that this RFC commits the project to.

**RFC overhead.** Codifying the ABI as a normative document creates ongoing maintenance: every change of a `direction` enum value requires a minor version bump and a published revision. Smaller projects do this implicitly via dependency versions; the RFC framing makes the cost explicit.

## Rationale and alternatives

### Alternative 1 — protobuf or flatbuffers

Self-describing schemas would relax the fixed-size constraint. They were rejected for three reasons: the wire size for a single `Navigation` event would inflate from 32 bytes to ~100 bytes; the decoder would require an arena allocator on the hot path (incompatible with RFC-0002 zero-copy ring buffer); and the schema evolution rules are weaker than the explicit per-field rules above.

### Alternative 2 — CBOR with cap on size

CBOR (RFC 8949) is already used in `axonos-consent` for the consent state machine. Extending it to the per-event wire format was considered. Rejected because CBOR decoding adds a variable-time cost incompatible with the WCET budget on the M4F (see RFC-0002 § Reference-level explanation), and because CBOR's flexibility on the wire is precisely what the fixed format eliminates.

### Alternative 3 — JSON

Rejected as a serious consideration only by virtue of completeness. JSON encoding/decoding has unbounded cost, requires a heap allocator, and the smallest meaningful per-event payload exceeds 64 bytes. JSON remains appropriate for the consent channel where audit-readability matters; it is not appropriate for the per-event ABI.

### Alternative 4 — no published wire format

Leave the ABI implicit in the SDK source. Rejected because it precludes independent replication (RFC-0003), creates soft vendor lock-in, and provides no auditable artifact for regulatory traceability under IEC 62304 § 5.3.

### Alternative 5 — full 32-byte HMAC tag (no truncation)

Rejected because it requires either a larger record size (breaks cache-line discipline from RFC-0002) or removing other fields. The hybrid approach — 64-bit truncation on the wire with full tag available out-of-band — preserves the per-event size budget while admitting stronger verification where required.

## Prior art

This RFC draws on three traditions. From IETF: fixed-size headers with explicit extension space (RFC 791 IPv4, RFC 8200 IPv6), explicit byte-order conventions, and the version-discovery handshake. From CAN bus and automotive embedded networking: 8-byte payload discipline as a means of pinning the cost of message handling. From safety-critical real-time systems (ARINC 653, AUTOSAR): formal specification of inter-partition message formats as a precondition for verification.

The truncated HMAC approach is established in NIST SP 800-107 § 5.3.4 and in IETF practice (RFC 4868 § 2.6 for IPsec). The 64-bit truncation length is at the lower bound of NIST recommendations and is justified here by the supplementary full-tag verification path; in contexts where the full-tag path is unavailable, this RFC's truncation length would be insufficient.

The capability bitfield design with reserved bits and explicit non-renumbering is consistent with the approach used in Linux `prctl` and `seccomp` flag fields.

## Unresolved questions

1. **Conformance test vector format.** The vectors will live under `vectors/rfc-0006/` in this repository. The exact encoding (raw binary, hex with comments, or both) is under discussion; the working assumption is "both, in parallel directories." Decision before promotion to active.
2. **HKDF salt source.** The current `axonos-consent` implementation uses a fixed salt; whether to derive the salt from the session ID instead is open. Decision before promotion to active.
3. **Confidence range adjustment.** Whether to extend `quality_raw` to Q1.15
   signed in a future minor revision (allowing negative confidence as a
   signal-quality indicator) is deferred to RFC-0008 or later, not committed.
4. **Replay and drop detection — no mechanism is specified.** Earlier revisions
   of this RFC placed a `sequence` counter at offset 12 and required consumers
   to "detect dropped observations by tracking expected vs. received sequence
   numbers", treating a non-monotonic value as a security event. **The shipped
   record has no such field**, so that requirement could not be met by any
   implementation, and it has been withdrawn rather than left standing.

   What remains is the gap it was covering. `timestamp_us` is monotonic within a
   session and gives a *weak* ordering signal — a receiver can notice time going
   backwards, but cannot distinguish a dropped observation from an epoch in
   which the classifier emitted nothing. The attestation tag makes a forged
   record infeasible; it does not make a *replayed* one detectable, because a
   replay is a byte-identical record with a valid tag.

   Three options, none yet chosen: spend `payload[1..=3]` on a 24-bit counter,
   which fits the existing record but caps the wrap period; move the counter
   into a v2 record, which is a breaking change; or specify replay detection at
   the transport layer in RFC-0007 and state here that the wire record does not
   carry it. **Until this is decided, this RFC makes no replay-detection claim
   and a consumer MUST NOT assume one.** Decision before promotion to active.
5. **Whether `session_id` is the HKDF derivation input.** The key schedule
   derives the per-session HMAC key from a kernel-generated session identifier;
   the record carries an 8-byte `session_id`. Whether these are the same value,
   and whether that should be normative, is unsettled — binding them would let a
   receiver recompute the key schedule's input from the wire, which may be
   undesirable. Decision before promotion to active.

## Future possibilities

A second wire-format RFC could specify the consent channel's per-message format (currently CBOR-encoded under `axonos-consent`) with the same level of normative detail. This would put the AxonOS Consent Protocol on equally portable footing.

A capability-payload RFC could specify how a future capability with a richer payload (e.g., a 12-byte velocity vector for a hypothetical continuous-control capability) would consume the `reserved` field block, including the version bump rules and the consumer-side parsing requirements.

The same wire-format discipline could be applied to the `axonos-swarm` peer-to-peer protocol (RFC-0004 § Future possibilities), giving the swarm tier an ABI as portable as the local kernel-to-application interface.

## Validation evidence level

Per RFC-0003, the claims in this RFC stand at the following levels:

- **Wire format size and alignment** — L1. Compile-time assertions in `axonos-sdk` enforce `size_of::<IntentObservation>() == 32` and `align_of::<IntentObservation>() == 8`. The assertions are evaluated at build time on every CI run.
- **Decoder rejection of malformed records** — L2. Unit tests in `axonos-sdk` cover the malformed-vector cases; runtime measurement on the reference platform is in progress as part of Phase 1.
- **HMAC verification correctness** — L1. The HMAC-SHA256 construction is delegated to a verified primitive; the truncation and key derivation are specified by reference to RFC 2104 / RFC 5869 / RFC 6234 and verified by test vectors against those RFCs.
- **End-to-end WCRT under this wire format** — pending L3 oscilloscope validation per RFC-0003 (target pending an instrumented evaluation-board fixture, which is not yet procured; no date is given because one would be invented).

The RFC will not be promoted from draft to active until the pending L3 claim is resolved, regardless of the schedule for other items.

## References

1. RFC 2119 — *Key words for use in RFCs to Indicate Requirement Levels.* IETF, 1997.
2. RFC 2104 — *HMAC: Keyed-Hashing for Message Authentication.* IETF, 1997.
3. RFC 5869 — *HMAC-based Extract-and-Expand Key Derivation Function (HKDF).* IETF, 2010.
4. RFC 6234 — *US Secure Hash Algorithms (SHA, HMAC-SHA).* IETF, 2011.
5. RFC 8949 — *Concise Binary Object Representation (CBOR).* IETF, 2020.
6. NIST SP 800-107 Rev. 1 — *Recommendation for Applications Using Approved Hash Algorithms.* NIST, 2012.
7. IEC 62304:2006/AMD1:2015 — *Medical device software — Software life cycle processes.*
8. AxonOS Article #30 — *The Developer Ecosystem.* Medium, 2026.

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). Implementations referenced from this RFC ([axonos-sdk](https://github.com/AxonOS-org/axonos-sdk), [axonos-consent](https://github.com/AxonOS-org/axonos-consent)) are licensed under Apache-2.0 OR MIT.*
