# Githena — Business Overview

## What it is

Githena is a config-driven, event-driven, multi-agent GitHub PR reviewer. It listens for pull requests marked ready for review, runs the diff through a small team of specialist AI agents in parallel, and posts a single supervisor-gated review (Approve, Request Changes, or Comment) back to the PR within seconds.

It is not a general-purpose code reviewer. It is a *policy* reviewer. Every project's review behavior — what counts as a block, what language the team uses, what the domain is about — is declared in three markdown files in the target repo. Adding a new project or changing an existing one's policy is a config edit, never a code change.

## Problem

Code review in a fast-moving monorepo is the bottleneck.

- A human reviewer is one person, one context window, one set of biases.
- A single general-purpose LLM reviewer is too coarse. It does not know the project's vocabulary, does not enforce org-specific rules, conflates style with correctness with security, and gives inconsistent output across PRs.
- Existing tools (CodeRabbit, Sourcery, et al.) treat every project the same. They are capability-driven, not policy-driven.

The gap: there is no review tool whose behavior is *defined by the project it reviews* and *auditable from git history alone*.

## Solution

Githena replaces the single reviewer with a small, observable system:

- A webhook listener that fires on `pull_request.ready_for_review` (and `reopened`, `synchronize`).
- Five specialist agents running in parallel: nomenclature, style, correctness, test, security.
- One supervisor agent that sees all five verdicts, checks cross-cutting rules, and emits one final review event.
- A private review log repo recording every diff, every verdict, and the supervisor's reasoning — full auditability.

Each project is governed by three config files: `project-context.md` (domain), `git-nomenclature.md` (conventions), `rules.md` (thresholds and hard blocks). The agent's behavior is shaped entirely by these files plus the agent's role prompt. Change the policy, the review changes. The agents do not need to be retrained, recompiled, or redeployed.

## Tech stack

Minimal by design.

- **Runtime:** GitHub Actions on a self-hosted runner. The "service" is a workflow file, not a long-lived process.
- **Listener:** Python 3.11+ / Flask (~200 lines). Validates HMAC, dispatches the workflow.
- **Workflow scripts:** TypeScript / Node 20. Compiled to `dist/`, executed by the workflow steps.
- **Multi-agent layer:** Six Mavis sub-agents (one supervisor, five specialists) called via the Mavis API.
- **Storage:** Two GitHub repos — the Githena source repo, and a private review log repo. No database, no queue, no external state store.
- **Config:** YAML + markdown. Version-controlled in the same repo as the source.

## Vendor-agnostic multi-agent layer

The specialist and supervisor agents are the only component tied to a model vendor, and even that is loose.

**The contract is the API, not the model.** The workflow calls the Mavis API by agent name and passes a task input. The agent's behavior is defined by its system prompt (in Mavis) plus the JSON output schema (in the prompt). The workflow does not know which model is behind the agent.

**Swapping a vendor is a Mavis-side change, not a code change.** Want to use Claude for correctness and GPT for security and a local model for nomenclature? Configure the Mavis agents accordingly and Githena calls them by the same names. The workflow is unchanged.

**Swapping the entire Mavis layer is a single config line.** The API base URL and auth header are read from a single place. Pointing Githena at a different agent platform — or a self-hosted model gateway that speaks the same shape — is one environment variable.

**Model upgrades are free.** When Mavis updates the underlying model, the agents' behavior may improve but their *contract* (JSON verdict, fixed schema) does not change. Githena's workflow validation logic does not need to change. The only artifact that may need updating is the agent's system prompt, and that is a Mavis-side edit, not a Githena release.

**Lock-in surface is one prompt file per agent.** Githena's git history contains the system prompts. If you leave Mavis, you have the prompts. Pointing them at a different platform is a copy-paste.

## Architecture

