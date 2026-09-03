# Record Field Sets

Applies to implementations claiming **Class A**. Conformance keywords per
BCP 14 (`00-overview.md`). Requirement IDs: `PR-n` (push record), `AU-n`
(audit view), `FR-n` (fetch record).

## 1. Model: one record, two views

**Push record.** The lifecycle record of `02` §1: exactly one per push
submission, created at `received` and carried through to a terminal
state. It is the unit of storage, of API access, and of review.

**Audit view.** The reconstruction that `02` LC-10 requires: for any
submission, the ordered sequence of state transitions with their times,
actors, and triggers. The audit view is normative as a view. An
implementation MAY store transitions as an explicit event log, or MAY
derive the view from evidence embedded in the push record (step
timestamps, attestations, terminal markers), provided the derivation is
total. No second stored object is required.

**Fetch operation record.** Fetch has no lifecycle — disposition is
inline allow/deny (`02` scope note). Its record is therefore flat: one
operation, one decision, no states.

## 2. Push record — minimum field set

**PR-1** — A conforming implementation MUST persist, for every push
submission, at least the fields marked MUST below, and MUST be able to
return them through its policy/audit interface.

Fields are normative as reportable content — what the policy/audit
interface can return. An implementation MAY store a field directly or
derive it at query time (for example, by joining a referenced provider
record), provided the derivation is deterministic and stable for the life
of the record. Derivation through configuration that can be edited after
the record is written does not qualify unless the referenced configuration
is immutable or versioned.

### 2.1 Submission identity and context

| Field                        | Level            | Notes                                                                                                                                           |
| ---------------------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                         | MUST             | Opaque, unique per submission; this is the correlation identifier (`03` §1)                                                                     |
| `receivedAt`                 | MUST             | Timestamp of submission acceptance                                                                                                              |
| repository identity          | MUST             | The canonical identity of `01` §6.2 — namespace tuple or (host, slug); the raw request URL MAY be retained in addition                          |
| target ref                   | MUST             | Full ref name (`refs/heads/...`, `refs/tags/...`)                                                                                               |
| old / new object ids         | MUST             | The commit range (`commitFrom` / `commitTo`); zero-id conventions denote ref creation and deletion                                              |
| submitter presented identity | MUST             | The username presented to the intermediary                                                                                                      |
| submitter email              | SHOULD           | Where user management is in use                                                                                                                 |
| resolved upstream identity   | conditional MUST | Where the identity-resolution capability (`01` §6.1) is in use, both presented and resolved identity MUST be recorded; otherwise not applicable |
| transport                    | SHOULD           | e.g. `https` / `ssh`; MAY be derived, e.g. from a provider reference where each provider entry binds exactly one transport                      |
| canonical state              | MUST             | Exactly one of the `02` §2 states (LC-1)                                                                                                        |

### 2.2 Content

| Field                  | Level  | Notes                                                                                                           |
| ---------------------- | ------ | --------------------------------------------------------------------------------------------------------------- |
| `commits[]`            | MUST   | Per commit: object id, parent id(s), author name and email, committer name and email, message, commit timestamp |
| signature material     | SHOULD | Commit signatures and sign-off trailers, where present                                                          |
| annotated tag metadata | SHOULD | For tag pushes; ref _type_ itself is derivable from the target ref name prefix and needs no separate field      |

### 2.3 Evaluation evidence

| Field                     | Level  | Notes                                                                                                                                                             |
| ------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `steps[]`                 | MUST   | One entry per executed policy check                                                                                                                               |
| step: name                | MUST   | Which check ran                                                                                                                                                   |
| step: outcome             | MUST   | Three-way at minimum: passed / policy violation / could-not-run — the §6.1 distinction; a check that could not run MUST NOT be conflated with either pass or fail |
| step: message             | MUST   | Human-readable result                                                                                                                                             |
| step: evidence content    | SHOULD | Structured findings, diff excerpts, scanner output                                                                                                                |
| step: order and timestamp | SHOULD | Execution order explicit; per-step time feeds the audit view                                                                                                      |
| step: logs                | MAY    |                                                                                                                                                                   |

### 2.4 Decision evidence

