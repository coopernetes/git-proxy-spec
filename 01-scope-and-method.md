# Scope, Conformance, and Method

## 1. Purpose and framing

The specification describes the **observable behaviour** of a Git reverse
proxy that performs policy enforcement, plus the **policy management layer**
beneath it (approvals, audit records, identity). It is written so that
multiple independent implementations — including a minimal wireline-only
proxy with no dashboard and no approval workflow — can conform, each to the
conformance class matching its ambitions.

**Derivation.** The draft is derived from the observed behaviour of a
working reference implementation, constrained to what is externally
observable (§2). Behaviour found in other implementations is treated as
prior art and evaluated for inclusion by the same criterion (§9).

**The specification's own scope boundary:**

> This specification does **not** specify the Git wire protocol. Git
> specifies what a compliant *server* does. This document specifies what a
> policy-enforcing *intermediary* does — behaviour that arises only because
> something sits in the middle and can say no. Where the upstream Git
> documentation is ambiguous or incomplete, we file upstream rather than
> specify around it.

---

## 2. The core discipline

Specify the capability that must exist and how it is observed from outside;
leave the mechanism to the implementation. A requirement qualifies as a
MUST only when a conformance test can observe it from outside — through a
Git client or through the audit trail.

Policy rejection is an example. A push that violates a rule is refused, and
the Git client receives a readable reason for the refusal. One
implementation produces this by terminating the protocol in a local
receive-pack; another relays it through a git subprocess. The
specification constrains only the observable result: the client learns
that the push was refused, and why.

---

## 3. Conformance classes

### Class P — Protocol conformance (mandatory for every implementation)

Correct Git smart-protocol behaviour through an intermediary: the
normative reference (§4) plus the proxy delta (§5). Every conforming
implementation, however minimal, claims Class P; a fetch-only deployment
(e.g. a read-only mirror for due-diligence access) conforms at Class P
alone.

### Class A — Policy and audit (optional profile)

The contract that makes audit portable across implementations and lets a
compliance auditor trust any conforming implementation equally:

- The interface a policy plugin/hook is presented with
- Minimum field set of a push record
- Minimum field set of an audit record
- Approval state machine: legal states and transitions (`02`)
- Deferral interaction models (`03`)
- Identity/authorization concepts referenced by the above

Storage engine, schema DDL, table layout, and dashboard design are **out of
scope**. An implementation claiming Class A also claims Class P.

### Out of scope entirely

- Dashboard / UI of any kind
- Specific content scanners (secret scanning, OCR/PII, DLP integrations)
- SCM OAuth integrations and API proxying (PR creation, issue management)
- Availability and performance functions — caching, rate limiting, quota
  enforcement — including the recording and audit of the decisions they
  make. The audit model of this specification (`04`) covers policy
  decisions; availability decisions and their records are
  implementation-specific.
- Any implementation-specific extension

---

## 4. Class P — Normative reference (do not restate)

A conforming implementation MUST correctly implement the Git smart protocol
over HTTPS and SSH, as defined by:

| Document | Covers |
| --- | --- |
| `gitprotocol-http(5)` | Git over HTTP: ref discovery, URL format, auth, session state, status codes |
| `gitprotocol-v2(5)` | v2 command-oriented protocol, capability advertisement, `ls-refs`/`fetch`/`object-info` |
| `gitprotocol-pack(5)` | pkt-line framing, v0/v1 exchange |
| `gitprotocol-capabilities(5)` | v0/v1 capability semantics |

These documents are referenced, not restated; a copy here would only drift
from the originals.

The specification's protocol surface is the **smart protocol only**; the
dumb protocol is out of scope entirely (§5.8). More generally, mechanisms
production upstreams have retired are out of scope rather than specified
defensively — the surface tracks what upstreams actually serve (§5.1).
The boundary of that pruning: protocol v2 covers fetch only, and push
remains a v0/v1 `receive-pack` exchange, so v0/v1 stays in scope for as
long as push itself depends on it.

### URL paths

`$GIT_URL` is opaque. Git states servers SHOULD handle all requests
matching `$GIT_URL`, because both protocols work by appending path
components; documented examples include catch-all CGI gateways with query
strings and nested submodule paths. Git does not constrain route shape, and
a specification mandating a route layout would be narrower than Git itself.
The only defensible requirement is prefix-completeness:

