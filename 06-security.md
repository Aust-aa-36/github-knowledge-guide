# GitHub Security

> Authentication, access control, branch protection, secret management, and code/dependency scanning.

---

## Authentication Methods

### Personal Access Tokens (PATs)

| Type | Scope | Lifetime | Best for |
|---|---|---|---|
| **Classic PAT** | Coarse GitHub-level scopes (`repo`, `workflow`, `read:org`…) | Until revoked | Legacy scripts; avoid for new work |
| **Fine-grained PAT** | Per-repo + per-permission | Configurable expiry (required for orgs) | Preferred for scripts/CI |

**Classic PAT scopes** (key ones):
```
repo              Full repo access (code, issues, PRs, releases)
repo:status       Commit statuses
read:org          Read org membership
admin:org         Org management
workflow           Manage Actions workflows
packages          Read/write Packages
gist              Create/update gists
notifications     Read notifications
delete_repo       Delete repos (not in repo scope!)
```

**Fine-grained PAT permissions** (per-repo):
```
Contents: read / write
Issues: read / write
Pull requests: read / write
Actions: read / write
Secrets: read / write
Metadata: read (always required)
...and 20+ more resource types
```

**Generate PAT**:
- Settings → Developer settings → Personal access tokens
- Fine-grained: Settings → Developer settings → Personal access tokens → Fine-grained tokens
- Set expiry; store in password manager or secrets vault

### SSH Keys

```bash
# Generate
ssh-keygen -t ed25519 -C "your_email@example.com"

# Add to agent
ssh-add ~/.ssh/id_ed25519

# Add public key to GitHub
# Settings → SSH and GPG keys → New SSH key
```

Use SSH for git operations (`git clone git@github.com:owner/repo.git`). Cannot use SSH for API calls.

### Deploy Keys

- Repo-scoped SSH key (read-only or read-write)
- Set in: Repo Settings → Security → Deploy keys
- Use for: CI systems, servers that need git access to one specific repo
- Different from personal SSH keys — not tied to a user account

### GITHUB_TOKEN (Actions)

```yaml
permissions:
  contents: write       # Push commits, create releases
  issues: write         # Comment, label, close issues
  pull-requests: write  # Comment, approve, merge PRs
  packages: write       # Push to GHCR
  id-token: write       # OIDC for cloud providers (Azure, AWS, GCP)
  checks: write         # Create check runs
  statuses: write       # Create commit statuses
  deployments: write    # Create deployments
  actions: read         # Read workflow runs
```

Default is `read` for most; `write` only when declared.

### OIDC (OpenID Connect) for Cloud Auth

Prefer OIDC over long-lived cloud credentials in Actions:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  # Azure
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  # AWS
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/my-role
      aws-region: ap-southeast-2
```

No static secrets stored — GitHub exchanges the OIDC token with the cloud provider at runtime.

---

## Branch Protection Rules

### Classic Branch Protection
Set at: Repo Settings → Branches → Add rule

| Rule | Effect |
|---|---|
| **Require pull request before merging** | No direct push to branch |
| **Required approvals (N)** | N approvals needed to merge |
| **Dismiss stale reviews** | Re-approve after new commits |
| **Require code owner review** | CODEOWNERS must approve for touched paths |
| **Require status checks** | Named checks must pass (green) |
| **Require branches to be up to date** | Must be current with base before merge |
| **Require conversation resolution** | All PR comments must be resolved |
| **Require signed commits** | GPG/SSH-signed commits only |
| **Require linear history** | Only squash/rebase merges allowed |
| **Require merge queue** | Queue-based merging to prevent conflicts |
| **Include administrators** | Admins cannot bypass rules |
| **Restrict who can push** | Allowlist of users/teams/apps |
| **Block force pushes** | Prevent rewriting history |
| **Restrict deletions** | Prevent branch deletion |

### Rulesets (next-gen, preferred for orgs)

Set at: Repo/Org Settings → Rules → Rulesets

**Advantages over classic branch protection**:
- Multiple rulesets can apply simultaneously (layered)
- Org-wide rules that apply to all repos
- Apply to tags as well as branches
- Import/export ruleset JSON
- Bypass list with roles (not just individuals)
- Available on all plans (classic = paid for some features)

```json
{
  "name": "protect-main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main", "refs/heads/release/**"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "required_linear_history" },
    { "type": "required_signatures" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "required_status_checks": [
          { "context": "ci/build", "integration_id": 0 },
          { "context": "ci/test", "integration_id": 0 }
        ],
        "strict_required_status_checks_policy": true
      }
    }
  ]
}
```

---

## CODEOWNERS

```
# .github/CODEOWNERS

