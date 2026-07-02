# Enterprise Context Layer (ECL)

A self-maintaining Git repository that encodes how your company actually works. Not a search engine. Not a chatbot. A living, cited, conflict-aware institutional memory built and kept current by LLM agents running continuously against your real data sources.

Andy Chen (@andychen32), who built the first production ECL at Abnormal Security, described the insight simply: retrieval and synthesis are different problems. Glean finds the best matching document. The ECL builds the reasoning framework an expert uses, and tells you which questions should never be answered at all.

---

## What Problem This Solves

Every company has the same problem: critical knowledge is scattered. The answer to _"how long do we keep customer data after churn?"_ lives in four different places: a legal policy doc, an engineering runbook, a Slack thread from eight months ago, and the head of one senior engineer. They don't all agree. Nobody knows which is right.

A retrieval system (search, RAG) returns the closest document. The ECL returns: _"These two sources conflict. Here's both positions, cited. Route this to the security team where they've had this debate before. Here's why, with three incidents where reps got it wrong."_

The difference is **synthesis over retrieval**. The ECL encodes the reasoning frameworks your experts use, not just the raw facts they work with.

---

## How It Works

The ECL is a Git repository of Markdown files. LLM agents read your company's source systems (Slack, Jira, Confluence, source code, Gong calls, Salesforce) and write synthesised, cited knowledge files into the repo. Every claim has an inline citation. Every conflict between sources is documented explicitly rather than silently resolved. The git history is the audit trail.

Three design patterns make this work:

**ECL Pattern** (Andy Chen, [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer)): Knowledge is synthesised into Markdown files organised by domain folder. The folder structure is the taxonomy. Backlinks between files are the knowledge graph. No ontology engine, no graph database. Just folders and plain text that any LLM reads natively.

**C-Compiler Pattern** (Nicholas Carlini, Anthropic, [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)): Multiple agents coordinate without a central broker. Each task is a YAML file in `tasks/`. An agent claims a task by writing a `.LOCKED` sidecar and pushing to git. If the push is rejected, another agent got there first. Git's push-rejection is the distributed mutex. No message broker, no database, no coordinator.

