# GitHub Platform — Complete Overview

> **Purpose**: Master reference for agents and developers. Describes every major GitHub platform component, how they relate, and when to use each.

---

## Platform Map

```
GitHub Platform
├── Code Hosting
│   ├── Repositories (git + metadata)
│   ├── Branches, Commits, Tags
│   ├── Releases & Packages
│   └── Codespaces (cloud dev environments)
│
├── Collaboration
│   ├── Pull Requests
│   ├── Issues
│   ├── Discussions
│   └── Projects v2 (planning boards)
│
├── Automation
│   ├── GitHub Actions (CI/CD)
│   ├── Webhooks (event push)
│   └── GitHub Apps / OAuth Apps
│
├── Security
│   ├── Branch Protection / Rulesets
│   ├── Secrets & Environments
│   ├── Dependabot
│   ├── Secret Scanning
│   ├── Code Scanning (CodeQL)
│   └── CODEOWNERS
│
├── Publishing
│   ├── GitHub Pages (static sites)
│   └── GitHub Packages (container/npm/etc registry)
│
└── API Surface
    ├── REST API (v3)
    └── GraphQL API (v4)
```

---

## Component Quick Reference

| Component | What it is | Primary use case |
|---|---|---|
| **Repository** | Git repo + GitHub metadata (issues, PRs, wiki, settings) | Source of truth for code |
| **Branch** | Named pointer to a commit chain | Isolate work; protect main |
| **Pull Request** | Proposed merge from branch → base, with review workflow | Code review & controlled merging |
| **Issue** | Trackable work item (bug, feature, task) | Planning, bug tracking |
| **Discussion** | Threaded forum-style conversation | Q&A, announcements, RFCs |
| **Project (v2)** | Flexible board/table/roadmap view over issues+PRs | Sprint planning, roadmaps |
| **Milestone** | Group of issues/PRs with a due date | Release tracking |
| **Label** | Coloured tag on issues/PRs | Filtering, triage, automation |
| **GitHub Actions** | YAML-defined CI/CD workflows running on runners | Build, test, deploy, automation |
| **Runner** | Compute that executes Actions jobs | GitHub-hosted or self-hosted |
| **Workflow** | `.github/workflows/*.yml` — one or more jobs triggered by events | CI pipeline, release pipeline |
| **Secret** | Encrypted value available to Actions | API keys, tokens, passwords |
| **Environment** | Deployment target with protection rules & secrets | staging, production gates |
| **GITHUB_TOKEN** | Auto-issued short-lived token per workflow run | Repo operations within Actions |
| **PAT (classic)** | User-scoped long-lived token | Cross-repo automation, CLI |
| **Fine-grained PAT** | Resource-scoped short-lived token | Preferred over classic PAT |
| **GitHub App** | Installable integration with granular permissions | Third-party services, bots |
| **OAuth App** | User-delegated access via OAuth flow | User-facing integrations |
| **Webhook** | HTTP POST on GitHub events → external URL | Real-time event integration |
| **REST API** | HTTP endpoints for all GitHub resources | Scripting, integrations |
| **GraphQL API** | Flexible query language — fetch exactly what you need | Projects v2, dashboards |
| **Releases** | Tagged snapshot with binaries and release notes | Software distribution |
| **Packages** | GitHub-hosted artifact registry (npm, Docker, Maven, etc.) | Internal package distribution |
| **Pages** | Static site hosting from repo | Docs, project sites |
| **Codespaces** | Cloud VS Code environment pre-configured from repo | Instant dev environments |
| **Dependabot** | Automated dependency PR creation + security alerts | Dependency management |
| **Branch Protection** | Rules enforced before merge (reviews, checks, signing) | Code quality gates |
| **Ruleset** | New generation of branch/tag protection (org-wide, layered) | Enterprise branch governance |
| **CODEOWNERS** | File mapping paths → required reviewers | Ownership enforcement |
| **Code Scanning** | Static analysis (CodeQL + 3rd party) | Security vulnerability detection |
| **Secret Scanning** | Detects accidentally committed secrets | Security compliance |

