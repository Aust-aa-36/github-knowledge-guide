# GitHub API, Apps & Webhooks

> Three complementary surfaces: REST API (resource-oriented), GraphQL API (flexible queries), Webhooks (push notifications), and GitHub Apps (installable integrations).

---

## REST API (v3)

### Base URL
```
https://api.github.com
```

### Authentication
```bash
# Personal Access Token (PAT) — header
curl -H "Authorization: Bearer ghp_xxxx" https://api.github.com/user

# GITHUB_TOKEN in Actions
curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" ...

# GitHub App installation token
curl -H "Authorization: Bearer <installation_token>" ...
```

### Rate Limits

| Auth type | Rate limit |
|---|---|
| Unauthenticated | 60 requests/hour (per IP) |
| PAT / OAuth token | 5,000 requests/hour |
| GITHUB_TOKEN (Actions) | 1,000 requests/hour (per repo) |
| GitHub App installation token | 5,000–15,000 requests/hour |

Check remaining rate limit:
```bash
curl -H "Authorization: Bearer TOKEN" https://api.github.com/rate_limit
```

### Pagination
```bash
# Page-based (most endpoints)
GET /repos/{owner}/{repo}/issues?per_page=100&page=2

# Link header navigation
Link: <https://api.github.com/...?page=3>; rel="next",
      <https://api.github.com/...?page=10>; rel="last"
```

### API Versioning
```bash
# Pin to specific API version (recommended)
-H "X-GitHub-Api-Version: 2022-11-28"
```

### Key Endpoint Categories

#### Repositories
```
GET    /repos/{owner}/{repo}
POST   /user/repos                          # Create repo
PATCH  /repos/{owner}/{repo}               # Update repo
DELETE /repos/{owner}/{repo}               # Delete repo
GET    /repos/{owner}/{repo}/contents/{path}
PUT    /repos/{owner}/{repo}/contents/{path}   # Create/update file
DELETE /repos/{owner}/{repo}/contents/{path}
GET    /repos/{owner}/{repo}/git/trees/{sha}?recursive=1  # Full tree
```

#### Branches & Refs
```
GET  /repos/{owner}/{repo}/branches
GET  /repos/{owner}/{repo}/branches/{branch}
POST /repos/{owner}/{repo}/git/refs         # Create branch
PATCH /repos/{owner}/{repo}/git/refs/{ref}  # Update ref (fast-forward)
```

#### Commits
```
GET  /repos/{owner}/{repo}/commits
GET  /repos/{owner}/{repo}/commits/{sha}
GET  /repos/{owner}/{repo}/compare/{base}...{head}
```

#### Issues
```
GET  /repos/{owner}/{repo}/issues
POST /repos/{owner}/{repo}/issues
PATCH /repos/{owner}/{repo}/issues/{number}
POST /repos/{owner}/{repo}/issues/{number}/comments
POST /repos/{owner}/{repo}/issues/{number}/labels
POST /repos/{owner}/{repo}/labels           # Create label
```

#### Pull Requests
```
GET  /repos/{owner}/{repo}/pulls
POST /repos/{owner}/{repo}/pulls
PATCH /repos/{owner}/{repo}/pulls/{number}
PUT  /repos/{owner}/{repo}/pulls/{number}/merge
POST /repos/{owner}/{repo}/pulls/{number}/reviews
GET  /repos/{owner}/{repo}/pulls/{number}/files
```

#### Releases
```
GET  /repos/{owner}/{repo}/releases
POST /repos/{owner}/{repo}/releases
GET  /repos/{owner}/{repo}/releases/latest
POST /repos/{owner}/{repo}/releases/{id}/assets  # Upload asset
```