**Lean Skills Pattern** (tw93, [Waza](https://github.com/tw93/Waza)): Agent quality comes from process discipline, not just prompt quality, but discipline should be scoped to exactly one workflow at a time. Team workflows are stored as small, single-purpose `SKILL.md` files, one skill per folder, with plain frontmatter naming its trigger phrases. An agent loads a skill only when a task matches, follows it as the mandatory workflow for that task, and stops; skills don't chain into an automatic multi-step pipeline. Team workflows are themselves stored as skill files inside the ECL, making process a first-class, versioned, citable artifact without the overhead of a heavier framework.

---

## The Three Documents in This Project

This project contains three documents, each written for a different reader:

| File                        | Written for                           | Purpose                                                                                                           |
| --------------------------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **`README.md`** (this file) | Humans                                | Overview, how to run, what each file does                                                                         |
| **`readme-for-agents.md`**  | LLM agents (Claude Code, Codex, etc.) | Complete build specification with runnable code, the blueprint an agent follows to construct the ECL from scratch |
| **`readme-for-humans.md`**  | Engineers and architects              | Deep conceptual background, design rationale, and the thinking behind each of the three patterns                  |

**If you want to understand the system:** read `readme-for-humans.md`.

**If you want to build the system using an LLM agent:** give `readme-for-agents.md` to Claude Code or Codex and tell it to follow the Quick-Start Checklist. The document contains complete, runnable Python for every component.

**If you want to deploy and operate a running ECL:** continue reading this file.

---

## Repository Structure (Once Built)

Once an agent has constructed the ECL for your company, the repository looks like this:

```
ecl-repo/
│
├── README.md                          ← This file
├── readme-for-agents.md               ← Build specification for LLM agents
├── readme-for-humans.md               ← Conceptual background and design rationale
│
├── meta/                              ← The three files every agent reads before every run
│   ├── system-prompt.md               ← Agent identity, rules, citation format, source authority
│   ├── how-to-get-accurate-information.md   ← Living source reliability guide (grows from agent experience)
│   └── domain-index.md                ← Map of all knowledge domains, owners, and primary sources
│
├── tasks/                             ← The task queue
│   ├── 20260321T091500-product-synthesise-a3f9b2c1.yaml    ← pending task
│   └── 20260321T091500-product-synthesise-a3f9b2c1.LOCKED  ← claimed by an agent
│
├── domains/                           ← All synthesised knowledge
│   ├── product/
│   │   ├── README.md                  ← Domain overview and sub-topic index
│   │   ├── feature-flags.md           ← A synthesised topic file with inline citations
│   │   ├── pricing.md
│   │   └── mapping-notes.md           ← Log of every agent run in this domain
│   ├── engineering/
│   ├── gtm/
│   ├── legal/
│   ├── people/
│   ├── security/
│   └── skills/                        ← Lean SKILL.md files for team workflows (see Lean Skills Pattern)
│       ├── incident-response/
│       │   └── SKILL.md
│       ├── closing-a-deal/
│       │   └── SKILL.md
│       └── mapping-notes.md
│
├── sources/                           ← Cited source snapshots (read-only; never edit directly)
│   ├── slack/
│   ├── jira/
│   ├── confluence/
│   ├── code/
│   ├── calls/
│   └── external/
│
└── logs/
    ├── drift-2026-03-21.md            ← Drift detection reports
    └── ERROR-product-synthesise-....md ← Agent error logs
```

---

## Key Files Explained

### `meta/system-prompt.md`

The identity and rules file that every worker agent reads at the start of every run. It contains the company name and context, the mandatory citation format, conflict handling rules, the source authority hierarchy (which source wins when two disagree), a list of sensitive topics that must be routed rather than answered, and the list of tools the agent has access to. You write this once during setup; agents follow it on every run.

### `meta/how-to-get-accurate-information.md`

The most important file in the ECL, and it starts completely empty. You do not pre-fill it. Agents add to it over time as they discover which sources are stale, which tool calls return unreliable results, and which questions should always be escalated. After thousands of agent runs it becomes a dense, experience-grounded guide to navigating your company's specific information landscape. Invented wisdom is worse than none; real experience accumulated by agents is the point.

### `meta/domain-index.md`

A table mapping every knowledge domain to its folder, its human owner, and its primary source systems. Agents use this to decide where to write new content and who to flag for conflict resolution.

### `domains/{domain}/mapping-notes.md`

The running log for a domain. Every time an agent runs a task in a domain, it appends a record: what it read, what it wrote, what conflicts it found, what citations it used. This is how you see what the agent fleet has been doing and how knowledge in a domain has evolved.

### `domains/{domain}/{topic}.md`

A synthesised topic file. Every factual claim has an inline citation tracing it to a primary source. Conflicting sources are documented explicitly with a conflict note. Sensitive questions have routing notes instead of answers. Each file has a `last_verified` date in its front matter so the maintenance agent knows when to re-verify it.

### `domains/skills/{name}/SKILL.md`

A lean workflow file in the spirit of Waza's skill format: YAML frontmatter naming the skill and its trigger phrases, followed by a single concise playbook for one workflow. When an agent encounters a task type that has a corresponding skill (incident response, closing a deal, customer data requests), it loads the skill file and follows it as a mandatory workflow for that task, it is not a suggestion. Skills stay narrow and don't chain automatically into a larger pipeline. Skills are maintained like any other ECL content: they have citations, `last_verified` dates, and get flagged for re-verification when the processes they describe change.

### `tasks/*.yaml` and `tasks/*.LOCKED`

The distributed task queue. The maintenance agent creates YAML task files. Worker agents claim them by writing a `.LOCKED` sidecar and pushing to git, if the push is rejected, another agent won the race and this agent moves on. When a task is complete, both files are deleted in a single commit. No message broker, no database, no coordinator, git's push rejection is the mutex.

### `sources/`

Read-only snapshots of the raw source material that domain files were synthesised from. When a topic file cites a Slack thread or a Jira ticket, the snapshot lives here. Sources are never edited, they are the evidentiary record.

### `logs/`

Drift detection reports and agent error logs. Drift reports summarise which files have fallen out of date relative to their source systems. Error logs capture full tracebacks when an agent task fails, committed to the repo so nothing is lost even if the agent process crashes.

---

## How to Run It

### Prerequisites

- Python 3.11+
- `uv` (recommended) or `pip`
- Git configured with push access to your ECL repo
- An Anthropic API key (or your preferred LLM provider's credentials)
- API access to your company's source systems (Slack, Jira, Confluence, etc.)

### Step 1 Bootstrap with an LLM agent

The ECL is built by an LLM agent following `readme-for-agents.md`. Open Claude Code (or Codex) in this repository and give it this prompt:

```
Read readme-for-agents.md fully. Then follow the Quick-Start Checklist at the bottom of that document.
Before you write any files, ask me the discovery questions in Step 1.
```

The agent will interview you about your company's knowledge domains, map your data sources, and scaffold the entire repository structure including meta files, domain folders, and initial skill stubs.

**This takes 30–90 minutes of interactive session with a human providing answers.** Do not skip the interview phase; the ECL's quality depends entirely on accurate domain mapping and source authority definitions.

### Step 2 Review `meta/system-prompt.md`

After the agent completes Step 4 of the checklist, **stop and review `meta/system-prompt.md` yourself** before letting any workers run. This file governs all agent behaviour. Check that:

- The sensitive topics list is complete for your company
- The source authority hierarchy reflects your actual trust hierarchy
- The routing rules are accurate

This is the one step that requires human sign-off.

### Step 3 Run the worker

Workers are stateless, so you can run as many as you want in parallel. Start with one worker to populate the initial task queue, then scale out.

```bash
# Single worker (start here)
uv run ecl-runner.py worker --repo .

# Multiple workers in parallel (after initial population)
for i in $(seq 1 5); do
  ECL_AGENT_ID="agent-$i" uv run ecl-runner.py worker --repo . &
done
```

Workers run continuously: claim a task, execute it, release it, repeat. They sleep with exponential backoff when the queue is empty (30s → 60s → ... → 5 min cap).

### Step 4 Run the maintenance agent

```bash
# Run once manually to create the initial task batch
uv run ecl-runner.py maintenance --repo .

# Schedule to run every 6 hours (cron example)
0 */6 * * * cd /path/to/ecl-repo && uv run ecl-runner.py maintenance --repo .
```

The maintenance agent scans every domain for stale content, missing files, unresolved conflicts, and outdated skill files, then creates appropriately prioritised tasks for workers to address.

### Step 5 Query the ECL

**Recommended:** Give Claude Code access to the repository with this system prompt:

```
You are answering questions using the Enterprise Context Layer in this repository.

Before answering any question:
1. Read meta/system-prompt.md
2. Read meta/domain-index.md
3. Check domains/skills/ for any SKILL.md relevant to the task type

For every claim in your answer, cite the ECL file using [[description]](path).
If the ECL documents a conflict on this topic, present both sides.
If the ECL says to route a question, tell the user who to route to and why.
Surface last_verified dates for any time-sensitive claim.
```

**Lightweight alternative:** Use `ripgrep` to find relevant files, then pass them to your LLM:

```bash
rg -l --type md -i "data retention" domains/ | head -10
```

---

## What Good ECL Output Looks Like

A well-maintained topic file looks like this:

```markdown
---
last_verified: 2026-03-15
confidence: high
agent: agent-3
---

# Customer Data Retention After Churn

Customer PII is deleted 90 days after a churn event, as defined in the GDPR
compliance schedule [[Legal: Data Retention Policy v3.1, 2026-02-01]](../../sources/legal/data-retention-v3.1.md).
Non-PII data is retained for 180 days for analytics purposes
[[Legal: Data Retention Policy v3.1, 2026-02-01]](../../sources/legal/data-retention-v3.1.md).

> ⚠️ **Conflict (resolved 2026-03-15):** The engineering deletion runbook previously
> described a 120-day pipeline for all data. This conflicted with the legal policy because the
> runbook lagged a Q4 2025 policy update. See [[INFRA-4892]](../../sources/jira/INFRA-4892.md).
> Runbook updated 2026-03-10.

**Do not answer data deletion timeline questions directly** in customer-facing contexts.
Route to the security team. See [[GTM: Routing Rules]](../gtm/routing-rules.md#data-deletion).

## Related

- [[Legal: Data Retention Policy]](../legal/data-retention-policy.md)
- [[Engineering: Deletion Pipeline Runbook]](../engineering/deletion-pipeline-runbook.md)
- [[Skill: customer-data-request]](../skills/customer-data-request/SKILL.md)
- [[Security: Escalation Routing]](../security/escalation-routing.md)
```

Every claim is cited. The conflict was documented and resolved, with the original conflict text preserved, not deleted. Sensitive questions have routing instructions. Related files are cross-linked.

---

## What the ECL Is Not

**Not a search engine.** Raw documents live in `sources/` as read-only snapshots. The ECL synthesises conclusions from them; it does not index or search them.

**Not a real-time system.** Content has staleness SLAs measured in days, not seconds. For live operational data, use the source systems directly.

**Not a policy enforcement system.** The ECL documents policies and routing rules. It does not enforce them. That is the job of the systems and people it routes to.

**Not finished.** The target state is not "ECL is complete." It is "ECL is continuously maintained." The maintenance agent and worker loop run indefinitely.

**Not a replacement for experts.** Sensitive questions (legal liability, security architecture, pricing exceptions) are documented as routing notes, not answers. The ECL tells you who to ask. It does not replace asking them.

---

## Recommended Agent Counts by Phase

| Phase                          | Workers | Notes                                                           |
| ------------------------------ | ------- | --------------------------------------------------------------- |
| Initial seeding                | 1       | Keep git history clean; easy to debug                           |
| First population (3–4 domains) | 3–5     | Enough to parallelise without overloading sources               |
| Full population (all domains)  | 10–20   | Matches production usage in Andy Chen's original implementation |
| Maintenance mode               | 2–5     | Mostly verify and backlink tasks; lower parallelism needed      |

All coordination happens through git. Workers on Modal, AWS Lambda, Kubernetes, or local machines are identical. There is no central coordinator to configure.

---

## Credits and Prior Art

This project is an implementation template synthesising three published ideas:

- **[The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer)**: Andy Chen's original design: synthesis over retrieval, git as single source of truth, the folder-as-taxonomy and backlinks-as-context-graph pattern.
- **[Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)**: Nicholas Carlini's C-Compiler pattern: file-based distributed locking via git push-rejection, enabling parallel agents with no central broker.
- **[Waza](https://github.com/tw93/Waza)**: tw93's lean skills framework: single-purpose `SKILL.md` files with trigger-phrase frontmatter, loaded on match and followed once rather than chained into a mandatory multi-skill pipeline. This project's Lean Skills Pattern adapts that shape for ECL's own domain workflows.

The 10-step build process, task schema, staleness SLA tables, and repository structure are this project's extrapolation from those sources. This is one way to implement the pattern, but not the only way.

---

## Licence

GPL-3.0: free for all uses, but improvements must be contributed back to the community.
