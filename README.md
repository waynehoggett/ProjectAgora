# Project Agora

> The production readiness platform for AI-built apps. From zipped archive or GitHub repository, right through to deployment.

<!-- Badges (to be completed)
![License](https://img.shields.io/badge/license-GPL--3.0-blue)
![Status](https://img.shields.io/badge/status-early--development-orange)
![Powered by Claude](https://img.shields.io/badge/powered%20by-Claude-purple)
-->

Project Agora is an open-source platform that takes shadow-IT applications — typically built quickly with AI coding assistants like Claude Code or Codex — and runs them through a fleet of specialist AI agents that handle the production-readiness work the original developer didn't.

## Two interfaces, one job

Agora has two audiences:

- **Admins** define what *production-ready* means for their company — the checks, the agents that run, the gates that need a reviewer, and who reviews them. Admins author every agent in their [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview) workspace once, configure Agora's pipeline template, and approve at any gates the pipeline pauses on.
- **End users** (the "vibe-coders") drop their app in (zip or GitHub URL), provide credentials, and watch the run progress. They see what's happening to their code at every stage. They don't configure agents and they don't approve gates.

**Agora is fully agentic by default and uses humans-in-the-loop sparingly.** A run that goes from drop-zone to production-ready PRs without anyone touching it is the goal. Reality is that some decisions — the plan being correct before any code is changed, an irreversible deploy — warrant a human reviewer. Gates exist for those moments, and admins choose where to place them. Treating every transition as a gate defeats the point.

Agora itself is small on purpose: it picks agents the admin has already defined in the Claude Console, runs them in the configured order, and gets out of the way. Orchestration, success criteria, persistent memory, and credential storage all live in Managed Agents primitives — not in Agora's source.

## About the name

*Agora* — the Greek word for a public gathering place. In ancient Athens, the agora was where citizens, craftspeople, and philosophers came together to trade, debate, and prepare for the work ahead. In Project Agora, it's where shadow-IT apps gather their fellowship of specialist agents before crossing into production.

---

## The problem

Modern AI coding assistants are very good at producing working software. They are not as good at surfacing the non-functional requirements that an experienced engineering team would treat as non-negotiable before a production launch:

- Secrets management, authentication, and dependency security
- Infrastructure as code, CI/CD, and environment configuration
- Observability — logs, metrics, traces, alerts, runbooks
- Tests, code review, and deploy gates
- Compliance and data-flow documentation

The deepest problem is usually that the builder doesn't know what they don't know. Project Agora makes that "last mile" concrete, automatable, and auditable — without taking ownership of your code, infrastructure, or data.

---

## How it works

Agora takes your shadow-IT app — as a **zipped archive or a GitHub repository** — and walks it through every stage from gap analysis to deployment.

For the end user it's one action: drop the app in. They watch the run progress; admins (or admin-designated reviewers) approve at any gates the pipeline pauses on. Behind the scenes, every Agora pipeline begins with two **mandatory** agent stages:

1. **Source Capture** — Agora always creates a fresh GitHub repository in an admin-configured managed location and clones your code into it, regardless of whether you uploaded a zip or pasted a GitHub URL. Your original source is never written to. From this point on, the **project repository** (Agora-managed) is the single source of truth.
2. **Gap Analysis** — Agora's analyst agent reviews the captured code against the production-readiness checklist and hands you back a scorecard, an **effort report** estimating the work needed to close each gap, and the list of specialist agents that apply to your project.

After Gate 1, a single **Specialist Coordinator** stage runs: an admin-authored multiagent coordinator that delegates in parallel to whichever specialists Gap Analysis flagged — opening pull requests, provisioning infrastructure, instrumenting observability, and so on. Every specialist works on a shared filesystem within one session, and every change is recorded in `.agora/readiness.yaml` in your repo as a portable, human-readable audit trail.

Each agent has a defined success outcome, evaluated by an automated grader before the run advances. Agents iterate until the rubric is satisfied or hand back specific gaps for the next attempt — so "production-ready" is checked, not assumed.

---

## For admins: configure once

Admins author agents directly in the Claude Console — system prompts, tools, MCP servers, skills, models. Agora ships **sample prompts and rubrics** in [`examples/`](examples/) for every stage; the admin UI walks you through copying each one into the Console, customising it, and pointing Agora's pipeline template at the agents you've created.

The pipeline template itself is a short YAML on first boot, then editable in the admin UI:

| Stage | What it does |
| --- | --- |
| **Source Capture** *(mandatory)* | Creates a fresh project repository in the managed destination and clones the user's source into it |
| **Gap Analysis** *(mandatory)* | Produces scorecard, effort report, and the list of applicable specialists |
| **Gate: plan_review** *(optional, recommended)* | Pause for an admin to approve the plan before any changes |
| **Specialist Coordinator** | Multiagent coordinator delegating in parallel to applicable specialists |
| **Gate: changes_review** *(optional, recommended)* | Pause for an admin to review the resulting PRs |

Gates are optional. An admin who trusts the pipeline can remove both and run fully autonomously; an admin who needs tighter review can add more.

The "applicable specialists" set comes from Gap Analysis, not from admin config — so a project that already has Terraform doesn't waste a session on infrastructure work, and one that's missing observability gets it.

Common specialist roles admins author:

- **Security** — secrets, dependency audit, auth hardening
- **Infrastructure** — Terraform/Bicep generation, provisioning plans
- **CI/CD** — pipelines, environments, deployment gates
- **Observability** — logs, metrics, traces, alerts, runbook stubs
- **Testing** — test scaffolding and coverage analysis
- **Compliance** — data-flow mapping and framework gap analysis

Sample prompts for each ship in [`examples/agent-prompts/specialists/`](examples/agent-prompts/specialists/).

---

## For end users: drop and go

You provide:

1. Your app — zip upload or GitHub URL. If your source is a private GitHub repo, a read-only PAT so Agora can clone it once.
2. Any target-environment credentials the agents will need — cloud account, observability token, etc. (stored in an Anthropic vault, never in Agora's database)

You watch your app move through three states in the dashboard:

- **Being worked on by agents** — capture, analysis, and remediation. The dashboard shows which stage is active and the findings as they appear.
- **Awaiting admin approval** — if the admin's pipeline includes review gates, the run pauses here. The dashboard tells you what's being reviewed and by whom; the approve button is theirs.
- **Deployed** — if the admin's pipeline includes a deploy stage, the run finishes with your app live in its target environment. Otherwise the run finishes production-ready, sitting in an Agora-managed project repository awaiting your own deploy.

Either way you get a complete audit trail in `.agora/readiness.yaml` and your original source is untouched throughout.

---

## Quickstart

> **🚧 To be completed.** This section will be filled in as the v0.1 release stabilises.

```bash
# Placeholder
git clone https://github.com/<org>/agora.git
cd agora
cp config/example.env .env
# Add your ANTHROPIC_API_KEY, AGORA_API_TOKEN, and ANTHROPIC_WEBHOOK_SIGNING_KEY
docker compose -f deploy/docker-compose.yml up -d
# Open http://localhost:8080 — the pipeline editor walks you through
# creating the sample agents in your Claude Console workspace.
```

---

## Self-hosting

Project Agora is designed to be run in your own environment. The repo ships with:

- **Docker Compose** for local single-machine deployment
- **Helm chart** for Kubernetes

> **🚧 To be completed.** Full self-hosting guides will live in [`docs/self-hosting.md`](docs/self-hosting.md).

---

## Documentation

- [`SPEC.md`](SPEC.md) — full technical specification
- [`docs/architecture.md`](docs/architecture.md) — *to be completed*
- [`docs/self-hosting.md`](docs/self-hosting.md) — *to be completed*
- [`docs/admin-guide.md`](docs/admin-guide.md) — authoring agents in the Console + configuring Agora's pipeline — *to be completed*
- [`docs/manifest-schema.md`](docs/manifest-schema.md) — readiness manifest reference — *to be completed*

---

## Status

Project Agora is in early development. The MVP targets a security-hardened repo with PRs ready to merge for a Claude-Code-built Node.js or Python web app, with no manual configuration beyond admin onboarding (one-time) and the user's initial credentials.

See [`SPEC.md`](SPEC.md) §12 for the phased delivery plan.

---

## Contributing

Contributions may be accepted at later date. Please read [`CONTRIBUTING.md`](CONTRIBUTING.md) — *to be completed*.

For significant changes, open an issue first to discuss what you'd like to change.

---

## Security

Found a vulnerability? Please follow the disclosure process in [`SECURITY.md`](SECURITY.md) — *to be completed* — rather than opening a public issue.

---

## License

Project Agora is licensed under the [GNU General Public License v3.0](LICENSE).

This is a strong copyleft licence: you are free to run, study, modify, and redistribute Project Agora, but any derivative work must also be released under the GPL-3.0.

---

## Acknowledgements

Built on [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview) by Anthropic — multiagent orchestration, outcomes, memory stores, vaults, and webhooks.

*Powered by Claude.*
