# Project Agora — Specification

> **Note.** This document is the detailed technical specification handed to implementers. For the public-facing introduction, see [`README.md`](README.md). The codebase uses `agora` (lowercase) for package names, file paths, and environment variables; "Project Agora" is the public-facing name.

## 1. Project Overview

Project Agora is an open-source platform that takes shadow-IT applications — typically built quickly with AI coding assistants like Claude Code or Codex — and runs them through a fleet of specialist AI agents that handle the production-readiness work the original developer didn't.

Agora has **two audiences**:

- **Admins** define what "production-ready" means for their company — the checks, the agents that run, the gates that require review, and who approves at each. Admins author every agent in their Anthropic [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview) workspace and configure Agora's pipeline template once. Admins (or a delegated reviewer pool) approve at any gates the pipeline pauses on.
- **End users** (the "vibe-coders") drop their app in (zip or GitHub URL), provide credentials, and watch the run progress. They see what's happening to their code at every stage, but they don't configure agents and they don't approve gates.

**Agora aims to be fully agentic by default and uses human-in-the-loop sparingly.** A pipeline that runs end-to-end without human intervention is the goal. Reality is that some decisions — the plan being correct before any code is changed, an irreversible deploy — warrant a reviewer. Gates exist for those moments, and admins decide where to place them. Treating every transition as a gate defeats the point; the pipeline should pause for what genuinely needs review and otherwise keep going.

Agora itself is intentionally thin: it picks agents the admin has already defined in the Claude Console, runs them in the configured order, persists state in the user's repo and Postgres, and surfaces progress in a dashboard. Every agent capability — orchestration, success criteria, persistent memory, credential storage — leans on a Managed Agents primitive rather than an Agora-side reimplementation.

Every Agora pipeline begins with two **mandatory** agent stages:

1. **Source Capture** — always creates a fresh GitHub repository in an Agora-managed location and clones the user's code into it, regardless of input type. Zip uploads are extracted and committed; GitHub URLs are cloned (using the user-supplied read-only PAT if the source is private). The user's original source is never written to. From this point on, the **project repository** (Agora-managed) is the single source of truth.
2. **Gap Analysis** — analyses the captured code against the production-readiness checklist, produces a scorecard and an **effort report**, and writes the initial manifest including the list of specialist stages applicable to this project.

After Gate 1, a single **Specialist Coordinator** stage runs: an admin-authored Claude multiagent coordinator that delegates in parallel to whichever specialists Gap Analysis flagged as applicable. Common specialist tasks include:

- Scan and remediate security issues (secrets, dependencies, auth gaps)
- Generate Infrastructure as Code (Terraform/Bicep)
- Wire up CI/CD pipelines
- Instrument observability (logs, metrics, traces, alerts)
- Generate test scaffolding
- Produce a compliance/data-flow register
- Provision infrastructure and deploy to the user's cloud account
- Open pull requests for every change, gated by human approval

State for each run is persisted in the project's GitHub repository as a structured manifest (`.agora/readiness.yaml`), making the audit trail portable, human-readable, and outliving any single Agora instance.

### 1.1 Vision

The "last mile" between an AI-generated app and production is currently invisible to most builders. Agora makes that last mile concrete, automatable, and auditable — without taking ownership of the user's code, infrastructure, or data, and without re-implementing the agent runtime.

The default mode is autonomous: agents capture, analyse, remediate, and prepare changes without human input. Human-in-the-loop is a deliberate design choice, applied where the cost of a wrong autonomous decision outweighs the cost of waiting for a reviewer. Admins decide where those points are; Agora gives them the controls to place gates precisely and to remove them when the pipeline is trusted enough to run open-loop.

### 1.2 Non-Goals

- **Not a code generator.** Agora works on existing apps; it does not write features.
- **Not a hosting platform.** It provisions infrastructure into the user's cloud accounts; it does not run user workloads.
- **Not an agent runtime.** Agents run in Claude Managed Agents; Agora coordinates them. Agora does not author, version, or host agent prompts as part of its source.
- **Not a SaaS.** This repo is the self-hosted product with deployment instructions; a proof of concept will be deployed.

### 1.3 License

GNU General Public License v3.0. Include `LICENSE` (full GPL-3.0 text) and `COPYING` files. Every source file should carry the standard GPL header.

---

## 2. Glossary

| Term | Meaning |
| --- | --- |
| **Project** | A user-submitted application registered with Agora. Defined by its source (zip or GitHub URL) and its project repository (Agora-managed, created during Source Capture). |
| **Source** | The user's original input — a zip file or a GitHub URL (with an optional read-only PAT for private repos). Agora reads from it once, during Source Capture, and never writes to it. |
| **Project Repository** | The Agora-managed GitHub repository created by Source Capture in the admin-configured destination (typically a dedicated Agora-managed organisation). All agent work, the manifest, and PRs live here. |
| **Admin** | Operator of the Agora instance. Authors agents in the Claude Console workspace, seeds the checklist memory store, configures Agora's pipeline template, approves at gates. In single-user mode, the same person as the end user. |
| **End User** | The "vibe-coder" who submits projects, supplies target-environment credentials at registration, triggers runs, and watches progress. Cannot configure the pipeline and does not approve gates. |
| **Run** | One end-to-end execution of the readiness pipeline against a project. |
| **Pipeline Template** | The admin-configured ordering of pipeline stages, plus rubrics and gate placement. Lives in Postgres; seeded from `config/pipeline.example.yaml` on first boot. |
| **Stage** | One step in the pipeline. Mandatory stages: `source_capture`, `gap_analysis`. Optional stages: `coordinator` (specialists), `gate`, future deploy stages. |
| **Agent** | A Claude Managed Agent created by the admin in their workspace. Agora references agents by ID; it does not author them. |
| **Coordinator** | A multiagent agent (admin-authored) whose roster includes specialist agents. The pipeline's `coordinator` stage points at one. |
| **Pipeline Runner** | The Agora API component that executes the pipeline — creates sessions, tracks state, records gate approvals. Not itself an LLM agent. |
| **Readiness Manifest** | The `.agora/readiness.yaml` file in the project's repo tracking state. |
| **Memory Store** | A workspace-scoped, persistent collection of text documents Agora attaches to agent sessions. Two stores are in scope (§6.4). Distinct from the manifest. |
| **Checklist Store** | The workspace-scoped, **read-only** memory store holding the production-readiness checklist and shared reference material. Attached to every agent session. |
| **Project Memory Store** | A per-project, **read-write** memory store holding soft context across re-runs (preferences, prior rejection reasons). Created at registration. |
| **Vault** | An Anthropic-managed, workspace-scoped credential store. Per-project; holds the end user's target-environment credentials (GitHub PAT, AWS keys, etc.). Replaces Agora-side credential encryption. |
| **Outcome** | A success rubric (markdown) attached to a session via `user.define_outcome`. The Managed Agents grader evaluates the agent's deliverable against it and either passes or returns specific gaps for another iteration. |
| **Effort Report** | Output of the Gap Analysis agent: per-finding estimate of the work needed to close each gap, presented at Gate 1. |
| **Applicable Stages** | The list Gap Analysis writes to the manifest indicating which specialist agents should run for this project. The coordinator reads it to scope its delegation. |
| **Gate** | A human-approval checkpoint between stages. Used sparingly — only for decisions where the cost of an autonomous mistake outweighs the cost of waiting for a reviewer. Approved by admins (or admin-designated reviewers in multi-user mode). |
| **Finding** | A single gap identified by an agent, with severity and proposed action. |
| **Action** | A concrete change (open PR, provision resource, create alert) tied to a finding. |
| **Sample** | A reference prompt or rubric shipped in `examples/`. Documentation for admins to copy into the Claude Console; never loaded by Agora. |