> An implementation MUST, for whatever prefix it serves as `$GIT_URL`,
> handle all protocol sub-paths appended to that prefix.

### Upstream is incomplete — a contribution opportunity

The http-protocol page carries literal `TODO: Document this further`
markers in both the `git-upload-pack` and `git-receive-pack` sections, and
leaves undefined: error behaviour when no "want" lines are sent; error
behaviour for an unreachable "want"; the pack-based and non-pack response
formats; client-side response parsing. Where deriving the proxy delta
surfaces genuine ambiguity here, **file upstream patches rather than
encoding one implementation's reading into this spec.**

---

## 5. Class P — The proxy delta (normative, MUST)

Behaviour that exists *only* because an intermediary is present. None of it
is specified upstream; all of it is externally testable.

### 5.1 Capability advertisement mediation — highest priority

The proxy relays the upstream capability advertisement. Two capabilities
are outright **policy-bypass vectors**:

| Capability | Bypass risk |
| --- | --- |
| `packfile-uris` | Server returns URIs the client downloads over http/https *in place of objects in the packfile*. That content never transits the proxy and is never scanned. |
| `filter` (partial clone) | Objects omitted from the pack; changes what the proxy can observe and inspect. |

Also requiring a stated position: `sideband-all`, `shallow`/`deepen*`,
`thin-pack` (bases not present in the pack), `object-format`.

> A conforming implementation MUST remove from the relayed advertisement
> any capability it cannot mediate, and MUST NOT relay `packfile-uris`
> where content inspection is in force.

The effective protocol surface through an intermediary is therefore the
**intersection** of what the upstream offers and what the intermediary can
mediate. Upstream hosts commonly implement a subset of the documented
protocol; nothing in this specification requires an intermediary to
supplement capabilities its upstream lacks.

### 5.2 Ref-level authorization is insufficient

Protocol v2 states wants "can be anything and are not limited to advertised
objects". In v1 the equivalent is gated behind `allow-tip-sha1-in-want` /
`allow-reachable-sha1-in-want`.

> Authorization decisions MUST be made on object reachability, not on
> refname alone. A refname-only policy is walkable via raw OID requests.

### 5.3 Inspection completeness

Content inspection is sound only if the intermediary can see the complete
set of objects a push introduces. A push that defeats that visibility MUST
be rejected.

- **Reachability.** Every object in the pushed pack MUST be reachable from
  the ref update being performed. An implementation MUST reject a push
  whose pack contains a commit not reachable from the claimed old→new range
  — a hidden or unreferenced commit — because such an object would enter
  the repository without passing reachability-based inspection. Where the
  reachable set cannot be determined, the push MUST be rejected.
- **Object coverage.** Inspection MUST cover every kind of object the push
  introduces, including annotated tag objects — their tagger identity,
  message, and signature. Dereferencing a tag to its target commit and
  inspecting only the commit leaves the tag a bypass channel.

A conforming implementation MUST reject a push that updates more than one
ref. A single-ref push has one old→new range whose reachable set bounds
what inspection must cover; multiple ref updates in one push multiply that
surface and its object-smuggling exposure without a corresponding need.

The capability positions of §5.1 (`packfile-uris`, `filter`) and the
dumb-protocol refusal of §5.8 enforce this same completeness principle at
the transport level.

### 5.4 Protocol version passthrough

v2 is negotiated via the `Git-Protocol` HTTP header, and over SSH via the
`GIT_PROTOCOL` environment variable — which upstream notes the server may
need explicit configuration to permit. Handling of these headers is a
common and easily overlooked source of proxy bugs, and is verifiable from
outside on each transport by a conformance test.

> An implementation MUST propagate protocol version negotiation on both
> transports. Silent downgrade to v0 is a conformance failure.

### 5.5 Denial semantics

Denials reach the client from two sources, and the intermediary treats
them differently.

**Intermediary-originated denials** — its own policy rejection (§5.6), its
own authorization decision, an unrecognized or administratively disabled
service, or an unparseable path. The intermediary is the authority for
these and MUST signal them clearly:

- refusal by the intermediary's own authorization → `403 Forbidden` on HTTP
- unrecognized or administratively disabled service name → `403 Forbidden`
- a policy denial after pack transfer has begun → the readable mid-stream
  error of §5.6