| Field                         | Level               | Notes                                                                                                                                     |
| ----------------------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| approval attestation          | MUST on `approved`  | Reviewer identity (username, email), timestamp, automated flag; question/answer content where attestation questions are configured (LC-6) |
| rejection evidence            | MUST on `rejected`  | Actor type; reviewer identity when human; reason; timestamp (LC-5)                                                                        |
| cancellation evidence         | MUST on `canceled`  | Same shape as rejection (LC-5)                                                                                                            |
| self-approval override marker | MUST when exercised | LC-8: every use of the override identifiable as such                                                                                      |
| error cause                   | MUST on `error`     | LC-9                                                                                                                                      |
| summary / blocked message     | SHOULD              | One-line disposition for list views                                                                                                       |
| `forwardedAt`                 | MUST on `forwarded` | Terminal transition time for the audit view                                                                                               |

**PR-2** — Attestation content shape (map vs. list, question wording) is
non-normative. Each configured question and the reviewer's response to it
MUST be individually recoverable.

**PR-3** — A push carries exactly one (ref, old, new) triple; multi-ref
pushes are rejected (`01` §5.3).

## 3. Audit view — minimum field set

**AU-1** — For every submission, the audit interface MUST produce the
ordered transition sequence, each entry carrying: submission `id`,
from-state, to-state, occurred-at, actor type (`automated` / `human`),
actor identity when human, and the trigger or reason. Where the
transition was rule-driven, the matched rule MUST be identifiable
(`01` §6.2).

**AU-2** — Storage form is non-normative (explicit event log or
derivation from the push record), provided AU-1 is satisfiable for every
submission, including those that ended in `error`.

## 4. Fetch operation record

**FR-1** — An implementation that applies policy to fetch operations
SHOULD record, per operation: `id`, timestamp, canonical repository
identity, presented identity, resolved identity where available,
disposition (allowed / denied), and the matched rule for denials.

## 5. Open issues

- **Correlation identifier derivation.** An identifier derived purely
  from push content (e.g. the commit range) is not unique across
  repositories — the same range pushed to two repositories collides.
  Interacts directly with `03` DF-10 scoping and `01` §6.2 binding. The
  draft's `id` is "opaque, unique per submission"; whether content-derived
  identifiers can conform needs an explicit position.
- **Redaction vs. completeness.** Evidence content can itself contain
  secret material. Post-terminal redaction of stored evidence conflicts
  with audit-trail immutability expectations; the spec needs a stated
  position on what may be redacted and what must survive.
- **Retention.** Submitter and commit emails are personal data; evidence
  content can be large. Whether retention schedules are in scope at all,
  or an operator matter, needs deciding.

<details>
<summary>Internal working note — remove before any publication</summary>

Field derivation, 2026-08-09, fogwall `main` vs finos/git-proxy
`origin/main` 790f7535.

Present in both (→ MUST candidates): id, timestamp, url + derived
repo/project/name, branch, commitFrom/commitTo, user, userEmail, state
(enum vs flags), errorMessage, blockedMessage, autoApproved,
autoRejected, steps (name/error/blocked/messages/content/logs), commits
(author/committer/emails/message/parent/timestamp), approval attestation
(reviewer username+email, timestamp, automated, answers), rejection
reason with reviewer.

fogwall-only: Attestation.Type incl. CANCELLATION (reasons on cancel);
selfApproval marker; resolvedUser + scmUsername (identity resolution);
per-step stepOrder + timestamp; upstreamUrl distinct from request url;
commit signature + signedOffBy; forwardedAt; FetchRecord
(provider/owner/repo, result ALLOWED|BLOCKED, pushUsername marked
untrusted, resolvedUser); StepStatus enum incl. could-not-run
distinction.

git-proxy-only: explicit protocol field (https|ssh) — correction: NOT a
gap in fogwall; fogwall derives transport via PushRecord.provider → the
provider entry's single URI scheme (one transport per provider entry).
Sourced the reportable-content/derivation-stability note instead.
tagData (annotated tag metadata) — genuinely gp-only; ref _type_ is
derivable in both (fogwall stores the full refname incl. refs/tags/ in
`branch`, PushStorePersistenceHook builder.branch(cmd.getRefName())).
capabilities[]; pullAuthStrategy; content-derived id
(`commitFrom__commitTo`) → sourced the correlation-id open issue. Their
audit.ts skips RequestType.PULL entirely → fetch record is fogwall-only,
hence FR-1 SHOULD.

Neither: session-id (AU-3 is aspirational SHOULD, protocol-derived);
explicit transition-event log (both derive history from embedded
evidence — hence AU-2).

</details>
