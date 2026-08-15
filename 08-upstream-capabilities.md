# Upstream Capability Designations (informative)

This document maps upstream SCM platforms to the capability-conditional
features of `01` §6.1. It is **informative** reference material: the
normative requirement is that an implementation documents, per feature,
the upstream capabilities that feature depends on (`01` §6.1), and degrades
observably where a configured upstream lacks them. The matrix here records
what widely-deployed upstreams offer, so that dependence can be judged
against real platforms rather than in the abstract.

Named platforms are the environment the intermediary operates in, not
subjects of the specification (`01` §6). GitHub and GitLab expose the
fullest capability surface and serve as the reference points; Forgejo and
Gitea share a common API; Bitbucket differs in identity handling; a bare
git server offers none of the API-backed capabilities.

## 1. Capability taxonomy

- **Token identity resolution** — an authenticated endpoint that maps a
  presented credential (a PAT or bearer token) to the account it belongs
  to. This is what lets the intermediary resolve the pusher from the
  credential rather than the presented username (`07` ID-2, ID-3).
- **Key identity resolution** — an endpoint listing a *named* account's
  registered SSH public keys (`.../users/{login}/keys`). Because the login
  is an input, this endpoint **confirms rather than discovers**: the
  intermediary must already hold candidate logins for the pusher, fetch
  each login's keys, and match the connecting key's fingerprint. It
  therefore presupposes an established pusher-to-account link — a key
  alone cannot be reversed to its owner. This is the asymmetry with token
  resolution, where the credential is self-identifying: a token push can
  bootstrap identity, an SSH-key push can only verify an identity already
  linked. A first-time SSH pusher with no prior link is unresolvable even
  where the platform offers a key-listing API.
- **OAuth identity linking** — a browser-based OAuth2 authorization-code
  flow, out of band from any git push, in which a user proves ownership of
  their upstream account. It establishes the pusher-to-account link
  verifiably and with no manual administrator mapping. The SSH key is never
  part of this exchange; what OAuth supplies is the link itself, so that
  once an account is linked, key resolution has a candidate login to match
  the connecting key against — the missing piece for an SSH-first pusher.
  Separately, with appropriate scopes, the same flow can yield a token the
  intermediary forwards on the user's behalf over HTTP (`01` §5.10); this
  is distinct from SSH on-behalf-of forwarding, which uses agent
  forwarding, not a token.
- **Verified email** — whether the identity response carries a usable
  account email, which a commit author/committer verification policy can
  match against (`07` ID-6).
- **Namespace-bearing URL** — whether ownership is encoded in the request
  path (owner, workspace, or group), from which the namespace tuple can be
  derived (`01` §6.2).
- **Credential-form constraint** — whether the upstream's git endpoint
  requires a specific credential form, obliging the intermediary to
  rewrite the client's credential before forwarding (`01` §5.10).

## 2. Platform matrix

| Platform | Token identity | Key identity | OAuth linking | Verified email | Namespace |
| --- | --- | --- | --- | --- | --- |
| GitHub | yes (`GET /user`) | yes (`GET /users/{login}/keys`) | yes | weak — often absent by privacy default | `owner/repo` |
| GitLab | yes (`GET /api/v4/user`) | yes (`username → id → /keys`) | yes | yes (primary email) | `group[/subgroup…]/repo` (nested) |
| Forgejo / Gitea | yes (`GET /api/v1/user`) | yes (`GET /api/v1/users/{login}/keys`) | yes | yes (login and email) | `owner/repo` |
| Bitbucket | yes (`GET /2.0/user`) | no | yes | account email is the lookup input | `workspace/repo` |
| Bare git server | no | no | no | no | path only, no semantic namespace |

Every hosted platform above provides an OAuth2 flow, so establishing the
pusher-to-account link is straightforward wherever one of them is the
upstream. A bare git server has no such flow; linking there is manual or
unavailable.

### Per-platform notes

- **GitHub.** Token identity via `GET /user`; account emails are commonly
  empty because users hide them, so an email-matching policy is weaker
  here than the presence of the field suggests. Key listing is public and
  returns no fingerprint field, so fingerprints are computed from the raw
  key.
- **GitLab.** Key resolution is two-step (resolve the username to a
  numeric id, then fetch that id's keys). Namespaces nest arbitrarily
  through subgroups, so the "owner" element of the namespace tuple may be
  a multi-segment path rather than a single account.
- **Forgejo / Gitea.** A common API surface; the key-listing response
  includes a fingerprint field, though an implementation may recompute
  from the raw key for consistency across platforms.
- **Bitbucket.** Provides token identity but no public key-listing API, so
  SSH pushes cannot be resolved to an account by key — identity resolution
  is available for token-authenticated pushes only. Bitbucket's git
  endpoint also does not accept the account email as the credential
  username; it requires the account's generated username. An
  implementation must look that username up (`GET /2.0/user` with the
  email) and rewrite the forwarded credential from email to username — a
  credential-form constraint in the sense of `01` §5.10.

## 3. Bare and API-less hosts

A plain git server — SSH- or HTTP-served repositories with no management
API — offers none of the API-backed capabilities. Token and key identity
resolution are unavailable, no account email can be fetched, and the URL
path is a filesystem location rather than a semantic namespace.

Such a host remains a legitimate upstream. The features that do not depend
on these capabilities work unchanged; the features that do must degrade
observably (`01` §6.1). For identity specifically, the pusher stays
credential-anchored but account-unresolved (`07` ID-3): the intermediary
either fails closed on controls that require a named identity, or acts as
the identity authority itself and forwards under an intermediary-controlled
identity (`07` §4, on-behalf-of).
