---
rfc: 0012
title: Consent withdrawal reaches every disclosure channel
status: draft
track: security
authors:
  - Denis Yermakou <connect@axonos.org>
created: 2026-10-08
updated: 2026-10-08
implementation:
  - axonos-stack v0.4.0 — the reference BCI wires withdrawal to revocation and verifies it at the receiver (W1, W5 by variant); see "Conformance status"
  - axonos-kernel — consent service — not yet implemented
references:
  - RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels (IETF, 1997)
  - RFC-0003 — Validation Status Framework (AxonOS)
  - RFC-0005 — Capability-Based Application Manifest (AxonOS)
  - RFC-0009 — Bounded Disclosure of Sealed Neural Data (AxonOS)
  - STANDARD.md Sections 15.1, 15.4, 20
  - axonos-consent SPEC.md §9 — the publication gate
  - RFC 7009 — OAuth 2.0 Token Revocation (IETF, 2013)
  - D. D. Redell, "Naming and Protection in Extendable Operating Systems", PhD thesis, MIT, 1974 — revocable capabilities through an indirection
  - 'Zero After Withdrawal — the AxonOS Reference BCI, long read: https://gist.github.com/AxonOS-BCI/b5cf55b5ce6a901bbeb0a34faaa1fd8a'
---

# RFC-0012: Consent withdrawal reaches every disclosure channel

## Summary

AxonOS has two mechanisms that stop neural information from reaching an
application. The consent gate of `axonos-consent` stops **intent observations**
the instant a withdrawal is admitted. The release predicate of RFC-0009,
implemented by `axonos-vault`, stops **derived disclosures** once a grant is
withdrawn. Each is specified, implemented and tested. **Nothing connects them.**
A consent withdrawal does not withdraw a single vault grant, so an
implementation that wires both in the obvious way stops the intents and keeps
releasing derived data.

This RFC makes the connection normative. An admitted withdrawal MUST revoke
every grant issued for the manifest installation before any further release is
evaluated; a suspension MUST refuse releases without revoking; the ten-millisecond
bound of Standard §15.4 applies to disclosures as it does to observation
streams; and the consent errors reach the application under the codes of
Standard §20.

## Motivation

### What was found

`axonos-stack` v0.4.0 added a reference BCI: synthetic EEG carried through
`axonos-hal`, `axonos-signal-pipeline`, `axonos-supervisor`, `axonos-vault`,
`axonos-consent` and `axonos-sdk`, with a signed withdrawal at frame 9 000 and a
verifier that counts, at the application, everything delivered at or after that
frame.

Wired with each crate used exactly as its own documentation describes, and
nothing more, the run withdraws consent and the gate refuses every later intent,
as specified. The vault goes on releasing contact-quality readings under a live
grant: **twelve releases, 384 bits, after the withdrawal**. The reference BCI
keeps that wiring as a configuration of its tests, so the number is reproduced
rather than remembered. Neither crate was wrong. The consent crate
governs the gate; the vault governs its grants; no code in the organisation told
the second about the first.

The reference BCI revokes the grant in the same step as the admitted
withdrawal, and the unconnected wiring is a test that must fail. That is a
fix in a demonstration. It is the integrator's job, and an integrator can forget
it. This RFC moves the obligation to where it cannot be forgotten.

### Why the existing texts do not already cover it

Standard §15.1 says that in **Withdrawn** "all observation streams for the
manifest are terminated", and §15.4 bounds that at ten milliseconds. A vault
release is not an observation stream in the sense of §19: it is a reduction
returned by `Vault::release`, not a record published through the ring. A careful
reader of §15 can conclude, correctly by the letter, that derived disclosures are
out of scope.

RFC-0009 N7 requires that grant withdrawal be terminal and take precedence over
every other refusal reason, and says this "mirrors consent withdrawal exactly".
It mirrors the *semantics*. It does not require that consent withdrawal *cause*
grant withdrawal, and RFC-0009 never mentions the manifest installation a grant
belongs to.

