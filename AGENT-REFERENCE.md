# GitHub Agent Quick Reference

> Fast lookup for agents performing GitHub automation tasks. No fluff — direct answers.

---

## Common Tasks → Right Tool

| Task | Use |
|---|---|
| Read file from repo | REST: `GET /repos/{owner}/{repo}/contents/{path}` |
| Write/update file | REST: `PUT /repos/{owner}/{repo}/contents/{path}` (requires base64 + sha) |
| Create issue | REST: `POST /repos/{owner}/{repo}/issues` |
| Close issue | REST: `PATCH /repos/{owner}/{repo}/issues/{n}` `{state: "closed"}` |
| Comment on issue/PR | REST: `POST /repos/{owner}/{repo}/issues/{n}/comments` |
| Create PR | REST: `POST /repos/{owner}/{repo}/pulls` |
| Merge PR | REST: `PUT /repos/{owner}/{repo}/pulls/{n}/merge` |
| Add label to issue | REST: `POST /repos/{owner}/{repo}/issues/{n}/labels` |
| Get all open issues with label | REST: `GET /repos/{owner}/{repo}/issues?state=open&labels=bug` |
| Add issue to Projects v2 | GraphQL: `addProjectV2ItemById` mutation |
| Update Projects v2 field | GraphQL: `updateProjectV2ItemFieldValue` mutation |
| Get project items | GraphQL: query `node(id: "PVT_xxx") { ... on ProjectV2 { items } }` |
| Trigger workflow | REST: `POST /repos/{owner}/{repo}/actions/workflows/{id}/dispatches` |
| Get workflow run status | REST: `GET /repos/{owner}/{repo}/actions/runs/{run_id}` |
| List repo workflows | REST: `GET /repos/{owner}/{repo}/actions/workflows` |
| Create release | REST: `POST /repos/{owner}/{repo}/releases` |
| Get latest release | REST: `GET /repos/{owner}/{repo}/releases/latest` |
| List branches | REST: `GET /repos/{owner}/{repo}/branches` |
| Create branch | REST: `POST /repos/{owner}/{repo}/git/refs` |
| Push commit (file update) | REST: `PUT /repos/{owner}/{repo}/contents/{path}` |

---

## gh CLI Cheatsheet

```bash
# Issues
gh issue create --title "Title" --body "Body" --label "bug"
gh issue close 42 --reason "not planned"
gh issue comment 42 --body "Fixed in PR #43"
gh issue list --state open --label "triage"
gh issue view 42

# PRs
gh pr create --title "feat: ..." --body "Closes #42" --base main
gh pr merge 42 --squash --auto
gh pr close 42
gh pr review 42 --approve --body "LGTM"
gh pr review 42 --request-changes --body "Please fix X"
gh pr list --state open --label "ready"
gh pr view 42
gh pr checks 42

# Actions
gh workflow run deploy.yml --ref main -f environment=production
gh run list --workflow=ci.yml --limit 5
gh run view 1234567890
gh run rerun 1234567890 --failed   # only re-run failed jobs
gh run cancel 1234567890

# Releases
gh release create v1.2.3 --title "v1.2.3" --notes "Changes..." ./dist/*.zip
gh release view v1.2.3

# Repos
gh repo clone owner/repo
gh repo create org/repo --private
gh repo fork owner/repo --clone

# REST API
gh api /repos/owner/repo
gh api -X POST /repos/owner/repo/issues -f title="Bug" -f body="Details"
gh api -X PATCH /repos/owner/repo/issues/1 -f state=closed

# GraphQL
gh api graphql -f query='query { viewer { login } }'
```

---

## Projects v2 — Node ID Lookup

```bash
# Find project node ID (user)
gh api graphql -f query='
  query {
    user(login: "USERNAME") {
      projectsV2(first: 20) {
        nodes { id number title }
      }
    }
  }'

# Find project node ID (org)
gh api graphql -f query='
  query {
    organization(login: "ORGNAME") {
      projectsV2(first: 20) {
        nodes { id number title }
      }
    }
  }'

# Find issue node ID
gh api graphql -f query='
  query($owner:String! $name:String! $number:Int!) {
    repository(owner:$owner name:$name) {
      issue(number:$number) { id title }
    }
  }' -f owner=OWNER -f name=REPO -F number=42

# Find field IDs in a project
gh api graphql -f query='
  query {
    node(id: "PVT_xxx") {
      ... on ProjectV2 {
        fields(first: 30) {
          nodes {
            ... on ProjectV2Field { id name }
            ... on ProjectV2SingleSelectField {
              id name
              options { id name }
            }
            ... on ProjectV2IterationField { id name }
          }
        }
      }
    }
  }'
```

---

## Projects v2 — Mutations

```bash
# Add issue to project
gh api graphql -f query='
  mutation {
    addProjectV2ItemById(input: {
      projectId: "PVT_xxx"
      contentId: "I_xxx"
    }) {
      item { id }
    }
  }'

# Set Status field (single select)
gh api graphql -f query='
  mutation {
    updateProjectV2ItemFieldValue(input: {
      projectId: "PVT_xxx"
      itemId: "PVTI_xxx"
      fieldId: "PVTF_xxx"
      value: { singleSelectOptionId: "opt_xxx" }
    }) {
      projectV2Item { id }
    }
  }'

# Set date field
# value: { date: "2026-06-01" }

# Set number field
# value: { number: 5 }

# Set text field
# value: { text: "my text" }
```

