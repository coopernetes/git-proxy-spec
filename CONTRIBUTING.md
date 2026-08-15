# Contributing

This repository holds a working-draft specification. These notes are for
people adding to or revising it; they are not part of the specification.

## Drafting discipline

The specification constrains observable behaviour and nothing else. The
core rule and the derivation method are stated in the spec itself
(`01-scope-and-method.md`, §2 and §9). Three failure modes to watch for
when adding or changing normative text:

1. **Creeping inward** — mandating internals (architecture, storage,
   libraries, process topology). If a requirement cannot be observed from
   outside the implementation, it is not a MUST.
2. **Creeping outward** — normative requirements for optional or
   product-specific features. A minimal conforming implementation must not
   be forced to carry them.
3. **Restating Git** — duplicating what the Git protocol documentation
   already specifies. Reference it (`01`, §4); a copy here only drifts.

A behaviour becomes normative once a working implementation exhibits it.
The one exception is the narrow set of security-critical proxy-delta items
derived directly from Git's documented protocol surface (`01`, §9).

## Status and remaining work

Drafted:

- Scope, conformance classes, protocol baseline, proxy delta (`01`)
- Push approval state machine (`02`)
- Deferral interaction models (`03`)
- Push record, audit view, fetch record (`04`)
- Capability mediation positions (`05`)
- Policy decision contract (`06`)
- Identity resolution (`07`)
- SCM platform capability matrix (`08`, informative)

Open:

- Conformance test outline — one test per MUST, observed externally
- Class P normative-reference table — needs a citation check against the
  current `gitprotocol-*` man pages
- Upstream patch candidates — ambiguities found in the Git documentation
  during derivation, to be filed with the Git project rather than
  specified around

Per-document open issues are listed in each document's own "Open issues"
section.

## Publishing

See `PUBLISHING.md` before making the repository, or any document in it,
public — several documents carry internal working notes that must be
stripped first.