Between the two documents, the property a person withdrawing consent would assume
— that nothing about them reaches the application any more — was nobody's
requirement.

### Who feels it

The person who withdraws consent, first. Then the application author, who
receives `ConsentWithdrawn` from the SDK and reasonably tears down a subscription
while a second channel keeps delivering. Then the regulator or auditor, for whom
"withdrawal is honoured" has to mean every channel, and who has no document to
point to that says so.

## Guide-level explanation

An application can learn about the person through two kinds of output:

- **Intents** — `IntentObservation` records, published one by one through the
  consent gate. The gate is a single atomic word that holds the consent state and
  the publication count, so a publication and a withdrawal are totally ordered.
- **Derived disclosures** — reductions of the sealed raw window (a contact-quality
  count, a channel mean), released under a grant that carries a purpose, an expiry
  and a budget in bits.

Raw samples never leave the sealed container (RFC-0009 N1), so they are not a
channel.

After this RFC, consent is the master switch for both kinds:

| Consent | Intents | Derived disclosures |
|:--|:--|:--|
| Granted | published | released under the grant's terms |
| Suspended | refused, consent-suspended | refused, consent-suspended; grant kept, budget not charged |
| Withdrawn | refused, consent-withdrawn, for ever | every grant for the installation revoked; refused, consent-withdrawn, for ever |

The rule an implementer has to remember is short: **a withdrawal is not complete
until the last channel is closed**, and the ten milliseconds of §15.4 are counted
to that point.

## Reference-level explanation

### 1. Definitions

- *m* — a manifest installation (Standard §15.1). Consent state is per *m*.
- *C(m)* — the consent machine for *m*; *G(m)* its publication gate.
- *Γ(m)* — the set of grants (RFC-0009 §1) whose disclosures reach the
  application installed as *m*. An implementation MUST be able to compute *Γ(m)*;
  a grant that cannot be attributed to an installation is a defect under W2.
- *Admission* of a frame — the return of `ConsentMachine::handle` with a new
  state (axonos-consent SPEC §7.6).
- A *disclosure channel* — any path by which information derived from the sealed
  data of *m*'s subject reaches *m*.

### 2. Normative requirements

**W1. Withdrawal revokes.** When *C(m)* admits a transition into **Withdrawn**,
every grant in *Γ(m)* MUST be withdrawn in the sense of RFC-0009 N7. No release
under any grant in *Γ(m)* may be evaluated between the admission and the
revocation. The revocation is part of the withdrawal, not a consequence observed
later.

**W2. No exempt channel.** Every disclosure channel to *m* MUST be either a
publication through *G(m)* or a release under a grant in *Γ(m)*. An
implementation MUST enumerate its disclosure channels in its documentation, as
RFC-0009 N8 requires for uncharged channels; a channel that is neither gated nor
granted is a defect regardless of its rate.

**W3. Suspension refuses without revoking.** While *C(m)* is **Suspended**,
every release under a grant in *Γ(m)* MUST be refused with the consent-suspended
error. The grants MUST NOT be revoked, and the refusal MUST NOT be charged against
any budget: it reveals only the consent state, which the application is told
anyway (RFC-0009 §7, uncharged state-only refusals). On resumption the grants
continue under their original terms.

**W4. The bound covers every channel.** The ten-millisecond bound of Standard
§15.4 MUST be measured from receipt of the trusted-path withdrawal event to the
moment *after which no channel of W2 can deliver*. An implementation that closes
the gate within the bound and revokes later does not meet §15.4.

**W5. One numbering.** A refusal caused by consent MUST reach the application
under the codes of Standard §20: `0x05` consent-suspended, `0x06`
consent-withdrawn, on both channels. `axonos-consent` already documents these
codes for `Suppressed::abi_code()`. `axonos-sdk` v0.3.5 numbers the same errors
`0x0301` and `0x0302` in `ErrorCode`, which does not conform to §20; until that
is corrected, an implementation MUST map the refusal by variant, not by code.

