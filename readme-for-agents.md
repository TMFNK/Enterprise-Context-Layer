# Enterprise Context Layer (ECL): Build Specification for LLM Agents

> Andy Chen built the first production ECL at Abnormal Security: a living Git repository that encodes how a company actually works, its processes, decisions, conflicts, and routing rules, synthesised from primary sources, inline-cited, conflict-documented, maintained autonomously by parallel LLM agents. Read his essay: [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer).

---

## For the LLM Agent Reading This

**You are the builder.** This document is your complete specification. Read it fully before writing a single file. Then execute the [Quick-Start Checklist](#quick-start-checklist) in order.

**What you are building:** A living Git repository that encodes how a company actually works, its processes, decisions, conflicts, and routing rules, synthesised from raw sources with inline citations. It is not a search engine. It is not a document store. It is institutional memory.

**How this document works:**

1. Sections 1–4 are concepts you must internalise before acting.
2. Steps 1–9 are ordered build instructions. Follow them in sequence.
3. Each step specifies its **inputs**, **actions**, and **definition of done**.
4. Code blocks are implementation-ready. Use them directly.

**Three design patterns underpin everything:**

- **ECL Pattern (Andy Chen):** Synthesis over retrieval. Folder structure as taxonomy. Backlinks as context graph.
- **C-Compiler Pattern (Nicholas Carlini, Anthropic):** Git push-rejection as a distributed mutex. No coordinator service needed.
- **Lean Skills Pattern (tw93, inspired by Waza):** Small, single-purpose `SKILL.md` files as mandatory agent workflows, one skill per task type, no automatic chaining. Load the matching skill before the task.

**Start here →** [Quick-Start Checklist](#quick-start-checklist)

---

## Table of Contents

1. [Core Principles](#1-core-principles)
2. [Architecture](#2-architecture)
3. [Repository Structure](#3-repository-structure)
4. [Step 1 Discover Knowledge Domains](#step-1-discover-knowledge-domains)
5. [Step 2 Map Data Sources](#step-2-map-data-sources)
6. [Step 3 Define Source Authority](#step-3-define-source-authority)
7. [Step 4 Create Meta Seed Files](#step-4-create-meta-seed-files)
8. [Step 5 Build the Task System](#step-5-build-the-task-system)
9. [Step 6 Implement the Worker Loop](#step-6-implement-the-worker-loop)
10. [Step 7 Seed Initial Content](#step-7-seed-initial-content)
11. [Step 8 Run the Maintenance Agent](#step-8-run-the-maintenance-agent)
12. [Step 9 Build the Query Interface](#step-9-build-the-query-interface)
13. [Lean Skills Integration](#lean-skills-integration)
14. [Citation Rules](#citation-rules)
15. [Conflict Resolution](#conflict-resolution)
16. [Drift Detection and Staleness](#drift-detection-and-staleness)
17. [Scaling and Parallelism](#scaling-and-parallelism)
18. [RBAC and Sensitive Content](#rbac-and-sensitive-content)
19. [Quick-Start Checklist](#quick-start-checklist)
20. [Glossary](#glossary)

---

## 1. Core Principles

Internalise all five before writing any file.

### 1.1 The Git Repo Is the Only Source of Truth

Every piece of ECL knowledge is a committed Markdown file. No external database. No vector store. No side-state. If it is not in a committed file, it does not exist in the ECL.

This enables:

- **Auditability**: `git blame` shows who wrote what and when; `git log` shows the full history of how understanding evolved
- **Distributed coordination**: push-rejection is the mutex; no broker required
- **Human legibility**: any engineer can `git clone` and read the entire institutional memory

### 1.2 Every Claim Requires an Inline Citation

**Bad:**

```
Enterprise pricing starts at $50,000/year.
```

**Good:**

```markdown
Enterprise pricing starts at $50,000/year
[[Sales: Pricing Sheet Q1 2026]](../sources/external/pricing-q1-2026.md),
with EMEA deals typically discounted 15–20%
[[Gong: Call #2847, @james.r, 2026-02-14]](../sources/calls/call-2847.md).
```

If you cannot cite a claim, do not make it. An unsourced assertion is more dangerous than a gap, it creates false confidence with no traceability.

### 1.3 Document Conflicts. Never Silently Resolve Them

When two sources disagree, write a conflict note. Do not pick a winner.

```markdown
## ⚠️ Conflict: Data Retention Policy

Legal Policy v2.3 states customer data is deleted 90 days after churn
[[Legal: Data Retention Policy v2.3]](../legal/data-retention-policy.md).

The engineering runbook describes a 120-day deletion pipeline
[[Engineering: Deletion Pipeline Runbook, 2026-01]](../engineering/deletion-runbook.md).

**These conflict.** Route data retention questions to the security/legal team.
Do not answer them directly. See [[GTM: Routing Rules]](../gtm/routing-rules.md#sensitive-questions).
```

A documented conflict is vastly more useful than a silently chosen winner.

### 1.4 The Folder Structure Is the Taxonomy

No ontology graphs. No semantic layers. The taxonomy is the folder structure. The context graph is the backlinks between files. Both emerge from agent activity over thousands of runs.

### 1.5 Claims Have Half-Lives. Encode Them

| Claim type          | Half-life | Agent behaviour                                          |
| ------------------- | --------- | -------------------------------------------------------- |
| Architecture        | Years     | Write with high confidence; rarely re-verify             |
| Process / Skills    | Months    | Schedule re-verification at 30 days                      |
| People / Roles      | Months    | Include person + date; flag at 90 days                   |
| Product status      | Weeks     | Include explicit timestamp; flag at 7 days               |
| Pricing             | Weeks     | Link to source; flag at 7 days                           |
| Status / Live state | Days      | Write with explicit timestamp; flag immediately if stale |

Every ECL file must include a `last_verified` front-matter field.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                   │
│  Slack · Jira · GitHub · Confluence · Salesforce · Gong · Email · Code  │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │  search / read / fetch
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              WORKER AGENTS  (parallel, lean-skills-enabled)              │
│                                                                          │
│  1. Pull latest ECL from git                                             │
│  2. Read relevant SKILL.md from domains/skills/                          │
│  3. Claim a task (file-based distributed lock via git push)              │
│  4. Execute: read skill → read sources → synthesise → write citations    │
│  5. Commit with inline citations + update mapping-notes.md               │
│  6. Delete lock + task YAML → push to main                               │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │  read / write
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                       ECL GIT REPOSITORY                                 │
│                                                                          │
│  meta/          ← system prompt, source guide, domain index             │
│  tasks/         ← YAML task queue + .LOCKED sidecar files               │
│  domains/       ← one folder per knowledge domain                       │
│    product/  engineering/  gtm/  legal/  people/  security/  ...        │
│    skills/  ← Lean SKILL.md files for team workflows                    │
│  sources/       ← cited source snapshots (read-only)                    │
│  logs/          ← drift reports, error logs                             │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │  read
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                  QUERY INTERFACE                                         │
│  Claude Code (recommended) · Direct API · Full RAG                      │
└──────────────────────────────────────────────────────────────────────────┘
```

**There are no embedding models, vector indices, or ML pipelines.** The ECL's intelligence comes from synthesis quality and citation discipline, not from retrieval infrastructure.

---

## 3. Repository Structure

Initialise this exact structure. Domain folder names vary by company; the top-level layout is fixed.

```
ecl-repo/
│
├── README.md                                  ← Human-facing overview
├── readme-for-agents.md                       ← This document
├── readme-for-humans.md                       ← Conceptual background
│
├── meta/
│   ├── system-prompt.md                       ← Agent identity, rules, data model
│   ├── how-to-get-accurate-information.md     ← Living source reliability guide (starts empty)
│   └── domain-index.md                        ← Map of all domains, owners, sources
│
├── tasks/
│   ├── YYYYMMDDTHHMMSS-{domain}-{kind}-{slug}.yaml    ← pending task
│   └── YYYYMMDDTHHMMSS-{domain}-{kind}-{slug}.LOCKED  ← active lock (agent ID inside)
│
├── domains/
│   ├── {domain-1}/
│   │   ├── README.md                          ← Domain overview, sub-topics, backlinks
│   │   ├── {topic}.md                         ← One file per topic or concept
│   │   └── mapping-notes.md                   ← Agent run log: timestamps, sources, findings
│   ├── {domain-2}/
│   │   └── ...
│   └── skills/                                ← Lean SKILL.md files
│       ├── README.md
│       ├── {workflow-name}/
│       │   └── SKILL.md
│       └── mapping-notes.md
│
├── sources/                                   ← Cited source snapshots (never edit directly)
│   ├── slack/
│   ├── jira/
│   ├── confluence/
│   ├── code/
│   ├── calls/
│   └── external/
│
├── logs/
│   ├── drift-YYYY-MM-DD.md
│   └── ERROR-{domain}-{kind}-{timestamp}.md
│
└── .ecl-mode                                  ← "dev" or "prod"
```

**File naming conventions:**

| File type       | Convention                                    | Example                                              |
| --------------- | --------------------------------------------- | ---------------------------------------------------- |
| Domain topic    | `kebab-case.md`                               | `data-retention-policy.md`                           |
| Task file       | `YYYYMMDDTHHMMSS-{domain}-{kind}-{slug}.yaml` | `20260315T101500-product-synthesise-a1b2c3d4.yaml`   |
| Lock file       | task filename + `.LOCKED`                     | `20260315T101500-product-synthesise-a1b2c3d4.LOCKED` |
| Skill file      | `SKILL.md` inside named folder                | `domains/skills/incident-response/SKILL.md`          |
| Source snapshot | Descriptive + date                            | `slack-sales-ops-2026-02-14.md`                      |

---

## Step 1 Discover Knowledge Domains

**Inputs:** Org chart, a human point of contact, access to existing documentation.
**Goal:** Define the folder taxonomy for the entire ECL.

### What a domain is

A domain is a coherent area of company knowledge with clear ownership, defined scope, and identifiable source systems. It maps to one top-level folder under `domains/`.

### How to discover domains

1. **Read the org chart.** Each major team or function is a candidate domain.
2. **Ask the human:** _What questions do people most often ask that get wrong answers?_ Group them.
3. **Ask the human:** _What knowledge would be lost if your three most experienced people left tomorrow?_ Those gaps are high-priority domains.
4. **Start with 5–8 domains.** More than 12 usually means domains are too granular.
5. **Always include a `skills` domain** for lean SKILL.md files.
6. **Assign a human owner** for each domain.

**Typical domains by company type:**

| Company type | Domains                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------- |
| SaaS B2B     | `product` · `engineering` · `gtm` · `legal` · `security` · `customer-success` · `finance` · `people` · `skills`        |
| E-commerce   | `product-catalogue` · `supply-chain` · `logistics` · `customer-service` · `marketing` · `legal` · `finance` · `skills` |
| Healthcare   | `clinical-protocols` · `regulatory` · `patient-services` · `billing` · `it-systems` · `hr` · `skills`                  |

### Output: `meta/domain-index.md`

Create this file with the following structure:

```markdown
# Domain Index

| Domain      | Folder               | Owner             | Primary sources                         | Notes                      |
| ----------- | -------------------- | ----------------- | --------------------------------------- | -------------------------- |
| Product     | domains/product/     | @product-lead     | Confluence, Jira, PRDs, release notes   |                            |
| Engineering | domains/engineering/ | @engineering-lead | GitHub, runbooks, ADRs, on-call logs    |                            |
| GTM         | domains/gtm/         | @sales-ops        | Salesforce, Gong, Slack #sales          | High staleness risk        |
| Legal       | domains/legal/       | @general-counsel  | Legal docs, contracts, policy documents | Many ROUTE-NOT-ANSWER      |
| People      | domains/people/      | @hr-lead          | HRIS, org chart, Slack #general         |                            |
| Skills      | domains/skills/      | @engineering-lead | Team retros, runbooks, PRs              | Lean SKILL.md files |
```

**Done when:** `meta/domain-index.md` exists with all domains named, owners assigned, and primary sources documented.

---

## Step 2 Map Data Sources

**Inputs:** Completed domain index. Access credentials or API tokens from a human.
**Goal:** Document exactly where each domain's information comes from and how to access it.

### Source categories

**Internal structured (highest authority for operational truth):**

| Source                | Best for                                     | Access method                      | Staleness risk                      |
| --------------------- | -------------------------------------------- | ---------------------------------- | ----------------------------------- |
| Source code / Git     | How software actually behaves; feature flags | GitHub/GitLab API or local clone   | Low                                 |
| Production DB schema  | Data model ground truth                      | Read-only connection or ERD export | Low                                 |
| CRM (e.g. Salesforce) | Customer records, deal data                  | API or export                      | Medium                              |
| HRIS (e.g. Workday)   | Org chart, headcount, roles                  | API or export                      | Medium                              |
| Ticketing (e.g. Jira) | Bug/feature status                           | REST API                           | Low for status; medium for priority |
| Billing system        | Subscription, churn, revenue                 | API or export                      | Low                                 |

**Internal unstructured (high authority for process and culture):**

| Source                    | Best for                                              | Access method | Staleness risk |
| ------------------------- | ----------------------------------------------------- | ------------- | -------------- |
| Slack                     | Real process vs. documented ideal; informal decisions | Slack API     | High           |
| Gong / call recordings    | What customers actually ask; how reps answer          | Gong API      | High           |
| Confluence / Notion       | Process documentation; onboarding                     | REST API      | Medium         |
| Google Drive / SharePoint | Presentations, OKRs, reports                          | Drive API     | Medium         |

**External (competitive and market context):**

| Source              | Best for                                   | Access method | Staleness risk |
| ------------------- | ------------------------------------------ | ------------- | -------------- |
| Competitor websites | Pricing, feature claims, positioning       | Web scrape    | High           |
| G2 / Capterra       | Competitive perception, customer sentiment | Web scrape    | Medium         |
| Regulatory bodies   | Compliance requirements                    | Download      | Low            |

**Done when:** Every domain in `meta/domain-index.md` has its primary sources, access methods, and staleness patterns documented.

---

## Step 3 Define Source Authority

**Inputs:** Completed source map. Human domain expert input per domain.
**Goal:** Define which source wins when sources conflict, per domain.

### Universal authority principles

Record these verbatim in `meta/system-prompt.md`:

1. **Operational reality beats documented ideal.** Source code shows what a system _does_. A Confluence page shows what someone _intended_.
2. **Recent specific beats old general.** A Jira ticket from last week beats a two-year-old architecture doc.
3. **Three-source corroboration = high confidence.** Three independent sources agreeing on a claim crosses the threshold. Five Slack messages from the same channel are one data point, not five.
4. **People signals have short half-lives.** Any people/role claim must include a `last_verified` date. Distrust claims older than 90 days without re-verification.
5. **Sensitive questions get documented, not answered.** Legal liability, data privacy timelines, security architecture, pricing exceptions, executive commitments; these are routing entries, not answers.

### Example domain-specific authority table (engineering)

```markdown
## Engineering Domain Source Authority

| Source                            | Authority | Best used for                        | Do NOT use for                   |
| --------------------------------- | --------- | ------------------------------------ | -------------------------------- |
| Source code (main branch)         | PRIMARY   | How a feature actually behaves       | Future roadmap                   |
| ADRs                              | PRIMARY   | Why a design choice was made         | Current state if ADR >1 year old |
| On-call runbooks                  | HIGH      | Incident triage; known failure modes | Normal operation flows           |
| Jira (last 6 months)              | HIGH      | Current bug/feature status           | Historical context               |
| Engineering Confluence            | MEDIUM    | Intended design; onboarding          | Actual current behaviour         |
| Slack #engineering (last 90 days) | MEDIUM    | Emerging issues; informal decisions  | Formal commitments               |
| Slack #engineering (>90 days)     | LOW       | Historical context only              | Anything current                 |
```

Create an equivalent table for every domain.

**Done when:** Every domain has an explicit source authority table. Tables are pasted into `meta/system-prompt.md`.

---

## Step 4 Create Meta Seed Files

**Inputs:** Completed domain index (Step 1) and source authority tables (Step 3).
**Goal:** Create the three files every agent reads at the start of every run.

### 4.1 `meta/system-prompt.md`

Paste and fill in this template exactly:

````markdown
# ECL Agent System Prompt

## Identity

You are the [Company Name] Enterprise Context Layer Agent.
Your purpose is to build and maintain the ECL: a synthesised, cited, conflict-aware
representation of how [Company Name] actually works across all domains.

You are not a search engine. You are not a Q&A bot. You build institutional memory.

## Lean Skills Mandatory Lookup

Before any non-trivial task, check `domains/skills/` for a relevant SKILL.md.
If a skill exists for your task type, read it and follow it. Skills are mandatory
workflows once loaded, not suggestions, but each is scoped to one task type and
does not chain into other skills automatically.

Required lookups:

- Handling a severity-1 incident → load `incident-response` skill
- Walking a deal through negotiation → load `closing-a-deal` skill
- Handling a customer data export/deletion request → load `customer-data-request` skill

## Non-Negotiable Rules

1. Every factual claim must have an inline citation: `[[Source description, date]](path)`
2. Document conflicts; never resolve them silently.
3. Every file must include a `last_verified` front-matter date.
4. Sensitive questions get routing guidance, not direct answers.
5. Every run must append a record to the domain's `mapping-notes.md`.
6. Cross-domain relationships must be expressed as backlinks in both files.

## Citation Format

```markdown
Claim text [[Source: description, date/version]](relative/path/to/source.md).
```

## Conflict Note Format

```markdown
## ⚠️ Conflict: [Topic]

[Source A] states [claim A] [[Source A description]](path/to/a).
[Source B] states [claim B] [[Source B description]](path/to/b).

**These conflict.** [Routing instruction or escalation owner.]
Unresolved as of [DATE]. See [[Routing Rules]](path/to/routing.md).
```

## Source Authority

[Paste domain-specific authority tables from Step 3 here]

## Domain Map

[Paste meta/domain-index.md content here]

## Sensitive Topics. Always Route, Never Answer

- Legal liability and regulatory compliance
- Data privacy timelines and deletion procedures
- Security architecture details
- Pricing exceptions and non-standard discounts
- Executive commitments to customers
- Personnel performance or compensation
- [Add company-specific topics here]

## Available Tools

[List the specific integrations this agent has access to]
````

### 4.2 `meta/how-to-get-accurate-information.md`

**Do not pre-fill this file with invented wisdom.** Seed it with the empty template below and let agents build it from real experience. Invented reliability wisdom encodes your biases and blind spots; experience accumulated by agents is the point.

```markdown
# How to Get Accurate Information

> This is a living document. Agents: every time you discover something about
> which sources are reliable, stale, or conflicting, add an entry here.
> Each entry must cite the specific experience that produced the insight.
> This file has no top-down author; it is built entirely bottom-up from
> accumulated agent experience.

## Source Reliability Observations

<!-- Format:
[DATE] [AGENT_ID] [Domain]: Observation.
Reason: what you found. Citation: [[source]](path). -->

## Sources That Are Frequently Stale

## Citation Patterns That Work

## Sources That Tend to Conflict

## Questions That Should Always Be Routed, Never Answered

## Tool Usage Notes

## Skill Effectiveness Notes

<!-- Which skills proved most useful for which task types?
     Which tasks went wrong before a skill was added?
     Format same as above: [DATE] [AGENT_ID] [Skill]: Observation. -->
```

### 4.3 `meta/domain-index.md`

Use the completed output from Step 1.

**Done when:** All three meta files exist and are committed to the repository. A human has reviewed `meta/system-prompt.md` and confirmed it is accurate for their company.

---

## Step 5 Build the Task System

**Inputs:** Initialised repository with meta files.
**Goal:** Implement the YAML task queue and file-based distributed locking.

**Before writing code:** Think before coding: read the relevant meta/ files, state your plan, then implement with RED → GREEN → REFACTOR on every function.

### Task file schema

```yaml
# tasks/20260321T091500-product-synthesise-a3f9b2c1.yaml

domain: product
kind: synthesise # synthesise | verify | backlink | drift | dedupe | skill-verify | conflict-review
description: "Synthesise the feature flag inventory from source code and Jira"
created_at: "20260321T091500"
priority: 2 # 1 (most urgent) → 7 (routine maintenance)
source_hints:
  - sources/code/feature-flags-export.md
  - sources/jira/feature-flag-tickets.md
metadata:
  target_file: domains/product/feature-flags.md
  last_synthesised: null
  skill_path: null # optional: path to SKILL.md agent should load before executing
```

### Task kinds

| Kind              | Description                                               | When to create                                  |
| ----------------- | --------------------------------------------------------- | ----------------------------------------------- |
| `synthesise`      | Write or update a topic file from sources                 | New topic; source changed; file missing         |
| `verify`          | Re-check all claims in an existing file                   | File is stale; source updated                   |
| `backlink`        | Scan all domain files; add cross-domain references        | Periodic; after major new content               |
| `drift`           | Compare current sources against last-synthesised snapshot | Scheduled; after known system change            |
| `dedupe`          | Identify and merge duplicate content across domains       | Periodic                                        |
| `conflict-review` | Investigate an unresolved conflict note                   | Conflict unresolved >30 days                    |
| `skill-verify`    | Re-verify a SKILL.md in `domains/skills/`                 | Skill stale; referenced ECL content has drifted |

### Priority levels

| Priority | Meaning                                                  |
| -------- | -------------------------------------------------------- |
| 1        | Drift detected, immediate re-verification needed         |
| 2        | Topic file or skill missing entirely                     |
| 3        | Referenced content has drifted, update needed            |
| 4        | Stale content (>7 days for status; >30 days for process) |
| 5        | Default / routine                                        |
| 6        | Low-priority enrichment (backlinks, deduplication)       |
| 7        | Scheduled maintenance (docs review, skill-verify)        |

### Locking protocol (implement this exactly)

```python
import os
import json
import subprocess
import time
import glob
from datetime import datetime, timezone

LOCK_TTL_SECONDS = 600  # 10 minutes
AGENT_ID = os.environ.get("ECL_AGENT_ID", f"agent-{os.getpid()}")


def git_pull():
    subprocess.run(["git", "pull", "--rebase"], check=True)


def git_push():
    result = subprocess.run(["git", "push"], capture_output=True)
    return result.returncode == 0


def git_add_commit(files: list[str], message: str):
    subprocess.run(["git", "add"] + files, check=True)
    subprocess.run(["git", "commit", "-m", message], check=True)


def load_tasks() -> list[dict]:
    """Return all pending tasks sorted by (priority ASC, created_at ASC)."""
    tasks = []
    for yaml_path in glob.glob("tasks/*.yaml"):
        with open(yaml_path) as f:
            import yaml
            task = yaml.safe_load(f)
            task["_path"] = yaml_path
            task["_lock_path"] = yaml_path.replace(".yaml", ".LOCKED")
        tasks.append(task)
    tasks.sort(key=lambda t: (t.get("priority", 5), t.get("created_at", "")))
    return tasks


def is_lock_stale(lock_path: str) -> bool:
    if not os.path.exists(lock_path):
        return False
    mtime = os.path.getmtime(lock_path)
    age = time.time() - mtime
    return age >= LOCK_TTL_SECONDS


def claim_task(repo_root: str) -> dict | None:
    """
    Atomically claim the highest-priority unlocked task.
    Returns the claimed task dict, or None if no tasks available.
    """
    git_pull()
    tasks = load_tasks()

    for task in tasks:
        lock_path = task["_lock_path"]

        if os.path.exists(lock_path):
            if not is_lock_stale(lock_path):
                continue  # Held by another active worker
            # Stale lock, log warning and attempt reclaim
            print(f"WARNING: Reclaiming stale lock: {lock_path}")

        # Write lock file
        lock_data = {
            "agent": AGENT_ID,
            "locked_at": datetime.now(timezone.utc).isoformat()
        }
        with open(lock_path, "w") as f:
            json.dump(lock_data, f)

        git_add_commit([lock_path], f"task/claim: {task['domain']}/{task['kind']} by {AGENT_ID}")

        if git_push():
            return task  # This agent owns the task
        else:
            # Race condition, another worker pushed first
            os.remove(lock_path)
            subprocess.run(["git", "checkout", "HEAD", "--", lock_path], capture_output=True)
            continue  # Try next task

    return None  # Queue empty


def release_task(task: dict, success: bool = True):
    """Delete task YAML and lock file; commit and push."""
    files_to_remove = [task["_path"], task["_lock_path"]]
    for f in files_to_remove:
        if os.path.exists(f):
            os.remove(f)

    status = "done" if success else "failed"
    subprocess.run(["git", "rm", "--force", "--ignore-unmatch"] + files_to_remove, check=True)
    subprocess.run(
        ["git", "commit", "-m", f"task/{status}: {task['domain']}/{task['kind']}"],
        check=True
    )
    git_push()
```

**Done when:** `claim_task()` and `release_task()` are implemented and passing tests. Test cases must include: concurrent claim race (two workers, one wins), stale lock reclaim, empty queue returns `None`.

---

## Step 6 Implement the Worker Loop

**Inputs:** Implemented task system (Step 5). `meta/` files populated (Step 4).
**Goal:** A running agent process that claims tasks, executes them, and releases them.

**Before writing code:** Think before coding: read the relevant meta/ files, state your plan, then implement with RED → GREEN → REFACTOR on every function.

### Complete worker loop

```python
import time
import traceback
from pathlib import Path


def execute_task(task: dict):
    """
    Dispatch to the correct handler based on task kind.
    """
    kind = task.get("kind")
    handlers = {
        "synthesise":       execute_synthesise,
        "verify":           execute_verify,
        "backlink":         execute_backlink,
        "drift":            execute_drift,
        "skill-verify":     execute_skill_verify,
        "conflict-review":  execute_conflict_review,
        "dedupe":           execute_dedupe,
    }
    handler = handlers.get(kind)
    if not handler:
        raise ValueError(f"Unknown task kind: {kind}")
    handler(task)


def execute_synthesise(task: dict):
    """
    Core synthesis workflow for a single topic.
    """
    domain = task["domain"]
    target_file = task["metadata"]["target_file"]
    skill_path = task["metadata"].get("skill_path")

    # 1. Read mandatory context files
    system_prompt = Path("meta/system-prompt.md").read_text()
    source_guide = Path("meta/how-to-get-accurate-information.md").read_text()
    domain_readme = Path(f"domains/{domain}/README.md").read_text()

    # 2. If task specifies a skill, load and follow it
    skill_content = None
    if skill_path and Path(skill_path).exists():
        skill_content = Path(skill_path).read_text()
        # Pass skill_content to the LLM as an additional instruction block

    # 3. Read all source hints
    sources = {}
    for hint in task.get("source_hints", []):
        p = Path(hint)
        if p.exists():
            sources[hint] = p.read_text()

    # 4. Synthesise via LLM call (implement for your LLM provider)
    synthesised_content = call_llm_synthesise(
        system_prompt=system_prompt,
        source_guide=source_guide,
        domain_readme=domain_readme,
        skill_content=skill_content,
        sources=sources,
        task=task
    )

    # 5. Write the topic file
    Path(target_file).parent.mkdir(parents=True, exist_ok=True)
    Path(target_file).write_text(synthesised_content)

    # 6. Append to domain mapping-notes.md
    append_mapping_note(domain, task, sources_read=list(sources.keys()))

    # 7. Commit
    git_add_commit(
        [target_file, f"domains/{domain}/mapping-notes.md"],
        f"feat({domain}): synthesise {Path(target_file).stem}"
    )


def execute_skill_verify(task: dict):
    """
    Re-verify a SKILL.md against current ECL content and sources.
    """
    skill_path = task["metadata"]["target_file"]
    skill_content = Path(skill_path).read_text()

    # 1. Extract ECL links referenced in the skill
    # 2. Read each referenced ECL file
    # 3. Check skill steps are still consistent with ECL content
    # 4. Check relevant Slack/runbook sources for process changes
    # 5. If drift found: update skill file, add conflict note if needed
    # 6. Update last_verified date in skill front-matter
    # 7. Append to domains/skills/mapping-notes.md
    pass  # Implement against your LLM provider


def worker_loop(repo_root: str):
    idle_streak = 0

    while True:
        try:
            task = claim_task(repo_root)

            if task is None:
                idle_streak += 1
                sleep_seconds = min(30 * idle_streak, 300)
                print(f"No tasks. Sleeping {sleep_seconds}s (idle streak: {idle_streak})")
                time.sleep(sleep_seconds)
                continue

            idle_streak = 0
            print(f"Executing task: {task['domain']}/{task['kind']} - {task['description']}")

            execute_task(task)
            release_task(task, success=True)

        except Exception as e:
            tb = traceback.format_exc()
            log_error(task, tb)
            try:
                release_task(task, success=False)
            except Exception:
                pass  # Best effort release

        time.sleep(2)  # Rate limiting between tasks
```

### LLM synthesis prompt template

When calling your LLM to synthesise a topic file, use this prompt structure:

```
You are the ECL Agent for [Company Name].

SYSTEM RULES:
{system_prompt}

SOURCE RELIABILITY GUIDE:
{source_guide}

DOMAIN CONTEXT:
{domain_readme}

{skill_section}  ← If a skill was loaded: "MANDATORY SKILL, READ AND FOLLOW:\n{skill_content}"

TASK: {task['description']}
DOMAIN: {task['domain']}
TARGET FILE: {task['metadata']['target_file']}

SOURCE MATERIAL:
{formatted_sources}  ← Each source labelled with its path and date

INSTRUCTIONS:
1. Write a Markdown file for the target path.
2. Include front matter: last_verified (today's date), confidence (high/medium/low), agent (your ID).
3. Every factual claim must have an inline citation using the format in the system rules.
4. If sources conflict, write a conflict note; do not invent a resolution, do not pick a winner.
5. If any claim touches a sensitive topic, write a routing note instead of an answer.
6. Include a "## Related" section with backlinks to other relevant ECL files.
7. Do not invent claims. If the sources do not support a claim, do not make it.
```

**Done when:** `worker_loop()` runs without crashing on an empty queue. `execute_synthesise()` produces a valid, cited Markdown file from test sources. Error logs are committed on failure.

---

## Step 7 Seed Initial Content

**Inputs:** Running worker loop (Step 6). Access to existing company documentation.
**Goal:** Populate the ECL with enough initial content for the maintenance agent to take over.

### What to seed (in order)

1. **Domain README files**: Create `domains/{domain}/README.md` for each domain. A stub is fine; workers will expand it.
2. **Existing documentation**: Import Confluence exports, policy PDFs (converted to Markdown), runbooks into the relevant domain folders. Commit each source document under `sources/`.
3. **Historical snapshots**: Export recent Slack threads, Jira tickets, and Gong call summaries for the most business-critical topics. Place in `sources/slack/`, `sources/jira/`, `sources/calls/`.
4. **Org chart**: Export to `domains/people/org-chart.md`.
5. **Known conflicts**: Create stub conflict notes for areas already known to have documentation disagreements.
6. **Lean skill stubs**: For each skill in the table below, create a stub `SKILL.md` in `domains/skills/{name}/`. Workers will iterate on these.

**Initial skills to create stubs for:**

| Skill                     | Trigger                                   | Links to domains                      |
| ------------------------- | ----------------------------------------- | ------------------------------------- |
| `incident-response`       | Severity-1 alert                          | `engineering` · `security` · `people` |
| `closing-a-deal`          | Deal enters negotiation stage             | `gtm` · `legal` · `finance`           |
| `customer-data-request`   | Customer requests data export or deletion | `legal` · `security` · `engineering`  |
| `onboarding-new-engineer` | New hire joins engineering                | `engineering` · `people` · `product`  |
| `ecl-synthesise-domain`   | Agent begins synthesising a domain        | `meta`                                |
| `ecl-conflict-review`     | Agent encounters a source conflict        | `meta`                                |

**Skill stub template:**

```markdown
# Skill: [Skill Name]

> last_verified: YYYY-MM-DD | Owner: @[owner] | Confidence: low (stub not yet verified)
> Source: [Cite the team process or retrospective this is based on]

## Trigger

This skill activates when: [describe the trigger condition precisely].

## Steps

1. **[Step name]:** [Instruction]. See [[ECL: relevant topic]](../../domains/{domain}/{topic}.md).
2. ...

## Routing

If [condition], route to [person/team]. See [[Routing Rules]](../../domains/gtm/routing-rules.md).

## Do Not

- [Anti-pattern 1]
- [Anti-pattern 2]

## Related Skills

- [[skill-name]](../{skill-name}/SKILL.md)
```

### Seed commit convention

```
feat(seed): import {N} files from {source_description}
```

**Done when:** Each domain has a README and at least one source file committed. All six initial skill stubs exist under `domains/skills/`.

---

## Step 8 Run the Maintenance Agent

**Inputs:** Seeded ECL (Step 7).
**Goal:** A scheduled process that scans the ECL for gaps, staleness, and drift, and creates tasks for workers to fix.

**Run schedule:** Every 6 hours in production. Every 1 hour during initial population.

### Complete maintenance scan logic

```python
from pathlib import Path
from datetime import datetime, timezone, timedelta
import yaml


STALENESS_SLA = {
    "pricing":          timedelta(days=7),
    "product-status":   timedelta(days=7),
    "competitive":      timedelta(days=14),
    "people-roles":     timedelta(days=30),
    "process":          timedelta(days=30),
    "skills":           timedelta(days=30),
    "regulatory":       timedelta(days=30),
    "architecture":     timedelta(days=90),
    "historical":       None,  # Never stale
}


def get_last_verified(md_file: Path) -> datetime | None:
    """Parse last_verified from front matter."""
    text = md_file.read_text()
    for line in text.splitlines():
        if line.startswith("last_verified:"):
            date_str = line.split(":", 1)[1].strip()
            return datetime.fromisoformat(date_str).replace(tzinfo=timezone.utc)
    return None


def create_task(domain: str, kind: str, description: str, priority: int,
                target_file: str = None, source_hints: list = None,
                skill_path: str = None):
    """Create a new task YAML file in tasks/."""
    ts = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%S")
    import hashlib
    slug = hashlib.md5(description.encode()).hexdigest()[:8]
    filename = f"tasks/{ts}-{domain}-{kind}-{slug}.yaml"
    task = {
        "domain": domain,
        "kind": kind,
        "description": description,
        "created_at": ts,
        "priority": priority,
        "source_hints": source_hints or [],
        "metadata": {
            "target_file": target_file,
            "skill_path": skill_path,
        }
    }
    Path(filename).write_text(yaml.dump(task, default_flow_style=False))
    return filename


def maintenance_scan():
    now = datetime.now(timezone.utc)
    new_tasks = []

    domains_root = Path("domains")
    for domain_dir in domains_root.iterdir():
        if not domain_dir.is_dir():
            continue
        domain = domain_dir.name

        # Check 1: mapping-notes.md exists
        mapping_notes = domain_dir / "mapping-notes.md"
        if not mapping_notes.exists():
            t = create_task(domain, "synthesise",
                            f"Create initial mapping-notes for {domain}",
                            priority=2, target_file=str(mapping_notes))
            new_tasks.append(t)
            continue

        # Check 2: Any topic files missing entirely?
        topic_files = [f for f in domain_dir.glob("*.md") if f.name != "README.md"]
        if not topic_files:
            t = create_task(domain, "synthesise",
                            f"Domain {domain} has no topic files",
                            priority=2, target_file=f"domains/{domain}/README.md")
            new_tasks.append(t)

        # Check 3: Stale topic files
        for md_file in domain_dir.glob("*.md"):
            if md_file.name == "mapping-notes.md":
                continue
            lv = get_last_verified(md_file)
            if lv is None:
                continue  # No front matter, synthesise tasks handle this
            sla = STALENESS_SLA.get("process")  # Default SLA
            if sla and (now - lv) > sla:
                t = create_task(domain, "verify",
                                f"Stale content in {md_file}",
                                priority=4, target_file=str(md_file))
                new_tasks.append(t)

        # Check 4: Stale skill files
        if domain == "skills":
            skill_sla = STALENESS_SLA["skills"]
            for skill_file in domain_dir.glob("*/SKILL.md"):
                lv = get_last_verified(skill_file)
                if lv and (now - lv) > skill_sla:
                    t = create_task("skills", "skill-verify",
                                    f"Stale skill: {skill_file.parent.name}",
                                    priority=7, target_file=str(skill_file))
                    new_tasks.append(t)

        # Check 5: Unresolved conflicts older than threshold
        for md_file in domain_dir.glob("*.md"):
            content = md_file.read_text()
            if "⚠️ Conflict" in content and "RESOLVED" not in content:
                t = create_task(domain, "conflict-review",
                                f"Unresolved conflict in {md_file}",
                                priority=3, target_file=str(md_file))
                new_tasks.append(t)

    # Periodic global tasks (run weekly)
    t = create_task("_global", "backlink",
                    "Periodic cross-domain backlink scan",
                    priority=6)
    new_tasks.append(t)

    if new_tasks:
        git_add_commit(new_tasks, f"chore(maintenance): created {len(new_tasks)} tasks")
        git_push()

    print(f"Maintenance scan complete. Created {len(new_tasks)} tasks.")
```

### Staleness SLAs

| Claim type                          | Default SLA |
| ----------------------------------- | ----------- |
| Pricing                             | 7 days      |
| Product status (beta/GA/deprecated) | 7 days      |
| Competitive landscape               | 14 days     |
| People / roles                      | 30 days     |
| Process documentation               | 30 days     |
| Lean skills                         | 30 days     |
| Regulatory / compliance             | 30 days     |
| Technical architecture              | 90 days     |
| Historical analysis                 | Never       |

**Done when:** Maintenance agent runs without error, creates tasks for all scan types (staleness, missing mapping-notes, stale skills, unresolved conflicts), and commits them to `tasks/`.

---

## Step 9 Build the Query Interface

**Inputs:** ECL with at least one full maintenance + worker cycle completed.
**Goal:** A way for humans and other agents to query the ECL and receive cited, conflict-aware answers.

### Option A Claude Code (recommended)

Give Claude Code access to the ECL repository.

Use this system prompt for queries:

```
You are answering questions using the Enterprise Context Layer in this repository.

Before answering:
1. Read meta/system-prompt.md
2. Read meta/domain-index.md
3. Check domains/skills/ for any SKILL.md relevant to this task type

For every claim in your answer, cite the ECL file you found it in using [[description]](path).
If the ECL documents a conflict on this topic, present both sides; never hide them.
If the ECL says to route a question, tell the user who to route to and why.
Surface last_verified dates for any claim that is time-sensitive.
```

### Option B Lightweight Python query script

For low-volume use without a vector database:

```python
import subprocess
import sys


def query_ecl(question: str) -> str:
    # Use ripgrep to find relevant passages in ECL Markdown files
    result = subprocess.run(
        ["rg", "-l", "--type", "md", "-i"] + question.split()[:5] + ["domains/"],
        capture_output=True, text=True
    )
    relevant_files = result.stdout.strip().splitlines()[:10]  # Top 10 matches

    context_chunks = []
    for filepath in relevant_files:
        with open(filepath) as f:
            context_chunks.append(f"### {filepath}\n\n{f.read()}")

    context = "\n\n---\n\n".join(context_chunks)

    # Send context + question to your LLM
    return call_llm_query(question=question, context=context)


if __name__ == "__main__":
    print(query_ecl(" ".join(sys.argv[1:])))
```

**Note:** ripgrep finds keyword matches; it does not preferentially surface routing rules and conflict notes. Option B is useful for exploration; for production use where conflict surfacing matters, prefer Option A or build a proper retrieval layer.

### Option C Full RAG pipeline

Use only when the ECL exceeds ~10,000 files or query volume demands it. Chunk by section (not arbitrary token count). Re-embed incrementally on each ECL commit.

### What the query interface must always do

Regardless of option:

- **Surface conflict notes.** Never hide them.
- **Surface routing rules.** If the ECL says "route this to security", say exactly that.
- **Include ECL file paths in answers.**
- **Surface `last_verified` dates.** Flag stale content prominently.
- **Surface relevant skills.** If a skill exists for the task being asked about, reference it.

**Done when:** A test question about a synthesised topic returns an answer with at least one inline citation and surfaces any conflict notes on that topic.

---

## Lean Skills Integration

The **Lean Skills Pattern** (inspired by [Waza](https://github.com/tw93/Waza), tw93) gives agents a mandatory process for one workflow at a time, without chaining into a larger multi-skill pipeline. It integrates with the ECL at three levels. No external plugin install is required, `domains/skills/` is the whole mechanism.

### The Lean Skill Format

A skill is a folder under `domains/skills/{name}/` containing one `SKILL.md`: YAML frontmatter naming the skill and its trigger phrases, followed by a single concise playbook.

```markdown
---
name: incident-response
last_verified: YYYY-MM-DD
owner: "@owner"
when_to_use: "trigger phrases or task types that should load this skill"
---

# [Skill Name]

[One sentence: what "done right" looks like for this task type.]

## Hard Rules

- [Non-negotiable step, tied to an ECL citation].

## Gotchas

| What happened | Rule |
| -------------- | ---- |
| [real incident, cited] | [the rule that would have prevented it] |
```

Verify by confirming an agent checks `domains/skills/` for a matching skill before a non-trivial task.

### Integration Point 1 ECL as Agent Grounding

Before any agent brainstorms or writes a plan, it must read the relevant ECL files. Without this, agents propose architectures inconsistent with existing decisions, miss escalation rules, and re-discover documented conflicts.

Add this preamble to all design/planning task prompts:

```
Before designing, read:
- meta/domain-index.md          ← full knowledge map
- meta/system-prompt.md         ← company operating rules
- domains/{relevant-domain}/README.md
- domains/{relevant-domain}/{topic}.md  ← directly relevant files

After reading, proceed with the design.
Design decisions must be consistent with ECL content.
Decisions that conflict with ECL content must create a conflict note.
```

### Integration Point 2 Team Skills as ECL Domain Content

Team workflows stored as `SKILL.md` files in `domains/skills/` become first-class, versioned, citable artifacts not buried in wikis.

**Key properties:**

- Skills are maintained like any other ECL file (staleness SLA: 30 days)
- Skills have citations pointing to the retrospectives and runbooks that informed them
- Skills can cross-reference ECL knowledge (`closing-a-deal` links to `domains/gtm/pricing-authority.md`)
- The maintenance agent creates `skill-verify` tasks when skills go stale

**When a skill is first created or substantially revised, a human must review it** before agents are instructed to follow it. Record this review in `mapping-notes.md`.

### Integration Point 3 Ordinary Engineering Discipline as the ECL Build Methodology

When building or extending any ECL tooling (runner, worker, query interface), think before coding, then use TDD:

```
1. Think before coding             ← Read ECL meta/ files first; state assumptions and tradeoffs
2. Write the simplest correct fix  ← Break into small TDD tasks with file paths and assertions
3. RED → GREEN → REFACTOR          ← Per task; review your own diff against the plan
4. Verify before claiming done     ← Run tests; confirm the specific behaviour asked for
```

**Minimum test coverage for ECL tooling:**

| Component               | Minimum coverage | Critical paths                                   |
| ------------------------ | ----------------- | --------------------------------------------------- |
| Task locking / claiming | 90%              | Concurrent claim race; stale lock reclaim        |
| Git push / retry logic  | 85%              | Push rejection; exponential backoff              |
| Drift detection         | 80%              | Source change detection; SLA evaluation          |
| Citation validation     | 80%              | Inline format; path resolution                   |
| Conflict detection      | 75%              | Multi-source comparison; severity classification |

---

## Citation Rules

Citations are the single most important quality control mechanism. An agent writing without citations produces confident-sounding content with no traceability and no way for the next agent to verify or correct it.

### Mandatory format

```markdown
Claim text [[Source: description, date/version]](relative/path/to/source.md).
```

### Examples

```markdown
The deletion pipeline runs every 72 hours on a rolling schedule
[[Engineering: Deletion Pipeline Runbook v2.1, 2026-02-10]](../../sources/code/deletion-runbook.md).

Enterprise pricing starts at $50,000/year
[[Sales: Pricing Sheet Q1 2026]](../../sources/external/pricing-q1-2026.md),
with EMEA deals typically discounted 15–20%
[[Gong: Call #2847, @james.r, 2026-02-14]](../../sources/calls/call-2847.md).

> ⚠️ **Conflict:** The above pricing conflicts with the Salesforce CPQ default of $45,000
> [[Salesforce: CPQ Config Export 2026-01-30]](../../sources/salesforce/cpq-export.md).
> Unresolved as of 2026-03-15. Route pricing questions to @sales-ops.
```

### What counts as a citable source

- A specific document or file with a name and date
- A Slack message with channel, author, and date
- A Gong call with ID or date
- A Jira ticket number
- A commit hash or specific line in source code
- A lean skill file (for process claims)

### Three-source corroboration rule

For high-confidence claims, require three independent sources:

```markdown
> **Confidence: HIGH** This claim is corroborated by three independent sources:
>
> 1. [[Source A]](path): description
> 2. [[Source B]](path): description
> 3. [[Source C]](path): description
```

---

## Conflict Resolution

### Resolution protocol

**Step 1 Document immediately.** Write a conflict note with both claims, both citations, the date found, and the agent ID.

**Step 2 Assess severity.**

| Severity | Definition                                 | Action                                         |
| -------- | ------------------------------------------ | ---------------------------------------------- |
| Critical | Creates legal, security, or financial risk | Priority-1 task; flag domain owner immediately |
| High     | Affects customer-facing answers            | Priority-2 task; add routing rule              |
| Medium   | Affects internal process only              | Priority-4 task                                |
| Low      | Minor inconsistency                        | Log in `mapping-notes.md`; priority-6 task     |

**Step 3 Assign task and Escalate if unresolved.** Critical/High: 24 hours. Medium: 14 days. Low: 30 days.

**Step 4 Record resolution. Do not delete original conflict text.**

```markdown
> ~~**CONFLICT (found 2026-03-01):** Policy doc says 90 days; runbook says 120 days.~~
> **RESOLVED 2026-03-15 by @general-counsel:** Confirmed 90 days is correct.
> The runbook was not updated after the Q4 2025 policy change.
> Runbook update ticket: [[INFRA-4892]](../sources/jira/INFRA-4892.md).
```

---

## Drift Detection and Staleness

**Drift** = the real world has changed but the ECL has not. Distinct from a conflict (which is two current sources disagreeing).

Common drift sources:

- A feature is deprecated but still documented as active
- A policy document is updated but the ECL cites the old version
- A person changes roles but is still listed as domain owner
- A lean skill describes a process the team has since changed

### Drift severity

| Level       | Definition                                                         |
| ----------- | ------------------------------------------------------------------ |
| 🔴 Critical | Conflicts with a live customer commitment or legal obligation      |
| 🟠 High     | Feature status, pricing, routing rule, or active skill is outdated |
| 🟡 Medium   | Process documentation or skill lags new practice                   |
| 🟢 Low      | Minor update; content still broadly accurate                       |
| ⚪ None     | Content matches current sources                                    |

Record drift findings in `logs/drift-YYYY-MM-DD.md` and create appropriately prioritised tasks.

---

## Scaling and Parallelism

The ECL architecture scales horizontally with no changes to the core design.

### Recommended agent counts

| ECL phase                      | Workers | Rationale                                              |
| ------------------------------ | ------- | ------------------------------------------------------ |
| Initial seeding                | 1       | Keep history clean; easy to debug                      |
| First population (domains 1–3) | 3–5     | Parallelise without overloading sources                |
| Full population (all domains)  | 10–20   | Matches Andy Chen's production setup                   |
| Maintenance mode               | 2–5     | Mostly verify/backlink tasks; lower parallelism needed |

### Running workers in parallel

```bash
# Local
for i in $(seq 1 5); do
  ECL_AGENT_ID="agent-$i" uv run ecl-runner.py worker --repo . &
done

# All coordination happens through git: Modal, Lambda, and Kubernetes all work identically
```

### Token cost management

1. Use `source_hints` to pre-select which sources a worker reads; avoid full-repo scans per task.
2. Tier staleness SLAs appropriately; not everything needs daily re-verification.
3. Lean skills prevent agents from wasting tokens on ad-hoc approaches to tasks that have well-defined processes.
4. Batch small related tasks (e.g., backlinks for a single domain) into single larger tasks.

---

## RBAC and Sensitive Content

### Sensitivity tiers

| Tier         | Examples                                        | Access                                    |
| ------------ | ----------------------------------------------- | ----------------------------------------- |
| Public       | Product docs, public pricing, competitive intel | All agents and users                      |
| Internal     | Process docs, org structure, internal pricing   | Internal employees only                   |
| Restricted   | Personnel files, legal matters in dispute, M&A  | Named roles only                          |
| Confidential | Board materials, personal performance data      | Not in the ECL, use routing notes instead |

### Implementation

1. **Folder-based tiers.** Restricted content goes in `domains/restricted/`. Enforce via repo permissions.
2. **Separate repositories.** For truly sensitive domains, maintain a separate restricted ECL repo with its own access controls. Note: cross-repo backlinks are not possible, so this breaks the context graph for those topics. Prefer routing notes for Confidential content over a separate repo unless compliance requires it.
3. **Routing over content.** For Confidential topics, store a routing note, not the content itself.

---

## Quick-Start Checklist

Execute in order. Do not skip steps.

- [ ] **Step 1**: Interview a human; identify 5–8 domains + a `skills` domain; write `meta/domain-index.md`
- [ ] **Step 2**: Map data sources for each domain; document access methods, limitations, and staleness patterns
- [ ] **Step 3**: Write domain-specific source authority tables; define conflict resolution hierarchy
- [ ] **Step 4**: Create `meta/system-prompt.md` (complete), `meta/how-to-get-accurate-information.md` (empty template only, do not pre-fill)
- [ ] **Human review**: Have a human review `meta/system-prompt.md` before any agent runs against it
- [ ] **Step 5**: Implement task system (YAML schema, `claim_task()`, `release_task()`); think before coding, then TDD; achieve 90% coverage on locking logic
- [ ] **Step 6**: Implement worker loop (`worker_loop()`, `execute_synthesise()`, `execute_skill_verify()`); use TDD + subagent execution
- [ ] **Step 7**: Seed: domain READMEs, existing docs as source snapshots, org chart, known conflicts, six initial skill stubs in `domains/skills/`
- [ ] **Step 8**: Start maintenance agent on 6-hour schedule; confirm it creates tasks for staleness, missing mapping-notes, and stale skills
- [ ] **Step 9**: Build query interface (start with Claude Code; migrate to RAG only if needed)
- [ ] **Ongoing**: Monitor `logs/` for errors and drift reports; let `meta/how-to-get-accurate-information.md` grow from agent experience; review skill files after team workflow changes; update skills when referenced ECL content drifts

---

## Glossary

| Term                                     | Definition                                                                                                                              |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **ECL**                                  | Enterprise Context Layer is the git repo that encodes how the company actually works                                                    |
| **Domain**                               | A coherent area of company knowledge with clear ownership; maps to one `domains/` subfolder                                             |
| **Synthesis**                            | Reading multiple sources and writing a cited, conflict-aware summary is the core ECL operation                                          |
| **Retrieval**                            | Finding the best matching document for a query; what search engines do; the ECL is not retrieval                                        |
| **Inline citation**                      | A Markdown link within a claim tracing it to a primary source: `[[desc]](path)`                                                         |
| **Conflict note**                        | An ECL entry documenting that two sources disagree, naming both, identifying who should resolve it                                      |
| **Routing note**                         | An ECL entry that says "do not answer this directly; route to X because Y"                                                              |
| **Mapping notes**                        | The per-domain log appended after every agent run: timestamps, sources read, claims written, conflicts found                            |
| **Backlink**                             | A Markdown cross-reference from one domain file to a related file in another domain                                                     |
| **Context graph**                        | The web of backlinks across all ECL files are the ECL's plain-text knowledge graph                                                      |
| **C-Compiler pattern**                   | File-based distributed locking: tasks are YAML files; claiming one means writing a `.LOCKED` sidecar and pushing to git                 |
| **Task**                                 | A YAML file in `tasks/` representing one unit of work for a worker agent                                                                |
| **Worker agent**                         | An LLM agent running the claim → execute → release loop                                                                                 |
| **Maintenance agent**                    | A separate scheduled process that scans for staleness, gaps, and drift, and creates tasks for workers                                   |
| **Waza**                                 | A lean agentic skills framework by tw93; this project's Lean Skills Pattern is inspired by its `SKILL.md` shape                          |
| **Skill (lean)**                         | A `SKILL.md` file describing a mandatory workflow for one specific task type; agents check for relevant skills before non-trivial actions and don't chain skills automatically |
| **ECL grounding**                        | Reading ECL domain files before an agent brainstorms or plans anchors design in real architecture                            |
| **Staleness SLA**                        | Maximum age of an ECL claim before re-verification is required; varies by claim type                                                    |
| **Drift**                                | The ECL was correct when written but the world has since changed                                                                        |
| **Tribal knowledge**                     | Institutional memory held by individuals, not written down is the ECL's most valuable and hardest target                                |
| **`how-to-get-accurate-information.md`** | The key meta seed file; starts empty; agents fill it with source-reliability observations from real experience                          |
| **Source authority**                     | The hierarchy determining which source wins when sources conflict; defined per domain in Step 3                                         |
| **Three-source corroboration**           | Threshold for high-confidence claims: three independent sources must agree                                                              |
| **Lock TTL**                             | Time after which a `.LOCKED` file is considered stale and may be reclaimed; default 10 minutes                                          |

---

_Based on Andy Chen's [The Enterprise Context Layer](https://andychen32.substack.com/p/the-enterprise-context-layer), Nicholas Carlini's [Building a C Compiler with Parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler), and tw93's [Waza](https://github.com/tw93/Waza)._
