---
description: Check the documentation's invariants — links, single-home facts, indexes, status markers
---

Run the documentation checker from the repository root:

```bash
python3 scripts/doc-assert.py
```

It is stdlib-only and needs no network, no build and no database. Every check it runs exists
because the defect it catches has already happened here — a broken link that reached `develop` in
feature 03, a test count written into five documents and stale in four, a PR checklist requiring a
CI job that does not exist, a command missing from the README's list.

**A failure is a finding, not a formatting nit.** Fix the document the check names; do not weaken the
check to make it pass. If a check is genuinely wrong — it flags something that is not drift — say so
and change the check deliberately, in its own commit, rather than deleting the rule.

If it reports nothing, say exactly that. Do not re-run it to be sure, and do not report a check that
was skipped as one that passed.