---

## Tier / Plan Matrix

| Feature | Free (personal) | Free (org) | Team | Enterprise Cloud |
|---|---|---|---|---|
| Public repos | ✅ Unlimited | ✅ Unlimited | ✅ | ✅ |
| Private repos | ✅ Unlimited | ✅ Unlimited | ✅ | ✅ |
| Actions minutes/mo | 2,000 | 2,000 | 3,000 | 50,000 |
| Packages storage | 500 MB | 500 MB | 2 GB | 50 GB |
| Branch protection | ✅ | ✅ | ✅ | ✅ |
| Required reviewers | ✅ | ✅ | ✅ | ✅ |
| Internal repos | ❌ | ❌ | ❌ | ✅ |
| SAML SSO | ❌ | ❌ | ❌ | ✅ |
| Audit log API | ❌ | ❌ | ❌ | ✅ |
| Advanced Security (GHAS) | ❌ | ❌ | ❌ | ✅ add-on |

---

## Key Relationships

```
Organization
└── Teams (groups of members with repo permissions)
    └── Repositories (owned by org or user)
        ├── Branches → Pull Requests → Merges
        ├── Issues ←→ Projects (cross-repo boards)
        ├── .github/workflows/*.yml → Actions Runs
        │   └── Uses: Secrets, Environments, GITHUB_TOKEN
        ├── Releases → Packages (artifacts)
        ├── Webhooks → External services
        └── Apps (GitHub App installations)
```

---

## Decision Tree — Which auth token to use?

```
Need to authenticate to GitHub API or git?
│
├── In a GitHub Actions workflow?
│   └── Use GITHUB_TOKEN (auto-provided, least privilege)
│       └── Need cross-repo or admin perms?
│           └── Store PAT or App token as repo/org secret
│
├── Building a bot/integration that acts as itself (not a user)?
│   └── Use GitHub App (preferred — fine-grained, short-lived tokens)
│
├── CLI / personal scripting?
│   └── Use Fine-grained PAT (prefer over classic PAT)
│       └── Legacy/broad-access need?
│           └── Classic PAT (avoid if possible)
│
└── User-facing web app needing GitHub access?
    └── OAuth App (user delegates access via browser flow)
```

---

## Decision Tree — Which event/automation to use?

```
Need to react to a GitHub event?
│
├── Within GitHub (trigger workflow)?
│   └── GitHub Actions with event trigger (push, pull_request, issues…)
│
├── Send to external system in real-time?
│   └── Webhook → external HTTP endpoint
│
├── Poll or act on schedule?
│   └── GitHub Actions `schedule:` cron trigger
│
├── Trigger from external system into GitHub?
│   └── repository_dispatch event via REST API
│       OR workflow_dispatch via REST API
│
└── Complex integration (read+write, many repos)?
    └── GitHub App with webhook + API calls
```

---

## GitHub Docs Quick Links

| Topic | URL |
|---|---|
| Repositories | https://docs.github.com/en/repositories |
| Actions | https://docs.github.com/en/actions |
| Issues | https://docs.github.com/en/issues |
| Projects v2 | https://docs.github.com/en/issues/planning-and-tracking-with-projects |
| Pull Requests | https://docs.github.com/en/pull-requests |
| REST API | https://docs.github.com/en/rest |
| GraphQL API | https://docs.github.com/en/graphql |
| Authentication | https://docs.github.com/en/authentication |
| GitHub Apps | https://docs.github.com/en/apps |
| Actions Security | https://docs.github.com/en/actions/security-for-github-actions |
| Branch Protection | https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches |
| GitHub Pages | https://docs.github.com/en/pages |
| GitHub Packages | https://docs.github.com/en/packages |
| Codespaces | https://docs.github.com/en/codespaces |
