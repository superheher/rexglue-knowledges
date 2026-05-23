# Contributing to the Xbox 360 Recompilation Knowledge Base

This KB is meant to **compound**: every port should leave it more useful for the
next one. It is written to be **published and reused** by other people and by
AI/agent sessions.

## Principles

- **English only.** All content in this repository is written in English.
- **General vs title-specific.** Put transferable knowledge in `general/`; put
  one port's live findings in `titles/<id>/`. Keep `general/` clean and citable
  so it can stand alone if published.
- **Promote findings.** When a title-specific discovery is actually general (a new
  pitfall, a reusable shim/hook pattern, a jump-table variant), **promote** it
  into `general/95-pitfalls-and-patterns.md` or the relevant `general/` doc. This
  promotion step is the whole point.
- **Method & measurements, not content.** Document *how* and *what you measured*.
  **Never** commit copyrighted game code or assets, or extracted files. The dump,
  `private/`, `extracted/`, `generated/`, `*.xex*` and asset roots are git-ignored.
- **Cite for empirical claims.** When you assert a specific behaviour/value, link
  the source (a tool README section, a Xenia file, a Free60 page) next to it.
  Upstream docs win over this KB where they disagree.
- **Be honest about confidence.** Mark unverified specifics as "to verify"; don't
  invent ordinal numbers, addresses, or API tables.

## How to add things

- **New title:** copy `templates/title-feasibility.md` and
  `templates/title-onboarding-checklist.md` into `titles/<id>/`; follow
  `general/90-new-title-onboarding-playbook.md`.
- **New general doc:** slot it into the numbered scheme (foundations 00–25, core
  40–60, runtime 70–80, meta 90–99) and add it to the `README.md` index with a
  status flag (✅ / ✍️ / ⏳).
- **Pitfall entries** use the **Symptom → Cause → Fix** format, with the relevant
  deep-dive doc in parentheses.
- **Boot/render logs** record `address/hash → cause → fix`; these journals are the
  most reusable artifact a port produces.

## Style

- Markdown; wrap prose ~80 cols; use tables for comparisons and triage.
- Cross-link with `general/NN-…` paths or `[[slug]]` wikilinks; keep slugs unique.
- Prefer short, dense, skimmable sections over long prose.

## Publishing

The intent is to be able to publish (at least) `general/` + `templates/` as a
standalone reference. Keep them free of machine-specific or title-specific
details so that split stays clean. See `LICENSE`.
