# Publishing checklist

This repository is a working draft and currently holds internal working notes in version control. Before making the
repository — or any individual document — public, work through this list.

## 1. Strip internal working notes

Every document may carry a collapsed `<details>` block labelled **"remove before any publication"**. These contain
derivation analysis (comparisons against reference implementations, issue cross-references, and other internal context)
that is deliberately not part of the specification text. Remove each block before publication.

Files currently carrying an internal note (re-check with the grep below, as documents are added):

- `02-push-approval-state-machine.md`
- `04-record-field-sets.md`
- `05-capability-positions.md`
- `06-policy-hook-contract.md`
- `07-identity-resolution.md`

Find every occurrence:

```sh
grep -rl "remove before any publication" .
```

Verify none remain after stripping:

```sh
grep -rn "remove before any publication" . && echo "STILL PRESENT" || echo "clean"
```

## 2. Register conventions

- Confirm the specification text itself carries no comparison to, or narration of, any specific implementation or
  project — only functional statements. The internal notes are the only place such analysis belongs, and step 1 removes
  them.
- Confirm every "Open issue" marker is either still genuinely open or has been resolved in the text.

## 3. Licence

The repository is licensed under the Apache License 2.0 (`LICENSE`). No action needed at publication time; confirm the
copyright line is correct.
