# Capability Mediation

Elaborates `01` §5.1 into per-capability positions. Conformance keywords per BCP 14 (`00-overview.md`). Requirement IDs:
`CP-n`. Source documents: `gitprotocol-capabilities(5)` (v0/v1), `gitprotocol-v2(5)`. Positions marked **TODO** are
deliberately unresolved pending protocol research or a design decision.

## 1. Position vocabulary

| Position    | Meaning                                                                           |
| ----------- | --------------------------------------------------------------------------------- |
| **Support** | The intermediary MUST implement the capability itself; stock clients depend on it |
| **Relay**   | Pass through between client and upstream unmodified; no mediation needed          |
| **Strip**   | Remove from the relayed advertisement; the client never sees it offered           |
| **Refuse**  | Additionally, reject the request if a client attempts it anyway                   |
| **TODO**    | Position pending; treated as Strip until resolved                                 |

## 2. Default rules

**CP-1** — **Fidelity by default.** In a transparent relay, the upstream advertisement passes through unmodified except
for the positions enumerated in this document. Every Strip or Refuse position MUST be justified by one of exactly two
grounds: (a) it closes a channel by which content evades inspection or authorization, or (b) the intermediary cannot
correctly process the capability's effect on the stream it must parse. Modifying the advertisement on any other ground
is a conformance failure.

**CP-2** — Stripping MUST operate on the advertisement in both directions where applicable (server capability lines, v2
capability advertisement), and a stripped capability attempted by a client anyway MUST result in a protocol-valid error,
not undefined behaviour.

### 2.1 The subset principle, per proxy mode

Two proxy modes negotiate capabilities differently, and the subset relationship to the upstream differs accordingly.

**CP-5** — In a transparent relay, the client-facing advertisement MUST be a subset of the upstream's own advertisement:
the intermediary filters, it never adds.

**CP-6** — In a store-and-forward mode there are two independent negotiations — client↔intermediary and
intermediary↔upstream — and the client-facing advertisement is the intermediary's own. Capabilities divide into two
kinds:

- **Connection-local** — properties of one hop, free to differ between hops: `side-band`/`side-band-64k`, `quiet`,
  `no-progress`, `agent`, `ofs-delta`, thin-pack transfer, negotiation capabilities. An intermediary MAY offer these to
  the client regardless of the upstream's advertisement (this is what enables intermediary-generated progress
  streaming).
- **End-to-end** — promises about the upstream repository or about what reaches it: `object-format`, `atomic`,
  `delete-refs`, `push-options`, `push-cert`, and on the fetch side `filter` and `packfile-uris`. An intermediary MUST
  NOT offer an end-to-end capability to the client unless it can honour it against the configured upstream — for these,
  the subset rule of CP-5 applies in every mode.

A store-and-forward advertisement is an allowlist by construction — the intermediary offers what its own server
implementation supports — so no fail-closed rule is needed there.

**CP-7** — **Unknown capabilities.** In a transparent relay, a capability the implementation does not recognize is
relayed by default: content enforcement happens at the pack layer, which fails closed on input it cannot parse, and
stripping unknowns by default would freeze protocol evolution for every client behind the intermediary. An
implementation MAY offer a strict mode that strips unrecognized capabilities where its inspection guarantees depend on
full comprehension of the negotiated stream; strict mode is explicit configuration, not the default.

## 3. receive-pack (push) — the core service

The baseline tier is what a stock `git push` over HTTPS or SSH exercises against any mainstream host: `report-status`,
`side-band-64k`, `delete-refs`, `ofs-delta`, `quiet`, `agent`, and thin-pack transfer.

| Capability         | Typical usage                                  | Position                   | Notes                                                                                                                                                                                                                                                                                                                                                  |
| ------------------ | ---------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `report-status`    | Universal; how the client learns per-ref ok/ng | **Support**                | Required by `01` §5.6 and `03` DF-9 — this is the denial-signalling channel                                                                                                                                                                                                                                                                            |
| `report-status-v2` | Newer clients/hosts                            | TODO                       | Extended status with option lines; verify semantics before relaying                                                                                                                                                                                                                                                                                    |
| `side-band-64k`    | Universal; progress and messages during push   | **Support**                | Required for mid-push messaging (deferral notices, validation output)                                                                                                                                                                                                                                                                                  |
| `delete-refs`      | Branch/tag deletion pushes                     | **Support**                | Deletions are policy-relevant operations: record with zero-id new-value per `04` §2.1. Disallowing deletion is a policy decision and SHOULD be enforced as an auditable per-ref rejection at the policy layer, not by withholding the capability in a transparent relay (CP-1); a store-and-forward advertisement MAY additionally decline to offer it |
| `ofs-delta`        | Universal pack encoding                        | **Support**                | Pack parsing concern; no policy surface                                                                                                                                                                                                                                                                                                                |
| `quiet`            | `git push -q`                                  | **Support**                | Client preference; suppresses band-2 progress, leaves band-3 errors                                                                                                                                                                                                                                                                                    |
| `agent`            | Universal (client version string)              | Relay                      | MAY be recorded on the push record (client identification aids audit)                                                                                                                                                                                                                                                                                  |
| `atomic`           | `git push --atomic`                            | Relay                      | Multi-ref pushes are rejected (`01` §5.3), so this applies only to the trivially-atomic single-ref case                                                                                                                                                                                                                                                |
| `push-options`     | `git push -o <opt>`                            | TODO                       | An arbitrary client→server string channel that bypasses nothing but is invisible to policy today; if relayed, the options SHOULD be recorded on the push record                                                                                                                                                                                        |
| `object-format`    | SHA-256 repositories                           | **Support**                | Both algorithms MUST be handled (`01` §5.9); MUST NOT relay a format the inspection pipeline cannot parse                                                                                                                                                                                                                                              |
| `session-id`       | Modern hosts                                   | Relay                      | Passed through unchanged                                                                                                                                                                                                                                                                                                                               |
| `push-cert`        | Signed pushes (`git push --signed`)            | TODO, Strip until resolved | The certificate signs the server-advertised nonce; a store-and-forward intermediary cannot replay it upstream (the upstream nonce differs). Position needed: terminate-and-record vs. unsupported                                                                                                                                                      |
| thin-pack transfer | Default for stock clients                      | **Support**                | Not an advertised capability on receive-pack — clients send thin packs by default; the intermediary MUST be able to resolve them (base objects available) to inspect content                                                                                                                                                                           |

