# Identity Resolution

Applies to implementations claiming **Class A**. Conformance keywords per BCP 14 (`00-overview.md`). Requirement IDs:
`ID-n`.

Establishes whose action a push is, so that authorization (LC-8), attribution (LC-5, LC-6), and scoped decisions (`03`
DF-10) rest on an identity that cannot be trivially forged.

## 1. Two kinds of identity

A push carries two unrelated kinds of identity, which MUST NOT be conflated.

- **The pusher** — the principal authenticated on the connection to the intermediary. This is the identity that maps to
  access on the upstream: it is established by credential material the client must possess. It is _the_ identity for
  authorization and approval.
- **Commit metadata identity** — the author and committer recorded in each commit object. This is self-asserted content:
  it is written by whoever produced the commit, is not authenticated by anything, and legitimately differs from the
  pusher and between commits. A single push may carry commits authored by several people, committed by another, and
  produced by automation with no human identity at all.

The requirements below treat these two kinds with deliberately different strictness: the pusher is resolved and trusted;
commit metadata is recorded and, at most, checked.

## 2. The pusher

**ID-1** — An implementation MUST NOT treat a client-presented identifier — the HTTP Basic username, the SSH login name
— as authenticated identity for any authorization, attribution, or approval decision. Such identifiers are
client-controlled input and are trivially forged.

**ID-2** — The pusher's identity MUST be anchored to the credential the client authenticated with (a bearer token, an
SSH public key, an OIDC assertion), never to a claimed name. The anchoring mechanism is implementation-defined and not
observable; the observable, testable property is that a request presenting a valid credential together with a false
username is attributed to the credential's true owner, not to the username.

**ID-3** — Mapping the authenticated credential to a named upstream account is a capability-conditional operation (`01`
§6.1): it depends on the upstream offering a means to resolve a token or key to an account. Where that capability is
present, the implementation MUST record the resolved account. Where it is absent — a host with no identity API — the
implementation MUST record the pusher as credential-anchored but account-unresolved, MUST NOT substitute the presented
username for the missing account, and MUST either fail closed on any control that requires a named identity (LC-8
self-approval; any policy naming specific reviewers or pushers) or anchor the pusher's identity internally rather than
at the upstream (§4).

**ID-4** — Where the pusher is resolved, the resolved identity MUST appear on the push record (`04`) and on every
decision attestation the push produces (LC-5, LC-6). The presented identifier MAY additionally be recorded, but MUST be
distinguishable from the resolved identity.

## 3. Commit metadata identity

**ID-5** — Author and committer identities are self-asserted commit metadata. An implementation MUST NOT treat them as
the pusher's authenticated identity, and MUST NOT require them to match the pusher as a condition of accepting a push by
default.

**ID-6** — Verifying commit metadata — matching author or committer email against the resolved pusher, against a
known-identity set, or against signature material — is optional policy. An implementation MAY offer it; where offered,
an unverified author or committer is a conforming outcome and SHOULD be surfaced as an advisory signal on the push
record rather than blocking the push by default. Distributed authorship, history rewriting (rebase, amend), and commits
produced by automation are ordinary and MUST NOT be treated as policy failures in the absence of an explicit rule
configured to reject them.

## 4. Intermediary-anchored identity (on-behalf-of)

Where the upstream cannot resolve a credential to a named account (ID-3), failing closed is one response. The other is
for the intermediary to be the identity authority itself: to authenticate the pusher against a credential it issued and
validates, and to perform the upstream operation under a separate identity it controls. The resolved pusher remains the
principal; the upstream sees the intermediary. This is an on-behalf-of relationship, and it relocates the attribution
guarantee from the upstream to the intermediary rather than abandoning it.

**ID-7** — An implementation MAY offer an on-behalf-of mode in which it authenticates the pusher against a
locally-issued credential and performs the upstream operation under an intermediary-controlled identity. This is
permitted where the upstream cannot support per-client credentials.

**ID-8** — In this mode the locally-issued credential is the identity anchor; ID-1 (a presented name is never trusted)
and ID-2 (identity is anchored to the credential) apply unchanged. The internal attribution guarantee holds; only the
upstream's view of the actor degrades.

