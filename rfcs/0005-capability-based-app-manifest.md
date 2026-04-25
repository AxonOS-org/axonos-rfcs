---
rfc: 0005
title: Capability-Based Application Manifest
status: active
track: security
authors:
  - Denis Yermakou <axonosorg@gmail.com>
created: 2026-04-25
updated: 2026-04-25
implementation:
  - axonos-sdk capability-manifest module (in development)
references:
  - Saltzer, J. H. & Schroeder, M. D. (1975). The Protection of Information in Computer Systems. Proc. IEEE 63(9).
  - UNESCO (2025). Recommendation on the Ethics of Neurotechnology.
  - WebAssembly Component Model. https://github.com/WebAssembly/component-model
  - AxonOS Article #17 — Anatomy of an AxonOS App.
  - AxonOS Article #30 — The Developer Ecosystem.
---

# RFC-0005: Capability-Based Application Manifest

## Summary

AxonOS applications declare their permissions in a signed **capability manifest** at install time. The kernel rejects manifests that request prohibited capabilities (raw EEG, continuous emotion inference, cognitive profile reads, re-identification). Permitted capabilities are typed and rate-limited at the SDK boundary. There is no kernel code path that exposes raw neural signal to an application — not because of a runtime permission check, but because the data does not cross the partition boundary in raw form. Applications receive typed, attested intent observations.

## Motivation

The default position of operating systems is to grant applications broad access to underlying data and let users restrict it via permission prompts. This works adequately for filesystem and camera access, where the user has intuition about what the data is. It does not work for neural data — users have no intuition about what 250 SPS × 8 channels of EEG can be inferred to, what re-identification it permits, or what side-channels emerge.

The capability model in AxonOS inverts the default: applications by structure cannot access raw neural data. The set of capabilities is a small, deliberately curated list of typed events. Adding a new capability requires an RFC and a kernel update — not a developer's choice.

This is the design that the UNESCO 2025 Recommendation on the Ethics of Neurotechnology (§III.4) anticipates when it calls for "structural data minimisation as a primary protection rather than user-mediated permission management."

## Guide-level explanation

Every AxonOS application ships with a manifest file in its package. The manifest declares which capabilities the application uses. The manifest is cryptographically signed by the developer, countersigned by the AxonOS application registry on submission, and verified by the kernel at install time.

The capability set is small and curated:

- `Navigation` — typed direction events (e.g., motor-imagery left/right/up/down) at ≤ 50 Hz
- `WorkloadAdvisory` — cognitive-load advisories (low/medium/high band) at ≤ 1 Hz
- `SessionQuality` — signal-quality indicators (good/degraded/lost) at ≤ 2 Hz
- `ArtifactEvents` — electrode/artifact notifications at ≤ 10 Hz

Capabilities not on the list are **structurally inaccessible** — they have no SDK surface, no kernel implementation path, no event class. An application "wanting" raw EEG cannot get raw EEG; the data simply does not flow to user-space.

Each event the application receives carries an attestation tag (HMAC-SHA256 over the event payload, computed in hardware via ATECC608B per RFC-0004 § DC6). The SDK verifies the tag before delivery. Unattested events are dropped before user-space ever sees them.

Applications run inside a WebAssembly sandbox on the A53 application core. The sandbox's host imports are exactly the capability set — there are no escape hatches.

## Reference-level explanation

### Manifest format

Manifests are CBOR-encoded with a deterministic field ordering. The CBOR choice (over JSON) is for the same reason as RFC-0002's bounds: deterministic encoding, bounded decoder (compatible with `MAX_MAP_FIELDS = 8`, `MAX_STRING_LEN = 128`, `MAX_NESTING_DEPTH = 4`).

```cbor
{
  "manifest_version":  1,                  // u32
  "app_id":            "com.example.bci.write",   // string ≤ 128
  "app_version":       "1.2.3",            // string ≤ 32
  "publisher_id":      "did:key:z6Mk...",  // string ≤ 128
  "capabilities":      [                   // array ≤ 8 entries
    {
      "name":      "Navigation",            // enum {one of permitted}
      "max_rate_hz": 50                    // u16, ≤ capability ceiling
    },
    {
      "name":      "SessionQuality",
      "max_rate_hz": 2
    }
  ],
  "wasm_sha256":       <h(32)>,            // 32-byte SHA-256 of the WASM blob
  "publisher_sig":     <h(64)>,            // Ed25519 over canonical CBOR of all preceding fields
  "registry_sig":      <h(64)>             // Ed25519 by AxonOS registry over all preceding fields
}
```

The manifest is bounded in size (≤ 1024 bytes). The kernel's manifest decoder is the same hardened CBOR decoder used for consent frames (per `axonos-consent`).

### Capability catalogue (v1)