**Upstream-originated denials** — whether a repository exists, and whether
the client may access it, are determinations only the upstream can make.
The intermediary MUST relay the upstream's response faithfully and MUST NOT
resolve it into a finer distinction than the upstream provides. An upstream
commonly returns "not found" for a private repository the client cannot
access, concealing whether the repository exists; the intermediary MUST
relay that response unchanged, MUST NOT convert it into a "forbidden" that
would disclose the repository's existence, and MUST NOT present an upstream
denial as success (`200`).

The intermediary cannot itself tell whether an upstream "not found" means
the repository is absent or the client lacks access. It MAY attach a
descriptive message directing the client to verify their credentials and
access with the upstream, but MUST NOT assert a cause the upstream did not
state.

**Cross-transport consistency.** SSH carries no HTTP status codes. For
intermediary-originated denials, an implementation MUST present an
equivalent readable refusal on each transport it serves; for
upstream-originated denials, it relays what the upstream returns on that
transport. Where the SSH legs use agent forwarding (§5.10), the upstream
authenticates the client's own key and evaluates existence and access
against the client's identity; its denial is authoritative for that
client, and the intermediary MUST relay it unmodified.

### 5.6 Mid-stream rejection signalling

Once a smart-HTTP request has been dispatched to a git service the HTTP
status is `200`, and the outcome is carried in the pkt-line body. A denial
reached at this point MUST be delivered in that body; closing the
connection instead produces "the remote end hung up unexpectedly" on the
client and loses the reason.

Git provides three in-body mechanisms, and the one to use depends on the
phase and the operation:

- **Ref advertisement and negotiation** (`info/refs`, `ls-refs`,
  `upload-pack` want/have): a pkt-line `ERR <message>` packet. Sideband is
  not available in this phase.
- **Push (`receive-pack`)**: a report-status — `unpack ok` followed by
  per-ref `ng <ref> <reason>` — carried on sideband band 1, with
  human-readable detail on band 2. The client renders each `ng` as
  `! [remote rejected] <ref> (<reason>)`, which reads as a policy decision
  rather than a failure. It requires the client to have advertised
  `report-status` (or `report-status-v2`) and `side-band-64k`.
- **Fetch (`upload-pack`) after packfile transfer has begun**: a sideband
  band-3 fatal error, which aborts the stream and surfaces as
  `fatal: <message>`.

> A denial reached after the git service has been dispatched MUST be
> delivered to the client in the pkt-line body, never as a transport hang
> or reset. On push, where `report-status` and `side-band-64k` were
> negotiated, the denial MUST be a per-ref `ng <ref> <reason>`
> report-status; a band-3 fatal error is used only where those
> capabilities were not negotiated. The reason carried to the client MUST
> match the reason recorded in the audit trail.

An implementation MUST define how it treats the inbound request body on
denial — whether it drains or resets it — and any size or time bound at
which it aborts reception.

### 5.7 Deferral and statelessness

Holding a push for review introduces server-side state that Git-over-HTTP
does not assume. Reconciling that with the protocol's stateless model is a
proxy-delta concern; the interaction models for it — what the client sees,
whether the pack is retained, how a retry is correlated — are specified in
`03-deferral-interaction-models.md`.

### 5.8 Dumb protocol — out of scope, refused

The dumb protocol serves loose objects and packfiles over plain `GET` with
no Git-aware server process — bypassing `upload-pack` and therefore **all**
inspection. This specification's protocol surface is the smart protocol
only.

> A conforming implementation MUST NOT serve or relay dumb-protocol
> object paths. Requests for them are refused per the denial semantics of
> §5.5.

### 5.9 Smaller normative items

- **`object-format`** — SHA-256 repositories exist; both hash algorithms
  MUST be handled.
- **HTTP 1.0/1.1 and chunked encoding** — both SHOULD be supported on
  request and response bodies; interacts with streaming vs. buffering
  inspection.
- **Extra Parameters** — colon-separated via the `Git-Protocol` header.
- **Cookies** — servers MUST NOT require them and SHOULD ignore them; a
  proxy adding session cookies for its dashboard MUST NOT let them leak
  into Git request handling.
- **Inspection surface completeness** — a push introduces object content
  beyond the commits reachable from branch refs: annotated tag objects
  carry their own tagger identity, message, and optional signature.
  Where content inspection is in force, it MUST cover all object content
  the push introduces, including annotated tag messages — inspection
  that dereferences tags to their target commit and inspects only the
  commit leaves the tag object itself a bypass channel.