#### Actions
```
GET  /repos/{owner}/{repo}/actions/workflows
POST /repos/{owner}/{repo}/actions/workflows/{id}/dispatches   # Trigger workflow
GET  /repos/{owner}/{repo}/actions/runs
POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun
DELETE /repos/{owner}/{repo}/actions/runs/{run_id}
GET  /repos/{owner}/{repo}/actions/secrets
PUT  /repos/{owner}/{repo}/actions/secrets/{name}
```

#### Organisations
```
GET  /orgs/{org}
GET  /orgs/{org}/members
GET  /orgs/{org}/repos
GET  /orgs/{org}/teams
POST /orgs/{org}/repos
```

#### Users
```
GET  /user                  # Authenticated user
GET  /users/{username}      # Any user
GET  /user/repos
```

---

## GraphQL API (v4)

### Endpoint
```
POST https://api.github.com/graphql
```

### Why GraphQL over REST?
- Single request to fetch related data (e.g., issue + comments + labels + project fields)
- Strongly typed schema — discover available fields
- Mutations for all write operations
- Required for: Projects v2 full CRUD
- Better for: dashboards, reporting, complex data aggregation

### Authentication
Same as REST — Bearer token in Authorization header.

### Rate Limits
- 5,000 points/hour (complexity-based, not request count)
- Each node queried = ~1 point; check `rateLimit` in response

### Basic Query Pattern
```graphql
query {
  repository(owner: "octocat", name: "Hello-World") {
    name
    description
    defaultBranchRef {
      name
    }
    issues(first: 10, states: OPEN) {
      totalCount
      nodes {
        number
        title
        labels(first: 5) {
          nodes { name color }
        }
        assignees(first: 3) {
          nodes { login }
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

### Pagination in GraphQL
```graphql
issues(first: 20, after: "cursor_string") {
  nodes { ... }
  pageInfo {
    hasNextPage
    endCursor
  }
}
```

### Key Mutations

```graphql
# Create issue
mutation {
  createIssue(input: {
    repositoryId: "R_xxx"
    title: "New bug"
    body: "Description"
    labelIds: ["LA_xxx"]
    assigneeIds: ["U_xxx"]
  }) {
    issue { number url }
  }
}

# Add PR comment
mutation {
  addComment(input: {
    subjectId: "PR_xxx"
    body: "LGTM!"
  }) {
    commentEdge { node { id } }
  }
}

# Add item to Projects v2
mutation {
  addProjectV2ItemById(input: {
    projectId: "PVT_xxx"
    contentId: "I_xxx"
  }) {
    item { id }
  }
}

# Update Projects v2 field
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "PVT_xxx"
    itemId: "PVTI_xxx"
    fieldId: "PVTF_xxx"
    value: { singleSelectOptionId: "opt_xxx" }
  }) {
    projectV2Item { id }
  }
}

# Merge PR
mutation {
  mergePullRequest(input: {
    pullRequestId: "PR_xxx"
    mergeMethod: SQUASH
  }) {
    pullRequest { merged mergedAt }
  }
}
```

### Getting Node IDs

```bash
# Via CLI
gh api graphql -f query='
  query($owner:String! $name:String!) {
    repository(owner:$owner name:$name) {
      id
      issue(number: 1) { id }
    }
  }' -f owner=myorg -f name=myrepo

# Via REST (items include node_id field)
curl https://api.github.com/repos/owner/repo/issues/1
# Returns: { "node_id": "I_xxxx", ... }
```

### Schema Introspection
```graphql
query {
  __schema {
    types { name kind }
  }
}
# Or explore at: https://docs.github.com/en/graphql/overview/explorer
```

---

## GitHub Apps

### What They Are
- Installable integrations that act as their own identity ("My Bot", not "john's token")
- Can be installed on orgs, users, or specific repos
- Fine-grained permissions per resource type
- Receive webhooks for subscribed events

### GitHub App vs OAuth App vs PAT

| Feature | GitHub App | OAuth App | PAT |
|---|---|---|---|
| Acts as | App identity | User identity | User identity |
| Permissions | Fine-grained per resource | Coarse OAuth scopes | Classic: coarse; Fine-grained: per-repo |
| Token lifetime | Short-lived (1 hour) | Until revoked | Until revoked |
| Installation scope | Org / repo / user | Entire account | Entire account |
| Rate limit | 5,000–15,000/hr | 5,000/hr | 5,000/hr |
| Webhooks | Built-in subscription | Separate setup | N/A |
| Best for | Production bots/integrations | User-facing web apps | Personal scripts, CI tokens |

### GitHub App Auth Flow
```
1. Register app → get App ID + private key (PEM)
2. Generate JWT from private key (expires in 10 min):
   { iss: APP_ID, iat: now, exp: now+600 }