## 4. upload-pack (fetch) — v0/v1

Fetch payload flows upstream→client and is deliberately uninspected (`01` §5.7 boundary discussion); most fetch
capabilities therefore Relay. The exceptions are the bypass vectors of `01` §5.1.

| Capability                                                 | Typical usage                       | Position                        | Notes                                                                                                                           |
| ---------------------------------------------------------- | ----------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `multi_ack`, `multi_ack_detailed`, `no-done`               | Universal negotiation               | Relay                           |                                                                                                                                 |
| `side-band`, `side-band-64k`                               | Universal                           | Relay                           | Band 3 remains available for intermediary error signalling (`01` §5.6)                                                          |
| `thin-pack`                                                | Universal                           | Relay                           | Fetch direction; uninspected by design                                                                                          |
| `shallow`, `deepen-since`, `deepen-not`, `deepen-relative` | CI and shallow clones — very common | Relay                           | Fetch-side only; no push-side reachability impact                                                                               |
| `no-progress`, `include-tag`                               | Common                              | Relay                           |                                                                                                                                 |
| `allow-tip-sha1-in-want`, `allow-reachable-sha1-in-want`   | Host-dependent                      | Strip unless reachability-aware | Relay only where authorization is object-reachability-based (`01` §5.2); refname-only authorization + raw-OID wants is walkable |
| `filter` (partial clone)                                   | Increasingly common                 | Relay, revisit                  | Acceptable while fetch responses are uninspected; becomes a Strip if response-side inspection is ever specified                 |
| `packfile-uris`                                            | Rare                                | **Strip**                       | Content transits outside the proxied connection entirely (`01` §5.1); strip by default in all modes                             |
| `symref`                                                   | Universal (HEAD advertisement)      | Relay                           |                                                                                                                                 |

## 5. Protocol v2 (fetch side)

| Item                | Typical usage             | Position              | Notes                                                                                                |
| ------------------- | ------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------- |
| `ls-refs`           | Universal                 | Relay                 | Apply ref-visibility filtering here if any is in force                                               |
| `fetch`             | Universal                 | Relay                 | Same argument treatment as §4 (`filter`, `packfile-uris` args mirror the capability positions above) |
| `object-info`       | Rare (client tooling)     | **Refuse** by default | Leaks object sizes without transferring content (`01` §5.9); enable per explicit configuration       |
| `server-option`     | Advertised by major hosts | TODO                  | v2's generic client→server option channel — same policy questions as `push-options`                  |
| `sideband-all`      | Host-dependent            | TODO                  | Changes response framing the intermediary must re-frame; verify before relaying                      |
| `wait-for-done`     | Rare                      | TODO                  |                                                                                                      |
| `session-id`        | Modern hosts              | Relay                 | Passed through unchanged                                                                             |
| unknown v2 commands | —                         | **Refuse**            | CP-1 applied to the command list: the v2 command surface is an allowlist                             |

## 6. Conformance consequences

**CP-3** — The positions above define the conforming advertisement for each service. A conformance test can diff the
intermediary's relayed advertisement against the upstream's raw advertisement and verify that exactly the
Strip/Refuse/TODO items are absent and the Support items are present.

**CP-4** — Where a Support-tier capability is absent from the upstream's own advertisement, the intersection rule of
`01` §5.1 applies: the intermediary MUST NOT advertise what the upstream cannot honour.

## 7. Open issues

- `push-cert` end-to-end story (the nonce problem above) — the only baseline-adjacent capability that is structurally
  incompatible with a store-and-forward intermediary as specified.
- `push-options` as a policy input: relay-and-record vs. strip. If relayed, are options visible to policy rules?
- `report-status-v2` semantics review.
- v2 for push does not exist today; if the Git project ever extends v2 to `receive-pack`, this section needs a new
  table.

<details>
<summary>Internal working note — remove before any publication</summary>

Live advertisement sample, 2026-08-09, github.com (coopernetes/test-repo):

- receive-pack:
  `report-status report-status-v2 delete-refs side-band-64k ofs-delta atomic object-format=sha1 quiet agent=... session-id=... push-options`
- upload-pack v0:
  `multi_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed allow-tip-sha1-in-want allow-reachable-sha1-in-want no-done symref=HEAD:... filter object-format=sha1 agent=...`
- v2: `ls-refs=unborn`, `fetch=shallow wait-for-done filter`, `server-option`, `object-format=sha1`

Observations: the §3 baseline Support tier matches the live receive-pack advertisement exactly. `push-cert` is NOT
advertised — Strip-until- resolved costs nothing against this host. `report-status-v2`, `push-options`, `session-id` are
all live → those TODOs are priority-ordered ahead of the others. Both `allow-*-sha1-in-want` variants are live on fetch
→ the §4 strip-unless-reachability-aware rule is not theoretical. No `packfile-uris` in either advertisement. No
`sideband-all`.

</details>