---

## 3. Architecture

### 3.1 High-Level

```
┌────────────────────────────────────────────────────────────────┐
│                         Web UI (HTMX)                           │
│  Project list • Run progress • Gates • Pipeline editor (admin) │
└────────────────────────────┬───────────────────────────────────┘
                             │ REST + SSE
┌────────────────────────────▼───────────────────────────────────┐
│                      Agora API (Go)                            │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ Project mgmt │  │ Pipeline runner │  │ Manifest service │  │
│  └──────────────┘  └────────┬────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌────────────────────────────────────┐  │
│  │ Webhook ingest   │  │ Pipeline template editor (admin)   │  │
│  └────────┬─────────┘  └────────────────────────────────────┘  │
└───────────┼──────────────────┬─────────────────────────────────┘
            │  webhooks (HTTPS) │ Anthropic Go SDK
            ▲                   ▼
┌───────────────────────────────────────────────────────────────┐
│              Claude Managed Agents (Anthropic-hosted)         │
│  Agents • Sessions • Multiagent threads • Memory • Vaults     │
└────────────────────────────┬──────────────────────────────────┘
                             │ MCP (per agent)
        ┌──────────┬─────────┼──────────┬──────────┐
        ▼          ▼         ▼          ▼          ▼
     GitHub    Cloud APIs  Datadog/Grafana   Jira/Linear
```

### 3.2 Components