**W6. Linearisation under concurrency.** Where releases and consent transitions
can run concurrently, every release under a grant in *Γ(m)* MUST be linearisable
with respect to the transitions of *C(m)*: there MUST be a single point after
which a withdrawal is in effect for both channels. Two conforming designs are
known, and §3 compares them; this RFC does not yet choose between them.

### 3. Two ways to meet W6

**(a) Revoke under the transition.** The kernel's consent service holds a single
lock (or runs on a single core) across `handle` and the revocation of *Γ(m)*, and
releases take the same lock. Simple, and enough for a single-core target, but it
puts the vault on the consent service's critical path.

**(b) Release through the gate's word.** A release reads *G(m)*'s state through
the same atomic word that intent publication uses, and commits only if that word
still reads `Granted` (a compare-and-swap on the state bits, without incrementing
the publication count, or a dedicated disclosure count in a second field of a
wider word). The linearisation point is then the one SPEC §9.2 already provides
and `loom` already checks. Revocation under W1 becomes bookkeeping for the audit
log rather than the mechanism that stops disclosure.

Option (b) reuses a proven mechanism. Option (a) reuses nothing and is easier to
read. The choice is the first unresolved question.

### 4. How conformance is checked

A conforming implementation MUST be able to run a session in which consent is
withdrawn part-way, and report, **counted at the receiver**:

1. the number of intents and derived disclosures delivered at or after the
   withdrawal — which MUST be zero;
2. the gate's publication count against the intents the application received,
   and the bits released against the bits the application received — which MUST
   be equal;
3. the result of the same session with the revocation of W1 deliberately
   omitted — which MUST show leakage. A check that cannot fail is not evidence.

The reference BCI in `axonos-stack` is this check for the host-side reference
wiring. Its default run is the conformance vector for W1:

```text
cargo run --locked --release --bin reference_bci
# … post-withdrawal leakage 0 · RESULT: VERIFIED
# trace SHA-256 cfaa12273ba1de9093cf9ea6a044d7fc9b47480cf0f893a69d5b623265a20629
```

## Drawbacks

- **Coupling.** The vault learns about manifest installations, and the consent
  service learns about the vault. RFC-0009 deliberately kept the vault free of
  consent semantics; this RFC gives up that separation in exchange for a
  property neither could provide alone.
- **Attribution.** *Γ(m)* has to be computable. A grant issued for a purpose that
  serves several installations — a shared calibration routine, for example — has
  no single *m*, and W2 makes it a defect until it is split or attributed.
- **Critical path.** Under §3 (a), revocation time counts against §15.4's ten
  milliseconds. Under (b), every release pays one atomic read-modify-write.

## Rationale and alternatives

**Leave it to the integrator.** This is the current state, and it is how the gap
was found: correct components, wired correctly by their own documentation, with a
leak between them. Rejected.

**Make the vault consult the consent state on each release, and never revoke.**
This is §3 (b) without W1. It stops disclosure, but the audit log then shows a
grant still live after a withdrawal, and an auditor reading the log reads the
wrong story. Kept as half of option (b), not as the whole.

**Erase the sealed window on withdrawal.** A stronger property, and probably the
right one, but it is about retention rather than disclosure, and Standard §15
does not require it today. Deferred to a future RFC; see *Unresolved questions*.

**Treat derived disclosures as observation streams in §15 directly.** An
amendment to the Standard instead of an RFC. That will be the end state, but the
Standard changes through RFCs (Standard §27), and this is that RFC.

## Prior art

Token revocation in OAuth 2.0 (RFC 7009) revokes a refresh token and, at the
server's discretion, the access tokens issued from it. That discretion is exactly
what this RFC removes: here, revoking the source revokes everything derived from
it. Redell's revocable capabilities (1974) route every capability through an
indirection that the grantor can sever. Option (b) of §3 is that design, with the
gate's atomic word as the indirection. Data-protection law states the user's side
of the same property: withdrawal of consent is to be as easy as giving it, and
takes effect for the processing it covered.