---

## Actions — Key Patterns for Agents

### Post comment on PR from workflow
```yaml
- uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: '✅ Deployment complete!'
      })
```

### Get PR number in workflow
```yaml
${{ github.event.pull_request.number }}    # In pull_request event
${{ github.event.issue.number }}           # In issue_comment event
```

### Trigger workflow from another workflow
```yaml
- name: Trigger deploy
  run: |
    gh workflow run deploy.yml \
      --ref main \
      -f environment=production
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Create issue from workflow
```yaml
- uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.create({
        owner: context.repo.owner,
        repo: context.repo.repo,
        title: 'Automated: Deploy failed',
        body: `Run: ${context.runId}`,
        labels: ['bug', 'automated']
      })
```

### Set job output
```yaml
- id: my-step
  run: echo "result=hello" >> $GITHUB_OUTPUT
# Use: ${{ steps.my-step.outputs.result }}
```

### Conditional steps
```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: failure()           # Only run on failure
if: always()            # Always run (even if prior steps failed)
if: success()           # Only on success (default)
if: cancelled()         # Only if workflow was cancelled
```

---

## REST API — File Operations

### Read file content
```bash
# Returns base64-encoded content + sha (needed for updates)
gh api /repos/OWNER/REPO/contents/path/to/file.js

# Decode content
gh api /repos/OWNER/REPO/contents/path/to/file.js \
  --jq '.content' | base64 --decode
```

### Create or update file
```bash
CONTENT=$(echo "file contents here" | base64)
SHA=$(gh api /repos/OWNER/REPO/contents/path/file.txt --jq '.sha' 2>/dev/null || echo "")

if [ -n "$SHA" ]; then
  # Update existing file
  gh api -X PUT /repos/OWNER/REPO/contents/path/file.txt \
    -f message="Update file" \
    -f content="$CONTENT" \
    -f sha="$SHA"
else
  # Create new file
  gh api -X PUT /repos/OWNER/REPO/contents/path/file.txt \
    -f message="Create file" \
    -f content="$CONTENT"
fi
```

---

## Rate Limit Handling

```bash
# Check remaining requests
gh api /rate_limit --jq '.rate'

# Response headers (in curl)
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1372700873    # Unix timestamp when limit resets
X-RateLimit-Used: 1
```

**When rate limited (HTTP 429 or 403 with rate limit message)**:
1. Check `X-RateLimit-Reset` header
2. Wait until reset time
3. For secondary rate limits: add 1 minute delay before retry
4. Do NOT hammer the endpoint — exponential backoff

---

## Common Error Codes

| Code | Meaning | Fix |
|---|---|---|
| 401 | Unauthenticated | Check token is valid and in header |
| 403 | Forbidden | Check token has required scope/permission |
| 404 | Not found or no access | Check repo/resource exists and token can see it |
| 409 | Conflict | Branch exists, file SHA mismatch, merge conflict |
| 422 | Unprocessable entity | Invalid parameters; check response body |
| 429 | Rate limited | Wait and retry with backoff |
| 451 | DMCA takedown | Cannot access |

---

## Webhook Event Payload Quick Ref

```
X-GitHub-Event: push
  .ref                     # "refs/heads/main"
  .commits[].message
  .repository.full_name
  .pusher.name

X-GitHub-Event: pull_request
  .action                  # opened | closed | merged | synchronize | labeled
  .pull_request.number
  .pull_request.merged     # true if action=closed + merged
  .pull_request.head.sha
  .pull_request.base.ref   # target branch

X-GitHub-Event: issues
  .action                  # opened | closed | labeled | assigned
  .issue.number
  .issue.title
  .issue.labels[].name

X-GitHub-Event: issue_comment
  .action                  # created | edited | deleted
  .comment.body
  .issue.number
  .issue.pull_request      # exists if comment is on a PR

X-GitHub-Event: workflow_run
  .action                  # completed | requested
  .workflow_run.status     # completed
  .workflow_run.conclusion # success | failure | cancelled
  .workflow_run.name
```

---

## Gotchas & Warnings

| Gotcha | Detail |
|---|---|
| `addProjectV2Item` is deprecated | Use `addProjectV2ItemById` |
| `workflow_dispatch` has no event context | `github.event.pull_request` is null; don't use PR-specific expressions |
| GITHUB_TOKEN can't trigger other workflows | Use PAT or App token to avoid recursive trigger prevention |
| File update requires SHA | GET the file first to get its sha, or the PUT will fail with 409 |
| GraphQL node IDs are not numeric issue numbers | Issue #42 ≠ `I_xxx`; query for the node ID separately |
| Fork PRs have no access to upstream secrets | `pull_request_target` is needed for fork PR automation (but use with caution) |
| Branch protection includes admins | Only if "Include administrators" is explicitly enabled |
| Stale review dismissal | Pushing new commits will invalidate approvals if this rule is on |
| Rate limit resets by bucket | Primary (5000/hr) and secondary (concurrent/abuse) are separate |
| Secrets are masked in logs | But don't print them with echo — they may appear partially |
| `continue-on-error: true` | Marks step as passed even if it fails; job still runs but step shows yellow |
