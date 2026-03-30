# GitHub Issues & Projects

> Issues = trackable work items. Projects v2 = flexible planning boards that aggregate issues and PRs across repos.

---

## Issues

### What Issues Are For
- Bug reports
- Feature requests
- Tasks / to-dos
- Questions (better in Discussions, but common in Issues)
- Tracking work that doesn't have code yet

### Issue Anatomy

| Field | Description |
|---|---|
| **Title** | Short summary |
| **Body** | Markdown description (supports checkboxes, code blocks, images) |
| **Assignees** | Up to 10 users responsible for the issue |
| **Labels** | Coloured tags for filtering/triage (multiple allowed) |
| **Milestone** | Group issues into a release/sprint with optional due date |
| **Projects** | Add to one or more Projects v2 boards |
| **Linked PRs** | PRs that reference or close this issue |
| **Sub-issues** | Child issues for hierarchical breakdown |
| **Development** | Linked branches/PRs created from this issue |
| **Type** | Custom issue type (org-level configuration) |

### Issue States
- **Open** — active work item
- **Closed as completed** — resolved
- **Closed as not planned** — won't fix / duplicate / out of scope

### Labels (Built-in defaults)

| Label | Meaning |
|---|---|
| `bug` | Something isn't working |
| `documentation` | Improvements or additions to docs |
| `duplicate` | Already reported |
| `enhancement` | New feature or request |
| `good first issue` | Good for newcomers |
| `help wanted` | Extra attention needed |
| `invalid` | Not a real issue |
| `question` | Further information requested |
| `wontfix` | Not going to be addressed |

> Labels are repo-specific. Org-level labels can be pushed to all repos via API.

### Milestones
- Group of issues/PRs targeting a deadline (e.g., `v2.0`, `Q3 Sprint`)
- Show % complete (closed / total)
- Do NOT cascade to Projects automatically (add items to both)
- Due date shown; does not auto-close issues

### Issue Templates & Forms

**Markdown templates** (`.github/ISSUE_TEMPLATE/*.md`):
- Pre-filled body text
- Can set default labels, assignees, title

**Issue Forms** (`.github/ISSUE_TEMPLATE/*.yml`) — preferred:
- Structured fields: text input, dropdown, checkboxes, textarea
- Required field enforcement
- Renders as a form UI, not free text
- Example:
```yaml
name: Bug Report
description: File a bug report
labels: ["bug", "triage"]
body:
  - type: markdown
    attributes:
      value: "## Bug Report"
  - type: input
    id: version
    attributes:
      label: Version
      placeholder: "e.g. 1.2.3"
    validations:
      required: true
  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options: [Critical, High, Medium, Low]
    validations:
      required: true
```

### Closing Keywords (in PR body or commit message)
When a PR/commit is merged into the default branch, these auto-close the linked issue:

```
close #123      closes #123      closed #123
fix #123        fixes #123       fixed #123
resolve #123    resolves #123    resolved #123
```

Also works cross-repo:
```
fixes owner/repo#123
```

### Cross-References
- Mention `#123` anywhere in a comment/body → creates a reference link
- GitHub shows the reference in the linked issue's timeline
- Mention `@username` → notifies the user

### Sub-Issues
- Issues can have child issues (nested hierarchy)
- Parent shows progress bar based on child states
- Multiple levels supported
- Useful for: epics → stories → tasks

---

## GitHub Projects v2

### What it is
A flexible, spreadsheet-meets-board planning tool that lives at the user or organisation level (not repo level) and can aggregate items from multiple repos.

### Views

| View | Description | Best for |
|---|---|---|
| **Table** | Spreadsheet-style with all fields as columns | Sprint management, bulk editing |
| **Board** | Kanban-style swimlanes by a field value | Workflow visualisation |
| **Roadmap** | Timeline/Gantt view using date fields | Release planning |

Multiple views of the same data can exist in one project — toggle between them.

### Built-in Fields

| Field | Type | Notes |
|---|---|---|
| Title | Text | Issue/PR title (read-only from source) |
| Assignees | User list | From issue/PR |
| Status | Single select | Default: Todo / In Progress / Done |
| Labels | Label list | From issue/PR |
| Repository | Repo | Source repo |
| Milestone | Milestone | From issue/PR |
| Linked PRs | PR list | PRs linked to issue |

