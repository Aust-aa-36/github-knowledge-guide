# GitHub Knowledge Guide

Comprehensive reference for all GitHub platform components. Built for agent consumption and developer reference.

> Source: GitHub official documentation (https://docs.github.com/en)
> Generated: 2026-03-30

---

## Contents

| File | What it covers |
|---|---|
| [00-overview.md](00-overview.md) | Platform map, component quick-reference, decision trees, tier matrix |
| [01-repositories.md](01-repositories.md) | Repos, branches, forks, templates, releases, settings, limits |
| [02-issues-projects.md](02-issues-projects.md) | Issues, labels, milestones, Projects v2 GraphQL, automation |
| [03-pull-requests.md](03-pull-requests.md) | PRs, draft PRs, reviews, merge strategies, CODEOWNERS, templates |
| [04-actions.md](04-actions.md) | Workflows, triggers, runners, secrets, environments, caching, limits |
| [05-api.md](05-api.md) | REST API, GraphQL API, GitHub Apps, OAuth Apps, Webhooks |
| [06-security.md](06-security.md) | Auth tokens, branch protection, rulesets, Dependabot, CodeQL, OIDC |
| [07-packages-pages-codespaces.md](07-packages-pages-codespaces.md) | Packages/GHCR, Pages hosting, Codespaces, Copilot |
| [AGENT-REFERENCE.md](AGENT-REFERENCE.md) | **Fast lookup**: task→API mapping, gh CLI, GraphQL mutations, gotchas |

### Diagrams

| File | Diagrams included |
|---|---|
| [diagrams/01-platform-architecture.md](diagrams/01-platform-architecture.md) | Platform component map, CI/CD pipeline, PR lifecycle, auth decision tree, Projects v2 schema, branch protection flow, secret precedence, App auth sequence, branching strategy, webhook processing |

---

## Quick Start for Agents

**"What API call do I need?"** → [AGENT-REFERENCE.md](AGENT-REFERENCE.md)

**"How does X work?"** → corresponding numbered guide above

**"Show me the architecture"** → [diagrams/01-platform-architecture.md](diagrams/01-platform-architecture.md)

---

## Key Facts (Memorise These)

```
REST API base:        https://api.github.com
GraphQL endpoint:     POST https://api.github.com/graphql
Rate limit (PAT):     5,000 req/hour
Rate limit (Actions): 1,000 req/hour per repo
Rate limit (App):     5,000–15,000 req/hour

GITHUB_TOKEN:         auto-provided, repo-scoped, expires at job end
PAT fine-grained:     preferred over classic PAT
GitHub App:           preferred over PAT for production bots

addProjectV2ItemById: ✅ correct mutation (NOT addProjectV2Item)
File update via REST: requires sha from prior GET
Fork PRs:             NO access to upstream secrets
workflow_dispatch:    NO pull_request event context
```

---

## Common Gotchas

- `addProjectV2Item` is **deprecated** → use `addProjectV2ItemById`
- File `PUT` needs the current file's `sha` (GET it first)
- `GITHUB_TOKEN` **cannot** trigger new workflow runs (prevents loops)
- Fork PRs cannot access upstream repo secrets — use `pull_request_target` carefully
- GraphQL node IDs (`I_xxx`, `PVT_xxx`) are different from numeric issue/project numbers
- Branch protection rules with "Include administrators" must be explicitly enabled
- `continue-on-error: true` masks step failures — the job still passes

---

## Install as a Claude Code Skill

Pull this repo and install the `/github-guide` skill so any Claude Code session can use it:

```bash
# 1. Clone into your Claude skills directory
git clone https://github.com/Aust-aa-36/github-knowledge-guide.git ~/.claude/skills/github-knowledge-guide

# 2. Symlink the skill file so Claude Code picks it up automatically
ln -s ~/.claude/skills/github-knowledge-guide/.claude/skills/github-guide.md ~/.claude/skills/github-guide.md
```

**Windows (PowerShell)**:
```powershell
git clone https://github.com/Aust-aa-36/github-knowledge-guide.git "$env:USERPROFILE\.claude\skills\github-knowledge-guide"
New-Item -ItemType SymbolicLink `
  -Path "$env:USERPROFILE\.claude\skills\github-guide.md" `
  -Target "$env:USERPROFILE\.claude\skills\github-knowledge-guide\.claude\skills\github-guide.md"
```

Once installed, use in any Claude Code session:
```
/github-guide                   # list available topics
/github-guide actions           # Actions workflows reference
/github-guide graphql mutation  # Projects v2 GraphQL mutations
/github-guide api file update   # REST file update with SHA
/github-guide diagram           # render architecture diagrams
```

**Update the knowledge base** at any time:
```bash
git -C ~/.claude/skills/github-knowledge-guide pull
```

---

## Docs Reference

| Topic | URL |
|---|---|
| Full docs | https://docs.github.com/en |
| Actions events | https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows |
| GraphQL Explorer | https://docs.github.com/en/graphql/overview/explorer |
| REST API Reference | https://docs.github.com/en/rest |
| Actions Marketplace | https://github.com/marketplace?type=actions |
| GitHub Status | https://www.githubstatus.com |