| Name | Description | Max rate | Event payload |
|:---|:---|---:|:---|
| `Navigation` | Motor-imagery direction events | 50 Hz | enum {Left, Right, Up, Down, Idle} |
| `WorkloadAdvisory` | Cognitive load band | 1 Hz | enum {Low, Medium, High} |
| `SessionQuality` | Signal quality indicator | 2 Hz | enum {Good, Degraded, Lost} |
| `ArtifactEvents` | Artifact notifications | 10 Hz | enum {Eye, Muscle, Movement, Electrode} |

Each event additionally carries: a 64-bit monotonic timestamp, a 16-byte HMAC tag, an 8-byte session ID.

### Prohibited capabilities

The following capabilities are explicitly **not in the catalogue and cannot be added without an RFC**:

- **`RawEEG`** — direct access to the ADC samples. Prohibited because the entire architectural premise of AxonOS rests on raw neural data not crossing the partition boundary.
- **`ContinuousEmotionInference`** — inferred emotional state. Prohibited as a privacy hazard — emotion inference from EEG is technically possible but its long-term collection enables re-identification and behavioural profiling at a level for which no consent regime is yet adequate.
- **`CognitiveProfile`** — long-term aggregated cognitive characteristics. Prohibited for the same reason.
- **`Reidentification`** — derived features that distinguish individuals. Prohibited.

A future RFC may, with explicit governance review, introduce an opt-in capability for one of these classes under tightly scoped clinical research conditions. Such an RFC would need to address: data-retention policy, consent revocation pathway, federated-only processing constraint, and external review.

### Manifest verification path

Manifest verification at install time:

1. Decode the CBOR with the bounded decoder. Reject if any bound is violated.
2. Verify `manifest_version == 1`.
3. Verify each capability name is in the catalogue.
4. Verify each capability's `max_rate_hz` does not exceed the catalogue ceiling.
5. Verify the WASM blob's SHA-256 matches `wasm_sha256`.
6. Verify `publisher_sig` over the canonical CBOR of preceding fields with the Ed25519 public key bound to `publisher_id`.
7. Verify `registry_sig` over the canonical CBOR of preceding fields (including `publisher_sig`) with the registry's published Ed25519 public key.

Any verification failure rejects the manifest and the application is not installed.

### Runtime enforcement

The application runs inside a WASM sandbox on the A53. The sandbox is configured at startup with **exactly** the import set declared in its capabilities. Imports for capabilities not declared are not bound — calls to undeclared capabilities trap as "unknown import."

Per-capability rate limits are enforced by the SDK. If an application's effective subscription rate to a capability exceeds the declared `max_rate_hz`, the SDK drops events (with a `RateLimitExceeded` diagnostic) until the rate falls back into bound. The kernel is not involved in rate-limit enforcement — the rate-limit invariant is preserved by the SDK because the SDK is the only path through which events reach the WASM sandbox.

### Attestation tag verification

Every event the SDK delivers to the WASM sandbox has been attestation-verified by the SDK. The verification is:

```
expected_tag = HMAC_SHA256_truncate(per_session_key, canonical_event_bytes)[..16]
delivered_tag == expected_tag
```

The per-session key is rotated on every session boundary. Keys reside in the ATECC608B and never leave the secure element; the HMAC computation is performed on-chip per RFC-0004 § DC6.

If verification fails, the event is dropped. A `Error::AttestationFailed` diagnostic is emitted to the developer console (with rate-limiting on the diagnostic to prevent log floods). Repeated attestation failures escalate to a session reset.

### Application identity and Ed25519 keys

Application publishers identify themselves via Ed25519 keypairs registered at the AxonOS application registry. The registry maintains a mapping `publisher_id → public_key`. Publishers are responsible for safeguarding their private keys; key rotation is supported by submitting a new manifest signed by the new key, countersigned by the old key for one rotation cycle.

The registry's own signing key is documented in the AxonOS root-trust manifest and is rotated on a 24-month cadence with overlap.

### Capability evolution

Adding a capability to the catalogue requires:

1. A new RFC describing the capability, its event class, its max rate, its threat model.
2. Implementation in the kernel and SDK.
3. Update of the manifest format if necessary (manifest_version increment).

Removing a capability requires:

1. A new RFC describing the deprecation rationale.
2. A grace period during which existing manifests continue to function.
3. Eventual rejection of manifests claiming the deprecated capability.

## Drawbacks

The small capability catalogue limits what applications can do. An application that wants raw EEG to do a novel signal-processing algorithm cannot exist in this model. This is **deliberate** — but it is a real constraint, and the AxonOS application ecosystem will be smaller than a permissive ecosystem.

The CBOR + Ed25519 + countersignature scheme adds non-trivial install-time complexity. The cost is paid once at install; runtime cost is zero (no per-event manifest check).

