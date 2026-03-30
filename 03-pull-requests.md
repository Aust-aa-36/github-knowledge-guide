# GitHub Pull Requests

> A Pull Request (PR) is a proposal to merge one branch into another, with a structured review workflow.

---

## PR Anatomy

| Field | Description |
|---|---|
| **Title** | Summary of change |
| **Body** | Markdown description; should explain what and why; links issues |
| **Base branch** | Target branch (where changes will land) |
| **Compare branch** | Source branch (your changes) |
| **Reviewers** | Users/teams requested to review |
| **Assignees** | Who is responsible for completing the PR |
| **Labels** | Tags for filtering/automation |
| **Milestone** | Release tracking |
| **Projects** | Planning board membership |
| **Linked issues** | Issues that will close when PR merges |
| **Checks** | CI status from Actions or third-party |

---

## PR States

| State | Meaning |
|---|---|
| **Open (draft)** | Work in progress; cannot be merged; code owners not notified |
| **Open (ready)** | Ready for review; checks run; can be merged when approved |
| **Merged** | Changes applied to base branch; permanently closed |
| **Closed (unmerged)** | Abandoned without merging |

### Draft PRs
- Mark as draft to signal "not ready"
- Convert to ready: PR page → "Ready for review"
- Code owners NOT auto-requested until marked ready
- All checks still run on drafts

---

## Merge Strategies

| Strategy | What happens | Commit history | When to use |
|---|---|---|---|
| **Merge commit** | Creates a merge commit joining both branches | Preserves full branch history | Default; good for long-lived feature branches |
| **Squash and merge** | All commits squashed into one commit on base | Clean linear history | Feature branches with messy WIP commits |
| **Rebase and merge** | Commits replayed on top of base, no merge commit | Linear, preserves individual commits | Clean feature branches wanting linear history |

> Configure which merge strategies are allowed: Settings → General → Pull Requests

### Auto-merge
- Enable on a PR: automatically merges when all required checks pass + approvals received
- Requires: auto-merge enabled in repo settings, at least one approval required, all checks green
- Disabled if new commits pushed (re-enables manually)

---

## Reviews

### Review States

| State | Meaning |
|---|---|
| **Comment** | General feedback; does not approve or block |
| **Approve** | LGTM; counts toward required approvals |
| **Request changes** | Blocks merge until reviewer re-reviews or dismisses |

### Review Workflow
1. Author opens PR, requests reviewers (or CODEOWNERS auto-requests)
2. Reviewer reads diff → adds inline comments → submits review
3. Author addresses feedback, pushes new commits
4. Reviewer re-reviews, approves or requests more changes
5. All required reviews + checks pass → merge

### Dismissing Stale Reviews
- When new commits are pushed, "Dismiss stale reviews" setting invalidates prior approvals
- Requires re-approval after every new push
- Configurable per branch protection rule

### Re-requesting Reviews
- After addressing feedback: re-request review from the reviewer (sync icon next to their name)

---

## CODEOWNERS

File location: `.github/CODEOWNERS`, `CODEOWNERS`, or `docs/CODEOWNERS`

```
# Syntax: <path pattern> <@user or @org/team>

# Global fallback
*                   @org/core-team

# Specific directories
/frontend/          @org/frontend-team
/backend/           @alice @bob
*.tf                @org/infra-team

# Specific files
/docs/SECURITY.md   @security-team
package.json        @org/dependency-guardians
```

**Behaviour**:
- Code owners are automatically requested as reviewers when PR touches their paths
- With branch protection "Require review from code owners", their approval is mandatory
- CODEOWNERS file is processed top-to-bottom; last matching pattern wins

---

## Branch Protection & Required Checks

Common settings that affect PRs:

| Rule | Effect |
|---|---|
| Require approvals (N) | Cannot merge until N users approved |
| Dismiss stale reviews | Re-approval needed after new commits |
| Require code owner review | CODEOWNERS must approve for owned paths |
| Require status checks | Named CI checks must pass before merge |
| Require branches up to date | Branch must be current with base before merge |
| Require linear history | Only squash/rebase allowed (no merge commits) |
| Require signed commits | All commits must be GPG-signed |
| Include administrators | Rules apply even to repo admins |
| Restrict who can push | Only listed users/teams can merge |

---

## PR Templates

File: `.github/PULL_REQUEST_TEMPLATE.md`

```markdown
## Summary
<!-- What does this PR do? -->

## Type of change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Related Issues
Closes #

## Testing
- [ ] Unit tests added/updated
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] No new warnings introduced
```

Multiple templates supported via `.github/PULL_REQUEST_TEMPLATE/` directory + `?template=name.md` URL param.

---

## Checks vs Statuses

| Type | Source | Shown as |
|---|---|---|
| **Check runs** | GitHub Actions, GitHub Apps | ✅ / ❌ with details + annotations |
| **Commit statuses** | External CI via REST API | ✅ / ❌ simple pass/fail |

Required status checks in branch protection reference check run names (from Actions `name:` field).

---

## Conflict Resolution

When base branch has diverged from compare branch:

**Option 1 — Merge base into feature (keeps merge commit)**:
```bash
git fetch origin
git checkout feature-branch
git merge origin/main
# resolve conflicts
git push
```

**Option 2 — Rebase feature onto base (linear history)**:
```bash
git fetch origin
git checkout feature-branch
git rebase origin/main
# resolve conflicts during rebase
git push --force-with-lease
```

**Option 3 — GitHub UI conflict editor**:
- PR page shows "Resolve conflicts" button when conflicts are simple text conflicts
- Not available for binary files

---

## PR Size Best Practices

| Size | Lines changed | Recommendation |
|---|---|---|
| XS | < 10 | Ideal for hot fixes |
| S | 10–100 | Good; fast review |
| M | 100–500 | Acceptable; reviewers need ~30 min |
| L | 500–1000 | Consider splitting |
| XL | > 1000 | Almost always should be split |

---

## Useful PR Automation (Actions)

```yaml
# Auto-label PRs based on file paths changed
- uses: actions/labeler@v5
  with:
    repo-token: ${{ secrets.GITHUB_TOKEN }}
    configuration-path: .github/labeler.yml

# Auto-assign reviewers (beyond CODEOWNERS)
- uses: kentaro-m/auto-assign-action@v1

# Enforce PR title format (conventional commits)
- uses: amannn/action-semantic-pull-request@v5
```

---

## Key API Endpoints (REST)

```
GET  /repos/{owner}/{repo}/pulls                      # List PRs
POST /repos/{owner}/{repo}/pulls                      # Create PR
GET  /repos/{owner}/{repo}/pulls/{number}             # Get PR
PATCH /repos/{owner}/{repo}/pulls/{number}            # Update PR
PUT  /repos/{owner}/{repo}/pulls/{number}/merge       # Merge PR
POST /repos/{owner}/{repo}/pulls/{number}/reviews     # Submit review
GET  /repos/{owner}/{repo}/pulls/{number}/files       # List changed files
GET  /repos/{owner}/{repo}/pulls/{number}/commits     # List commits
```

---

## PR Merge via gh CLI

```bash
# Merge with default strategy
gh pr merge 42

# Squash merge
gh pr merge 42 --squash

# Rebase merge
gh pr merge 42 --rebase

# Auto-merge (waits for checks)
gh pr merge 42 --auto --squash

# Create PR
gh pr create --title "feat: add login" --body "Closes #10" --base main
```