### 5.10 Credential custody and authorization layering

The intermediary sits between two authorization contexts: its own policy
decisions and the upstream's. The guarantees:

> **Attribution.** Every operation an intermediary performs against the
> upstream on behalf of a client MUST be attributable to the resolved
> pusher (`07`) in a durable audit record. By default that attribution is
> carried at the upstream: the operation uses authorization material
> attributable to the client, and the intermediary does not substitute
> shared or service credentials. Where the upstream cannot support
> per-client credentials, an intermediary MAY instead operate an
> on-behalf-of mode (`07` §4) in which the upstream operation uses an
> intermediary-controlled identity; there the attribution requirement is
> met by the intermediary's own audit trail, and the loss of upstream-side
> attribution MUST be explicit and recorded.

Upstream-carried attribution keeps the upstream's own audit trail truthful
and makes credential revocation effective end-to-end: revoking a client's
credential revokes their access through the intermediary, observably. The
on-behalf-of mode trades that end-to-end property for reach onto hosts that
cannot resolve per-client credentials.

> **Passthrough mode.** A conforming intermediary MUST support a mode of
> operation in which the client's `Authorization` material (end-to-end
> per RFC 9110) is forwarded to the upstream unmodified. In this mode the
> intermediary MUST NOT alter the material, MUST NOT write it to durable
> storage, and MUST bound its in-memory retention by the resolution of
> the submission it authorizes.

> **Brokered modes.** An intermediary MAY additionally support modes in
> which it substitutes derived, per-client authorization material for the
> upstream connection — for example an OAuth-based token exchange, or a
> deferred park-and-forward capability where passthrough is not workable.
> The attribution requirement holds unchanged in every such mode; a
> per-client derived credential — such as the user's own OAuth token —
> remains attributable to the client at the upstream, unlike the shared
> intermediary-controlled identity of `07` §4. Token-based forwarding is
> an HTTP mechanism: an OAuth token authenticates HTTP git operations,
> and the SSH counterpart of on-behalf-of forwarding is agent forwarding
> (below). Which mode governs a given route MUST be explicit
> configuration, and the mode in effect for a submission MUST be
> determinable from its audit record.

> **Authorization layering.** The intermediary's policy authorization
> supplements upstream authorization and MUST NOT replace it: an upstream
> denial is final regardless of intermediary policy. The intermediary
> SHOULD derive its own authorization decisions from the client identity
> established by the same credential, and MAY consult upstream
> permissions in doing so.

> **SSH.** Where both the client and upstream connections use SSH, the
> intermediary MUST support upstream authentication via the client's
> forwarded agent (the SSH agent protocol), and MUST NOT require clients
> to deposit private key material with the intermediary.

**Open issue — brokered-mode requirements.** Brokered modes are permitted
above but not yet fully specified: scoping of derived credentials (per
repository? per submission?), their lifetime, and what the audit record
must capture about the exchange all need drafting before a brokered mode
can carry a conformance claim of its own.

**Open issue — retention during deferral.** Model H (`03`) holds the
client's credential in memory for the review window; a deferred-forwarding
capability would retain it longer, at rest. Bounds and protection
requirements for retained authorization material need their own section
before either is normative.

---

## 6. The upstream environment

Upstream Git hosts are the environment an intermediary operates in, not
subjects of this specification. Nothing here places a requirement on an
upstream, and no part of the specification is a proposal for upstream
adoption: the intermediary exists to provide behaviour the upstream does
not itself provide. Upstream capabilities nonetheless condition what an
intermediary can do, and two devices keep that dependency explicit
without coupling the specification to any host's API.

### 6.1 Capability-conditional features

Some Class A features depend on upstream capabilities that not every host
provides — for example, verifying that presented authorization material
corresponds to a claimed upstream identity (a user-identity endpoint), or
resolving an SSH public key to an upstream account (a key-listing
endpoint). Minimal hosts — plain SSH-based git hosting with no management
API — lack these, and remain fully legitimate upstreams for the features
that do not need them. Pusher identity resolution is the principal
capability-conditional feature; its requirements are in
`07-identity-resolution.md`.

