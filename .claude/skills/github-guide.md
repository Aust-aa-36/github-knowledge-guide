# Skill: github-guide

When the user invokes `/github-guide` (with or without arguments), load and apply the GitHub platform knowledge guide to assist with any GitHub-related task.

---

## Trigger Conditions

Activate this skill when the user:
- Explicitly runs `/github-guide`
- Asks about GitHub APIs, Actions workflows, Projects v2, authentication, webhooks, branch protection, or repository settings
- Needs a `gh` CLI command, REST endpoint, or GraphQL mutation
- Is debugging a GitHub Actions failure or a rate-limit issue
- Is building a GitHub App, OAuth App, or automation bot

---

## What This Skill Provides

This skill gives you access to a comprehensive GitHub platform knowledge base stored in the repo at `github-knowledge-guide/`. Use the files below as your primary reference — always read from them rather than relying on training memory alone, as they contain curated, gotcha-annotated content.

### File Map

| Question type | Read this file |
|---|---|
| "What GitHub feature should I use?" | `00-overview.md` — decision trees, tier matrix, component map |
| "How do repositories / branches / releases work?" | `01-repositories.md` |
| "How do Issues / Projects v2 work?" | `02-issues-projects.md` |
| "How do Pull Requests / CODEOWNERS / reviews work?" | `03-pull-requests.md` |
| "How do Actions / workflows / secrets / runners work?" | `04-actions.md` |
| "What REST / GraphQL call do I need?" | `05-api.md` and `AGENT-REFERENCE.md` |
| "How do I secure this / set up branch protection?" | `06-security.md` |
| "How do Packages / Pages / Codespaces / Copilot work?" | `07-packages-pages-codespaces.md` |
| "Show me the architecture" | `diagrams/01-platform-architecture.md` |
| **Fast lookup: task → exact API call** | `AGENT-REFERENCE.md` ← start here for automation tasks |

---

## Behaviour When Invoked

### `/github-guide` (no argument)
Print a summary of available topics and ask the user what they need help with. Offer the quick-reference categories above.

### `/github-guide <topic>`
Examples:
- `/github-guide actions` → read `04-actions.md`, answer question
- `/github-guide projects v2` → read `02-issues-projects.md` + `AGENT-REFERENCE.md`
- `/github-guide api file update` → read `AGENT-REFERENCE.md` REST file operations section
- `/github-guide branch protection` → read `06-security.md`
- `/github-guide graphql mutation` → read `AGENT-REFERENCE.md` Projects v2 mutations section
- `/github-guide diagram` → render the Mermaid diagrams from `diagrams/01-platform-architecture.md`

### Automation / agent tasks
When helping an agent write GitHub Actions YAML or API calls:
1. Always check `AGENT-REFERENCE.md` Gotchas table first
2. Use `addProjectV2ItemById` (NOT `addProjectV2Item`)
3. Always GET file SHA before PUT
4. Warn if the task needs `GITHUB_TOKEN` to trigger another workflow (it can't — needs PAT/App)
5. Warn if workflow uses `workflow_dispatch` and tries to access `github.event.pull_request`

---

## Key Facts (Always Apply)

```
REST base:            https://api.github.com
GraphQL endpoint:     POST https://api.github.com/graphql
Rate limit (PAT):     5,000 req/hour
Rate limit (Actions): 1,000 req/hour per repo
Rate limit (App):     5,000–15,000 req/hour

Correct mutation:     addProjectV2ItemById  (NOT addProjectV2Item — deprecated)
File update:          requires sha from prior GET /contents/{path}
GITHUB_TOKEN:         repo-scoped only, cannot trigger downstream workflows
Fork PRs:             no access to upstream secrets
workflow_dispatch:    no pull_request event context
```

---

## Installation

See repo README for full install instructions:
```
https://github.com/Aust-aa-36/github-knowledge-guide
```

Quick install:
```bash
# Clone the skill into your Claude skills directory
git clone https://github.com/Aust-aa-36/github-knowledge-guide.git ~/.claude/skills/github-knowledge-guide

# Symlink the skill file so Claude Code picks it up
ln -s ~/.claude/skills/github-knowledge-guide/.claude/skills/github-guide.md ~/.claude/skills/github-guide.md
```

Or manually copy `.claude/skills/github-guide.md` to `~/.claude/skills/`.

To update the knowledge base: `git -C ~/.claude/skills/github-knowledge-guide pull`
