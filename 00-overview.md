# Policy-Enforcing Git Proxy — Draft Specification

**Status:** Draft.

A policy-enforcing Git proxy is a reverse proxy between a Git client and an
upstream Git server that can refuse, defer, or permit operations according
to policy. This specification admits multiple independent implementations,
including minimal ones that implement only the protocol-conformance class.

## Goals

**G1 — Portable compliance.** A compliance auditor can trust any
conforming implementation equally: the evidence a push lifecycle produces
— states, attestations, reasons, actor types, correlation — has the same
shape and semantics regardless of whose software produced it. Audit
tooling written against the spec works against every conforming
implementation.

**G2 — The intermediary domain.** The specification covers exactly the
behaviour that exists in front of an upstream: policy enforcement,
deferral, and audit that the upstream does not itself provide. Upstream
Git hosts are the environment this behaviour operates in, not subjects of
the specification (`01`, §6).

**G3 — Implementation independence.** Conformance is defined entirely by
observable behaviour, so an implementation of any architecture, language,
or vintage can reach it without structural change, and two conforming
implementations are interchangeable from the auditor's and Git client's
point of view.

**G4 — A shared vocabulary.** Deferral, attestation, deciding actor,
correlation identifier, review window: today every product invents its
own words for these. A common vocabulary lets security teams, vendors,
and regulators specify requirements against the concept rather than
against one product's feature names.

## Non-goals

- Not an architecture standard: nothing here mandates process topology,
  storage, libraries, language, or runtime.
- Not a wire-protocol change: where the Git protocol itself would need
  extending (e.g. a machine-readable deferral signal), the path is an
  upstream proposal to the Git project, not a clause here.
- Not a product roadmap.

## Document set

| Document | Contents |
| --- | --- |
| `01-scope-and-method.md` | Scope, conformance classes, protocol baseline, derivation method |
| `02-push-approval-state-machine.md` | Push lifecycle: canonical states, transitions, evidence (`LC-n`) |
| `03-deferral-interaction-models.md` | The pending window on the wire: held-connection and reject-and-retry models (`DF-n`) |
| `04-record-field-sets.md` | Push record, audit view, and fetch record field sets (`PR-n`, `AU-n`, `FR-n`) |
| `05-capability-positions.md` | Per-capability mediation positions (`CP-n`) |
| `06-policy-hook-contract.md` | Policy decision contract and the optional external decision protocol (`HK-n`) |
| `07-identity-resolution.md` | Pusher and commit-metadata identity; resolution requirements (`ID-n`) |

## Conformance language

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when,
they appear in all capitals.

The governing discipline for every normative statement: **if a behaviour
cannot be observed and tested from outside the implementation — by a Git
client or by a reader of the audit interface — it is not eligible to be a
MUST.**

## Provenance

Requirements are derived from the observed behaviour of working
implementations, with the Git project's protocol documentation as the
baseline for everything Git already specifies (`01`, §9).

## Editing conventions

- Sections marked **(informative)** or **(non-normative)** never carry
  requirements.
- **Open issue:** markers flag questions the draft deliberately leaves
  unresolved rather than resolving by accident.
- Internal working notes that must not survive into any published draft are
  fenced and labelled as such.

## References

### Normative

- **BCP 14** — RFC 2119 and RFC 8174: requirement keywords.
- **Git protocol documentation** (canonical, maintained by the Git
  project, at `git-scm.com/docs/`):
  - `gitprotocol-http(5)` — Git over smart and dumb HTTP
  - `gitprotocol-v2(5)` — protocol version 2
  - `gitprotocol-pack(5)` — pack transfer, pkt-line framing, v0/v1
  - `gitprotocol-capabilities(5)` — v0/v1 capability semantics
  - `gitprotocol-common(5)` — shared wire-format definitions
- **RFC 9110** — HTTP Semantics: `Authorization`, status codes, caching.
- **RFC 4251–4254** — SSH protocol architecture, authentication,
  transport, and connection protocols.
- **SSH agent protocol** (`draft-miller-ssh-agent`) — agent forwarding.
- **OpenID Connect Core 1.0** and **OpenID Connect Discovery 1.0** —
  where OIDC authentication capability is claimed.

### Informative

- **OCI Distribution Specification** — comparison point for the scope
  decision on routes vs. protocol behaviour (`01`, §7).