The list of prohibited capabilities is opinionated. Other reasonable people could draw the line elsewhere. The defence of the line is in the threat-model analysis (RFC-0005 prohibited section above).

## Rationale and alternatives

### Alternative 1 — Permission prompts (Android/iOS style)

Applications request permissions at runtime; user grants or denies.

**Rejected because** users have no informed basis for granting "raw EEG" permission. The structural data minimisation of capability-only model is a stronger protection.

### Alternative 2 — Capabilities + opt-in raw access

Same as proposed, but with an "advanced user" toggle that grants raw EEG access.

**Rejected because** it creates a class of users with structurally weaker protection. The opt-in becomes a target for social-engineering attacks ("for full functionality, please enable advanced mode"). The structural-prohibition design is robust against this.

### Alternative 3 — Linux-style namespaces / cgroups

Process isolation via kernel namespaces. Standard Linux container approach.

**Rejected because** Linux's namespaces are isolation mechanisms, not capability declarations. They do not address the question "what classes of data does this application operate on" — they only address "what files / network / processes can this application see." The semantic gap is too large.

### Alternative 4 — eBPF-style programmable filtering

Application supplies an eBPF program that filters its own data; kernel verifies and runs it.

**Rejected because** eBPF in this context would require the application to receive the data first (to filter it). The structural protection is preserved by the application never receiving raw data.

### Alternative 5 — WebAssembly Component Model with capabilities

Use the [WebAssembly Component Model](https://github.com/WebAssembly/component-model) directly.

**Partially adopted.** AxonOS uses the WASM sandbox runtime and the import-binding mechanism. The AxonOS capability set is more constrained than Component Model would allow by default (no arbitrary host imports), but the underlying mechanism is compatible. A future RFC may align the surface more closely with Component Model.

## Prior art

- **Saltzer & Schroeder (1975)** — *The Protection of Information in Computer Systems*. The seminal paper articulating the principle of least privilege — what AxonOS's capability model implements at the application boundary.
- **Capability-based operating systems** — KeyKOS, EROS, CapROS, Genode. AxonOS's approach is in this lineage but at the application layer, not the kernel-object layer.
- **iOS App Store entitlements** — apps declare capabilities (Camera, Microphone, HealthKit) at install. Closest commercial analogue, but with permission prompts at runtime.
- **WebAssembly Component Model** — typed import/export interfaces for WASM modules. Mechanism overlap.
- **UNESCO 2025 Recommendation on the Ethics of Neurotechnology** — provides the ethical framework that motivates structural data minimisation over user-mediated permission management.

## Unresolved questions

- **Should capabilities be revocable mid-session?** Currently the manifest is fixed at install. A "I changed my mind" revocation pathway requires kernel work and is deferred.
- **How do capabilities compose for multi-application sessions?** Two apps each holding `Navigation` may interact in ways that are not the union of their individual permissions. Requires a separate RFC.
- **What is the long-term governance for the capability catalogue?** Currently the catalogue is maintained by the AxonOS project. A multi-party governance for the catalogue (analogous to standards-body custody) is desirable but not yet planned.

## Future possibilities

- WebAssembly Component Model alignment for the SDK surface.
- A federated capability extension model where research institutions can negotiate scoped extensions for clinical studies.
- A user-facing dashboard showing what capabilities each installed application holds and how often each event class fires.

## Validation evidence level

This RFC is largely a security and isolation specification rather than a performance specification. The performance claims it touches:

- **Manifest verification time** — bounded; expected ≤ 1 ms on the A53. **Pending** measurement (Level 2).
- **HMAC verification per event** — ≈ 18 µs per event on M4F via ATECC608B. **Level 1** instruction-count derived; Level 2 measurement pending.

The security claims (capability isolation, attestation correctness) require Level 3 evidence acquired via penetration testing, which is part of the Phase 2 validation programme.

## References

1. Saltzer, J. H. & Schroeder, M. D. (1975). The Protection of Information in Computer Systems. *Proceedings of the IEEE*, 63(9), 1278–1308.
2. UNESCO (2025). *Recommendation on the Ethics of Neurotechnology*. UNESCO General Conference, 43rd session.
3. WebAssembly Community Group. *Component Model design*. https://github.com/WebAssembly/component-model
4. RFC 8949 — *Concise Binary Object Representation (CBOR)*. IETF, 2020.
5. RFC 8032 — *Edwards-Curve Digital Signature Algorithm (EdDSA)*. IETF, 2017.
6. AxonOS Article #17 — *Anatomy of an AxonOS App*. Medium, 2026.
7. AxonOS Article #30 — *The Developer Ecosystem*. Medium, 2026.

---

*This RFC is licensed under [CC-BY-SA-4.0](../LICENSE). Implementation in [axonos-sdk](https://github.com/AxonOS-org/axonos-sdk) under Apache-2.0 OR MIT.*