3. GET /app/installations → get installation_id
4. POST /app/installations/{id}/access_tokens → get installation token (1 hour)
5. Use installation token as Bearer token for API calls
```

### Key App Permissions
```
Repository: contents, issues, pull_requests, actions, checks,
            deployments, environments, packages, projects,
            security_events, statuses, workflows
Organisation: members, projects, secrets, webhooks
Account: email, followers, profile
```

---

## OAuth Apps

```
1. Register OAuth App → get Client ID + Client Secret
2. Redirect user to:
   https://github.com/login/oauth/authorize?client_id=xxx&scope=repo,user&state=random
3. User authorises → GitHub redirects to callback with ?code=xxx&state=xxx
4. Exchange code for token:
   POST https://github.com/login/oauth/access_token
   { client_id, client_secret, code }
5. Use access_token as Bearer token
```

OAuth scopes: `repo`, `read:org`, `user`, `gist`, `notifications`, `workflow`, etc.

---

## Webhooks

### What Webhooks Do
GitHub sends an HTTP POST to your URL when subscribed events occur — real-time event push to external systems.

### Webhook Scopes
| Scope | Set in | Events available |
|---|---|---|
| Repository webhook | Repo Settings → Webhooks | Repo events |
| Organisation webhook | Org Settings → Webhooks | All org repo events |
| GitHub App webhook | App settings | Events the app subscribes to |

### Delivery Format
```http
POST https://your-server.com/webhook
Content-Type: application/json
X-GitHub-Event: pull_request
X-GitHub-Delivery: 72d3162e-cc78-11e3-81ab-4c9367dc0958
X-Hub-Signature-256: sha256=<HMAC>

{
  "action": "opened",
  "pull_request": { ... },
  "repository": { ... },
  "sender": { ... }
}
```

### Signature Verification (MUST implement for security)
```python
import hmac, hashlib

def verify_signature(payload: bytes, secret: str, signature: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

### Common Webhook Events
```
push                    # Commits pushed
pull_request            # PR opened/closed/merged/etc
pull_request_review     # Review submitted
issues                  # Issue created/closed/etc
issue_comment           # Comment on issue or PR
create / delete         # Branch or tag created/deleted
release                 # Release published
check_run / check_suite # CI status
workflow_run            # Actions workflow completed
deployment / deployment_status
repository              # Repo created/deleted/transferred
member                  # Collaborator added/removed
```

### Webhook Delivery Retries
- GitHub retries failed deliveries (non-2xx or timeout) up to 3 days
- 10 second timeout per delivery
- View delivery history + redeliver in Settings → Webhooks → Recent Deliveries

---

## gh CLI — API Shortcuts

```bash
# REST
gh api /repos/owner/repo/issues --jq '.[] | .title'
gh api -X POST /repos/owner/repo/issues -f title="Bug" -f body="Details"
gh api -X PATCH /repos/owner/repo/issues/1 -f state=closed

# GraphQL
gh api graphql -f query='query { viewer { login } }'
gh api graphql --paginate -f query='...'   # Auto-paginate

# Trigger workflow
gh workflow run my-workflow.yml --ref main -f env=production

# View Actions runs
gh run list --workflow=ci.yml
gh run view 1234567890
gh run rerun 1234567890
```