1. **Agora API** — Go service. Owns project registration, run lifecycle, manifest read/write, webhook ingest, pipeline template, and Managed Agents session management via the Anthropic Go SDK. Stateless except for run metadata in Postgres.
2. **Web UI** — HTMX + TailwindCSS, server-rendered from the Go API via [`templ`](https://github.com/a-h/templ). Communicates over REST + Server-Sent Events for live run progress.
3. **Postgres** — Stores projects, runs, run events, gate approvals, audit log, and the active pipeline template. Records `vault_id`, `memory_store_id`, and `repo_url` per project. Does not store the readiness manifest itself (that lives in the user's repo) and does not store user credentials (those live in vaults).
4. **Pipeline Template** — Admin-configured ordering of stages, persisted in Postgres. Seeded from `config/pipeline.example.yaml` on first boot if no template exists yet. Edited via the admin UI thereafter.
5. **Webhook ingest** — Agora exposes `POST /api/v1/webhooks/anthropic`, verifies payloads via the Go SDK's `Beta.Webhooks.Unwrap()` helper, and converts Managed Agents events into Agora's internal SSE stream.
6. **Agora Manifest MCP server** — Thin custom MCP server (`mcp/agora-manifest/`) giving agents typed `read_manifest` / `write_finding` / `update_checklist` / `mark_gate` / `set_applicable_stages` operations on top of the GitHub MCP. Without this layer, agents tend to write malformed YAML.
7. **Agents, Memory, Vaults** — All Anthropic-hosted. Agora references them by ID; it does not host them.

### 3.3 Run Lifecycle (Example)

```
submit project ──► trigger run ──► pipeline starts
       │                                     │
       │                          Source Capture (mandatory)
       │                          ──► create or link GitHub repo
       │                                     │
       │                          Gap Analysis (mandatory)
       │                          ──► scorecard + effort report
       │                          ──► writes initial manifest
       │                          ──► writes applicable_stages
       │                                     │
       │                          ◄──── GATE 1: review plan
       │                                     │
       │                          Specialist Coordinator
       │                          (one session, multiagent)
       │                          ──► delegates to applicable specialists
       │                          ──► parallel threads, shared filesystem
       │                          ──► each opens PRs & writes manifest
       │                                     │
       │                          ◄──── GATE 2: review changes
       │                                     │
       │                          (optional) Deploy stage
       │                                     │
       │                          ◄──── GATE 3: deploy approval
       │                                     │
       └──► run complete ◄────── final report
```

Gate 1 is mandatory in every pipeline. Gates after the coordinator and before deploy are admin-configured; gate any stage that touches paid infrastructure or production environments.

---

## 4. Tech Stack

Opinionated to keep the project moving:

- **Backend:** Go 1.23, `chi` (router), `pgx` (Postgres driver), `goose` (migrations), `sqlc` (typed queries)
- **Anthropic SDK:** `github.com/anthropics/anthropic-sdk-go` (first-party, Go 1.23+; covers agents, sessions, multiagent, outcomes, memory, vaults, webhooks, Files API)
- **Concurrency:** Goroutines and channels throughout. Standard `net/http` for HTTP.
- **Database:** PostgreSQL 16
- **Frontend:** HTMX + TailwindCSS, served as server-rendered HTML via `templ`
- **Realtime:** Server-Sent Events (SSE) — simpler than WebSockets and sufficient; HTMX has first-class SSE support
- **Container:** Docker, multi-stage builds, distroless or `scratch` final images
- **Orchestration:** Docker Compose (local) and Helm chart (Kubernetes)
- **CI:** GitHub Actions
- **Linting/formatting:** `gofmt` + `golangci-lint` (Go), `prettier` for templates and CSS
- **Testing:** standard `testing` package + `testify` (Go), `testcontainers-go` for integration tests
- **Logging:** `log/slog` (stdlib) with JSON handler

Avoid:

- Job queues (Asynq, NATS, RabbitMQ) — Managed Agents handles long-running work
- Redis — not needed for MVP; revisit if multi-replica API requires it
- Heavyweight ORMs (GORM, ent) — prefer `sqlc` on top of `pgx`
- CSS frameworks other than Tailwind
- Client-side SPA frameworks — the UI is server-rendered HTML
- Expression languages (CEL, etc.) — Gap Analysis declares applicable stages directly in the manifest (§5.4); Agora does not evaluate predicates
- Custom credential encryption — vaults handle it (§8.2)

---

## 5. Pipeline

### 5.1 Pipeline Template

Agora's pipeline is data, not code. The active template is a row tree in Postgres; on first boot, `config/pipeline.example.yaml` is imported as the seed if no template exists. After that, Postgres is the source of truth and the admin UI is the editor.

Template shape:

```yaml
# config/pipeline.example.yaml — first-boot seed
version: 1
stages:
  - kind: source_capture     # mandatory; must be first
    agent_id: agt_01ABC...   # admin picks from their workspace
    rubric_file: examples/rubrics/source-capture.md
    outcome_max_iterations: 3
  - kind: gap_analysis       # mandatory; must be second
    agent_id: agt_01DEF...
    rubric_file: examples/rubrics/gap-analysis.md
    outcome_max_iterations: 3
  - kind: gate
    name: plan_review
  - kind: coordinator        # admin-authored multiagent coordinator
    agent_id: agt_01GHI...
    rubric_file: examples/rubrics/specialists.md
    outcome_max_iterations: 5
  - kind: gate
    name: changes_review
```

Validation rules (enforced by the API on save):

- First stage must be `kind: source_capture`.
- Second stage must be `kind: gap_analysis`.
- Every non-gate stage must reference an `agent_id` that exists in the workspace (`client.Beta.Agents.List()`).
- Rubric files must exist on disk *or* the field may be omitted (graders are optional but recommended).
- Gate names within a template must be unique. No gate is *required* by the API — admins can author a fully autonomous pipeline (no gates at all) if they trust their agents and rubrics. The seed template (`config/pipeline.example.yaml`) includes `plan_review` and `changes_review` as recommended defaults.

### 5.2 Mandatory Stages

**Source Capture** (single-agent session). Always creates a fresh project repository in the destination the admin's Source Capture agent is configured to use (typically a dedicated Agora-managed organisation), regardless of input type:

- **Zip upload:** the agent extracts the archive and pushes it as the initial commit.
- **GitHub URL:** the agent clones the source (using the user-supplied PAT for private repos), strips the upstream remote, and pushes to the new project repository.

The agent then writes:

- `project.source` — origin (`type: zip` or `type: github` with the original URL and `cloned_at`).
- `project.repo` — the new project repository URL.
- `.agora/readiness.yaml` scaffolded with `schema_version: 1`.

All downstream stages read and write to `project.repo`. The user's original source is never modified.

At the end of the run, the user pulls changes from the project repository into their own workflow — either manually (`git remote add agora ...`) or by reviewing PRs in the project repository directly. An automated PR-back-to-source mechanism could be added later, but is out of scope for MVP.

**Gap Analysis** (single-agent session). Reads the captured code, evaluates against the production-readiness checklist (mounted from the read-only checklist memory store at `/mnt/memory/checklist/`), and writes:

- `runs[].findings[]` — each gap with severity, category, effort estimate, and proposed action.
- `checklist.<category>.<key>: pass|warn|fail` — the per-key scorecard.
- `gap_analysis.applicable_stages: [...]` — the canonical list of specialist names this project needs (e.g. `[security, infrastructure, observability]`).

If the pipeline includes a `plan_review` gate after Gap Analysis, the run pauses for an admin to approve before any specialist work begins. Approval is recorded in the manifest's `gates[]`. Pipelines without that gate proceed straight to the specialist phase.

### 5.3 Specialist Coordinator (Multiagent)

The specialist phase is one stage and one session. The admin authors a "Specialist Coordinator" agent in the Claude Console with `multiagent.type: coordinator` and a roster including every specialist (security, infrastructure, ci-cd, observability, testing, compliance, ...). Agora's pipeline references the coordinator by ID.

When the stage runs, Agora:

1. Creates a session against the coordinator agent, attaching the project memory store (`read_write`) and the checklist store (`read_only`), and the project's vault via `vault_ids: [...]`.
2. Sends a `user.message` event including the manifest's `gap_analysis.applicable_stages` list and a pointer to the current manifest.
3. Sends `user.define_outcome` with the specialists rubric.
4. Streams the primary thread; sub-thread events (`session.thread_created`, `agent.thread_message_received`, etc.) surface specialist activity.

The coordinator's prompt instructs it to delegate in parallel only to the agents named in `applicable_stages`, share results on the session filesystem, and converge on the rubric. Agora does not pick specialists itself — that's the coordinator's job, scoped by Gap Analysis's output.

For admins who want sequential single-agent specialists instead of multiagent, point the `coordinator` stage at a non-coordinator agent — Agora doesn't care; it just runs whatever session is configured.

### 5.4 Outcomes

Every non-gate stage may declare a `rubric_file`. After creating the session, Agora sends:

```go
client.Beta.Sessions.Events.Send(ctx, sessionID, anthropic.BetaSessionEventSendParams{
    Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
        OfUserDefineOutcome: &anthropic.BetaManagedAgentsUserDefineOutcomeEventParams{
            Description: "<stage description>",
            Rubric: anthropic.BetaManagedAgentsUserDefineOutcomeEventParamsRubricUnion{
                OfFile: &anthropic.BetaManagedAgentsFileRubricParams{FileID: rubricFileID},
            },
            MaxIterations: anthropic.Int(stage.OutcomeMaxIterations),
        },
    }},
})
```

The Managed Agents grader evaluates the deliverable in an isolated context window. Verdict (`satisfied | needs_revision | max_iterations_reached | failed | interrupted`) is captured via the `session.outcome_evaluation_ended` webhook and recorded on the run.

**Rubrics describe end state, not process.** A rubric criterion is something a human reviewer could verify against the codebase and manifest at the gate — never something about how the agent worked. Examples in §5.6.

Rubric files are uploaded once via the Files API (`files-api-2025-04-14` beta header) at startup or on template save, and referenced by file ID in the `define_outcome` event.

### 5.5 Sample Agents and Onboarding

Agora ships reference prompts and rubrics in `examples/`:

```
examples/
├── agent-prompts/
│   ├── source-capture.md
│   ├── gap-analysis.md
│   ├── specialist-coordinator.md
│   └── specialists/
│       ├── security.md
│       ├── infrastructure.md
│       └── ...
└── rubrics/
    ├── source-capture.md
    ├── gap-analysis.md
    ├── specialists.md
    └── ...
```

These are documentation, never loaded code. The admin UI's "Pipeline editor" walks first-time admins through:

1. *Create the Source Capture agent in your Claude Console.* Here is a sample system prompt — open it, copy, customize as needed.
2. *Create the Gap Analysis agent.* Sample prompt; sample rubric to upload via Files API.
3. *Create your specialist agents (security, infra, ...).* One sample per specialist.
4. *Create a Specialist Coordinator with the specialists in its roster.* Sample prompt.
5. *Pick the agents you just created in each pipeline stage below.*

Once the admin has authored their preferred set, the samples are out of the picture.

### 5.6 Example Rubric (Security Specialist)

```markdown
# Security specialist outcome

## Static analysis
- gitleaks against HEAD returns zero findings, OR every finding has a manifest entry
  with severity, file path, and an explicit justification.
- The configured dependency auditor (npm audit / pip-audit / equivalent) reports
  no high-severity vulnerabilities, OR each remaining high CVE has a manifest entry.

## Repository state
- No tracked file matches the high-confidence credential regex set
  (`/mnt/memory/checklist/credential-patterns.md`).
- `.env*` files are gitignored and absent from history.

## Manifest completeness
- Every key under `checklist.security.*` has a status of pass/warn/fail
  (no `unknown`).
- Every fail has at least one finding with severity, category, and effort estimate.
- Every finding marked `status: resolved` references a concrete action
  (PR URL, commit SHA, or manifest justification).
```

Note what's absent: no requirement to "open PRs," no count of commits, no opinion on agent process. The grader checks the codebase + manifest at the end of the iteration.

### 5.7 Phasing of Specialists

Specialists ship in three waves; each wave just adds more sample prompts/rubrics under `examples/specialists/` and is a documentation change rather than a code change:

- **Wave 1 (MVP):** Security
- **Wave 2:** Infrastructure, CI/CD
- **Wave 3:** Observability, Testing, Compliance

Admins can author additional specialists at any time; Agora has no enum of specialist types.

---

## 6. The Readiness Manifest

`.agora/readiness.yaml` lives in the project's GitHub repo — guaranteed to exist by the Source Capture mandatory agent that runs first in every pipeline. It is the source of truth across runs, the audit trail, and the state document agents read and write.

### 6.1 Schema

```yaml
schema_version: "1"
project:
  name: my-app
  source:
    type: github
    url: github.com/alice/my-app
    cloned_at: 2026-05-03T10:00:30Z
  repo: github.com/agora-managed/my-app-2026-05-03
  registered_at: 2026-05-03T10:00:00Z
runs:
  - id: run-2026-05-03-1
    started_at: 2026-05-03T10:01:00Z
    completed_at: 2026-05-03T11:14:00Z
    status: completed
    triggered_by: alice@acme.com
    agora_version: 0.1.0
    gap_analysis:
      applicable_stages: [security, observability]
      scorecard:
        passed: 12
        warn: 4
        fail: 5
    outcomes:
      - stage: gap_analysis
        result: satisfied
        iterations: 1
      - stage: coordinator
        result: satisfied
        iterations: 2
    findings:
      - id: sec-001
        agent: security
        severity: high
        title: "Hardcoded API key in src/config.py"
        category: secrets
        status: resolved
        action:
          type: pull_request
          url: https://github.com/acme/my-app/pull/42
        resolved_at: 2026-05-03T10:34:00Z
    gates:
      - name: plan_review
        approved_at: 2026-05-03T10:08:00Z
        approved_by: alice@acme.com
      - name: changes_review
        approved_at: 2026-05-03T11:02:00Z
        approved_by: alice@acme.com
checklist:
  security:
    secrets_scan: pass
    dependency_audit: pass
    auth_review: warn
  infrastructure:
    iac_present: pass
    backups_configured: fail
    ha_configured: warn
```

### 6.2 Schema Versioning

The `schema_version` key is mandatory. Migrations are forward-only and live in `schemas/manifest/migrations/`. Each agent must validate `schema_version` before reading.

### 6.3 Agora Manifest MCP Server

Ship a small MCP server (`mcp/agora-manifest/`) that exposes:

- `read_manifest(repo)` — fetch and parse current manifest
- `write_finding(repo, finding)` — append/update a finding
- `update_checklist(repo, category, key, status)` — update a checklist key
- `set_applicable_stages(repo, stages[])` — set `gap_analysis.applicable_stages` (Gap Analysis only)
- `mark_gate(repo, gate_name, approver)` — record gate approval

This wraps the GitHub MCP underneath but gives agents a typed, validated interface.

### 6.4 Memory Stores

Memory complements the manifest. Where the manifest is **structured, permanent state in the project's repo** (audit trail and source of truth across runs), memory is **unstructured, agent-managed context** held in [Anthropic's workspace-scoped memory stores](https://platform.claude.com/docs/en/managed-agents/memory).

**Two stores are in scope, both MVP:**

- **Checklist store** (workspace-scoped, **read-only**). Holds the production-readiness checklist, manifest schema, "good PR" exemplars, and known false-positive patterns. Mounted on every agent at `/mnt/memory/checklist/`. Admins update it via the Memory API; safe concurrent edits via `content_sha256` precondition. No agent redeploy required. Read-only access prevents prompt-injection corruption of trusted reference material.
- **Project memory store** (per-project, **read-write**). Created at project registration; `memory_store_id` recorded on the `projects` row. Holds soft context across re-runs: prior rejection rationales, user preferences ("prefers Terraform over Bicep"), environment hints, false positives surfaced earlier. Archived when the project is deregistered.

Optional Wave-2: a per-run scratch store attached to the coordinator session so specialists can leave files for each other within the run; archived when the run completes.

| | Manifest | Memory |
| --- | --- | --- |
| Location | Project's repo (`.agora/readiness.yaml`) | Anthropic-hosted memory store |
| Retention | Permanent | 30-day version retention with redact for compliance |
| Format | Structured YAML against a schema | Free-form markdown files |
| Audience | Compliance / audit | Agents |
| Portability | Outlives any Agora instance | Tied to the workspace |

**Operational constraints:**

- Each memory file is capped at 100 KB (~25K tokens). Structure as many small focused files.
- Maximum 8 memory stores per session.
- Memory stores attach **at session creation only**; add/remove during a run is not supported.
- The Memory API requires the `managed-agents-2026-04-01` beta header; the Go SDK sets this automatically.

Credentials never go in memory — those live in vaults (§8.2).

---

## 7. API Design

REST + SSE. JSON request/response. OpenAPI spec generated from `chi` route definitions via `swaggo/swag`.

### 7.1 Endpoints

```
POST   /api/v1/projects                 Register a project (creates vault + memory store)
GET    /api/v1/projects                 List projects
GET    /api/v1/projects/{id}            Project detail
DELETE /api/v1/projects/{id}            Deregister (archives vault + memory store)

POST   /api/v1/projects/{id}/runs       Trigger a new run
GET    /api/v1/projects/{id}/runs       List runs
GET    /api/v1/runs/{id}                Run detail (current state)
GET    /api/v1/runs/{id}/events         SSE stream of run events
POST   /api/v1/runs/{id}/gates/{name}   Approve a gate
POST   /api/v1/runs/{id}/cancel         Cancel an in-flight run

GET    /api/v1/agents                   List Managed Agents from the workspace (admin picker)
GET    /api/v1/pipeline                 Read the active pipeline template
PUT    /api/v1/pipeline                 Update the pipeline template (admin)
POST   /api/v1/pipeline/validate        Dry-run validation of a proposed template

POST   /api/v1/webhooks/anthropic       Ingest Managed Agents webhooks (signature-verified)

GET    /api/v1/health                   Liveness
GET    /api/v1/ready                    Readiness (DB + Anthropic API reachable)
```

`GET /api/v1/agents` proxies `client.Beta.Agents.List()` so the admin UI can populate stage pickers from agents the admin has already created in the Console.

### 7.2 SSE Event Schema

```json
{
  "type": "stage_started" | "agent_message" | "tool_call" | "thread_created" |
          "thread_message" | "finding" | "outcome_evaluation" | "gate_pending" |
          "stage_completed" | "run_completed" | "error",
  "run_id": "run-2026-05-03-1",
  "stage": "coordinator",
  "session_thread_id": "sth_01...",   // present for multiagent thread events
  "timestamp": "2026-05-03T10:05:00Z",
  "data": { ... }
}
```

Events are produced by the webhook ingest path (state transitions) and the Anthropic SSE stream (per-event detail).

### 7.3 Authentication and Roles

Agora has two roles: **Admin** (configures the pipeline template, manages memory stores) and **End User** (registers projects, triggers runs, approves gates). Two auth modes are supported, configured via `AGORA_AUTH_MODE`:

**Single-user mode** (`token`, default — MVP). One bootstrap token in `AGORA_API_TOKEN`, sent as `Authorization: Bearer <token>`. There is no interactive sign-in and no per-user identity; the holder of the token is implicitly both Admin and End User. This matches the §11 self-hosting flow where the developer running Agora is also its only user.

**Multi-user mode** (`github` — post-MVP). Sign-in via a **GitHub App** OAuth flow. Admins are identified by membership in `AGORA_ADMINS` — a comma-separated list of **numeric GitHub user IDs**. Anyone else who authenticates is an End User. The same GitHub App also holds the repo-write permissions Source Capture needs, so Agora maintains a single GitHub integration rather than two.

Admin-only endpoints (return 403 for End Users): `PUT /api/v1/pipeline`, `POST /api/v1/runs/{id}/gates/{name}` (gate approval), memory store management, and any future system-configuration endpoints. The remaining `/projects/*` and `/runs/*` endpoints — registration, run trigger, run detail, SSE — are available to both roles. End users see gates in the run UI as informational state ("waiting for reviewer"); the approve button only renders for admins.

The webhook endpoint `/api/v1/webhooks/anthropic` does not use Agora's auth — it verifies the Anthropic webhook signature via `client.Beta.Webhooks.Unwrap()` instead.

---

## 8. Configuration & Secrets

All configuration via environment variables. A `config/example.env` file documents every option. No secrets in code, ever.

### 8.1 Required

```
ANTHROPIC_API_KEY              Anthropic API key (admin's workspace)
ANTHROPIC_WEBHOOK_SIGNING_KEY  whsec_-prefixed secret from Console > Webhooks
DATABASE_URL                   postgresql://user:pass@host:5432/agora
AGORA_API_TOKEN                Bootstrap auth token (single-user mode)
AGORA_PUBLIC_URL               Public URL of the API (for SSE + webhook delivery)
```

### 8.2 Per-User Credentials (Vaults)

End users supply credentials when registering a project. These never live in Postgres. Agora creates a per-project Anthropic vault and stores credentials as `mcp_oauth` (with auto-refresh) or `static_bearer` per the target service. The session passes `vault_ids: [project.vault_id]` so the configured agents authenticate against MCP servers transparently.

Two categories:

```
Source-repo PAT (only for private GitHub sources): read-only → static_bearer
   Used once, by Source Capture, to clone the user's source.
   Agora's own GitHub identity (in the managed destination) handles all writes.

Target-environment credentials (provided as needed by the project):
   Cloud:         AWS / Azure / GCP credentials → mcp_oauth or static_bearer per MCP
   Observability: Datadog API key / Grafana token (optional) → static_bearer
   Issue tracker: Jira / Linear API token (optional) → mcp_oauth
```

**Authorization note:** vaults are workspace-scoped, so any holder of the Anthropic API key can reference any vault. Agora must enforce its own per-project authorization by checking that the `vault_id` on the run matches the requesting project. Never accept a user-supplied `vault_id` on the API; always look it up from `projects.vault_id`.

### 8.3 Optional

```
AGORA_AUTH_MODE            token | github                        (default: token)
AGORA_ADMINS               comma-separated numeric GitHub user IDs (multi-user mode only)
AGORA_GITHUB_APP_*         GitHub App config: id, client secret, private key path
AGORA_LOG_LEVEL            debug | info | warn | error           (default: info)
AGORA_PIPELINE_SEED_FILE   first-boot pipeline seed              (default: ./config/pipeline.example.yaml)
AGORA_MAX_CONCURRENT_RUNS  per-project run concurrency           (default: 1)
AGORA_GATE_TIMEOUT_HOURS   auto-cancel runs awaiting approval    (default: 72)
ANTHROPIC_BETA_HEADER      override the managed-agents beta header
ANTHROPIC_FILES_BETA       override the files-api beta header (for rubric uploads)
```

`AGORA_ENCRYPTION_KEY` is intentionally absent — vaults handle encryption.

---

## 9. Project Structure

Monorepo, single `git` repo, multiple deployable artifacts.

```
agora/
├── LICENSE                          GPL-3.0
├── COPYING
├── README.md                        Quickstart, screenshots, license
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md                      Vulnerability disclosure
├── SPEC.md                          This document
├── docs/
│   ├── architecture.md
│   ├── self-hosting.md
│   ├── admin-guide.md               Authoring agents in the Console + configuring Agora
│   ├── manifest-schema.md
│   └── adr/                         Architecture decision records
│
├── api/                             Agora API (Go)
│   ├── go.mod
│   ├── cmd/agora/main.go            HTTP server entrypoint
│   ├── internal/
│   │   ├── config/                  Settings loaded from env vars
│   │   ├── db/                      sqlc-generated code, goose migrations
│   │   ├── handlers/                chi handlers per resource
│   │   ├── services/                Business logic
│   │   │   ├── projects.go          Includes vault + memory store creation
│   │   │   ├── runs.go              Pipeline runner
│   │   │   ├── manifest.go
│   │   │   ├── pipeline.go          Pipeline template CRUD + validation
│   │   │   └── webhooks.go          Anthropic webhook ingest
│   │   ├── claude/                  Anthropic Go SDK wrappers
│   │   └── mcp/                     MCP server URLs/configs
│   └── tests/
│
├── ui/                              Web UI (HTMX templates)
│   ├── templates/                   templ components (.templ files)
│   │   ├── layout.templ
│   │   ├── pages/                   Projects, Runs, Run detail, Pipeline editor
│   │   └── components/              Shared partials
│   ├── static/
│   │   ├── htmx.min.js
│   │   └── tailwind.css             Built CSS bundle
│   └── tailwind.config.js
│
├── mcp/                             Custom MCP servers
│   └── agora-manifest/
│       ├── go.mod
│       └── main.go
│
├── examples/                        Reference samples for admins (NOT loaded code)
│   ├── agent-prompts/               System prompts to copy into the Claude Console
│   │   ├── source-capture.md
│   │   ├── gap-analysis.md
│   │   ├── specialist-coordinator.md
│   │   └── specialists/
│   │       ├── security.md
│   │       └── ...
│   └── rubrics/                     Outcome rubrics to upload via Files API
│       ├── source-capture.md
│       ├── gap-analysis.md
│       ├── specialists.md
│       └── ...
│
├── schemas/
│   ├── manifest/
│   │   ├── v1.json                  JSON Schema for the manifest
│   │   └── migrations/
│   └── pipeline/
│       └── v1.json                  JSON Schema for the pipeline template
│
├── deploy/
│   ├── docker-compose.yml           Local one-command stack
│   ├── docker-compose.dev.yml       Dev overrides (hot reload)
│   └── helm/                        Helm chart
│
├── config/
│   ├── example.env
│   └── pipeline.example.yaml        First-boot pipeline seed
│
├── scripts/
│   ├── bootstrap.sh                 First-run setup
│   └── seed-dev.go                  Local dev fixtures (run via `go run`)
│
└── .github/
    └── workflows/
        ├── ci.yml                   Lint, test, build
        ├── release.yml              Tag → GHCR images + Helm chart
        └── codeql.yml               Security scanning
```

The `agents/`, `prompts/`, and `crypto/` directories from earlier drafts are gone — agents are admin-authored in the Console, prompts ship as `examples/` documentation only, and credential encryption is handled by vaults.

---

## 10. Deployment

### 10.1 Local (Docker Compose)

A user clones the repo, copies `config/example.env` to `.env`, fills in `ANTHROPIC_API_KEY`, `AGORA_API_TOKEN`, and `ANTHROPIC_WEBHOOK_SIGNING_KEY`, configures their webhook endpoint in the Anthropic Console (`{AGORA_PUBLIC_URL}/api/v1/webhooks/anthropic`), and runs:

```bash
docker compose -f deploy/docker-compose.yml up -d
```

This starts: Postgres, the API, the UI (served from the API), and the Agora manifest MCP server. The UI is reachable at `http://localhost:8080`.

For local development without a public URL, use `ngrok` (or similar) to forward the webhook endpoint. The Anthropic Console must reach Agora's webhook URL or the run will appear stuck after the first session is created.

### 10.2 Kubernetes (Helm)

```bash
helm install agora deploy/helm \
  --set anthropic.apiKey=sk-ant-... \
  --set anthropic.webhookSigningKey=whsec_... \
  --set agora.apiToken=... \
  --set agora.publicUrl=https://agora.example.com \
  --set postgresql.auth.password=...
```

Helm chart requirements (unchanged plus webhook URL exposure):

- API `Deployment`, UI bundled into the API image
- `Ingress` template that exposes `/api/v1/webhooks/anthropic` publicly
- Postgres as a subchart dependency, or external via `postgresql.external.url`
- `Secret` for credentials, mountable from external secret managers via annotations
- `NetworkPolicy` allowing egress to `api.anthropic.com`, GitHub, and configured MCP endpoints
- Health and readiness probes wired to `/api/v1/health` and `/api/v1/ready`
- `PodDisruptionBudget` and resource requests/limits with sensible defaults

### 10.3 Container Images

Published to `ghcr.io/<org>/agora-api` and `ghcr.io/<org>/agora-mcp-manifest`. Tagged with semver and `latest`. Multi-arch (amd64 + arm64). Distroless final stage. SBOM via `syft`.

### 10.4 Upgrade Path

Database migrations run automatically on API startup via `goose`. Manifest schema migrations run on first manifest read per project. Pipeline template schema migrations run on template load. All forward-only. Document breaking changes in `CHANGELOG.md`.

---

## 11. MVP Acceptance Criteria

**Admin onboarding** (one-time):

1. Admin clones the repo, runs `docker compose up`, reaches the UI in under 5 minutes.
2. Admin opens the Pipeline editor; the seed template is already loaded.
3. For each stage, the editor shows: "Create this agent in the Claude Console — here's the sample prompt." Admin opens the Console, creates each agent, returns and pastes the agent IDs.
4. Admin uploads the rubric files via the editor (one click per file → Files API).
5. Admin saves the template; validation passes.

**End-user flow** (the vibe-coder):

6. End user registers a project, supplying either a zip upload or a GitHub URL (with a read-only PAT in the project's vault if the source is private).
7. End user triggers a run; Source Capture creates a fresh project repository in Agora's managed destination, clones the source into it, and Gap Analysis starts within 10 seconds.
8. End user watches live progress via SSE: stage status, tool calls, findings, outcome verdicts.
9. The pipeline reaches `plan_review`; the run UI shows the Gap Analysis output (scorecard, findings, effort report, applicable stages) and indicates "awaiting reviewer."
10. *(Admin action)* The admin reviews and approves `plan_review`. The Specialist Coordinator stage runs, delegating to applicable specialists in parallel.
11. Coordinator reaches `satisfied` outcome; PRs are visible in the user's repo. The pipeline reaches `changes_review`.
12. *(Admin action)* The admin reviews the PRs and approves `changes_review`. The run is marked complete.
13. The project repository contains a populated `.agora/readiness.yaml` and the specialist PRs.
14. Re-running against the same project: Source Capture refreshes the project repository from the source (or no-ops on an unchanged source), and Gap Analysis picks up prior context from the project memory store.

In single-user mode (MVP default), the admin and end user are the same person, so steps 10 and 12 are self-service. In multi-user mode the gates fan out to whoever holds an admin role.

The bar for "complete" on the MVP is: **a developer with a Claude-Code-built Node.js or Python web app can hand it to Agora and get back a security-hardened repo with PRs ready to merge, with no manual configuration beyond admin onboarding (one-time) and the user's initial credentials. An admin who trusts the pipeline can disable both gates and run fully autonomously; the platform supports that out of the box.**

---

## 12. Phased Delivery

### Phase 0 — Project Skeleton (Week 1)

- Repo scaffolding, monorepo tooling, CI green
- License, contributing, security policy, ADR template
- API skeleton: health, readiness, OpenAPI auto-docs
- UI skeleton: routing, theme, empty pages
- Postgres + goose, base migrations
- Docker Compose runs the empty stack
- Anthropic Go SDK wired in; `agents.list()` proxy works

### Phase 1 — Project Lifecycle + Vaults + Memory (Week 2)

- Project registration: create vault, create per-project memory store, encrypted-Postgres path *not* introduced
- Manifest service: read/write `.agora/readiness.yaml` via GitHub
- Agora manifest MCP server, end-to-end tested
- Run model in DB, run lifecycle state machine
- SSE event stream for clients

### Phase 2 — Pipeline Runner + Mandatory Stages + Outcomes + Webhooks (Weeks 3–4)

- Pipeline template: Postgres-backed, YAML seed, validation, admin UI editor
- Pipeline runner: creates sessions per stage via the Go SDK, attaches vault + memory, streams events
- Webhook ingest: `Beta.Webhooks.Unwrap()`, signature verification, idempotency by `event.id`
- Outcomes: rubric upload via Files API, `user.define_outcome` per stage, verdict captured on the run
- Source Capture (link existing GitHub repos; zip ingestion deferred)
- Gap Analysis with `applicable_stages` output
- Checklist memory store: seeded with the production-readiness checklist; mounted read-only on every agent
- Sample agent prompts and rubrics shipped in `examples/`
- Gates 1 and 2 fully working

### Phase 3 — Specialist Coordinator (Wave 1) + Hardening (Week 5)

- Sample Specialist Coordinator prompt with Security in its roster
- Sample Security specialist prompt + rubric
- E2E test: register a known-vulnerable test repo, run end-to-end, verify Security PRs and outcome `satisfied`
- Helm chart, NetworkPolicy, secrets handling
- Audit logging
- Run history, replay, cancellation
- Documentation: self-hosting guide, admin guide
- v0.1.0 release

### Phase 4 — Wave 2 Specialists (Weeks 6–8)

- Sample prompts + rubrics for Infrastructure (Terraform target first) and CI/CD (GitHub Actions target first)
- Gate 3 for paid infrastructure provisioning
- Per-run scratch memory store (optional Wave-2 feature)
- Zip ingestion in Source Capture

### Phase 5 — Wave 3 Specialists (Weeks 9–11)

- Sample prompts + rubrics for Observability (Grafana Cloud target first), Testing, Compliance

---

## 13. Testing Strategy

### 13.1 Layers

- **Unit tests** — pure logic, no I/O. Target ≥80% coverage on `services/`.
- **Integration tests** — API + Postgres via testcontainers. Real database, mocked Anthropic and GitHub.
- **Pipeline template tests** — validate every committed `pipeline.example.yaml` against the schema; assert stage `kind` ordering rules.
- **Manifest schema tests** — round-trip every fixture through read/write/validate.
- **Webhook tests** — feed canned Anthropic webhook payloads through `Beta.Webhooks.Unwrap()`; assert idempotency by `event.id`.
- **End-to-end tests** — Run against a vendored fixture repo (`fixtures/sample-vulnerable-app/`) using the real Anthropic API behind a feature flag. Disabled in standard CI; runs nightly with secrets.

### 13.2 Outcomes and Determinism

Agents are non-deterministic. With graders, tests assert on the **rubric verdict** (`satisfied` vs `needs_revision`) rather than reverse-engineering shape from agent output. For unit tests of pipeline transitions, mock the grader's response. For E2E, rely on the real grader.

For tests that must inspect agent shape, assert the existence of structural artefacts (a finding of severity X exists, a PR was opened) — not exact content. Use response recording for fast tests; record once with the real API, replay in CI.

---

## 14. Observability of Agora Itself

The platform that hardens other apps must hold itself to the same standard.

- **Structured JSON logs** via `log/slog`, every request tagged with `run_id`, `project_id`, `stage`, `session_id`
- **Prometheus metrics** at `/metrics`: run counts, durations, finding counts, agent token usage, gate wait times, webhook ingest rate, webhook signature failures
- **OpenTelemetry traces** — propagate trace context into Managed Agents calls via headers
- **Audit log** — separate Postgres table, append-only, captures every gate approval, credential write to vault, PR opened, pipeline template edit
- **Runbook** — `docs/runbook.md` covering common failures: webhook unreachable, vault credential refresh failure, grader `failed` verdict, Anthropic rate limit

---

## 15. Security Requirements

- All end-user credentials stored in Anthropic vaults — Agora never persists them in cleartext or in Postgres.
- Per-project vault authorization enforced server-side: never accept user-supplied `vault_id`; look up from `projects.vault_id`.
- Webhook payloads verified via `Beta.Webhooks.Unwrap()`; reject anything older than five minutes.
- No credentials in logs, ever — explicit redaction in the `log/slog` handler.
- Cosign-signed container images.
- Dependabot enabled on the repo; weekly dependency PRs.
- `SECURITY.md` documents the disclosure process.
- Regular `trivy` scans in CI on the built images.
- The principle that gates the platform also gates contributors: no PR merges to `main` without review.

---

## 16. Documentation Requirements

Treat documentation as a first-class deliverable. Each phase ships with its docs.

- `README.md` — two-interface pitch, screenshot, quickstart, link to full docs
- `docs/self-hosting.md` — Docker Compose and Helm walkthroughs, webhook URL setup
- `docs/admin-guide.md` — authoring agents in the Claude Console, configuring Agora's pipeline, copying samples
- `docs/manifest-schema.md` — full manifest reference including `gap_analysis.applicable_stages`
- `docs/api.md` — generated from OpenAPI
- `docs/architecture.md` — component diagram, data flow, threat model
- `docs/runbook.md` — common failures and remediations
- `docs/adr/` — significant decisions (one per file, dated)

---

## 17. Open Questions to Resolve During Phase 0

These are intentionally left to the implementer to decide and document via ADR:

1. Multi-tenancy story: do we add workspaces in v0.1, or wait?
2. How aggressively does Agora cache GitHub API responses? (rate limits matter)
3. Do we build a CLI alongside the UI in Phase 1, or defer?
4. User-facing webhook callbacks for run completion — separate from the Anthropic ingest path; v0.1 or later?
5. Multi-user mode and the GitHub App integration — ship in v0.1, or defer? Confirm the Admin/End User endpoint split documented in §7.3 via ADR before shipping.

Open questions resolved during the 2026-05-09 architecture pivot (no ADR needed):

- Pipeline configuration storage → hybrid YAML seed + Postgres truth (§5.1).
- Agent authoring → admin owns it in the Claude Console; Agora ships samples in `examples/`.
- Specialist orchestration → multiagent coordinator (admin-authored).
- "Which specialists run for this project" → Gap Analysis writes `applicable_stages` to the manifest.
- Credential storage → Anthropic vaults; no AES path.
- Source Capture's working location → always a fresh Agora-managed project repository. Existing GitHub source repos are cloned, not worked on in-place. The destination organisation is admin-configured at the agent prompt level (not Agora env vars).

---

## 18. Out of Scope (Explicit)

- A managed/hosted version of Agora
- Anything other than English in the UI for MVP
- Mobile UI (responsive desktop is enough)
- A plugin marketplace (admins add specialists by authoring agents in the Console)
- Replacing the user's existing CI — Agora generates pipelines, doesn't run them
- Code generation for new features

---

## 19. Branding Compliance

Project Agora is built on Claude Managed Agents. Per Anthropic's branding guidelines for Managed Agents partners, the UI may say "Powered by Claude" but must not adopt Claude Code or Claude Cowork branding or visual elements. Pick original iconography, an original colour palette, and an original logo. The agora-themed visual language (columns, gathering places, Greek-inspired motifs) is a natural starting point but not prescriptive.

---

## 20. Definition of Done (per PR)

A PR cannot merge to `main` unless:

- [ ] CI is green (lint, type-check, unit tests, integration tests)
- [ ] New code has tests
- [ ] Changed behaviour is reflected in the relevant doc
- [ ] If a public API or schema changed: ADR added, `CHANGELOG.md` updated, manifest or pipeline migration written
- [ ] Reviewer has signed off
- [ ] No new secrets, hardcoded credentials, or PII in logs

---

*End of specification. v0.2, 2026-05-09.*
