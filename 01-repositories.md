# GitHub Repositories

> Git repository + GitHub metadata layer: issues, PRs, wiki, settings, security, Actions.

---

## Repository Visibility Types

| Type | Who can see | Available on |
|---|---|---|
| **Public** | Anyone on the internet | All plans |
| **Private** | Owner + explicitly granted users/teams | All plans |
| **Internal** | All members of the enterprise | GitHub Enterprise Cloud only |

**Key point**: Forking a private repo creates a private fork. Forking a public repo creates a public fork by default.

---

## Core Repository Components

### Code & Git
| Component | Description |
|---|---|
| **Default branch** | Usually `main`; the branch PRs target and that is shown by default |
| **Branch** | Named pointer to a commit; lightweight; cheap to create |
| **Tag** | Immutable pointer to a commit; used for releases |
| **Commit** | Snapshot of tracked files + author/message metadata |
| **Tree / Blob** | Git internals — directory structure and file contents |
| **Submodule** | Reference to a commit in another repo; embedded at a path |
| **.gitignore** | Patterns of files git should not track |
| **.gitattributes** | Per-path git settings (line endings, diff drivers, LFS pointers) |

### GitHub Metadata Layer
| Component | Description |
|---|---|
| **Issues** | Bug reports, feature requests, tasks |
| **Pull Requests** | Proposed merges with review workflow |
| **Discussions** | Forum-style Q&A / announcements |
| **Wiki** | Freeform documentation pages (separate git repo under the hood) |
| **Projects** | Board/table views referencing issues+PRs (live in org or user account) |
| **Releases** | Tagged snapshots with notes and binary attachments |
| **Packages** | Artifact registry tied to the repo |
| **Actions** | CI/CD workflows in `.github/workflows/` |
| **Security** | Advisories, Dependabot, secret scanning, code scanning |
| **Settings** | Repo configuration: visibility, branch protection, deploy keys, webhooks |
| **Insights** | Traffic, contributors, dependency graph, pulse |

---

## Repository Structure Conventions

```
repo-root/
├── .github/
│   ├── workflows/          # Actions workflow YAML files
│   ├── ISSUE_TEMPLATE/     # Issue templates (.md or .yml forms)
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── CODEOWNERS          # Auto-assign reviewers by path
│   ├── dependabot.yml      # Dependabot config
│   └── FUNDING.yml         # Sponsor links
├── .gitignore
├── .gitattributes
├── README.md               # Shown on repo homepage
├── LICENSE
├── CONTRIBUTING.md
├── SECURITY.md             # Vulnerability reporting policy
└── CODE_OF_CONDUCT.md
```

---

## Branching Strategies

### Trunk-Based Development (recommended for CI/CD)
- Single `main` branch; short-lived feature branches merged frequently
- Feature flags for in-progress work
- Best for: teams with strong CI/CD and automated testing

### GitHub Flow
- `main` is always deployable
- Feature branches → PR → review → merge to `main` → deploy
- Best for: web apps, continuous deployment

### Git Flow
- Long-lived `develop` + `main`; `feature/`, `release/`, `hotfix/` branches
- Best for: versioned software with scheduled releases

### Release Branching
- `main` is development; `release/1.x` branches cut per release
- Hotfixes cherry-picked to both branches
- Best for: multi-version maintenance

---

## Forks

| Aspect | Detail |
|---|---|
| Purpose | Contribute to repos you don't have write access to |
| Relationship | Fork tracks upstream; can sync/pull upstream changes |
| PRs | Can open PRs from fork branch → upstream base branch |
| Visibility | Fork of public = public; fork of private = private |
| Actions | Actions in forks do NOT have access to upstream secrets |
| Limits | Organisation can restrict forking of private repos |

---

## Repository Templates

- Mark any repo as a **template** (Settings → Template repository)
- Users can generate a new repo with the same directory structure and files
- Does **not** copy: git history, issues, PRs, wiki, releases, stars
- Useful for: project scaffolding, standardised repo structures across org

---

## Repository Limits

| Resource | Limit |
|---|---|
| Recommended repo size | < 1 GB (soft) |
| Hard file size limit (without LFS) | 100 MB per file |
| Git LFS file size | Up to 2 GB per file |
| Max push size | 2 GB |
| Wikis | Treated as separate repo; same limits |
| Actions workflow files | 1,000 per repo |
| Actions concurrent jobs | 20 (free), 40 (pro), 60+ (team/enterprise) |

---

## Releases

- Created by tagging a commit (or creating a tag inline)
- Can attach binary assets (`.zip`, `.exe`, `.tar.gz`)
- Support Markdown release notes (auto-generated from PRs/commits)
- **Latest release** badge links to most recent non-prerelease
- Pre-release flag: marks alpha/beta; excluded from "latest"

### Release Process (best practice)
```
1. Tag commit:  git tag -a v1.2.0 -m "Release v1.2.0" && git push --tags
2. Create release from tag on GitHub UI or via API
3. Attach artifacts (CI can do this automatically via actions/upload-artifact + gh release upload)
4. Publish (draft → publish)
```

---

## Repository Settings — Key Options

| Setting | Location | Purpose |
|---|---|---|
| Default branch | General | Change from `main` to something else |
| Merge options | General | Allow/disable merge commits, squash, rebase |
| Auto-delete head branches | General | Clean up branches after PR merge |
| Vulnerability alerts | Security | Enable Dependabot alerts |
| Dependabot security updates | Security | Auto-PRs for CVE patches |
| Branch protection rules | Branches | Enforce checks before merge |
| Rulesets | Rules | Next-gen layered protection rules |
| Deploy keys | Security | Read-only or read-write SSH key for deploy systems |
| Webhooks | Webhooks | Push events to external URLs |
| GitHub Apps | Integrations | Install third-party apps |
| Actions permissions | Actions | Control which actions can run |
| Environments | Environments | Deployment targets with protection gates |

---

## Topics

- Repos can have up to **20 topics** (tags like `python`, `devops`, `azure`)
- Make repos discoverable via GitHub search and topic pages
- Set via Settings → General → Topics or via API

---

## Archived Repositories

- Read-only; no issues, PRs, pushes
- Still visible, cloneable, forkable
- Use for: end-of-life projects, keeping history without maintenance burden
- Unarchiving restores full functionality

---

## Repository Transfers

- Moves repo to another user or organisation
- All issues, PRs, labels, milestones, releases are preserved
- Redirects maintained for old URL (unless new repo created at old path)
- Stars, watchers **not** transferred
- Team access must be re-granted in new org

---

## Key API Endpoints (REST)

```
GET  /repos/{owner}/{repo}                   # Repo metadata
GET  /repos/{owner}/{repo}/branches          # List branches
GET  /repos/{owner}/{repo}/commits           # List commits
POST /repos/{owner}/{repo}/releases          # Create release
GET  /repos/{owner}/{repo}/contents/{path}   # Get file contents
PUT  /repos/{owner}/{repo}/contents/{path}   # Create/update file
GET  /repos/{owner}/{repo}/topics            # Get topics
```