### Custom Fields (up to 50 total)

| Field Type | Use case |
|---|---|
| **Text** | Notes, URLs, free text |
| **Number** | Story points, effort, priority score |
| **Date** | Target ship date, due date |
| **Single select** | Priority (P0/P1/P2), Phase, Team |
| **Iteration** | Sprint/iteration cycle with start date + duration; supports breaks |

### Automation / Workflows

Built-in auto-workflows (no code needed):
- Item added to project → set Status
- Item closed → set Status to Done
- PR merged → set Status to Done
- Auto-archive closed items after N days
- Auto-add issues/PRs matching repo + label filter

Custom automation via:
- **GitHub Actions** + GraphQL API mutations
- External scripts using GraphQL API

### Project Limits

| Resource | Limit |
|---|---|
| Items per project | 1,200 (soft) / 10,000 (archive) |
| Custom fields | 50 |
| Views | 50 |
| Workflows | 10 |
| Projects per org | 2,000 |
| Projects per user | 100 |

### GraphQL API — Key Operations

```graphql
# Add issue to project
mutation {
  addProjectV2ItemById(input: {
    projectId: "PVT_xxx"
    contentId: "I_xxx"
  }) {
    item { id }
  }
}

# Update a custom field value
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

# Query project items
query {
  node(id: "PVT_xxx") {
    ... on ProjectV2 {
      items(first: 20) {
        nodes {
          id
          content {
            ... on Issue { title number state }
            ... on PullRequest { title number state }
          }
        }
      }
    }
  }
}
```

**Important**: Use `addProjectV2ItemById` (NOT the deprecated `addProjectV2Item`).

### Getting Node IDs for GraphQL

```bash
# Get project node ID
gh api graphql -f query='
  query {
    user(login: "USERNAME") {
      projectsV2(first: 10) {
        nodes { id title }
      }
    }
  }'

# Get issue node ID
gh api graphql -f query='
  query {
    repository(owner: "OWNER", name: "REPO") {
      issue(number: 42) { id title }
    }
  }'
```

### Projects vs Classic Projects

| Feature | Projects v2 | Classic Projects |
|---|---|---|
| Location | User or Org level | Repo or Org level |
| Cross-repo | ✅ Yes | ✅ Org-level only |
| Custom fields | ✅ 50 fields | ❌ Notes only |
| Roadmap view | ✅ | ❌ |
| GraphQL API | ✅ Full | Limited |
| Automation | ✅ Built-in + API | Limited |
| Status | Current | Being deprecated |

---

## Discussions

| Aspect | Detail |
|---|---|
| Purpose | Community Q&A, announcements, ideas, show-and-tell |
| vs Issues | Not a work item; no assignees/milestones; has "answered" state |
| Categories | Customisable (Q&A, General, Ideas, etc.) |
| Polls | Built-in polling in discussions |
| Convert | Discussion can be converted to issue if it becomes actionable |
| Search | Full-text searchable |
| Notifications | Subscription-based like issues |

---

## Workflow: Issue → Project → PR → Close

```
1. Issue created (manually or via template)
   └── Triage: add labels, assignee, milestone
   └── Add to Project (manual or auto-workflow)

2. Developer picks up issue
   └── Create branch from issue (Development → Create branch)
   └── Project item moves to "In Progress" (via workflow or manually)

3. Developer opens PR
   └── PR body references issue: "Closes #123"
   └── PR linked in issue timeline

4. PR reviewed and merged
   └── Issue auto-closes
   └── Project item moves to "Done" (via built-in workflow)
   └── Milestone % complete updates
```

---

## Key API Endpoints (REST)

```
POST /repos/{owner}/{repo}/issues              # Create issue
PATCH /repos/{owner}/{repo}/issues/{number}    # Update issue (close, label, etc.)
POST /repos/{owner}/{repo}/issues/{number}/labels   # Add labels
POST /repos/{owner}/{repo}/milestones          # Create milestone
GET  /repos/{owner}/{repo}/issues?state=open&labels=bug  # Filter issues
```
