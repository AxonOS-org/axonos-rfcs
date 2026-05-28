# Security Policy — axonos-rfcs

This repository holds engineering specifications (RFCs), not executable
code. "Security" here means a flaw in a *specification* that would make
any conformant implementation unsafe.

## Scope

In scope: a specification-level defect in any RFC in this repository —
a timing contract that cannot actually be met, a capability or consent
model with a bypass, a wire-format ambiguity that a malicious peer could
exploit, or a validation rule that admits an unsafe claim.

Out of scope for this repository: defects in the *code* that implements
an RFC. Those belong to the implementing repository's security policy
(for example [`axonos-kernel`](https://github.com/AxonOS-org/axonos-kernel),
[`axonos-sdk`](https://github.com/AxonOS-org/axonos-sdk),
[`axonos-consent`](https://github.com/AxonOS-org/axonos-consent),
[`axonos-swarm`](https://github.com/AxonOS-org/axonos-swarm)).

## How to report

A specification-level security concern may be raised in two ways:

- Privately, by writing to **security@axonos.org**, if disclosing the
  flaw publicly before a fix would create risk.
- Publicly, as a [GitHub Discussion](../../discussions) or an issue, if
  the concern is a design question rather than an exploitable flaw — a
  public technical record is usually the point of an RFC process.

When in doubt, choose the private channel first.

## What to expect

The project acknowledges a security report within five business days.
A specification fix is handled like any other RFC change: a correcting
RFC or amendment is drafted, reviewed, and merged, and the reporter is
credited unless they ask to remain anonymous.

---

The AxonOS Project · https://axonos.org · security@axonos.org