**ID-9** — Every operation an intermediary performs under an intermediary-controlled identity on behalf of a pusher MUST
record, on the push record (`04`), both the resolved pusher and the upstream identity actually used, so the on-behalf-of
binding is reconstructable from the audit trail alone.

**ID-10** — Where the intermediary-controlled identity is shared across pushers (a single machine account on the
upstream), the resulting loss of upstream-side per-pusher attribution MUST be an explicit, recorded property of the
deployment. The intermediary's audit trail becomes the sole authority for per-pusher attribution; an implementation MUST
NOT represent the upstream's records as carrying attribution they do not hold.

In this mode the attribution requirement of `01` §5.10 is satisfied by the intermediary's audit trail (ID-9);
independent upstream-side attribution is forgone by design, and ID-10 requires that trade to be visible.

## 5. Why the asymmetry (informative)

The pusher is the one identity backed by something the actor must possess — a credential that will be honoured — so it
can carry authorization weight. Commit author and committer are labels attached to content; git neither authenticates
them nor prevents one person from writing another's name. Resting an authorization or approval decision on commit
metadata would let any client assert any identity by editing a commit.

## 6. Open issues

- **Signed commits.** A verified commit signature (GPG/SSH) does bind commit metadata to a key, which could raise
  author/committer from self-asserted toward authenticated for signed commits. Whether the spec should define a stronger
  posture for verified-signed commits, and how that interacts with ID-5/ID-6, is undecided.
- **On-behalf-of credential issuance.** §4 requires the intermediary to authenticate the pusher against a locally-issued
  credential but says nothing about issuing, rotating, or revoking those credentials, nor about how a developer enrols.
  That lifecycle needs its own treatment before §4 is implementable to a defined bar.
- **Automation identity.** A push produced entirely by automation still has a pusher (the automation's credential).
  Whether the spec should distinguish a human pusher from a service/automation pusher — for approval routing, for
  instance — is unaddressed.

<details>
<summary>Internal working note — remove before any publication</summary>

Derivation, 2026-08-15, fogwall `main` vs finos/git-proxy `origin/main`.

The pusher/metadata split and ID-5/ID-6's soft posture are fogwall's observed behaviour: `PushRecord` separates
`user`/`resolvedUser`/ `scmUsername` (pusher, resolved account, provider login) from
`author`/`authorEmail`/`committer`/`committerEmail` (commit metadata), and unverified author/committer email surfaces as
an advisory warning without blocking — sourced ID-4's distinguishability and ID-6's soft-signal default. ID-1 is the
never-trust-presented-name rule fogwall encodes as "git username is arbitrary" (SCM identity resolved from token, not
the Basic username); ID-3 credential-anchored-but-account-unresolved matches fogwall's null-`resolvedUser` path in
open/config-only mode. `resolvedUser` vs `scmUsername` (config resolution vs token-verified provider login) is the
mechanism detail ID-2 deliberately leaves unobservable.

finos/git-proxy: `User` carries `gitAccount`/`email` as single scalars; identity resolution from token was in progress
(PR #1604, `findUserByGitAccount`) and `gitAccount` isn't indexed in the file sink. This is the "fogwall drives the
baseline, git-proxy grapples" case — but per the functional-register rule, §§1-4 state only the behaviour, no
comparison, no history. Keep internal.

fogwall issue cross-refs (internal): identity model normalized across `proxy_users`/`user_emails`/`user_scm_identities`;
SSH pubkey→identity was the remaining gap noted in the SSH transport work (username always `git`, resolve from forwarded
key).

§4 on-behalf-of is the biggest architectural capability gap between the two reference implementations, and the reason it
is specified as MAY/observable only. Intermediary-anchored identity requires the proxy to be a real credential-issuing
git endpoint that terminates and authenticates on its own terms. fogwall has this natively — JGit `ReceivePack` makes it
a first-class git server, so issuing locally-significant credentials and pushing upstream under a separate identity is
within the existing model (cf. the OAuth-brokered push-forwarding design, fogwall #317, and credential-at-rest #6).
git-proxy has no native git server; to offer this it would need to either implement a Node git server or front the Git
project's own `git http-backend` CGI to terminate the protocol as an authenticating endpoint. §4 states only the
behaviour so either path conforms; the mechanism gap stays here, out of spec text, per the functional-register rule.

</details>