> An implementation MUST document, per policy feature, the upstream
> capabilities the feature requires. Where a configured upstream lacks a
> required capability, the implementation MUST either refuse the
> configuration or degrade observably: a check that cannot run MUST NOT
> be reported as passed.

Conformance tests for capability-conditional features run only against an
upstream that provides the capability; a test suite MUST distinguish "not
applicable in this environment" from "failed". `08-upstream-capabilities.md`
maps these capabilities onto widely-deployed platforms (informative).

### 6.2 Repository identity and namespace derivation

Policy scoping references a **canonical repository identity**. Where the
upstream encodes ownership in its URL structure, that identity is a
namespace tuple — at minimum (upstream host, owner, repository). Where it
does not (a plain git host, or a host serving repositories alongside
other services), the identity is (upstream host, canonical path): a
single opaque **slug** dimension. Namespace-bearing URLs are an upstream
capability in the §6.1 sense — guarantees scoped to an owner or
organization are conditional on it, and degrade explicitly to
repository-scoped where only a slug exists.

> Derivation of the canonical identity — tuple or slug — MUST be
> deterministic, documented, and normalizing (case, trailing `.git`,
> percent-encoding, path separators), such that equivalent request URLs
> yield the same identity.

Two operations consume the identity, with different strictness:

> **Rule selection.** Policy rules MAY match on any dimension of the
> canonical identity, with implementation-defined matcher semantics
> (literal, glob, regular expression). The match input MUST be the
> canonical identity, never the raw request URL, and the rule that
> matched MUST be identifiable in the audit record.

> **Submission binding.** Artifacts attached to a specific submission —
> approval consumption (`03` DF-10) above all — MUST bind by exact
> equality of canonical identity, never by pattern match. Entitlement
> *grants* may be pattern-scoped as rules; the *consumption* of a
> decision about one submission may not.

---

## 7. Explicitly excluded

Each of these is a good engineering opinion and a bad specification clause.

| Item | Why excluded |
| --- | --- |
| Specific URL paths | `$GIT_URL` is opaque upstream. Mandating a route would be narrower than Git itself permits. |
| Process topology (single server vs. two ports/apps) | No observable-behaviour consequence. Non-normative note at most. |
| Library choices | Implementation trade-offs. The spec cares that the capability works to a defined bar. |
| Storage engine / DB abstraction | Record *shape* is normative; persistence mechanism is not. |
| Restatement of pkt-line framing, negotiation, pack format | Normative reference only (§4). |
| Language or runtime | No observable-behaviour consequence. |

**On the OCI comparison:** OCI specifies its distribution API because
third-party clients depend on those endpoints. Here the Git client already
speaks a fixed protocol — it needs correct behaviour, not a particular
route layout. Specify protocol *behaviour*, not routes.

---

## 8. Normative language

IETF BCP 14 (RFC 2119 + RFC 8174), applied as:

- **MUST** — anything a Git client or a compliance auditor can observe
  from outside the black box.
- **SHOULD** — strong recommendations that don't affect interoperability.
- **MAY / non-normative** — internal structure, URL layout, process count,
  library selection, storage.

Example of well-formed MUST language:

> An approval MUST transition from `pending` to either `approved` or
> `rejected`. A rejected push MUST NOT reach the upstream.

Note what that says nothing about: which role enforces it, what table it
lives in, how a dashboard renders it.

---

## 9. Method: derive descriptively, with one named exception

Default rule: a behaviour becomes normative only after a working
implementation exhibits it. This keeps every requirement testable against
running code and prevents the specification drifting from practice.

**The one exception — security-critical proxy-delta items.** Some §5 items
(capability filtering, dumb-protocol refusal) are plausibly implemented by
*no* current implementation. They are normative anyway. The carve-out is
narrow: these requirements are derived from Git's own documented protocol
surface and describe bypass vectors of the proxy's entire reason for
existing — not product features. Product-feature aspirations (e.g.
structured findings/exemptions, `02` Appendix B) do not qualify and remain
non-normative until implemented.

**Inventory partition rule** (for auditing any implementation against the
draft):

| Finding | Becomes |
| --- | --- |
| Behaviour present in the reference implementation | Candidate MUST |
| Present only in another implementation | Candidate SHOULD; evaluate for inclusion |
| Present in no implementation, security-critical proxy-delta | MUST (the §9 carve-out) |
| Present in no implementation, product feature | Non-normative proposed extension |

