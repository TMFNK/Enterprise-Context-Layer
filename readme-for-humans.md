# Enterprise Context Layer (ECL)

> Andy Chen built the first production ECL at Abnormal Security: a living Git repository that encodes how a company actually works, maintained by parallel LLM agents reading primary sources and writing cited synthesis documents. His essay: [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer).

---

## What This Is

The Enterprise Context Layer is not a database. It is not a search engine. It is not a chatbot.

It is a **living, self-maintaining Git repository** that encodes how your company actually works; the institutional memory, judgment calls, org behaviour, process reality, and cross-domain relationships that no retrieval system can derive on its own.

Traditional RAG (Retrieval-Augmented Generation) fetches documents. The ECL synthesises understanding. When a sales rep asks "how long do we keep customer data after churn?", a retrieval system returns the closest policy document. The ECL returns: _"Don't answer this yourself; route to the security team. Here is why, with three cited incidents where reps got this wrong."_

The difference is synthesis over retrieval. The ECL contains the reasoning frameworks your experts use, not just the raw facts they work with.

**This repository is a generic implementation template.** An LLM agent following this document should be able to build, seed, and run a complete ECL for any company, regardless of industry, size, or tooling.

---

## ⚡ Superpowers Integration

This implementation adds [Superpowers](https://github.com/obra/superpowers) which is Jesse Vincent's agentic skills framework as an extension to the ECL architecture. Superpowers gives structure and discipline to the agents that _build_ the ECL, the agents that _query_ it, and the developers who _maintain_ the tooling itself.

The two systems are complementary at every layer:

| Layer               | ECL's role                      | Superpowers' role                                                                                               |
| ------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Knowledge           | Stores _what_ the company knows | Gives agents a _how-to_ for building that knowledge                                                             |
| Agent behaviour     | Defines worker task lifecycle   | Provides skills (`brainstorming` → `writing-plans` → `subagent-driven-development`) that execute each task well |
| Development process | Describes the runner codebase   | Enforces TDD, systematic debugging, and subagent-driven development on the codebase                             |
| Process encoding    | Stores process docs as Markdown | Stores team workflows as reusable Superpowers skill files                                                       |

There are three distinct integration points, described fully in [Section 11 (Superpowers Integration)](#11-superpowers-integration):

1. **ECL as grounding for Superpowers agents**: before a Superpowers agent brainstorms or writes a plan, it reads the ECL to anchor its design in real architectural decisions, existing patterns, and team conventions.
2. **Skills as ECL domain content** team workflows, engineering conventions, and operating procedures are stored as Superpowers-compatible `SKILL.md` files inside the ECL's `domains/` tree, making process a first-class, maintained, citable artifact.
3. **Superpowers as the build methodology for ECL tooling** when implementing or extending the runner, worker loop, or query interface, use Superpowers' brainstorming → writing-plans → subagent-driven-development → requesting-code-review workflow.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [When to Build an ECL](#2-when-to-build-an-ecl)
3. [Architecture Overview](#3-architecture-overview)
4. [Repository Structure](#4-repository-structure)
5. [The Three Foundational Patterns](#5-the-three-foundational-patterns)
6. [Step 1: Discover Your Company's Knowledge Domains](#step-1--discover-your-companys-knowledge-domains)
7. [Step 2: Map Your Data Sources](#step-2--map-your-data-sources)
8. [Step 3: Define Source Authority](#step-3--define-source-authority)
9. [Step 4: Create the Meta Seed Files](#step-4--create-the-meta-seed-files)
10. [Step 5: Write the System Prompt](#step-5--write-the-system-prompt)
11. [Superpowers Integration](#11-superpowers-integration)
12. [Step 6: Build the Task System](#step-6--build-the-task-system)
13. [Step 7: Implement the Worker Loop](#step-7--implement-the-worker-loop)
14. [Step 8: Seed the Initial Content](#step-8--seed-the-initial-content)
15. [Step 9: Run the Maintenance Agent](#step-9--run-the-maintenance-agent)
16. [Step 10: Build the Query Interface](#step-10--build-the-query-interface)
17. [Citation Rules](#17-citation-rules)
18. [The Context Graph: Backlinks and Cross-References](#18-the-context-graph-backlinks-and-cross-references)
19. [Source Reliability and Conflict Resolution](#19-source-reliability-and-conflict-resolution)
20. [Scaling and Multi-Agent Parallelism](#20-scaling-and-multi-agent-parallelism)
21. [Human-in-the-Loop Review](#21-human-in-the-loop-review)
22. [RBAC and Sensitive Content](#22-rbac-and-sensitive-content)
23. [Drift Detection and Staleness](#23-drift-detection-and-staleness)
24. [Adapting the ECL to Your Company](#24-adapting-the-ecl-to-your-company)
25. [What the ECL Is Not](#25-what-the-ecl-is-not)
26. [Glossary](#26-glossary)

---

## 1. Core Concepts

Before building anything, an LLM agent implementing this system must internalise these five principles. They govern every decision about structure, content, and behaviour.

### 1.1 The Git Repo Is the Single Source of Truth

Every piece of knowledge the ECL contains lives as a file in this repository. Every update, correction, addition, and deletion is a git commit. This is not a style preference; it is the architectural foundation that makes everything else possible:

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

```
## Data Retention Policy: Conflict Note

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

When an agent discovers that "data retention questions" are connected to "GTM routing rules" and "security escalation policy", it writes a backlink in each relevant file pointing to the others. Over thousands of agent runs, these backlinks accumulate into a rich, navigable web of cross-domain understanding, all built entirely from plain Markdown that any LLM can read and traverse.

### 1.5 Architecture Claims Are Durable; Status Claims Are Ephemeral

Different facts have different half-lives. The ECL must encode this explicitly:

| Claim type   | Example                                               | Half-life    | Agent behaviour                                                   |
| ------------ | ----------------------------------------------------- | ------------ | ----------------------------------------------------------------- |
| Architecture | "We use an event-driven microservices architecture"   | Years        | Write with high confidence; rarely re-verify                      |
| Process      | "Incident severity is determined by the on-call lead" | Months       | Write with medium confidence; schedule re-verification            |
| Status       | "Feature X is in beta for EU customers"               | Days–weeks   | Write with explicit timestamp; flag for immediate staleness check |
| People/roles | "Jane Smith owns the enterprise accounts"             | Months       | Write with person + date; flag for org-change triggers            |
| Pricing      | "Enterprise tier starts at $50k"                      | Weeks–months | Write with date; link to source                                   |

Every ECL file should note the confidence level and last-verified date of its claims. The maintenance agent uses this to schedule re-verification tasks.

---

## 2. When to Build an ECL

An ECL is the right tool when your organisation faces one or more of these conditions:

- **Tribal knowledge problem:** Critical institutional knowledge lives in the heads of a few people and is lost when they leave.
- **Source conflict problem:** Multiple systems (Confluence, Notion, Slack, email, code comments) describe the same process differently, and no one knows which is authoritative.
- **Escalation routing problem:** Customer-facing and internal teams frequently give wrong answers because they cannot reliably distinguish "answer this" from "route this to an expert."
- **Cross-domain gap problem:** Understanding a customer situation requires combining knowledge from product, legal, finance, and engineering; but these teams never produce a unified view.
- **Onboarding cost problem:** New employees take months to become productive because the knowledge they need is scattered across dozens of systems.
- **AI agent grounding problem:** You are building LLM-powered agents but they hallucinate or give outdated answers because there is no reliable, structured knowledge base to ground them.

The ECL is **not** the right tool as a replacement for a proper search engine over raw documents. Use a tool like Glean, Elastic, or a vector database for that. The ECL sits above retrieval: it is the synthesised, conflict-resolved, citations-verified layer that search cannot produce.

---

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                    │
│  Slack  Jira  GitHub  Confluence  Salesforce  Gong  Email  Code          │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │  search / read / fetch
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              WORKER AGENTS  (parallel, Superpowers-enabled)               │
│                                                                           │
│  1. Pull latest ECL from git                                              │
│  2. Read relevant Superpowers SKILL.md from domains/skills/               │
│  3. Claim a task (file-based distributed lock)                            │
│  4. Execute: brainstorm approach → read sources → synthesise              │
│  5. Commit with inline citations + update mapping-notes                   │
│  6. Push to main → release lock                                           │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │  read / write
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                       ECL GIT REPOSITORY                                  │
│                                                                           │
│  meta/              ← seed files, system prompt, source guide             │
│  tasks/             ← YAML task queue (C-Compiler locking pattern)        │
│  domains/           ← one folder per knowledge domain                    │
│    product/                                                               │
│    engineering/                                                           │
│    gtm/                                                                   │
│    legal/                                                                 │
│    people/                                                                │
│    skills/          ← NEW: Superpowers SKILL.md files for team workflows  │
│    ...                                                                    │
│  sources/           ← cited source snapshots                             │
│  logs/              ← drift reports, error logs                          │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │  read
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               QUERY INTERFACE (Superpowers-grounded)                      │
│                                                                           │
│  Claude Code with Superpowers plugin installed                            │
│  → reads ECL before brainstorming or planning any change                  │
│  → answers grounded in cited ECL context                                  │
│  Claude API / any LLM with file-system access                             │
└──────────────────────────────────────────────────────────────────────────┘
```

The architecture is intentionally minimal. There are no embedding models to fine-tune, no vector indices to maintain, no custom ML pipelines. The ECL's intelligence comes from the quality of the synthesis and citations written by agents, not from retrieval infrastructure. Superpowers adds disciplined agent behaviour on top of this foundation without adding infrastructure.

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
│   ├── skills/                        ← NEW: Superpowers skill files for team workflows
│   │   ├── README.md                  ← Index of available skills and their triggers
│   │   ├── [workflow-name]/
│   │   │   └── SKILL.md               ← Superpowers-compatible skill definition
│   │   └── mapping-notes.md
│   └── .../
│
├── sources/                           ← Cited source snapshots (read-only)
│   ├── slack/
│   ├── jira/
│   ├── confluence/
│   ├── code/
│   ├── calls/
│   └── external/
│
├── logs/                              ← System health and diagnostics
│   ├── drift-YYYY-MM-DD.md
│   ├── backlinks-YYYYMMDDTHHMMSS.md
│   └── ERROR-domain-kind-TS.md
│
└── .ecl-mode                          ← "dev" or "prod" (created on first run)
```

### File naming conventions

| File type       | Convention                                    | Example                                              |
| --------------- | --------------------------------------------- | ---------------------------------------------------- |
| Domain topic    | `kebab-case.md`                               | `data-retention-policy.md`                           |
| Mapping notes   | `mapping-notes.md` per domain                 | `domains/legal/mapping-notes.md`                     |
| Task file       | `YYYYMMDDTHHMMSS-{domain}-{kind}-{slug}.yaml` | `20260315T101500-product-synthesise-a1b2c3d4.yaml`   |
| Lock file       | Same as task + `.LOCKED`                      | `20260315T101500-product-synthesise-a1b2c3d4.LOCKED` |
| Source snapshot | Descriptive name with date                    | `slack-sales-ops-2026-02-14.md`                      |
| Drift report    | `drift-YYYY-MM-DD.md`                         | `drift-2026-03-21.md`                                |
| Skill file      | `SKILL.md` inside named folder                | `domains/skills/incident-response/SKILL.md`          |

---

## 5. The Three Foundational Patterns

The ECL implementation is built on three design patterns. Understanding all three before writing any code is essential.

### 5.1 ECL Pattern (Andy Chen)

The insight: synthesis and retrieval are different problems. Retrieval finds the best matching document. Synthesis builds a mental model where the document is just one input. The ECL captures the reasoning framework an expert uses, not just the facts they cite.

The ECL encodes synthesis as Markdown files with inline citations in a Git repo. Every run by every agent appends to the repo's knowledge. Over thousands of agent runs, the repo accumulates:

- Conflict documentation (two sources disagree → ECL explains both sides and says who to ask)
- Routing rules (this question should go to legal, not engineering)
- Tribal knowledge (this PM always announces early → don't commit to customers)
- Cross-domain context (churn data in Salesforce is 30 days stale relative to the billing system)

**The taxonomy is the folder structure. The context graph is the backlinks.**

No heavyweight graph database. No ontology engine. No semantic layer. Plain folders and plain Markdown that LLMs read natively.

### 5.2 C-Compiler Pattern (Nicholas Carlini, Anthropic)

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

### 5.3 Superpowers Skills Pattern (Jesse Vincent)

The insight: agent quality is determined by the discipline of its process, not just the quality of its prompt. Superpowers encodes that discipline as composable `SKILL.md` files that agents load and follow before executing a task.

Applied to the ECL:

- A worker about to synthesise a complex cross-domain topic loads the `brainstorming` skill to surface hidden assumptions before writing.
- A worker implementing a new source connector loads `writing-plans` to produce a task list before touching code.
- A worker that has been running for hours and is about to push loads `requesting-code-review` to check its own output before committing.
- A worker fixing a broken drift-detection query loads `systematic-debugging` to follow a root-cause process rather than guessing.

Skills are **mandatory workflows, not suggestions**. The agent checks for a relevant skill before any non-trivial action.

The ECL also _stores_ skills as domain content, making team processes first-class, maintained, citable artifacts in the knowledge layer, not buried in wikis.

---

## Step 1: Discover Your Company's Knowledge Domains

**Who does this:** An LLM agent with access to the company's documentation, org chart, and a human point of contact.

**When:** Before any files are written. This step defines the folder structure for the entire ECL.

### What a domain is

A domain is a coherent area of company knowledge that has clear ownership, a defined scope, and identifiable source systems. It maps to one top-level folder under `domains/`.

Examples by company type:

| Company type  | Typical domains                                                                                                 |
| ------------- | --------------------------------------------------------------------------------------------------------------- |
| SaaS B2B      | `product`, `engineering`, `gtm`, `legal`, `security`, `customer-success`, `finance`, `people`, `skills`         |
| E-commerce    | `product-catalogue`, `supply-chain`, `logistics`, `customer-service`, `marketing`, `legal`, `finance`, `skills` |
| Healthcare    | `clinical-protocols`, `regulatory`, `patient-services`, `billing`, `it-systems`, `hr`, `skills`                 |
| Manufacturing | `production`, `quality`, `supply-chain`, `safety`, `engineering`, `commercial`, `hr`, `skills`                  |

> **Note:** Always add a `skills` domain. This is where team workflows are stored as Superpowers skill files. See [Section 11](#11-superpowers-integration) for details.

### How to discover domains

An agent should follow this process:

1. **Read the org chart.** Each major team or function is a candidate domain.
2. **Ask: what questions do people ask most often?** Group them.
3. **Ask: where does knowledge live that is currently lost or siloed?** Those gaps are high-priority domains.
4. **Start with 5–8 domains.** More than 12 is usually a sign the domains are too granular.
5. **Assign a domain owner** (a human) for each domain. Record this in `meta/domain-index.md`.

### Output of this step

Create `meta/domain-index.md` with entries like:

```markdown
# Domain Index

| Domain      | Folder               | Owner             | Primary sources                         | Notes                      |
| ----------- | -------------------- | ----------------- | --------------------------------------- | -------------------------- |
| Product     | domains/product/     | @product-lead     | Confluence, Jira, PRDs, release notes   |                            |
| Engineering | domains/engineering/ | @engineering-lead | GitHub, runbooks, ADRs, on-call logs    |                            |
| GTM         | domains/gtm/         | @sales-ops        | Salesforce, Gong, Slack #sales          | High staleness risk        |
| Legal       | domains/legal/       | @general-counsel  | Legal docs, contracts, policy documents | Many ROUTE-NOT-ANSWER      |
| People      | domains/people/      | @hr-lead          | HRIS, org chart, Slack #general         |                            |
| Skills      | domains/skills/      | @engineering-lead | Team retrospectives, runbooks, PRs      | Superpowers SKILL.md files |
```

---

## Step 2: Map Your Data Sources

**Who does this:** An LLM agent with a human providing access credentials or API tokens.

**When:** After domains are defined, before the system prompt is written.

### Why source mapping matters

The ECL's quality is entirely determined by the quality and coverage of its sources. Source mapping ensures agents know exactly where to look for each type of information, what access they have, and what the limitations of each source are.

### Source categories

#### Internal structured sources (highest authority for operational truth)

| Source type                 | What it's best for                                   | Typical access method                 | Staleness risk                      |
| --------------------------- | ---------------------------------------------------- | ------------------------------------- | ----------------------------------- |
| Source code / Git           | How software actually behaves; feature flags; config | GitHub/GitLab API or local clone      | Low                                 |
| Production database schema  | Data model ground truth                              | Read-only DB connection or ERD export | Low                                 |
| ERP / CRM (e.g. Salesforce) | Customer records, deal data, pipeline                | API or export                         | Medium                              |
| HRIS (e.g. Workday)         | Org chart, headcount, roles                          | API or export                         | Medium                              |
| Ticketing (e.g. Jira)       | Bug status, feature status, sprint state             | REST API                              | Low for status; medium for priority |
| Billing system              | Subscription, churn, revenue data                    | API or export                         | Low                                 |

#### Internal unstructured sources (high authority for process and culture)

| Source type               | What it's best for                                      | Typical access method | Staleness risk |
| ------------------------- | ------------------------------------------------------- | --------------------- | -------------- |
| Slack                     | Real process (vs. documented ideal); informal decisions | Slack API + search    | High           |
| Gong / call recordings    | What customers actually ask; how reps answer            | Gong API              | High           |
| Email threads             | Executive decisions; legal discussions                  | Gmail/Outlook API     | High           |
| Confluence / Notion       | Process documentation; onboarding; policies             | REST API              | Medium         |
| Google Drive / SharePoint | Presentations, reports, OKRs                            | Drive API             | Medium         |

#### External sources (authority for competitive and market context)

| Source type                 | What it's best for                         | Access method         | Staleness risk |
| --------------------------- | ------------------------------------------ | --------------------- | -------------- |
| Competitor websites         | Pricing, feature claims, positioning       | Web scrape / manual   | High           |
| G2 / Capterra / TrustRadius | Competitive perception, customer sentiment | Web scrape / API      | Medium         |
| LinkedIn                    | Competitor headcount, leadership changes   | Manual / API          | Medium         |
| Regulatory bodies           | Compliance requirements                    | Web scrape / download | Low            |

### Output of this step

For each domain defined in Step 1, document the sources in `meta/domain-index.md`, including:

- Primary sources with access method and limitations
- Source authority hierarchy (highest → lowest)
- Known staleness patterns

---

## Step 3: Define Source Authority

**Who does this:** LLM agent, informed by a human domain expert for each domain.

**When:** Immediately after source mapping. Before any content is written.

### Universal source authority principles

These apply across all domains. Record them in `meta/system-prompt.md`:

**Principle 1: Operational reality beats documented ideal.**
Source code shows what the system _does_. A Confluence page shows what someone _intended_.

**Principle 2: Recent specific beats old general.**
A Jira ticket from last week describing a specific bug is more reliable than a two-year-old architecture document.

**Principle 3: Corroboration threshold.**
Three independent sources agreeing on a claim crosses the threshold for high confidence.

**Principle 4: People signals have short half-lives.**
Any people/role claim must include a last-verified date. Do not trust people claims older than 90 days without re-verification.

**Principle 5: Sensitive questions get documented, not answered.**
Common examples: legal liability, data privacy timelines, security architecture details, pricing exceptions, executive commitments.

### Domain-specific authority tables

For each domain, produce an explicit table. Example for the engineering domain:

```markdown
## Engineering Domain: Source Authority

| Source                            | Authority level | Best used for                        | Do NOT use for                   |
| --------------------------------- | --------------- | ------------------------------------ | -------------------------------- |
| Source code (main branch)         | PRIMARY         | How a feature actually behaves       | Future roadmap                   |
| ADRs                              | PRIMARY         | Why a design choice was made         | Current state if ADR >1 year old |
| On-call runbooks                  | HIGH            | Incident triage; known failure modes | Normal operation flows           |
| Jira (last 6 months)              | HIGH            | Current bug/feature status           | Historical context               |
| Engineering Confluence            | MEDIUM          | Intended design; onboarding          | Actual current behaviour         |
| Slack #engineering (last 90 days) | MEDIUM          | Emerging issues; informal decisions  | Formal commitments               |
| Slack #engineering (>90 days)     | LOW             | Historical context only              | Anything current                 |
```

---

## Step 4: Create the Meta Seed Files

**Who does this:** LLM agent.

**When:** Before any domain content is written.

### 4.1 `meta/system-prompt.md`

This file encodes the agent's identity, operating rules, and data model. Every worker agent reads this at the start of every run. It should be written specifically for your company but follow this template:

```markdown
# ECL Agent System Prompt

## Identity

You are the [Company Name] Enterprise Context Layer Agent.
Your purpose is to build and maintain the ECL: a synthesised, cited, conflict-aware
representation of how [Company Name] actually works across all domains.

You are not a search engine. You are not a Q&A bot. You build institutional memory.

## Superpowers skills

Before any non-trivial task, check domains/skills/ for a relevant SKILL.md.
If a skill exists for your task type, read and follow it. Skills are mandatory
workflows, not suggestions. Specifically:

- Before synthesising a complex cross-domain topic → brainstorming skill
- Before writing or modifying any ECL tooling code → writing-plans + TDD skill
- Before pushing a large synthesise batch → requesting-code-review skill
- When a source returns unexpected or contradictory results → systematic-debugging skill

## Non-negotiable rules

1. Every factual claim must have an inline citation: [[Source description, date]](path)
2. Document conflicts; do not resolve them silently.
3. Every file must include a last-verified date.
4. Sensitive questions get routing guidance, not answers.
5. Every run must update the domain's mapping-notes.md.
6. Backlink related files across domains.

## Source authority

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
> This file has no top-down author; it is built entirely bottom-up from
> accumulated agent experience.

## Source reliability observations

<!-- Add entries as you discover them. Format:
- [DATE] [AGENT_ID] [Domain] Observation about source reliability.
  Reason: what you found. Citation: [[source]](path). -->

## Sources that are frequently stale

## Citation patterns that work

## Sources that tend to conflict

## Questions that should always be routed, never answered

## Tool usage notes

## Superpowers skill effectiveness notes

<!-- Which skills have been most useful for which task types?
     Which tasks went wrong before a skill was added? -->
```

> **Note:** The final section _Superpowers skill effectiveness notes_ is new. It allows agents to record which skills proved valuable for which ECL task types. Over time this builds the same bottom-up reliability wisdom for process as the rest of the file builds for sources.

### 4.3 `meta/domain-index.md`

The completed domain index from Step 1. Used by agents to understand the full scope of the ECL, find where to write new content, and identify domain owners for escalation.

---

## Step 5: Write the System Prompt

**Who does this:** LLM agent in collaboration with a human reviewer.

**When:** After Steps 1–4. This is the only step where human review before first use is strongly recommended.

### Checklist for a good system prompt

- **Identity is specific.** The prompt names the company and describes what the ECL is for.
- **Citation format is explicit.** The exact Markdown format for inline citations is shown with an example.
- **Conflict handling is explicit.** The agent knows to write a conflict note rather than pick a winner.
- **Superpowers skill lookup is explicit.** The agent knows to check `domains/skills/` before non-trivial tasks.
- **Sensitive topics are enumerated.** A list of topic categories that always require routing, never direct answers.
- **Source authority table is included.** Copied from Step 3.
- **Staleness policy is specified.**
- **Mapping-notes requirement is explicit.**
- **Backlink requirement is explicit.**
- **Tool list is accurate.**
- **Out-of-scope is defined.**

---

## 11. Superpowers Integration

This section describes the three integration points between Superpowers and the ECL in full detail.

### 11.1 Installing Superpowers

For agents using Claude Code, Superpowers is installed via the official plugin marketplace:

```bash
/plugin install superpowers@claude-plugins-official
```

For agents using Codex, Cursor, or OpenCode, follow the respective install instructions in the [Superpowers README](https://github.com/obra/superpowers).

Verify installation by asking Claude Code to help plan a feature, the `brainstorming` skill should activate automatically. The canonical Superpowers workflow skills are: `brainstorming`, `using-git-worktrees`, `writing-plans`, `subagent-driven-development`, `requesting-code-review`, and `finishing-a-development-branch`.

### 11.2 Integration Point 1: ECL as Agent Grounding

The most powerful integration is also the simplest: before a Superpowers agent brainstorms a new feature or writes an implementation plan, it reads the relevant ECL domain files to ground itself in the company's actual architecture, conventions, and constraints.

Without ECL grounding, an agent will:

- Propose patterns inconsistent with the existing architecture
- Suggest tools the company has already rejected
- Miss routing rules for sensitive questions
- Re-discover conflicts already documented

With ECL grounding, an agent:

- Designs within the actual architectural context
- Follows already-decided conventions
- Knows which questions to escalate
- Builds on existing conflict documentation rather than creating new ones

**How to implement ECL grounding in a Superpowers workflow:**

Add an ECL pre-read step to your `brainstorming` and `writing-plans` skill invocations. The recommended pattern is a preamble in the agent's task prompt:

```
Before brainstorming, read the following ECL files:
- meta/domain-index.md         ← understand the full knowledge map
- meta/system-prompt.md        ← understand the company's operating rules
- domains/[relevant-domain]/README.md  ← understand the specific domain context
- domains/[relevant-domain]/[topic].md ← read any directly relevant topic files

After reading, proceed with the brainstorming skill.
Any design decisions you make must be consistent with the ECL content.
Any design decisions that conflict with ECL content must create a conflict note.
```

This is analogous to what the ECL README already describes in Step 10 (Query Interface): the same principle applied to design-time agents, not just query-time agents.

### 11.3 Integration Point 2: Skills as ECL Domain Content

Team workflows, operating procedures, and engineering conventions can be stored as Superpowers-compatible `SKILL.md` files inside the ECL's `domains/skills/` domain. This has several important consequences:

1. **Process becomes a first-class artifact.** Workflow knowledge is no longer buried in wikis, it lives in the same versioned, cited, conflict-aware layer as all other company knowledge.
2. **Skills are maintained like ECL content.** The maintenance agent can create `verify` tasks for stale skills, just like stale domain files. Skills that lag behind process change get flagged.
3. **Skills have citations.** A skill for "how to run an incident retrospective" cites the Slack threads and retrospective docs that informed it, making the reasoning behind the process traceable.
4. **Skills can cross-reference ECL knowledge.** A `closing-a-deal` skill can link to `domains/gtm/pricing-authority.md` and `domains/legal/contract-routing.md`, grounding the workflow in current facts.

**Recommended initial skills to encode in `domains/skills/`:**

| Skill name                | Trigger                                | Links to ECL domains                |
| ------------------------- | -------------------------------------- | ----------------------------------- |
| `incident-response`       | Severity-1 alert                       | `engineering`, `security`, `people` |
| `closing-a-deal`          | Deal moves to negotiation stage        | `gtm`, `legal`, `finance`           |
| `customer-data-request`   | Customer asks for data export/deletion | `legal`, `security`, `engineering`  |
| `onboarding-new-engineer` | New hire joins engineering             | `engineering`, `people`, `product`  |
| `ecl-synthesise-domain`   | Agent begins synthesising a domain     | `meta`                              |
| `ecl-conflict-review`     | Agent encounters a conflict            | `meta`                              |

**Skill file format (compatible with Superpowers):**

```markdown
# Skill: [Skill Name]

> Last verified: YYYY-MM-DD | Owner: @[owner] | Confidence: high/medium/low
> Source: [[Retrospective notes, 2026-02]](../../sources/slack/retro-feb-2026.md)

## Trigger

This skill activates when: [describe trigger condition].

## Steps

1. **[Step name]:** [Precise instruction]. See [[ECL: relevant topic]](../../domains/[domain]/[topic].md).
2. ...

## Routing

If [condition], route to [person/team]. See [[Routing rules]](../../domains/gtm/routing-rules.md).

## Do not

- [Anti-pattern 1]
- [Anti-pattern 2]

## Related skills

- [[skill-name]](../[skill-name]/SKILL.md)
```

**Maintenance of skills:**

The maintenance agent should check `domains/skills/` with the same staleness rules as other domains. Skills with a `last_verified` date older than 30 days should trigger a `verify` task. Skills that reference ECL content that has changed (based on drift detection) should trigger an update task with priority 3.

### 11.4 Integration Point 3: Superpowers as the ECL Build Methodology

When implementing or extending the ECL runner, worker loop, query interface, or any tooling, use the full Superpowers development workflow:

```
1. brainstorming skill
   → Read ECL meta/ files first to understand constraints
   → Refine the feature through questions
   → Produce a validated design document

2. writing-plans skill
   → Break work into 2–5 minute tasks
   → Every task has exact file paths and verification steps
   → Plan emphasises TDD, YAGNI, and DRY

3. subagent-driven-development skill
   → Each task: write failing test first → watch it fail → write minimal code → pass
   → Two-stage review after each task: spec compliance, then code quality
   → Critical issues block progress

4. requesting-code-review (between task batches)
   → Reviews against plan
   → Critical issues block progress

5. finishing-a-development-branch
   → Verifies all tests pass
   → Presents merge/PR/keep/discard options
   → Cleans up worktree
```

This workflow is particularly valuable for ECL tooling because the ECL worker loop's distributed nature makes bugs difficult to reproduce. Enforcing RED-GREEN-REFACTOR from the start prevents the class of hard-to-detect race conditions that arise when locking logic is written without tests.

**Minimum test coverage targets for ECL tooling:**

| Component               | Minimum coverage | Critical paths                                    |
| ----------------------- | ---------------- | ------------------------------------------------- |
| Task locking / claiming | 90%              | Concurrent claim race, stale lock reclaim         |
| Git push / retry logic  | 85%              | Push rejection handling, exponential backoff      |
| Drift detection         | 80%              | Source change detection, staleness SLA evaluation |
| Citation validation     | 80%              | Inline citation format, path resolution           |
| Conflict detection      | 75%              | Multi-source comparison, severity classification  |

---

## Step 6: Build the Task System

**Who does this:** Engineer or LLM agent writing the runner script.

**When:** Before any content agents run. The task system is the coordination layer.

### Task file structure

Tasks are YAML files in `tasks/`. A task represents one unit of work.

```yaml
# tasks/20260321T091500-product-synthesise-a3f9b2c1.yaml

domain: product
kind: synthesise # synthesise | verify | backlink | drift | dedupe | docs | skill-verify
description: "Synthesise the feature flag inventory from source code and Jira"
created_at: "20260321T091500"
priority: 2 # 1 = most urgent
source_hints:
  - sources/code/feature-flags-export.md
  - sources/jira/feature-flag-tickets.md
metadata:
  target_file: domains/product/feature-flags.md
  last_synthesised: null
  superpowers_skill: null # optional: path to a skill the agent should read before executing
```

### Task kinds

| Kind              | Description                                               | When to create                                     |
| ----------------- | --------------------------------------------------------- | -------------------------------------------------- |
| `synthesise`      | Write or update an ECL topic file from sources            | New topic; source has changed; file is missing     |
| `verify`          | Re-check all claims in an existing file                   | File is stale; source has been updated             |
| `backlink`        | Scan all domain files and add cross-domain references     | Periodically; after major new content added        |
| `drift`           | Compare current sources against last-synthesised snapshot | Scheduled; after a known system change             |
| `dedupe`          | Identify duplicate content across domains                 | Periodically                                       |
| `docs`            | AI review of mapping-notes to identify gaps               | Periodically; on request                           |
| `conflict-review` | Investigate a documented conflict                         | When a conflict note has been unresolved >30 days  |
| `skill-verify`    | Re-verify a Superpowers skill file in `domains/skills/`   | Skill is stale; referenced ECL content has drifted |

### Priority levels

| Priority | Meaning                                                                |
| -------- | ---------------------------------------------------------------------- |
| 1        | Drift detected; immediate re-verification needed                       |
| 2        | Topic file or skill missing entirely                                   |
| 3        | Proof/verification file missing                                        |
| 4        | Stale content (>7 days for status claims; >30 days for process claims) |
| 5        | Default / routine                                                      |
| 6        | Low-priority enrichment (backlinks, deduplication)                     |
| 7        | Scheduled maintenance (docs review, cross-domain scan, skill-verify)   |

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
       d. Push to git; if push succeeds: this agent owns the task
          If push fails (race condition): delete local .LOCKED, try next task
    4. Return claimed task, or None if queue is empty
    """
```

---

## Step 7: Implement the Worker Loop

**Who does this:** Engineer writing the runner script.

**When:** Alongside the task system implementation. Use the full Superpowers workflow (`brainstorming` → `writing-plans` → `subagent-driven-development`) to implement this component.

The worker loop is the core execution engine:

```
LOOP:
  1. Pull latest ECL from git
  2. Claim a task (using the locking protocol in Step 6)
  3. If no task: sleep with exponential backoff; continue
  4. If task has a superpowers_skill reference, read that skill file before executing
  5. Execute the task
  6. Delete task YAML + lock file → commit → push (release)
  7. Sleep 2 seconds (rate limiting)
  8. Repeat
```

### Task execution: what an agent does

For a `synthesise` task, the execution is:

```
a. Read meta/system-prompt.md
b. Read meta/how-to-get-accurate-information.md
c. If the task has a superpowers_skill reference, read that skill file and follow it
d. Read the target domain's README.md and mapping-notes.md
e. For each source in source_hints:
   - Fetch / read the source
   - Extract relevant claims
   - Note the source name, date, and access path
f. Synthesise: write a new or updated topic .md file with:
   - Front matter: last_verified, agent, confidence level
   - Body: synthesised claims with inline citations
   - Conflict notes where sources disagree
   - Backlinks to related topics
   - Routing notes for sensitive questions
g. Append a run record to the domain's mapping-notes.md
h. Commit all changed files
```

For a `skill-verify` task, the execution is:

```
a. Read the skill file at domains/skills/[skill-name]/SKILL.md
b. For each ECL link referenced in the skill:
   - Read the referenced file
   - Check whether the guidance in the skill is still consistent with the ECL content
c. For each process step in the skill:
   - Check relevant sources (Slack, retrospectives, runbooks) for changes
d. If drift found: update the skill file, write a conflict note if needed
e. Update last_verified date
f. Append to domains/skills/mapping-notes.md
```

### Exponential backoff when idle

```python
idle_streak = 0
while True:
    task = claim_task(repo)
    if task is None:
        idle_streak += 1
        sleep(min(30 * idle_streak, 300))  # 30s, 60s, ..., cap at 300s
        continue
    idle_streak = 0
    execute_task(task)
    release_task(task)
    sleep(2)
```

### Error handling

Every task execution is wrapped in a try/except. On failure:

1. Log the full traceback to `logs/ERROR-{domain}-{kind}-{timestamp}.md`
2. Commit the error log
3. Release the task with `success=False`; commit message: `task/failed:`
4. Continue the loop; do not crash the worker

---

## Step 8: Seed the Initial Content

**Who does this:** LLM agent, running as a `seed` command before the worker loop starts.

**When:** Once, during ECL initialisation.

### What to seed

1. **Existing documentation.** Copy Confluence exports, policy PDFs (converted to Markdown), runbooks into the relevant domain folders.
2. **Historical snapshots.** Export recent Slack threads, Jira tickets, Gong call summaries for the most business-critical topics.
3. **Org chart.** Export into `domains/people/org-chart.md`.
4. **Known conflicts.** Create stub conflict notes for areas already known to have documentation conflicts.
5. **README files for each domain.** Workers will expand these over time.
6. **Initial Superpowers skills.** For each skill listed in [Section 11.3](#113-integration-point-2-skills-as-ecl-domain-content), create a stub `SKILL.md` with the known process steps. Workers will iterate on these over time.

### Seed commit message convention

```
feat(seed): import {N} files from {source_description}
```

---

## Step 9: Run the Maintenance Agent

**Who does this:** A dedicated agent process running on a schedule (e.g., every 6 hours).

**When:** Continuously after the ECL is seeded.

### What the maintenance agent checks

```
For each domain (including domains/skills/):
  1. Does mapping-notes.md exist?
     NO  → create synthesise task (priority 2)

  2. How old is the newest entry in mapping-notes.md?
     > staleness threshold → create verify task (priority 4)

  3. Are there any topic files with last_verified older than their staleness SLA?
     YES → create verify task for each stale file (priority 4)

  4. Are there any skill files with last_verified older than 30 days?
     YES → create skill-verify task (priority 7)

  5. Are there any conflict notes unresolved for >30 days?
     YES → create conflict-review task (priority 3)

  6. Is there a pending backlink scan?
     NO → create backlink task (priority 7)

  7. Are there any domains with no topic files at all?
     YES → create synthesise task (priority 2)

Global checks:
  8. Are there any source files updated since last synthesise?
     YES → create synthesise task for affected domain (priority 3)

  9. Run drift check: compare current sources against last-synthesised state
     DRIFT detected → create synthesise task (priority 1)

  10. Are there any skill files that reference ECL content that has drifted?
      YES → create skill-verify task (priority 4)
```

### Staleness SLAs by claim type

| Claim type                          | Default SLA | Reasoning                                  |
| ----------------------------------- | ----------- | ------------------------------------------ |
| Pricing                             | 7 days      | Changes frequently; expensive to get wrong |
| Product status (beta/GA/deprecated) | 7 days      | Changes with each sprint                   |
| People / roles                      | 30 days     | Org changes happen monthly                 |
| Process documentation               | 30 days     | Process evolves but not daily              |
| Superpowers skills                  | 30 days     | Team workflows evolve with practice        |
| Technical architecture              | 90 days     | Evolves slowly                             |
| Competitive landscape               | 14 days     | Competitors ship fast                      |
| Regulatory / compliance             | 30 days     | Consequences are high                      |
| Historical analysis                 | Never       | Past events don't change                   |

---

## Step 10: Build the Query Interface

**Who does this:** Engineer, after the ECL has initial content.

**When:** After at least one full worker run has populated the major domain files.

### Option A: Claude Code with Superpowers (recommended)

Give Claude Code access to the ECL repository and ensure Superpowers is installed. The combined setup gives you:

- **ECL for grounding:** The agent reads ECL domain files to anchor its answers in cited company knowledge.
- **Superpowers for discipline:** The agent follows skills like `brainstorming` before designing solutions, and `systematic-debugging` before diagnosing problems.

Prompt for querying:

```
You are answering questions using the Enterprise Context Layer in this repository.
You have Superpowers installed.

Before answering:
1. Read meta/system-prompt.md
2. Read meta/domain-index.md
3. Check domains/skills/ for any relevant skill for this task type

For every claim in your answer, cite the ECL file you found it in.
If the ECL documents a conflict on this topic, present both sides.
If the ECL says to route this question, tell the user who to route to and why.
If you are designing a solution, follow the brainstorming skill before producing output.
```

### Option B: Direct API query agent

A lightweight Python script that:

1. Takes a question as input
2. Uses `grep` or `ripgrep` over the ECL's Markdown files to find relevant passages
3. Sends relevant passages + question to the LLM
4. Returns the answer with ECL file citations

Sufficient for most use cases. No vector database required for ECL repositories up to ~10,000 files.

### Option C: Full RAG pipeline

For large ECLs (>10,000 files) or high query volume:

1. Embed all ECL files (chunk by section, not by arbitrary token count)
2. Store embeddings in a vector database (Pinecone, Weaviate, pgvector)
3. At query time: retrieve top-k chunks → send to LLM with citation context
4. Re-embed incrementally on each ECL commit

Use Option A or B first and migrate only when the query volume or ECL size demands it.

### What the query interface must always do

Regardless of option chosen:

- **Pass through conflict notes.** Surface them, never hide them.
- **Pass through routing rules.** If the ECL says "route this to the security team", say exactly that.
- **Include ECL file paths in answers.**
- **Surface last-verified dates.** Flag stale content prominently.
- **Surface relevant skills.** If a skill exists for the task being asked about, reference it.

---

## 17. Citation Rules

Citations are the single most important quality control mechanism in the ECL. An agent writing without citations produces confident-sounding content with no traceability. This is harder to challenge and correct than a blank file.

### Mandatory citation format

```
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
> Unresolved as of 2026-03-15. Route pricing questions to @sales-ops.
```

### What counts as a citeable source

- A specific document, page, or file with a name and date
- A specific Slack message with channel, author, and date
- A specific Gong call with ID or date
- A specific Jira ticket by ticket number
- A specific commit hash or line number in source code
- A specific Superpowers skill file (for skills that encode a cited process)

### Three-source corroboration rule

For high-confidence claims, require three independent sources. Note the corroboration:

```markdown
> **Confidence: HIGH** corroborated by three independent sources:
>
> 1. [[Source A]](path): description
> 2. [[Source B]](path): description
> 3. [[Source C]](path): description
```

---

## 18. The Context Graph: Backlinks and Cross-References

The ECL does not use a graph database. The context graph is built from plain Markdown backlinks between files.

### What a backlink looks like

In `domains/gtm/customer-churn-handling.md`:

```markdown
## Related: Data Retention After Churn

When a customer churns, questions about data deletion timelines arise.
**Do not answer these directly.** See
[[Legal: Data Retention Policy]](../legal/data-retention-policy.md) for
the official policy, and [[Security: Escalation Routing]](../security/escalation-routing.md)
for the routing procedure.

When handling these requests operationally, follow:
[[Skill: customer-data-request]](../skills/customer-data-request/SKILL.md)

> Note: The legal policy and engineering runbook conflict on this topic.
> See the conflict note in [[Legal: Data Retention Policy]](../legal/data-retention-policy.md#conflict-note).
```

### When agents add backlinks

An agent should add a backlink whenever it encounters:

- A topic in domain A that is meaningfully affected by a topic in domain B
- A process step that should follow a Superpowers skill
- A question routing rule that crosses domain boundaries
- A conflict involving sources from different domains

---

## 19. Source Reliability and Conflict Resolution

### Why `how-to-get-accurate-information.md` is grown, not written

The source reliability guide starts empty because invented reliability wisdom is worse than none. Over thousands of agent runs, this file accumulates the distilled experience of the entire agent fleet, including experience with which Superpowers skills worked well for which ECL task types.

### Conflict resolution protocol

When a conflict is found between sources:

**Step 1: Document immediately.** Write a conflict note with both conflicting claims, both source citations, the date discovered, and the agent ID.

**Step 2: Assess severity.**

| Severity | Definition                                          | Action                                         |
| -------- | --------------------------------------------------- | ---------------------------------------------- |
| Critical | Conflict creates legal, security, or financial risk | Priority-1 task; flag domain owner immediately |
| High     | Conflict affects customer-facing answers            | Priority-2 task; add routing rule              |
| Medium   | Conflict affects internal process                   | Priority-4 task                                |
| Low      | Minor inconsistency                                 | Log in mapping-notes; priority-6 task          |

**Step 3: Escalate to domain owner if unresolved after threshold.** Critical/High: 24 hours. Medium: 14 days. Low: 30 days.

**Step 4: When resolved, update with resolution and mark closed.** Do not delete the original conflict text:

```markdown
> ~~**CONFLICT (found 2026-03-01):** Policy doc says 90 days; runbook says 120 days.~~
> **RESOLVED 2026-03-15 by @general-counsel:** Confirmed 90 days is correct.
> The runbook was not updated after the policy change in Q4 2025.
> Runbook update ticket: [[INFRA-4892]](../sources/jira/INFRA-4892.md).
```

---

## 20. Scaling and Multi-Agent Parallelism

The ECL architecture scales horizontally with no changes to the core design.

### Recommended agent counts by ECL state

| ECL phase                      | Recommended workers | Rationale                                                             |
| ------------------------------ | ------------------- | --------------------------------------------------------------------- |
| Initial seeding                | 1                   | Keep git history clean; easy to debug                                 |
| First population (domains 1–3) | 3–5                 | Enough to parallelise without overloading sources                     |
| Full population (all domains)  | 10–20               | Matches Andy Chen's production setup                                  |
| Maintenance mode               | 2–5                 | Most tasks are verify/backlink/skill-verify; lower parallelism needed |

### Token cost management

1. **Task batching:** Group small related tasks into single larger ones.
2. **Selective source reads:** The `source_hints` field pre-selects which sources to read.
3. **Staleness tiers:** Not everything needs daily verification.
4. **Skill-driven efficiency:** Superpowers skills prevent agents from wasting tokens on ad-hoc approaches to tasks that have well-defined processes.

### Running agents in parallel

```bash
# Local: run N workers in parallel
for i in $(seq 1 5); do
  uv run ecl-runner.py worker --source-dir ./sources &
done

# Modal / AWS Lambda / Kubernetes: all work identically
# All coordination happens through git
```

---

## 21. Human-in-the-Loop Review

### When humans should review

- Before any customer-facing deployment
- After detecting a conflict involving a sensitive topic
- For people and org-related content
- For legal, security, and compliance content
- Periodically for the highest-traffic topics
- **When a Superpowers skill is first created or substantially revised** skills encode team processes; humans must confirm the encoding is accurate before agents follow it

### Human review workflow

1. The maintenance agent creates `verify` or `skill-verify` tasks for content past its staleness SLA.
2. A worker agent re-verifies by re-reading relevant sources.
3. If content is still accurate, it updates `last_verified`.
4. If a discrepancy is found, it writes an updated version with a conflict note.
5. The domain owner reviews the conflict note and confirms or corrects.

### Recording human corrections

```markdown
> **Human review 2026-03-20 @jane.smith (Legal):** The above was incorrect.
> The 90-day figure applies to PII under GDPR. Non-PII data has a 180-day retention period.
> Corrected against Legal Policy v3.1 [[link]](../../sources/legal/data-retention-v3.1.md).
```

---

## 22. RBAC and Sensitive Content

### Sensitivity tiers

| Tier         | Examples                                                        | Access                  |
| ------------ | --------------------------------------------------------------- | ----------------------- |
| Public       | Product docs, public pricing, competitive intel                 | All agents and users    |
| Internal     | Process docs, org structure, internal pricing                   | Internal employees only |
| Restricted   | Personnel files, legal matters in dispute, M&A activity         | Named roles only        |
| Confidential | Board materials, acquisition targets, personal performance data | Not in the ECL          |

### Implementation approach

1. **Folder-based tiers.** Restricted content in `domains/restricted/`. Enforce via repo permissions and query-interface access control.
2. **Separate repositories.** For truly sensitive content, maintain a separate restricted ECL repo. Note: cross-repo backlinks are not possible, so this breaks the context graph for those topics. Prefer routing notes for Confidential content over a separate repo unless compliance requires it.
3. **Routing over content.** For the most sensitive topics, store a routing note, a pointer to the content, not the content itself. This is usually the right answer for Confidential material.
4. **Skill access control.** Skills in `domains/skills/` that reference restricted content should themselves be in restricted subfolders.

---

## 23. Drift Detection and Staleness

### What drift means in the ECL

Drift occurs when the real world changes but the ECL does not. Sources of drift:

- A product feature is deprecated but the ECL still documents it as active
- A policy document is updated but the ECL cites the old version
- A person changes roles but the ECL still lists them as the owner
- A Superpowers skill describes a process that the team has since changed
- A competitor makes a significant product announcement that invalidates battle card content

### Drift severity

```
🔴 Critical drift: Conflict with a live customer commitment or legal obligation
🟠 High drift: Feature status, pricing, routing rule, or active skill is outdated
🟡 Medium drift: Process documentation or skill lags new practice
🟢 Low drift: Minor update; content still broadly accurate
⚪ No drift: Content matches current sources
```

---

## 24. Adapting the ECL to Your Company

### What always stays the same

- Git as the single source of truth
- File-based task locking (C-Compiler pattern)
- Inline citations for every claim
- Conflict documentation (not resolution)
- `meta/` seed files, especially `how-to-get-accurate-information.md`
- Mapping-notes per domain
- `domains/skills/` for Superpowers skill files

### What you customise per company

| Element                    | How to customise                                       |
| -------------------------- | ------------------------------------------------------ |
| Domain names and structure | Step 1; discover from org chart and common questions   |
| Source systems             | Step 2; map to your actual tooling                     |
| Source authority table     | Step 3; must reflect your specific sources             |
| Staleness SLAs             | Step 9; adjust to your industry's pace of change       |
| Sensitivity tiers          | Step 22; adjust to your compliance requirements        |
| Agent tools                | Match to your available integrations                   |
| Superpowers skills         | Step 11.3; encode workflows most critical to your team |
| Routing rules              | Highly company-specific                                |

### Questions to ask a human before building

1. **What questions do people ask most often that get wrong answers today?**
2. **What knowledge would be lost if your top 3 most experienced people left tomorrow?**
3. **What topics have caused the most customer or legal problems when communicated incorrectly?**
4. **Which internal systems have the most up-to-date information?**
5. **Which Slack channels contain the most useful institutional knowledge?**
6. **Who owns each major domain?**
7. **What is the most common misunderstanding about your product or company?**
8. **Which team workflows are most often done inconsistently or incorrectly?** ← These become initial Superpowers skills.

---

## 25. What the ECL Is Not

**The ECL is not a document store.** Raw source documents live in `sources/` as read-only snapshots.

**The ECL is not a search engine.** The ECL synthesises conclusions from sources; it does not index or search raw source content.

**The ECL is not a real-time system.** ECL content has a staleness SLA measured in days, not seconds.

**The ECL is not a policy enforcement system.** The ECL documents policies and routing rules. It does not enforce them.

**The ECL is not a code execution engine.** Superpowers provides agent execution discipline; the ECL provides the knowledge those agents run on. They are not the same system, even when used together.

**The ECL is not finished.** It is never finished. The target state is not "ECL is complete": it is "ECL is continuously maintained."

---

## 26. Glossary

| Term                                     | Definition                                                                                                                              |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **ECL**                                  | Enterprise Context Layer; the git repo that encodes how the company actually works                                                      |
| **Domain**                               | A coherent area of company knowledge with clear ownership; maps to one `domains/` subfolder                                             |
| **Synthesis**                            | Reading multiple sources and writing a cited, conflict-aware summary; the core operation of the ECL                                     |
| **Retrieval**                            | Finding the best matching document for a query; what search engines do; the ECL is not retrieval                                        |
| **Inline citation**                      | A Markdown link within a claim that traces it to a primary source: `[[desc]](path)`                                                     |
| **Conflict note**                        | An ECL entry documenting that two sources disagree, naming both, and identifying who should resolve it                                  |
| **Routing note**                         | An ECL entry that says "do not answer this directly; route to X because Y"                                                              |
| **Mapping notes**                        | The per-domain log appended after every agent run: timestamps, sources, claims written, conflicts found                                 |
| **Backlink**                             | A Markdown cross-reference from one domain file to a related file in another domain                                                     |
| **Context graph**                        | The web of backlinks across all ECL files; the ECL's plain-text knowledge graph                                                         |
| **C-Compiler pattern**                   | Anthropic's file-based distributed locking: tasks are YAML files; claiming one means writing a `.LOCKED` sidecar and pushing to git     |
| **Task**                                 | A YAML file in `tasks/` representing one unit of work for a worker agent                                                                |
| **Worker agent**                         | An LLM agent running the claim → execute → release loop                                                                                 |
| **Maintenance agent**                    | A separate agent that scans the ECL for staleness, gaps, and conflicts, and creates tasks for workers to fix them                       |
| **Superpowers**                          | An agentic skills framework by Jesse Vincent; provides composable `SKILL.md` files that agents load and follow                          |
| **Skill (Superpowers)**                  | A `SKILL.md` file describing a mandatory workflow for a specific task type; agents check for relevant skills before non-trivial actions |
| **Skill domain**                         | The `domains/skills/` folder in the ECL; stores team workflows as Superpowers-compatible SKILL.md files                                 |
| **ECL grounding**                        | Reading ECL domain files before a Superpowers agent brainstorms or plans, to anchor design in real architecture and conventions         |
| **Staleness SLA**                        | The maximum age of an ECL claim before it should be re-verified; varies by claim type                                                   |
| **Drift**                                | The ECL was correct when written but the world has changed; different from a conflict                                                   |
| **Tribal knowledge**                     | Institutional memory held by individuals; not written down; the ECL's most valuable and most difficult target                           |
| **`how-to-get-accurate-information.md`** | The key meta seed file; starts empty; agents fill it with source-reliability and skill-effectiveness observations from real experience  |
| **Source authority**                     | The hierarchy determining which source wins when sources conflict; defined per domain in Step 3                                         |
| **Three-source corroboration**           | The threshold for high-confidence claims: three independent sources must agree                                                          |
| **Lock TTL**                             | The time after which a `.LOCKED` file is considered stale and can be reclaimed; default 10 minutes                                      |

---

## Quick-Start Checklist

An LLM agent starting a new ECL from scratch should complete these steps in order:

- **Step 1:** Interview a human; identify 5–8 knowledge domains plus a `skills` domain; write `meta/domain-index.md`
- **Step 2:** Map data sources for each domain; document access methods and limitations
- **Step 3:** Write source authority tables; define the conflict resolution hierarchy
- **Step 4:** Create `meta/system-prompt.md` (complete, including Superpowers skill lookup instructions), `meta/how-to-get-accurate-information.md` (empty template only, including the skills-effectiveness section)
- **Step 5:** Human review of system prompt
- **Superpowers:** Install Superpowers:
- **Step 11.3:** Create stub `SKILL.md` files for the team's highest-priority workflows in `domains/skills/`
- **Step 6:** Implement task system (YAML schema, locking protocol, priority levels, `skill-verify` task kind), use Superpowers' + workflow
- **Step 7:** Implement worker loop (pull → read skill → claim → execute → release → sleep), use Superpowers' skill
- **Step 8:** Seed existing docs, domain READMEs, source snapshots, and initial skill stubs
- **Step 9:** Start maintenance agent; define staleness SLAs including 30-day SLA for skills
- **Step 10:** Build query interface (start with Claude Code + Superpowers; migrate later if needed)
- **Ongoing:** Monitor `logs/` for errors and drift reports; let `how-to-get-accurate-information.md` grow; review skill files after team workflow changes; update skills when ECL content they reference drifts

---

_Based on Andy Chen's [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer), Nicholas Carlini's [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler), and Jesse Vincent's [Superpowers](https://github.com/obra/superpowers). The 10-step build process, task schema, staleness SLA tables, Superpowers integration, and repository structure are this project's extrapolation from those sources._