# Default reviewers for everything
*                   @org/core-team

# Frontend team owns all frontend code
/frontend/          @org/frontend-team @alice

# Infrastructure team owns Terraform
*.tf                @org/infra-team
**/*.tf              @org/infra-team

# Security team must review security policy
/SECURITY.md        @org/security-team

# Shared UI components
/src/components/    @bob @carol

# Last matching rule wins (top-to-bottom)
```

**Key behaviours**:
- Auto-request code owners as reviewers when PR touches owned paths
- With "Require code owner review" branch protection: approval is mandatory
- Teams must be visible to the repo (for org repos: team must have access)
- Does not work on forks from public repos (fork PRs use base repo CODEOWNERS)

---

## Secrets Management

### Secret Hierarchy (precedence: specific > general)
```
Environment secret    (highest precedence for jobs using that environment)
  └── Repository secret
        └── Organisation secret   (lowest precedence)
```

### Secret Best Practices
1. Never commit secrets to git (even `.env` files)
2. Use environment secrets for deployment credentials
3. Rotate secrets regularly
4. Use OIDC instead of static cloud credentials where possible
5. Scope to minimum required repos/environments

### Detecting Leaked Secrets
- **Push protection**: GitHub scans pushes for ~200 known secret patterns and blocks them (enabled by default on public repos)
- **Secret scanning**: Scans history and issues; alerts on matches
- If leaked: rotate immediately, then remove from git history if needed

### Removing Secrets from Git History
```bash
# Using git-filter-repo (preferred)
pip install git-filter-repo
git filter-repo --path-glob '*.env' --invert-paths

# Force push all branches and tags after
git push --force --tags origin 'refs/heads/*'
```

---

## Dependabot

### Configuration (`.github/dependabot.yml`)
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Australia/Sydney"
    open-pull-requests-limit: 5
    labels: ["dependencies", "automated"]
    reviewers: ["@org/dependency-team"]
    groups:
      production-dependencies:
        patterns: ["*"]
        exclude-patterns: ["@types/*"]
        update-types: ["minor", "patch"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"

  - package-ecosystem: "docker"
    directory: "/docker"
    schedule:
      interval: "weekly"
```

### Dependabot Alerts vs Security Updates
| Feature | Dependabot Alerts | Dependabot Security Updates |
|---|---|---|
| What | Notifies about vulnerable deps | Auto-creates PRs to fix CVEs |
| Trigger | NVD/GitHub Advisory Database | Same database |
| Action needed | Manual review | Auto-PR, you just merge |
| Config | Enable in Settings | Enable in Settings |

---

## Code Scanning (CodeQL)

```yaml
# .github/workflows/codeql.yml
name: CodeQL
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 1 * * 1'

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      actions: read
      contents: read

    strategy:
      matrix:
        language: ['javascript', 'python']

    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

Results appear in: Security → Code scanning alerts

---

## Audit Log

- Organisation-level event log: who did what, when
- Available via: Org Settings → Audit log, or REST/GraphQL API
- Events: repo access, team changes, permission changes, secret access, Actions runs
- Retention: 180 days (Enterprise: streaming to SIEM supported)

---

## Two-Factor Authentication (2FA)

- Org admins can require 2FA for all members
- Members without 2FA are removed from org automatically
- Supported: TOTP apps, hardware security keys (FIDO2/WebAuthn), SMS (less secure)

---

## Security Policy

File: `SECURITY.md` in repo root or `.github/`

```markdown
# Security Policy

## Supported Versions
| Version | Supported |
| ------- | --------- |
| 2.x     | ✅        |
| 1.x     | ❌        |

## Reporting a Vulnerability
Please report security vulnerabilities to security@example.com.
Do NOT open a public issue.
We will respond within 48 hours.
```

GitHub surfaces this in: Security tab → Security policy. Also shown in issue creation UI.

---

## Private Vulnerability Reporting

- Researchers can report vulnerabilities privately (no public issue)
- You receive private notification; can collaborate on fix before public disclosure
- Enable at: Security → Private vulnerability reporting

---

## Environment Protection Rules

```yaml
# In Actions workflow
jobs:
  deploy:
    environment:
      name: production
```

Set in: Repo Settings → Environments → production

| Rule | Description |
|---|---|
| Required reviewers | Named users/teams must approve before job runs |
| Wait timer | Delay N minutes before job starts |
| Deployment branches | Only allow from `main` or `release/**` branches |
| Deployment tags | Only allow from semver tags |
| Custom properties | Add metadata to environments |
