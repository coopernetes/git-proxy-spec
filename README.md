# git-proxy-spec

A working draft specification for a **policy-enforcing Git proxy** — a
reverse proxy that sits between a Git client and an upstream Git server and
can refuse, defer, or permit operations according to policy, while
producing a portable audit trail of every decision.

The specification is written against *observable behaviour*, so that
multiple independent implementations can conform, each to the conformance
class matching its ambitions. It does not restate the Git wire protocol; it
specifies the behaviour that exists only because an intermediary is present.

## Status

Draft. Wording, structure, and open questions are all in motion.

## How to read it

Start with **[00-overview.md](00-overview.md)** — the document map, the
conformance-keyword conventions (BCP 14), the goals, and the references.

| Document | Contents |
| --- | --- |
| [00-overview.md](00-overview.md) | Goals, conformance language, references, document map |
| [01-scope-and-method.md](01-scope-and-method.md) | Scope, conformance classes, protocol baseline (normative reference + proxy delta), upstream environment, method |
| [02-push-approval-state-machine.md](02-push-approval-state-machine.md) | Push lifecycle: canonical states, legal transitions, evidence requirements (`LC-n`) |
| [03-deferral-interaction-models.md](03-deferral-interaction-models.md) | The pending window on the wire: held-connection and reject-and-retry models (`DF-n`) |
| [04-record-field-sets.md](04-record-field-sets.md) | Push record, audit view, fetch record minimum field sets (`PR-n`, `AU-n`, `FR-n`) |
| [05-capability-positions.md](05-capability-positions.md) | Per-capability mediation positions (`CP-n`) |
| [06-policy-hook-contract.md](06-policy-hook-contract.md) | Policy decision contract and external decision protocol (`HK-n`) |
| [07-identity-resolution.md](07-identity-resolution.md) | Pusher vs. commit-metadata identity; resolution requirements (`ID-n`) |
| [08-scm-capabilities.md](08-scm-capabilities.md) | Informative: what SCM platforms offer (identity APIs, OAuth, namespaces), mapped to the capability-conditional features |

## A note on the working notes

Several documents carry collapsed **internal working notes** (fenced
`<details>` blocks labelled *"remove before any publication"*). These hold
derivation analysis and are tracked deliberately, not published prose. See
[PUBLISHING.md](PUBLISHING.md) before making this repository, or any
document in it, public.

## Contributing

[CONTRIBUTING.md](CONTRIBUTING.md) covers the drafting discipline and
tracks remaining work.

## Licence

[Apache License 2.0](LICENSE). Copyright 2026 Thomas Cooper.