## Unresolved questions

1. **§3 (a) or (b).** Option (b) is preferred on the evidence that its
   linearisation point is already model-checked. Deciding requires a prototype in
   the kernel's consent service and a measurement against the §15.4 bound.
2. **Retention.** Whether a withdrawal must also erase the sealed window and
   any reductions held for later release. That needs its own RFC.
3. **The numbering.** W5 states which codes conform. Whether `axonos-sdk` changes
   its `ErrorCode` values or adds a §20 mapping, and how the conformance vectors
   record it, is for the SDK's next minor release.
4. **Multi-subject grants.** How *Γ(m)* is computed for a grant serving more
   than one installation, if such grants are permitted at all.

## Future possibilities

- A conformance vector in `axonos-conformance` that encodes a withdrawal
  mid-session and the expected receiver-side counts, so that an independent
  implementation can be checked against the same numbers.
- An L3 measurement of W4 on the reference hardware: GPIO toggled at receipt of
  the trusted-path event, at admission, at revocation and at the last delivery on
  each channel, captured on an instrument with its own clock.
- Folding W1–W5 into Standard §15 once this RFC is final.

## Validation evidence level

Per RFC-0003:

- **Level 1 (formally verified)** — none for the property this RFC introduces.
  The mechanisms it builds on carry L1 evidence in `axonos-consent`: Kani
  harnesses for authentication, replay refusal and the absorbing withdrawal, and
  `loom` models for the gate's linearisation.
- **Level 2 (runtime measured)** — behaviour only. The reference BCI in
  `axonos-stack` v0.4.0 runs on a host from synthetic input and measures counts
  and SHA-256 digests: zero post-withdrawal leakage with W1 in place, twelve
  releases and 384 bits without it. **No timing is measured.** W4 has never been
  measured against the ten-millisecond bound.
- **Level 3 (independent instrument)** — none.

## Conformance status of the reference implementation

| Requirement | `axonos-stack` v0.4.0 (reference BCI) | `axonos-kernel` |
|:--|:--|:--|
| W1 withdrawal revokes | **conformant** for the single grant of the session, in the same step | not implemented |
| W2 no exempt channel | two channels, both enumerated; raw has no channel | not implemented |
| W3 suspension refuses without revoking | not exercised | not implemented |
| W4 bound covers every channel | not measured | not measured |
| W5 one numbering | mapped by variant; SDK codes nonconformant | — |
| W6 linearisation | single-threaded; satisfied trivially, not demonstrated | not implemented |

This RFC stays in draft until the kernel's consent service implements W1 and W3
and one of the designs of §3 has been chosen and measured.

## References

1. AxonOS Standard, Sections 15.1, 15.4 and 20 — https://github.com/AxonOS-org/axonos-standard
2. RFC-0009, Bounded Disclosure of Sealed Neural Data — `rfcs/0009-bounded-disclosure-sealed-neural-data.md`
3. `axonos-consent` SPEC §9, the publication gate — https://github.com/AxonOS-org/axonos-consent/blob/main/SPEC.md
4. `axonos-stack` v0.4.0, the reference BCI — https://github.com/AxonOS-org/axonos-stack/releases/tag/v0.4.0
5. Transcript of the conformance run — https://github.com/AxonOS-org/axonos-stack/blob/main/reference/reference-bci-7.txt
6. "Zero After Withdrawal — the AxonOS Reference BCI", long read — https://gist.github.com/AxonOS-BCI/b5cf55b5ce6a901bbeb0a34faaa1fd8a
7. T. Lodderstedt, S. Dronia, M. Scurtescu, RFC 7009, "OAuth 2.0 Token Revocation", IETF, 2013
8. D. D. Redell, "Naming and Protection in Extendable Operating Systems", PhD thesis, MIT, 1974

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). Implementations referenced from this RFC may be licensed differently — see the implementation repository for terms.*
