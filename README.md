# Enterprise Context Layer (ECL)

> *"The central intelligence that encompasses all knowledge for your company — able to answer any question, self-updating, built from ~1000 lines of Python and a Git repo."*
> — Andy Chen, [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer)

---

## What This Is

The Enterprise Context Layer is not a database. It is not a search engine. It is not a chatbot.

It is a **living, self-maintaining Git repository** that encodes how your company actually works — the institutional memory, judgment calls, org behaviour, process reality, and cross-domain relationships that no retrieval system can derive on its own.

Traditional RAG (Retrieval-Augmented Generation) fetches documents. The ECL synthesises understanding. When a sales rep asks "how long do we keep customer data after churn?", a retrieval system returns the closest policy document. The ECL returns: *"Don't answer this yourself — route to the security team. Here is why, with three cited incidents where reps got this wrong."*

The difference is synthesis over retrieval. The ECL contains the reasoning frameworks your experts use, not just the raw facts they work with.

**This repository is a generic implementation template.** An LLM agent following this document should be able to build, seed, and run a complete ECL for any company, regardless of industry, size, or tooling.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [When to Build an ECL](#2-when-to-build-an-ecl)
3. [Architecture Overview](#3-architecture-overview)
4. [Repository Structure](#4-repository-structure)
5. [The Two Foundational Patterns](#5-the-two-foundational-patterns)
6. [Step 1 — Discover Your Company's Knowledge Domains](#step-1--discover-your-companys-knowledge-domains)
7. [Step 2 — Map Your Data Sources](#step-2--map-your-data-sources)
8. [Step 3 — Define Source Authority](#step-3--define-source-authority)
9. [Step 4 — Create the Meta Seed Files](#step-4--create-the-meta-seed-files)
10. [Step 5 — Write the System Prompt](#step-5--write-the-system-prompt)
11. [Step 6 — Build the Task System](#step-6--build-the-task-system)
12. [Step 7 — Implement the Worker Loop](#step-7--implement-the-worker-loop)
13. [Step 8 — Seed the Initial Content](#step-8--seed-the-initial-content)
14. [Step 9 — Run the Maintenance Agent](#step-9--run-the-maintenance-agent)
15. [Step 10 — Build the Query Interface](#step-10--build-the-query-interface)
16. [Citation Rules](#16-citation-rules)
17. [The Context Graph: Backlinks and Cross-References](#17-the-context-graph-backlinks-and-cross-references)
18. [Source Reliability and Conflict Resolution](#18-source-reliability-and-conflict-resolution)
19. [Scaling and Multi-Agent Parallelism](#19-scaling-and-multi-agent-parallelism)
20. [Human-in-the-Loop Review](#20-human-in-the-loop-review)
21. [RBAC and Sensitive Content](#21-rbac-and-sensitive-content)
22. [Drift Detection and Staleness](#22-drift-detection-and-staleness)
23. [Adapting the ECL to Your Company](#23-adapting-the-ecl-to-your-company)
24. [What the ECL Is Not](#24-what-the-ecl-is-not)
25. [Glossary](#25-glossary)

---

## 1. Core Concepts

Before building anything, an LLM agent implementing this system must internalise these five principles. They govern every decision about structure, content, and behaviour.

### 1.1 The Git Repo Is the Single Source of Truth

Every piece of knowledge the ECL contains lives as a file in this repository. Every update, correction, addition, and deletion is a git commit. This is not a style preference — it is the architectural foundation that makes everything else possible:

- **Auditability:** `git log` gives a complete, timestamped history of how understanding evolved.
- **Conflict resolution:** When two agents disagree about a fact, the commit history shows which version came from which source at which time.
- **Multi-agent coordination:** Multiple workers writing simultaneously use git's push-rejection mechanism as a distributed mutex. The last clean push wins; others retry.
- **Human legibility:** A human engineer can `git clone` the repo and read the entire company's institutional memory as Markdown files.

There is no separate database. No vector store. No external state. If it is not in a committed file in this repo, it does not exist in the ECL.

### 1.2 The ECL Is Built for Traceability, Not Readability

Every factual claim in every ECL file must have an **inline citation** that traces it to a primary source. This is the single most important writing rule.

Bad:
```
Our enterprise pricing starts at $50,000 per year.
```

Good:
```
Our enterprise pricing starts at $50,000 per year [[Pricing page, Q1 2026]](../sources/pricing-q1-2026.md).
```

Better:
```
Our enterprise pricing starts at $50,000 per year [[Pricing page, Q1 2026]](../sources/pricing-q1-2026.md),
though sales ops noted on 2026-02-14 that EMEA deals are often discounted 15–20%
[[Slack #sales-ops, 2026-02-14, @jane.smith]](../sources/slack-sales-ops-feb-2026.md).
```

If an agent cannot cite a claim, it must not make it. Unsourced assertions are more dangerous than gaps because they create false confidence.

### 1.3 Document the Conflict, Not the Winner

When two sources disagree, the ECL does not pick a side. It documents the conflict explicitly:

```markdown
## Data Retention Policy — Conflict Note

The legal team's policy document states customer data is deleted after 90 days of churn
[[Legal Policy v2.3]](../legal/data-retention-policy.md).

The engineering runbook describes a 120-day deletion pipeline
[[Infra Runbook, 2026-01]](../engineering/deletion-pipeline-runbook.md).

**These conflict.** This question should be routed to the security/legal team, not answered by
GTM or support reps. See [[GTM Routing Rules]](../gtm/routing-rules.md#sensitive-questions).
```

A documented conflict is vastly more useful than a silently chosen winner, because it tells downstream agents and humans exactly why this topic requires escalation.

### 1.4 The Folder Structure IS the Taxonomy

There are no ontology graphs, no semantic layers, no knowledge base schemas to maintain. The taxonomy is the folder structure. The context graph is the backlinks between files.

When an agent discovers that "data retention questions" are connected to "GTM routing rules" and "security escalation policy", it writes a backlink in each relevant file pointing to the others. Over thousands of agent runs, these backlinks accumulate into a rich, navigable web of cross-domain understanding — built entirely from plain Markdown that any LLM can read and traverse.

### 1.5 Architecture Claims Are Durable; Status Claims Are Ephemeral

Different facts have different half-lives. The ECL must encode this explicitly:

| Claim type | Example | Half-life | Agent behaviour |
|------------|---------|-----------|-----------------|
| Architecture | "We use an event-driven microservices architecture" | Years | Write with high confidence; rarely re-verify |
| Process | "Incident severity is determined by the on-call lead" | Months | Write with medium confidence; schedule re-verification |
| Status | "Feature X is in beta for EU customers" | Days–weeks | Write with explicit timestamp; flag for immediate staleness check |
| People/roles | "Jane Smith owns the enterprise accounts" | Months | Write with person + date; flag for org-change triggers |
| Pricing | "Enterprise tier starts at $50k" | Weeks–months | Write with date; link to source |

Every ECL file should note the confidence level and last-verified date of its claims. The maintenance agent uses this to schedule re-verification tasks.

---

## 2. When to Build an ECL

An ECL is the right tool when your organisation faces one or more of these conditions:

- **Tribal knowledge problem:** Critical institutional knowledge lives in the heads of a few people and is lost when they leave.
- **Source conflict problem:** Multiple systems (Confluence, Notion, Slack, email, code comments) describe the same process differently, and no one knows which is authoritative.
- **Escalation routing problem:** Customer-facing and internal teams frequently give wrong answers because they cannot reliably distinguish "answer this" from "route this to an expert."
- **Cross-domain gap problem:** Understanding a customer situation requires combining knowledge from product, legal, finance, and engineering — but these teams never produce a unified view.
- **Onboarding cost problem:** New employees take months to become productive because the knowledge they need is scattered across dozens of systems.
- **AI agent grounding problem:** You are building LLM-powered agents but they hallucinate or give outdated answers because there is no reliable, structured knowledge base to ground them.

The ECL is **not** the right tool as a replacement for a proper search engine over raw documents. Use a tool like Glean, Elastic, or a vector database for that. The ECL sits above retrieval — it is the synthesised, conflict-resolved, citations-verified layer that search cannot produce.

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                                  │
│  Slack  Jira  GitHub  Confluence  Salesforce  Gong  Email  Code     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  search / read / fetch
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     WORKER AGENTS  (parallel)                        │
│                                                                      │
│  1. Pull latest ECL from git                                         │
│  2. Claim a task (file-based distributed lock)                       │
│  3. Execute: read sources → synthesise → write ECL files             │
│  4. Commit with inline citations                                      │
│  5. Push to main → release lock                                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  read / write
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     ECL GIT REPOSITORY                               │
│                                                                      │
│  meta/              ← seed files, system prompt, source guide        │
│  tasks/             ← YAML task queue (C-Compiler locking pattern)   │
│  domains/           ← one folder per knowledge domain                │
│    product/                                                          │
│    engineering/                                                      │
│    gtm/                                                              │
│    legal/                                                            │
│    people/                                                           │
│    ...                                                               │
│  sources/           ← cited source snapshots                         │
│  logs/              ← drift reports, error logs                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  read
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     QUERY INTERFACE                                   │
│                                                                      │
│  Claude Code / Claude API / Any LLM with file-system access          │
│  Reads ECL files directly → answers grounded in cited context        │
└─────────────────────────────────────────────────────────────────────┘
```

The architecture is intentionally minimal. There are no embedding models to fine-tune, no vector indices to maintain, no custom ML pipelines. The ECL's intelligence comes from the quality of the synthesis and citations written by agents, not from retrieval infrastructure.

---

## 4. Repository Structure

An LLM agent initialising a new ECL repository should create this exact structure. The names of domain folders will vary by company (see Step 1), but the top-level layout is fixed.

```
ecl-repo/
│
├── README.md                          ← This document (implementation guide)
│
├── meta/                              ← Seed files. Agents read these before every run.
│   ├── system-prompt.md               ← Agent identity, rules, and data model
│   ├── how-to-get-accurate-information.md   ← Living guide to source reliability
│   └── domain-index.md                ← Map of all knowledge domains and their owners
│
├── tasks/                             ← C-Compiler task queue
│   ├── YYYYMMDDTHHMMSS-domain-kind-slug.yaml    ← pending task
│   └── YYYYMMDDTHHMMSS-domain-kind-slug.LOCKED  ← active lock (agent ID inside)
│
├── domains/                           ← All synthesised knowledge lives here
│   ├── [domain-1]/
│   │   ├── README.md                  ← Domain overview, sub-topics, backlinks
│   │   ├── [topic].md                 ← One file per topic or concept
│   │   └── mapping-notes.md           ← Agent run log: timestamps, metrics, citations
│   ├── [domain-2]/
│   │   └── ...
│   └── .../
│
├── sources/                           ← Cited source snapshots (read-only)
│   ├── slack/                         ← Exported Slack thread snippets
│   ├── jira/                          ← Exported Jira tickets
│   ├── confluence/                    ← Exported Confluence pages
│   ├── code/                          ← Copied code references
│   ├── calls/                         ← Gong / meeting transcript excerpts
│   └── external/                      ← Competitor sites, public docs, etc.
│
├── logs/                              ← System health and diagnostics
│   ├── drift-YYYY-MM-DD.md            ← Drift detection reports
│   ├── backlinks-YYYYMMDDTHHMMSS.md   ← Backlink scan reports
│   └── ERROR-domain-kind-TS.md        ← Task failure reports
│
└── .ecl-mode                          ← "dev" or "prod" (created on first run)
```

### File naming conventions

| File type | Convention | Example |
|-----------|-----------|---------|
| Domain topic | `kebab-case.md` | `data-retention-policy.md` |
| Mapping notes | `mapping-notes.md` per domain | `domains/legal/mapping-notes.md` |
| Task file | `YYYYMMDDTHHMMSS-{domain}-{kind}-{slug}.yaml` | `20260315T101500-product-synthesise-a1b2c3d4.yaml` |
| Lock file | Same as task + `.LOCKED` | `20260315T101500-product-synthesise-a1b2c3d4.LOCKED` |
| Source snapshot | Descriptive name with date | `slack-sales-ops-2026-02-14.md` |
| Drift report | `drift-YYYY-MM-DD.md` | `drift-2026-03-21.md` |

---

## 5. The Two Foundational Patterns

The ECL implementation is built on exactly two design patterns. Understanding both fully before writing any code is essential.

### 5.1 ECL Pattern (Andy Chen)

The insight: synthesis and retrieval are different problems. Retrieval finds the best matching document. Synthesis builds a mental model — the reasoning framework an expert uses, not just the facts they cite.

The ECL encodes synthesis as Markdown files with inline citations in a Git repo. Every run by every agent appends to the repo's knowledge. Over thousands of agent runs, the repo accumulates:

- Conflict documentation (two sources disagree → ECL explains both sides and says who to ask)
- Routing rules (this question should go to legal, not engineering)
- Tribal knowledge (this PM always announces early → don't commit to customers)
- Cross-domain context (churn data in Salesforce is 30 days stale relative to the billing system)

**The taxonomy is the folder structure. The context graph is the backlinks.**

No heavyweight graph database. No ontology engine. No semantic layer. Plain folders and plain Markdown that LLMs read natively.

### 5.2 C-Compiler Pattern (Anthropic)

The insight: distributed agents need a mutex without a centralised coordinator. File-based locking via Git push-rejection provides exactly this.

The task lifecycle:

```
[1] Maintenance agent creates a task YAML file → commits to main
[2] Worker pulls main → finds unclaimed task → writes .LOCKED sidecar → pushes
    - If push succeeds: this worker owns the task
    - If push fails (another worker got there first): delete .LOCKED, try next task
[3] Worker executes the task (reads sources, synthesises, writes ECL files)
[4] Worker deletes both .yaml and .LOCKED → commits → pushes
```

This gives distributed mutual exclusion with no message broker, no database, no coordinator service. The git remote is the coordinator. This works for 1 worker or 20 workers running in parallel.

**Stale lock handling:** If a `.LOCKED` file is older than the lock TTL (default: 10 minutes), another worker may reclaim the task. This handles crashed workers.

---

## Step 1 — Discover Your Company's Knowledge Domains

**Who does this:** An LLM agent with access to the company's documentation, org chart, and a human point of contact.

**When:** Before any files are written. This step defines the folder structure for the entire ECL.

### What a domain is

A domain is a coherent area of company knowledge that has clear ownership, a defined scope, and identifiable source systems. It maps to one top-level folder under `domains/`.

Examples by company type:

| Company type | Typical domains |
|-------------|----------------|
| SaaS B2B | `product`, `engineering`, `gtm`, `legal`, `security`, `customer-success`, `finance`, `people` |
| E-commerce | `product-catalogue`, `supply-chain`, `logistics`, `customer-service`, `marketing`, `legal`, `finance` |
| Healthcare | `clinical-protocols`, `regulatory`, `patient-services`, `billing`, `it-systems`, `hr` |
| Manufacturing | `production`, `quality`, `supply-chain`, `safety`, `engineering`, `commercial`, `hr` |
| Financial services | `products`, `risk`, `compliance`, `operations`, `technology`, `client-services`, `hr` |

### How to discover domains

An agent should follow this process:

1. **Read the org chart.** Each major team or function is a candidate domain. If a team owns distinct knowledge that other teams query, it is a domain.

2. **Ask: what questions do people ask most often?** Group them. "How does feature X work?" → `product`. "What is our GDPR compliance status?" → `legal`/`compliance`. "How should I escalate this customer issue?" → `customer-success`.

3. **Ask: where does knowledge live that is currently lost or siloed?** Those gaps are high-priority domains.

4. **Start with 5–8 domains.** More than 12 is usually a sign the domains are too granular. Merge related areas. The ECL grows domains over time — it is easier to split a large domain later than to maintain too many thin ones from the start.

5. **Assign a domain owner** (a human) for each domain. They are the escalation point when an agent finds conflicting information or cannot verify a claim. Record this in `meta/domain-index.md`.

### Output of this step

Create `meta/domain-index.md` with entries like:

```markdown
# Domain Index

| Domain | Folder | Owner | Primary sources | Notes |
|--------|--------|-------|----------------|-------|
| Product | domains/product/ | @product-lead | Confluence, Jira, PRDs, release notes | Covers all product lines; separate sub-domains if >3 products |
| Engineering | domains/engineering/ | @engineering-lead | GitHub, runbooks, ADRs, on-call logs | Source code is ground truth for how things work |
| GTM | domains/gtm/ | @sales-ops | Salesforce, Gong, Slack #sales, battle cards | High staleness risk on competitive claims |
| Legal | domains/legal/ | @general-counsel | Legal docs, contracts, policy documents | Many topics are ROUTE-NOT-ANSWER |
| People | domains/people/ | @hr-lead | HRIS, org chart, Slack #general | Roles change frequently; always include last-verified date |
```

---

## Step 2 — Map Your Data Sources

**Who does this:** An LLM agent with a human providing access credentials or API tokens.

**When:** After domains are defined, before the system prompt is written.

### Why source mapping matters

The ECL's quality is entirely determined by the quality and coverage of its sources. An agent that cannot reach the primary source for a claim will either leave a gap or cite a secondary source — both are inferior. Source mapping ensures agents know exactly where to look for each type of information, what access they have, and what the limitations of each source are.

### Source categories

#### Internal structured sources (highest authority for operational truth)

| Source type | What it's best for | Typical access method | Staleness risk |
|------------|-------------------|----------------------|---------------|
| Source code / Git | How software actually behaves; feature flags; config | GitHub/GitLab API or local clone | Low — commits are authoritative |
| Production database schema | Data model ground truth | Read-only DB connection or ERD export | Low |
| ERP / CRM (e.g. Salesforce) | Customer records, deal data, pipeline | API or export | Medium — data entry lag |
| HRIS (e.g. Workday) | Org chart, headcount, roles | API or export | Medium |
| Ticketing (e.g. Jira) | Bug status, feature status, sprint state | REST API | Low for status; medium for priority |
| Billing system | Subscription, churn, revenue data | API or export | Low |

#### Internal unstructured sources (high authority for process and culture)

| Source type | What it's best for | Typical access method | Staleness risk |
|------------|-------------------|----------------------|---------------|
| Slack | Real process (vs. documented ideal); informal decisions; tribal knowledge | Slack API + search | High — messages age quickly |
| Gong / call recordings | What customers actually ask; how reps actually answer; deal dynamics | Gong API | High — each call is a snapshot |
| Email threads | Executive decisions; legal discussions; client commitments | Gmail/Outlook API | High |
| Confluence / Notion | Process documentation; onboarding; policies | REST API | Medium — docs often lag reality |
| Google Drive / SharePoint | Presentations, reports, OKRs, strategy docs | Drive API | Medium |

#### External sources (authority for competitive and market context)

| Source type | What it's best for | Access method | Staleness risk |
|------------|-------------------|--------------|---------------|
| Competitor websites | Pricing, feature claims, positioning | Web scrape / manual | High |
| G2 / Capterra / TrustRadius | Competitive perception, customer sentiment | Web scrape / API | Medium |
| LinkedIn | Competitor headcount, leadership changes | Manual / API | Medium |
| Industry analyst reports | Market positioning, buyer criteria | Manual download | Low (versioned) |
| Regulatory bodies | Compliance requirements | Web scrape / download | Low |

### Output of this step

For each domain defined in Step 1, document the sources in `meta/domain-index.md`:

```markdown
## GTM Domain — Source Map

### Primary sources (highest authority)
1. **Salesforce** — deal data, customer segments, churn records
   - Access: REST API via SALESFORCE_TOKEN env var
   - Limitation: 24–48h data entry lag from reps; field definitions vary by team

2. **Gong** — what customers actually ask and how reps answer
   - Access: Gong API via GONG_API_KEY env var
   - Limitation: only covers calls that were recorded; excludes email and chat

### Secondary sources
3. **Slack #sales** — informal decisions, competitive intel, escalation patterns
   - Access: Slack API via SLACK_BOT_TOKEN env var
   - Limitation: ephemeral; context collapses without thread; high noise

4. **Confluence /Sales Playbooks** — official process documentation
   - Access: Confluence REST API
   - Limitation: frequently out of date; treat as "ideal process", not "real process"

### Source authority hierarchy (highest → lowest)
1. Salesforce closed-won/lost data
2. Gong call recordings
3. Slack #sales (last 90 days)
4. Confluence playbooks
5. Marketing materials
```

---

## Step 3 — Define Source Authority

**Who does this:** LLM agent, informed by a human domain expert for each domain.

**When:** Immediately after source mapping. Before any content is written.

This step is critical and frequently skipped, which causes the ECL to accumulate quietly wrong content. Every domain needs an explicit answer to: *"When source A and source B disagree, which one wins?"*

### Universal source authority principles

These apply across all domains. Record them in `meta/system-prompt.md`:

**Principle 1: Operational reality beats documented ideal.**
Source code shows what the system *does*. A Confluence page shows what someone *intended*. When they conflict, the code is right and the doc is stale.

**Principle 2: Recent specific beats old general.**
A Jira ticket from last week describing a specific customer bug is more reliable than a two-year-old architecture document about how that area generally works.

**Principle 3: Corroboration threshold.**
Three independent sources agreeing on a claim crosses the threshold for high confidence. Five messages from the same Slack channel are one data point, not five — they share the same source bias.

**Principle 4: People signals have short half-lives.**
A statement like "Jane owns enterprise accounts" may have been true six months ago. Any people/role claim must include a last-verified date. Do not trust people claims older than 90 days without re-verification.

**Principle 5: Sensitive questions get documented, not answered.**
Some questions should never be answered directly — they should be routed to an expert. Common examples: legal liability, data privacy timelines, security architecture details, pricing exceptions, executive commitments. The ECL's job for these is to document *why* they are sensitive and *who* to route them to, not to provide the answer.

### Domain-specific authority tables

For each domain, produce an explicit table. This becomes the agent's decision rule when sources conflict:

```markdown
## Engineering Domain — Source Authority

| Source | Authority level | Best used for | Do NOT use for |
|--------|----------------|---------------|----------------|
| Source code (main branch) | PRIMARY | How a feature actually behaves; configuration; limits | Future roadmap; customer-facing descriptions |
| ADRs (Architecture Decision Records) | PRIMARY | Why a design choice was made | Current state if ADR is >1 year old without a revision |
| On-call runbooks | HIGH | Incident triage; known failure modes | Normal operation flows |
| Jira tickets (last 6 months) | HIGH | Current bug/feature status | Historical context |
| Engineering Confluence | MEDIUM | Intended design; onboarding context | Actual current behaviour |
| Slack #engineering (last 90 days) | MEDIUM | Emerging issues; informal decisions | Formal commitments |
| Slack #engineering (>90 days) | LOW | Historical context only | Anything current |
| Blog posts / talks by engineers | LOW | General philosophy | Specific product behaviour |
```

---

## Step 4 — Create the Meta Seed Files

**Who does this:** LLM agent.

**When:** Before any domain content is written. These files are read by every subsequent agent on every run.

The `meta/` directory is the ECL's self-guidance system. It does not contain company knowledge — it contains the rules and learned wisdom that agents use when building and maintaining company knowledge.

### 4.1 `meta/system-prompt.md`

This file encodes the agent's identity, operating rules, and data model. Every worker agent reads this at the start of every run. It should be written specifically for your company but follow this template:

```markdown
# ECL Agent — System Prompt

## Identity

You are the [Company Name] Enterprise Context Layer Agent.
Your purpose is to build and maintain the ECL: a synthesised, cited, conflict-aware
representation of how [Company Name] actually works across all domains.

You are not a search engine. You are not a Q&A bot. You build institutional memory.

## Non-negotiable rules

1. **Every factual claim must have an inline citation.** Format:
   [[Source description, date]](relative/path/to/source.md)
   If you cannot cite it, do not write it.

2. **Document conflicts, do not resolve them silently.** When two sources disagree,
   write a conflict note that names both sources, quotes the disagreement,
   and identifies who should resolve it.

3. **Every file must include a last-verified date.** Add this to the front matter
   or first line of every topic file:
   `> Last verified: YYYY-MM-DD | Agent: {AGENT_ID} | Confidence: high/medium/low`

4. **Sensitive questions get routing guidance, not answers.** If a topic is likely
   to be misused or requires expert judgment, write who to route to and why,
   not the answer itself.

5. **Every run must update the domain's mapping-notes.md.** Append your run metrics,
   sources consulted, and any anomalies found.

6. **Backlink related files.** If you write about topic A and discover it connects
   to topic B in another domain, add a backlink in both files.

## Source authority (highest → lowest)
[Paste the company-specific authority table from Step 3]

## Domain map
[Paste the domain index from Step 1]

## Available tools
[List the specific tools this agent has access to]
```

### 4.2 `meta/how-to-get-accurate-information.md`

This is the most important seed file in the entire ECL. Do not pre-fill it with invented wisdom. Seed it with only an empty template and let agents build it from real experience.

The initial seed content should be exactly this and nothing more:

```markdown
# How to Get Accurate Information

> This is a living document. Agents: every time you learn something about
> which sources are reliable, stale, or conflicting, add it here.
> Each entry should cite the specific experience that produced the insight.
> This file has no top-down author — it is built entirely bottom-up from
> accumulated agent experience.

## Source reliability observations

<!-- Add entries as you discover them. Format:
- [DATE] [AGENT_ID] [Domain] Observation about source reliability.
  Reason: what you found. Citation: [[source]](path). -->

## Sources that are frequently stale

<!-- Which sources consistently lag behind reality? -->

## Citation patterns that work

<!-- What citation formats have proven most useful for traceability? -->

## Sources that tend to conflict

<!-- Which source pairs frequently disagree? What does the conflict usually mean? -->

## Questions that should always be routed, never answered

<!-- Topics where direct answers have caused problems. -->

## Tool usage notes

<!-- How to use each tool effectively; known limitations; gotchas. -->
```

Over time, as agents run, they will fill this file with observations like: *"Confluence pages in the /sales-playbooks/ section are typically 6–9 months stale. Verify against recent Gong calls before citing."* These observations make every subsequent agent smarter without any changes to the code or prompt.

### 4.3 `meta/domain-index.md`

The completed domain index from Step 1. Used by agents to understand the full scope of the ECL, find where to write new content, and identify domain owners for escalation.

---

## Step 5 — Write the System Prompt

**Who does this:** LLM agent in collaboration with a human reviewer.

**When:** After Steps 1–4. This is the only step where human review before first use is strongly recommended.

The system prompt in `meta/system-prompt.md` controls agent behaviour across every run. A poorly written system prompt produces an ECL that accumulates confidently wrong or misleadingly cited content. Review it against these criteria:

### Checklist for a good system prompt

- [ ] **Identity is specific.** The prompt names the company and describes what the ECL is for, not just generically.
- [ ] **Citation format is explicit.** The exact Markdown format for inline citations is shown with an example.
- [ ] **Conflict handling is explicit.** The agent knows to write a conflict note rather than pick a winner.
- [ ] **Sensitive topics are enumerated.** A list of topic categories that always require routing, never direct answers (e.g., legal liability, security architecture, pricing exceptions).
- [ ] **Source authority table is included.** Copied from Step 3.
- [ ] **Staleness policy is specified.** What counts as a stale claim? How should agents flag it?
- [ ] **Mapping-notes requirement is explicit.** Agents must append to `mapping-notes.md` after every run.
- [ ] **Backlink requirement is explicit.** Agents must add cross-domain backlinks when they discover relationships.
- [ ] **Tool list is accurate.** Only tools the agent actually has access to are listed.
- [ ] **Out-of-scope is defined.** The prompt explains what the ECL does NOT contain (e.g., raw document storage, personal data, financial forecasts).

### What a confident system prompt does NOT contain

- Invented source reliability wisdom (this belongs in `how-to-get-accurate-information.md`, grown from experience)
- Hard-coded answers to specific questions (these belong in domain files, cited from sources)
- Business logic (this belongs in your application layer, not the ECL)

---

## Step 6 — Build the Task System

**Who does this:** Engineer or LLM agent writing the runner script.

**When:** Before any content agents run. The task system is the coordination layer.

### Task file structure

Tasks are YAML files in `tasks/`. A task represents one unit of work: synthesise a topic, verify a claim, scan for backlinks, check for drift, improve documentation, deduplicate content.

```yaml
# tasks/20260321T091500-product-synthesise-a3f9b2c1.yaml

domain: product
kind: synthesise          # synthesise | verify | backlink | drift | dedupe | docs
description: "Synthesise the feature flag inventory from source code and Jira"
created_at: "20260321T091500"
priority: 2               # 1 = most urgent
source_hints:             # Optional: specific sources agent should check
  - sources/code/feature-flags-export.md
  - sources/jira/feature-flag-tickets.md
metadata:
  target_file: domains/product/feature-flags.md
  last_synthesised: null  # null = never run before
```

### Task kinds

| Kind | Description | When to create |
|------|-------------|----------------|
| `synthesise` | Write or update an ECL topic file from sources | New topic; source has changed; file is missing |
| `verify` | Re-check all claims in an existing file against current sources | File is stale; source has been updated |
| `backlink` | Scan all domain files and add cross-domain references | Periodically; after major new content added |
| `drift` | Compare current sources against last-synthesised snapshot | Scheduled; after a known system change |
| `dedupe` | Identify duplicate or near-duplicate content across domains | Periodically; after large content additions |
| `docs` | AI review of mapping-notes to identify gaps and contradictions | Periodically; on request |
| `conflict-review` | Investigate a documented conflict and produce a resolution proposal | When a conflict note has been unresolved >30 days |

### Priority levels

| Priority | Meaning |
|----------|---------|
| 1 | Drift detected — immediate re-verification needed |
| 2 | Topic file missing entirely |
| 3 | Proof/verification file missing |
| 4 | Stale content (>7 days for status claims; >30 days for process claims) |
| 5 | Default / routine |
| 6 | Low-priority enrichment (backlinks, deduplication) |
| 7 | Scheduled maintenance (docs review, cross-domain scan) |

### Locking protocol

```python
def claim_task(repo):
    """
    Atomically claim the highest-priority unlocked task.

    1. Pull latest from git (get any newly created tasks)
    2. Sort tasks by (priority ASC, created_at ASC)
    3. For each task:
       a. If .LOCKED exists and is fresh (< LOCK_TTL): skip
       b. If .LOCKED exists and is stale (>= LOCK_TTL): reclaim (log warning)
       c. Write .LOCKED with {agent: AGENT_ID, locked_at: timestamp}
       d. Push to git — if push succeeds: this agent owns the task
          If push fails (race condition): delete local .LOCKED, try next task
    4. Return claimed task, or None if queue is empty
    """
```

The push is the atomic compare-and-swap. Git's remote will reject any push that is not a fast-forward — meaning if two agents try to claim the same task simultaneously, only one push succeeds. The other sees a `GitCommandError` and moves on to the next candidate.

---

## Step 7 — Implement the Worker Loop

**Who does this:** Engineer writing the runner script.

**When:** Alongside the task system implementation.

The worker loop is the core execution engine. It is intentionally simple:

```
LOOP:
  1. Pull latest ECL from git
  2. Claim a task (using the locking protocol in Step 6)
  3. If no task: sleep with exponential backoff; continue
  4. Execute the task
  5. Release the task (delete yaml + lock, commit, push)
  6. Sleep 2 seconds (rate limiting)
  7. Repeat
```

### Task execution: what an agent does

For a `synthesise` task, the execution is:

```
a. Read meta/system-prompt.md            ← understand rules and authority
b. Read meta/how-to-get-accurate-information.md  ← check for relevant source warnings
c. Read the target domain's README.md and mapping-notes.md  ← understand domain context
d. For each source in source_hints (and any other relevant sources discovered):
   - Fetch / read the source
   - Extract relevant claims
   - Note the source name, date, and access path
e. Synthesise: write a new or updated topic .md file with:
   - Front matter: last_verified, agent, confidence level
   - Body: synthesised claims with inline citations
   - Conflict notes where sources disagree
   - Backlinks to related topics in other domains
   - Routing notes for sensitive questions
f. Append a run record to the domain's mapping-notes.md:
   - Timestamp, agent ID
   - Sources consulted
   - Claims written / updated
   - Conflicts found
   - Confidence assessment
g. Commit all changed files with message: "synthesise({domain}): {description[:60]}"
```

### Exponential backoff when idle

When no tasks are available, the worker does not spin:

```python
idle_streak = 0
while True:
    task = claim_task(repo)
    if task is None:
        idle_streak += 1
        sleep(min(30 * idle_streak, 300))  # 30s, 60s, 90s, ... cap at 300s
        continue
    idle_streak = 0
    execute_task(task)
    release_task(task)
    sleep(2)
```

This means a single idle worker costs essentially nothing. A fleet of 20 idle workers sleeps and costs nothing. Work arrives via the maintenance agent creating new tasks.

### Error handling

Every task execution is wrapped in a try/except. On failure:

1. Log the full traceback to `logs/ERROR-{domain}-{kind}-{timestamp}.md`
2. Commit the error log
3. Call `release_task(task, success=False)` — the task YAML is deleted but the commit message is `task/failed:` rather than `task/done:`
4. Continue the loop — do not crash the worker

This prevents a single malformed source file from killing the entire worker fleet.

---

## Step 8 — Seed the Initial Content

**Who does this:** LLM agent, running as a `seed` command before the worker loop starts.

**When:** Once, during ECL initialisation.

Seeding does not write domain knowledge — that is the worker loop's job. Seeding copies existing artefacts into the repo structure so the workers have a starting point.

### What to seed

1. **Existing documentation.** Copy Confluence exports, Notion exports, policy PDFs (converted to Markdown), runbooks, onboarding docs into the relevant domain folders. These become the first source of truth for workers to synthesise from.

2. **Historical snapshots.** Export recent Slack threads, Jira tickets, Gong call summaries for the most business-critical topics. Save them as source snapshots in `sources/`.

3. **Org chart.** Export the current org chart (even as a simple list) into `domains/people/org-chart.md`. This is the starting point for the people domain.

4. **Known conflicts.** If you already know of specific areas where documentation conflicts with reality, create stub conflict notes immediately. Workers will investigate and fill these in.

5. **README files for each domain.** Create a `README.md` in each domain folder that describes the domain scope, links to sub-topics, lists the domain owner, and notes the primary sources. Workers will expand these over time.

### Seed commit message convention

```
feat(seed): import {N} files from {source_description}
```

The seed commit establishes the baseline for drift detection — subsequent runs compare against it.

---

## Step 9 — Run the Maintenance Agent

**Who does this:** A dedicated agent process running on a schedule (e.g., every 6 hours).

**When:** Continuously after the ECL is seeded. The maintenance agent is what keeps the ECL alive.

The maintenance agent does not write domain knowledge. It scans the ECL for health issues and creates tasks for worker agents to fix them.

### What the maintenance agent checks

```
For each domain:
  1. Does mapping-notes.md exist?
     NO  → create synthesise task (priority 2)

  2. How old is the newest entry in mapping-notes.md?
     > staleness threshold → create verify task (priority 4)

  3. Are there any topic files with last_verified older than their staleness SLA?
     (Status claims: 7 days; Process claims: 30 days; Architecture claims: 90 days)
     YES → create verify task for each stale file (priority 4)

  4. Are there any conflict notes unresolved for >30 days?
     YES → create conflict-review task (priority 3)

  5. Is there a pending backlink scan?
     NO → create backlink task (priority 7)

  6. Are there any domains with no topic files at all?
     YES → create synthesise task (priority 2)

Global checks:
  7. Are there any source files that have been updated since last synthesise?
     YES → create synthesise task for affected domain (priority 3)

  8. Run drift check: compare current source snapshots against last-synthesised state
     DRIFT detected → create synthesise task for drifted domains (priority 1)
```

### Staleness SLAs by claim type

Different claims need different maintenance intervals. Configure these for your company:

| Claim type | Default SLA | Reasoning |
|------------|------------|-----------|
| Pricing | 7 days | Changes frequently; expensive to get wrong |
| Product status (beta/GA/deprecated) | 7 days | Changes with each sprint |
| People / roles | 30 days | Org changes happen monthly |
| Process documentation | 30 days | Process evolves but not daily |
| Technical architecture | 90 days | Evolves slowly; refactors take time |
| Competitive landscape | 14 days | Competitors ship fast |
| Regulatory / compliance | 30 days | Regulations change slowly but consequences are high |
| Historical analysis | Never | Past events don't change |

---

## Step 10 — Build the Query Interface

**Who does this:** Engineer, after the ECL has initial content.

**When:** After at least one full worker run has populated the major domain files.

The ECL is optimised for *contribution* by agents. The query interface is what makes it useful for *consumption* by humans and downstream AI agents.

### Option A — Claude Code (simplest, recommended for getting started)

Simply give Claude Code access to the ECL repository. Its tool calls will navigate the folder structure, read relevant files, and compose answers grounded in the ECL content. This requires no additional infrastructure.

Prompt for querying:
```
You are answering questions using the Enterprise Context Layer in this repository.
Read meta/system-prompt.md first, then meta/domain-index.md to understand the structure.
For every claim in your answer, cite the ECL file you found it in.
If the ECL documents a conflict on this topic, present both sides.
If the ECL says to route this question, tell the user who to route to and why.
```

### Option B — Direct API query agent

A lightweight Python script that:
1. Takes a question as input
2. Uses semantic search or keyword search over the ECL's Markdown files to find relevant passages
3. Sends the relevant passages + question to the LLM
4. Returns the answer with ECL file citations

This is sufficient for most use cases and requires no vector database — simple `grep` or `ripgrep` over the Markdown files is fast enough for ECL repositories up to ~10,000 files.

### Option C — Full RAG pipeline

For large ECLs (>10,000 files) or high query volume, build a proper retrieval pipeline:
1. Embed all ECL files (chunk by section, not by arbitrary token count)
2. Store embeddings in a vector database (Pinecone, Weaviate, pgvector)
3. At query time: retrieve top-k relevant chunks → send to LLM with citation context
4. Re-embed incrementally on each ECL commit

This adds infrastructure complexity. Use Option A or B first and migrate only when the query volume or ECL size demands it.

### What the query interface must always do

Regardless of option chosen, the query interface must:

- **Pass through conflict notes.** If the ECL documents a conflict on the queried topic, the query interface must surface it, not hide it.
- **Pass through routing rules.** If the ECL says "route this to the security team", the query interface must say exactly that, not attempt to answer the underlying question.
- **Include ECL file paths in answers.** The user must be able to inspect the source of every claim.
- **Surface last-verified dates.** If the relevant ECL content is stale (past its SLA), the query interface should flag this prominently.

---

## 16. Citation Rules

Citations are the single most important quality control mechanism in the ECL. An agent writing without citations is worse than no agent at all — it produces confident-sounding content with no traceability, which is harder to challenge and correct than a blank file.

### Mandatory citation format

```markdown
Claim text [[Source: description, date/version]](relative/path/to/source.md).
```

Examples:

```markdown
The deletion pipeline runs every 72 hours on a rolling schedule
[[Engineering: deletion-pipeline runbook v2.1, 2026-02-10]](../../sources/code/deletion-runbook.md).

Product pricing for the Enterprise tier starts at $50,000/year
[[Sales: Pricing sheet Q1 2026]](../../sources/external/pricing-q1-2026.md),
with EMEA deals typically discounted 15–20%
[[Gong: Sales call #2847 with rep @james.r, 2026-02-14]](../../sources/calls/call-2847.md).

> ⚠️ **Conflict:** The above pricing conflicts with the Salesforce CPQ default of $45,000
> [[Salesforce: CPQ config export 2026-01-30]](../../sources/salesforce/cpq-export.md).
> This was unresolved as of 2026-03-15. Route pricing questions to @sales-ops.
```

### What counts as a citeable source

- A specific document, page, or file with a name and date
- A specific Slack message with channel, author, and date
- A specific Gong call with ID or date
- A specific Jira ticket by ticket number
- A specific commit hash or line number in source code
- A specific version of a policy document

### What does NOT count as a citeable source

- "According to engineering" (who? when? in what context?)
- "It is generally understood that..." (general understanding is not a source)
- "This is how it works" (show me where)
- A reference to a document without a date (the document may have changed)

### Three-source corroboration rule

For high-confidence claims (used in customer-facing scenarios, legal contexts, or security contexts), require three independent sources. Note the corroboration in the ECL file:

```markdown
> **Confidence: HIGH** — corroborated by three independent sources:
> 1. [[Source A]](path) — description
> 2. [[Source B]](path) — description
> 3. [[Source C]](path) — description
```

---

## 17. The Context Graph: Backlinks and Cross-References

The ECL does not use a graph database. The context graph is built from plain Markdown backlinks between files. This is simultaneously simpler, more maintainable, and more LLM-readable than any graph storage solution.

### What a backlink looks like

In `domains/gtm/customer-churn-handling.md`:
```markdown
## Related: Data Retention After Churn

When a customer churns, questions about data deletion timelines arise.
**Do not answer these directly.** See
[[Legal: Data Retention Policy]](../legal/data-retention-policy.md) for
the official policy, and [[Security: Escalation Routing]](../security/escalation-routing.md)
for the routing procedure.

> Note: The legal policy and engineering runbook conflict on this topic.
> See the conflict note in [[Legal: Data Retention Policy]](../legal/data-retention-policy.md#conflict-note).
```

### When agents add backlinks

An agent should add a backlink whenever it encounters:

- A topic in domain A that is meaningfully affected by a topic in domain B
- A question routing rule that crosses domain boundaries
- A conflict involving sources from different domains
- A process that spans multiple teams

### The backlink scan task

The backlink agent (`kind: backlink`) runs periodically. It reads summaries of all domain files and asks: *"What connections exist that are not yet captured as backlinks?"* It produces a report listing suggested backlinks and optionally writes them to the relevant files.

This is what produces the ECL's emergent meta-awareness: over many backlink scans, the repo builds a rich web of cross-domain context that mirrors how the company actually functions — with all its informal dependencies, shared concerns, and inter-team politics encoded in plain text.

---

## 18. Source Reliability and Conflict Resolution

### Why `how-to-get-accurate-information.md` is grown, not written

The source reliability guide starts empty (see Step 4.2) because invented reliability wisdom is worse than none. An agent that is told "Slack is unreliable" as a general rule will ignore useful Slack signals. An agent that has *observed* "the Slack #sales-engineering channel has consistently 2-week-stale competitive intel because the SE team posts after deals close, not during" has learned something specific and actionable.

The mechanism is: after every run, if an agent discovers something about source reliability (a Confluence page that was badly out of date, a Jira status that did not match the code, a Slack thread that turned out to be based on a misunderstanding), it appends to `how-to-get-accurate-information.md`. Over thousands of agent runs, this file accumulates the distilled reliability experience of the entire agent fleet.

### Conflict resolution protocol

When a conflict is found between sources:

**Step 1: Document immediately.** Write a conflict note in the relevant domain file. Include both conflicting claims, both source citations, the date the conflict was discovered, and the agent that found it.

**Step 2: Assess severity.**

| Severity | Definition | Action |
|----------|-----------|--------|
| Critical | Conflict creates legal, security, or financial risk | Create priority-1 task; flag domain owner immediately |
| High | Conflict affects customer-facing answers | Create priority-2 task; add routing rule |
| Medium | Conflict affects internal process | Create priority-4 task |
| Low | Minor inconsistency with no immediate impact | Log in mapping-notes; create priority-6 task |

**Step 3: Escalate to domain owner if unresolved after threshold.** For Critical and High severity: 24 hours. For Medium: 14 days. For Low: 30 days. Create a `conflict-review` task automatically.

**Step 4: When resolved, update with resolution and close the conflict note.** Do not delete the original conflict text — mark it resolved with the resolution and the date:

```markdown
> ~~**CONFLICT (found 2026-03-01):** Policy doc says 90 days; runbook says 120 days.~~
> **RESOLVED 2026-03-15 by @general-counsel:** Confirmed 90 days is correct.
> The runbook was not updated after the policy change in Q4 2025.
> Runbook update ticket: [[INFRA-4892]](../sources/jira/INFRA-4892.md).
```

---

## 19. Scaling and Multi-Agent Parallelism

The ECL architecture scales horizontally with no changes to the core design. More workers = faster ECL population. The git-based locking handles coordination transparently.

### Recommended agent counts by ECL state

| ECL phase | Recommended workers | Rationale |
|-----------|--------------------|-|
| Initial seeding | 1 | Keep git history clean; easy to debug |
| First population (domains 1–3) | 3–5 | Enough to parallelise without overloading sources |
| Full population (all domains) | 10–20 | Matches Andy Chen's production setup; saturates task queue efficiently |
| Maintenance mode | 2–5 | Most tasks are verify/backlink; lower parallelism needed |

### Token cost management

Running 20 agents continuously is expensive. Manage costs with:

1. **Task batching:** Group small related tasks into single larger ones. Synthesising three related topics in one task is cheaper than three separate tasks (system-prompt reading overhead paid once).

2. **Selective source reads:** The `source_hints` field in task YAML pre-selects which sources to read. A well-maintained task system avoids agents reading irrelevant sources.

3. **Staleness tiers:** Not everything needs daily verification. Use the staleness SLA table from Step 9 to avoid unnecessary re-verification of stable content.

4. **Maintenance mode cadence:** After initial population, switch to a much lower agent count and longer maintenance intervals (e.g., one backlink scan per week, verify runs only when a source changes).

### Running agents in parallel (Modal, AWS Lambda, local processes)

The ECL runner can be executed in any environment that has git access. Common approaches:

```bash
# Local: run N workers in parallel
for i in $(seq 1 5); do
  uv run ecl-runner.py worker --source-dir ./sources &
done

# Modal: run as parallel functions
# AWS Lambda: trigger from SQS queue of task notifications
# Kubernetes: deploy as a Deployment with N replicas
```

The runner does not care how it is hosted. All coordination happens through git.

---

## 20. Human-in-the-Loop Review

The ECL is built by machines but its accuracy depends on periodic human review, especially for high-stakes domains.

### When humans should review

- **Before any customer-facing deployment.** The ECL may be accurate for internal use but contain nuances inappropriate for direct customer answers.
- **After detecting a conflict involving a sensitive topic.** An agent that documents a conflict is correct to document it — but a human should verify the resolution.
- **For people and org-related content.** Agents can misread org signals. Role changes, reporting structures, and informal authority are better verified by HR or management.
- **For legal, security, and compliance content.** Agents write what the sources say. Whether that interpretation is legally correct requires expert judgment.
- **Periodically for the highest-traffic topics.** Whatever topics are queried most often should receive regular human spot-checks.

### Human review workflow

1. The maintenance agent creates `verify` tasks for content past its staleness SLA.
2. A worker agent re-verifies by re-reading the relevant sources and comparing to the ECL content.
3. If the worker finds the content still accurate, it updates the `last_verified` date.
4. If the worker finds a discrepancy, it writes an updated version with a conflict note.
5. The domain owner receives a notification (however your team communicates) to review the conflict note.
6. The domain owner confirms or corrects the content and records the resolution.

### Recording human corrections

When a human corrects or overrides an agent's synthesis, the correction must be committed to the repo with the human's identity and reasoning:

```markdown
> **Human review 2026-03-20 @jane.smith (Legal):** The above was incorrect.
> The 90-day figure applies to PII under GDPR. Non-PII data has a 180-day retention period.
> Corrected against Legal Policy v3.1 [[link]](../../sources/legal/data-retention-v3.1.md).
```

This teaches the next agent what the domain expert knows.

---

## 21. RBAC and Sensitive Content

The ECL in its basic form treats all content as uniformly accessible to all agents and users. For most organisations this is acceptable for internal use. For organisations with strict access controls, the following extensions apply.

### Sensitivity tiers

Define these for your organisation in `meta/system-prompt.md`:

| Tier | Examples | Access |
|------|---------|--------|
| Public | Product docs, public pricing, competitive intel | All agents and users |
| Internal | Process docs, org structure, internal pricing | Internal employees only |
| Restricted | Personnel files, legal matters in dispute, M&A activity | Named roles only |
| Confidential | Board materials, acquisition targets, personal performance data | Not in the ECL |

### Implementation approach

1. **Folder-based tiers.** Put restricted content in `domains/restricted/` and configure your query interface to check the user's role before allowing reads.

2. **Separate repositories.** For truly sensitive content, maintain a separate restricted ECL repo with its own worker agents that have appropriate access.

3. **Routing over content.** For the most sensitive topics, do not put the content in the ECL at all. Instead, put a routing note: *"This topic requires access to the restricted repo. Contact @security-team."*

4. **Agent credentials.** Different worker agents may have different levels of source access. A worker with Salesforce credentials can synthesise customer data. A worker without those credentials writes a note that the synthesis is incomplete pending source access.

---

## 22. Drift Detection and Staleness

### What drift means in the ECL

Drift occurs when the real world changes but the ECL does not. This is distinct from a conflict (two sources disagree now). Drift is: the ECL was correct when written but is now wrong because something changed.

Sources of drift:
- A product feature is deprecated but the ECL still documents it as active
- A policy document is updated but the ECL cites the old version
- A person changes roles but the ECL still lists them as the owner
- A competitor makes a significant product announcement that invalidates battle card content

### Drift detection implementation

For each domain, the drift check compares the current state of source documents against the state at the time of the most recent ECL synthesis:

```python
def run_drift_check(domain):
    # Read the domain's last_synthesised_at from mapping-notes.md
    # For each source file referenced in the domain's topic files:
    #   Check if the source file has changed since last_synthesised_at
    #   If yes: log as potential drift; create verify task
    # Generate a drift report in logs/drift-YYYY-MM-DD.md
```

For structured sources (databases, APIs), drift detection checks whether the data exported at seed time differs from the current export. For unstructured sources (Slack, documents), it checks modification timestamps.

### Drift severity

```markdown
🔴 Critical drift  — Conflict with a live customer commitment or legal obligation
🟠 High drift      — Feature status, pricing, or routing rule is outdated
🟡 Medium drift    — Process documentation lags new practice
🟢 Low drift       — Minor update; content still broadly accurate
⚪ No drift        — Content matches current sources
```

---

## 23. Adapting the ECL to Your Company

The steps above describe a generic ECL. Every company needs to adapt the following:

### What always stays the same

- Git as the single source of truth
- File-based task locking (C-Compiler pattern)
- Inline citations for every claim
- Conflict documentation (not resolution)
- `meta/` seed files, especially `how-to-get-accurate-information.md`
- Mapping-notes per domain

### What you customise per company

| Element | How to customise |
|---------|-----------------|
| Domain names and structure | Step 1 — discover from org chart and common questions |
| Source systems | Step 2 — map to your actual tooling (not every company has Gong) |
| Source authority table | Step 3 — must reflect your specific sources and their reliability |
| Staleness SLAs | Step 9 — adjust to your industry's pace of change |
| Sensitivity tiers | Step 21 — adjust to your compliance requirements |
| Agent tools | Match to your available integrations (Slack API, Jira API, GitHub API, etc.) |
| Routing rules | Highly company-specific — what is sensitive varies enormously |

### Questions to ask a human before building

Before writing a single ECL file, an LLM agent should ask a human representative these questions:

1. **What questions do people ask most often that get wrong answers today?** These are the highest-priority topics for initial synthesis.

2. **What knowledge would be lost if your top 3 most experienced people left tomorrow?** These are the tribal knowledge gaps the ECL must capture first.

3. **What topics have caused the most customer or legal problems when communicated incorrectly?** These become ROUTE-NOT-ANSWER entries.

4. **Which internal systems have the most up-to-date information?** This informs the source authority table.

5. **Which Slack channels or email threads contain the most useful institutional knowledge?** These are the richest unstructured sources.

6. **Who owns each major domain?** Required for the domain index and conflict escalation.

7. **What is the most common misunderstanding about your product or company?** The ECL should address this explicitly, with evidence.

---

## 24. What the ECL Is Not

Understanding what the ECL is not prevents a common class of implementation errors.

**The ECL is not a document store.** Raw source documents live in `sources/` as read-only snapshots. The ECL's domain files are syntheses of those sources, not the sources themselves. If someone asks "can you find the Q3 board presentation", they want a search tool, not the ECL.

**The ECL is not a search engine.** If someone asks "show me all Jira tickets from last week", they want Jira, not the ECL. The ECL synthesises conclusions from sources; it does not index or search raw source content.

**The ECL is not a real-time system.** ECL content has a staleness SLA measured in days, not seconds. For real-time operational data (current system status, live deal pipeline, active incident), query the source system directly.

**The ECL is not a policy enforcement system.** The ECL documents policies and routing rules. It does not enforce them. An ECL that says "route data deletion questions to the security team" is a guide for agents and humans — it does not technically prevent anyone from answering the question incorrectly.

**The ECL is not finished.** It is never finished. Companies change. Knowledge changes. The ECL's value compounds over time as the agent fleet runs more cycles, finds more conflicts, builds more backlinks, and accumulates more reliability wisdom in `how-to-get-accurate-information.md`. The target state is not "ECL is complete" — it is "ECL is continuously maintained."

---

## 25. Glossary

| Term | Definition |
|------|-----------|
| **ECL** | Enterprise Context Layer — the git repo that encodes how the company actually works |
| **Domain** | A coherent area of company knowledge with clear ownership; maps to one `domains/` subfolder |
| **Synthesis** | The act of reading multiple sources and writing a cited, conflict-aware summary — the core operation of the ECL |
| **Retrieval** | Finding the best matching document for a query; this is what search engines do; the ECL is not retrieval |
| **Inline citation** | A Markdown link within a claim that traces it to a primary source: `[[desc]](path)` |
| **Conflict note** | An ECL entry documenting that two sources disagree, naming both, and identifying who should resolve it |
| **Routing note** | An ECL entry that says "do not answer this directly; route to X because Y" |
| **Mapping notes** | The per-domain log appended after every agent run: timestamps, sources, claims written, conflicts found |
| **Backlink** | A Markdown cross-reference from one domain file to a related file in another domain; collectively forms the context graph |
| **Context graph** | The web of backlinks across all ECL files; the ECL's equivalent of a knowledge graph, implemented in plain text |
| **C-Compiler pattern** | Anthropic's file-based distributed locking mechanism: tasks are YAML files; claiming a task means writing a `.LOCKED` sidecar and pushing to git |
| **Task** | A YAML file in `tasks/` representing one unit of work for a worker agent |
| **Worker agent** | An LLM agent running the claim → execute → release loop |
| **Maintenance agent** | A separate agent that scans the ECL for staleness, gaps, and conflicts, and creates tasks for workers to fix them |
| **Staleness SLA** | The maximum age of an ECL claim before it should be re-verified; varies by claim type |
| **Drift** | The ECL was correct when written but the world has changed; different from a conflict |
| **Tribal knowledge** | Institutional memory held by individuals; not written down; the ECL's most valuable and most difficult target |
| **`how-to-get-accurate-information.md`** | The key meta seed file; starts empty; agents fill it with source-reliability observations from real experience |
| **Source authority** | The hierarchy determining which source wins when sources conflict; defined per domain in Step 3 |
| **Three-source corroboration** | The threshold for high-confidence claims: three independent sources must agree |
| **Lock TTL** | The time after which a `.LOCKED` file is considered stale and can be reclaimed; default 10 minutes |

---

## Quick-Start Checklist

An LLM agent starting a new ECL from scratch should complete these steps in order:

- [ ] **Step 1:** Interview a human; identify 5–8 knowledge domains; write `meta/domain-index.md`
- [ ] **Step 2:** Map data sources for each domain; document access methods and limitations
- [ ] **Step 3:** Write source authority tables; define the conflict resolution hierarchy
- [ ] **Step 4:** Create `meta/system-prompt.md` (complete), `meta/how-to-get-accurate-information.md` (empty template only)
- [ ] **Step 5:** Human review of system prompt
- [ ] **Step 6:** Implement task system (YAML schema, locking protocol, priority levels)
- [ ] **Step 7:** Implement worker loop (pull → claim → execute → release → sleep)
- [ ] **Step 8:** Seed existing docs; create domain READMEs; export source snapshots
- [ ] **Step 9:** Start maintenance agent; define staleness SLAs
- [ ] **Step 10:** Build query interface (start with Claude Code; migrate later if needed)
- [ ] **Ongoing:** Monitor `logs/` for errors and drift reports; review domain owner escalations; let `how-to-get-accurate-information.md` grow

---

*Based on Andy Chen's [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer) and Anthropic's [Building a C Compiler](https://www.anthropic.com/engineering/building-c-compiler).*
