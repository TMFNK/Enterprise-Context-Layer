# Agent Instructions

This repository is an Enterprise Context Layer (ECL) implementation template.

## If `meta/` does not exist yet

The ECL has not been built. Read `readme-for-agents.md` fully and follow its Quick-Start Checklist. Ask the human the discovery questions in Step 1 before writing any files.

## Once the ECL is built

Before acting on any task:

1. Read `meta/system-prompt.md` — agent identity, citation format, non-negotiable rules
2. Read `meta/domain-index.md` — knowledge domains, owners, primary sources
3. Check `domains/skills/` for a `SKILL.md` matching your task type; if one matches, follow it as the mandatory workflow

## Rules that always apply

- Every factual claim needs an inline citation: `[[Source, date]](path)`
- Document conflicts explicitly; never resolve them silently
- Sensitive topics get routing notes, not answers
- Append a record to the domain's `mapping-notes.md` after every run