```
        ┌──────────────────────────────────────────────────────────────┐
        │  GitHub (target repos)                                       │
        │  PR marked ready_for_review → webhook fired                 │
        └─────────────────────────────┬────────────────────────────────┘
                                      │ HMAC-signed POST
                                      ▼
        ┌──────────────────────────────────────────────────────────────┐
        │  Self-hosted runner                                          │
        │  ┌────────────────────────────────────────────────────────┐  │
        │  │  Listener (Flask)                                       │  │
        │  │  - validate HMAC                                        │  │
        │  │  - filter event (ready_for_review / reopened / sync)     │  │
        │  │  - call GitHub workflow_dispatch                        │  │
        │  └────────────────────────────┬───────────────────────────┘  │
        └─────────────────────────────┬─�────────────────────────────────┘
                                      │ trigger
                                      ▼
        ┌──────────────────────────────────────────────────────────────┐
        │  GitHub Actions workflow (review.yml)                        │
        │  on: workflow_dispatch                                        │
        │  runs-on: [self-hosted, githena]                             │
        │                                                              │
        │  ┌──────────────────────────────────────────────────────┐    │
        │  │  Step 1: load org.yaml → identify project, repo, PAT  │    │
        │  │  Step 2: fetch PR diff (using repo-specific PAT)      │    │
        │  │  Step 3: load project-context.md, nomenclature, rules │    │
        │  │  Step 4: snapshot merged config to review log         │    │
        │  └──────────────────────────────────────────────────────┘    │
        │                          │                                   │
        │                          ▼                                   │
        │  ┌──────────────────────────────────────────────────────┐    │
        │  │  Step 5: dispatch specialists in parallel             │    │
        │  │  (Mavis API, async)                                   │    │
        │  │  ┌───────────────┐  ┌───────────────┐  ┌────────────┐  │   │
        │  │  │ githena-      │  │ githena-      │  │ githena-   │  │   │
        │  │  │ nomenclature  │  │ style         │  │ correctness│  │   │
        │  │  └───────────────┘  └───────────────┘  └────────────┘  │   │
        │  │  ┌───────────────┐  ┌───────────────┐                  │   │
        │  │  │ githena-test   │  │ githena-      │                  │   │
        │  │  │               │  │ security      │                  │   │
        │  │  └───────────────┘  └───────────────┘                  │   │
        │  └──────────────────────────┬───────────────────────────┘    │
        │                             ▼                                │
        │  ┌──────────────────────────────────────────────────────┐    │
        │  │  Step 6: supervisor                                  │    │
        │  │  (Mavis API: githena-supervisor)                     │    │
        │  │  input: 5 verdicts + rules.md                       │    │
        │  │  output: APPROVE | REQUEST_CHANGES | COMMENT         │    │
        │  └──────────────────────────┬───────────────────────────┘    │
        │                             ▼                                │
        │  ┌──────────────────────────────────────────────────────┐    │
        │  │  Step 7: post review via GitHub API (repo PAT)        │    │
        │  │  Step 8: commit review log entry                      │    │
        │  │  (githena-review-log repo, separate PAT)              │    │
        │  └──────────────────────────────────────────────────────┘    │
        └──────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
        ┌──────────────────────────────────────────────────────────────┐
        │  Storage                                                     │
        │  githena/                  → source, workflows, config       │
        │  githena-review-log/       → reviews/, config-snapshots/,     │
        │                              audit/                          │
        └──────────────────────────────────────────────────────────────┘
```

## Why this works

1. **Latency.** Webhook to review in under 60 seconds. No polling, no queue.
2. **Onboarding.** New repo = three config files + one PAT + a Mavis agent check. No Githena code change.
3. **Auditability.** Every review has a corresponding log entry: full diff, all five verdicts, supervisor reasoning, final decision. Answerable from git history alone.
4. **Safety.** Per-repo mode (enforce or comment-only). Hard caps on review actions. PATs scoped per repo, never logged, never committed.
5. **Vendor optionality.** The multi-agent layer is one API call deep. Swap models, swap vendors, swap platforms — the Githena workflow does not change.
