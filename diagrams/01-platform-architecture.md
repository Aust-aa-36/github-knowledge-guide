# GitHub Platform Architecture Diagrams

All diagrams use Mermaid syntax. Render in GitHub, VS Code (Mermaid Preview), or mermaid.live.

---

## 1. Platform Component Map

```mermaid
graph TB
    subgraph GitHub["GitHub Platform"]
        subgraph Code["Code & Collaboration"]
            REPO[Repository]
            BRANCH[Branches / Tags]
            COMMIT[Commits]
            PR[Pull Requests]
            ISSUE[Issues]
            DISCUSS[Discussions]
            WIKI[Wiki]
        end

        subgraph Planning["Planning"]
            PROJECT[Projects v2]
            MILESTONE[Milestones]
            LABEL[Labels]
        end

        subgraph Automation["Automation & Integration"]
            ACTIONS[GitHub Actions]
            WEBHOOK[Webhooks]
            APP[GitHub Apps]
            OAUTHAPP[OAuth Apps]
        end

        subgraph Security["Security"]
            BPROTECT[Branch Protection / Rulesets]
            SECRETS[Secrets & Variables]
            CODEOWNERS[CODEOWNERS]
            DEPENDABOT[Dependabot]
            CODESCAN[Code Scanning]
            SECRETSCAN[Secret Scanning]
            OIDC[OIDC / Tokens]
        end

        subgraph Publishing["Publishing & Hosting"]
            PACKAGES[GitHub Packages / GHCR]
            PAGES[GitHub Pages]
            RELEASES[Releases]
        end

        subgraph DevEnv["Developer Experience"]
            CODESPACE[Codespaces]
            COPILOT[Copilot]
        end

        subgraph API["API Surface"]
            REST[REST API v3]
            GQL[GraphQL API v4]
        end
    end

    REPO --> BRANCH
    BRANCH --> COMMIT
    BRANCH --> PR
    PR --> ISSUE
    REPO --> ACTIONS
    REPO --> WEBHOOK
    ACTIONS --> SECRETS
    ACTIONS --> PACKAGES
    ACTIONS --> PAGES
    ACTIONS --> RELEASES
    PROJECT --> ISSUE
    PROJECT --> PR
    ISSUE --> MILESTONE
    ISSUE --> LABEL
    PR --> BPROTECT
    PR --> CODEOWNERS
    REST --> REPO
    GQL --> PROJECT
    GQL --> ISSUE
    APP --> REST
    APP --> GQL
    APP --> WEBHOOK
```

---

## 2. CI/CD Pipeline Flow

```mermaid
flowchart LR
    DEV([Developer]) -->|git push| REPO[(Repository)]
    REPO -->|triggers| WF[Workflow Run]

    subgraph Actions["GitHub Actions"]
        WF --> JOB1[Build Job\nubuntu-latest]
        JOB1 --> JOB2[Test Job\nmatrix: node 18/20/22]
        JOB2 -->|artifact| JOB3[Security Scan\nCodeQL]
        JOB3 -->|needs: test| JOB4[Build Docker Image\nghcr.io/owner/app]
        JOB4 -->|needs: build| JOB5{Branch?\nmain?}
    end

    JOB5 -->|yes| ENV1[Deploy to Staging\nenvironment: staging]
    JOB5 -->|release tag| ENV2[Deploy to Production\nenvironment: production\nrequires: approval]

    ENV1 --> SLACK([Slack/Teams\nNotification])
    ENV2 --> SLACK

    style Actions fill:#0d1117,color:#f0f6fc,stroke:#30363d
    style ENV2 fill:#238636,color:#fff
```

---

## 3. Pull Request Review Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Draft: Create draft PR
    Draft --> OpenReady: Mark ready for review
    OpenReady --> ReviewRequested: CODEOWNERS auto-requested\nor manually requested
    ReviewRequested --> ChangesRequested: Reviewer: Request changes
    ReviewRequested --> Approved: Reviewer: Approve
    ChangesRequested --> ReviewRequested: Author pushes fixes\n(stale reviews dismissed)
    Approved --> ChecksPending: Waiting for CI checks
    ChecksPending --> Mergeable: All checks pass\n+ required approvals met
    ChecksPending --> BlockedByChecks: Check failed
    BlockedByChecks --> ChecksPending: Author fixes + pushes
    Mergeable --> Merged: Merge (commit/squash/rebase)
    Merged --> [*]
    OpenReady --> Closed: Abandoned
    Closed --> [*]
```

---

## 4. GitHub Authentication Decision Tree

```mermaid
flowchart TD
    START([Need to authenticate\nto GitHub?]) --> Q1{Where are\nyou running?}

    Q1 -->|GitHub Actions workflow| TOKEN[GITHUB_TOKEN\nauto-provided\nleast privilege]
    TOKEN --> Q2{Need cross-repo\nor admin access?}
    Q2 -->|Yes| PATSTORE[Store PAT or\nApp token as\norg/repo secret]
    Q2 -->|No| TOKEN_DONE([Use GITHUB_TOKEN])

    Q1 -->|External system /\nthird-party bot| Q3{Act as a\nservice identity?}
    Q3 -->|Yes| GHAPP[GitHub App\nFine-grained\nshort-lived tokens\ninstallation scoped]
    Q3 -->|No, act as user| OAUTHAPP2[OAuth App\nbrowser flow\nuser delegates access]

    Q1 -->|Personal CLI\nor script| FGPAT[Fine-grained PAT\nper-repo + per-permission\nset expiry]
    FGPAT --> Q4{Legacy system\nor broad scope?}
    Q4 -->|Yes| CPAT[Classic PAT\navoid if possible]
    Q4 -->|No| FGPAT_DONE([Use fine-grained PAT])

    Q1 -->|git clone/push\nfrom terminal| SSH[SSH Key\nor HTTPS + PAT]

    style TOKEN_DONE fill:#238636,color:#fff
    style GHAPP fill:#1f6feb,color:#fff
    style FGPAT_DONE fill:#238636,color:#fff
```

---

## 5. GitHub Projects v2 Data Flow

```mermaid
erDiagram
    ORGANISATION ||--o{ PROJECT : owns
    USER ||--o{ PROJECT : owns
    PROJECT ||--o{ VIEW : has
    PROJECT ||--o{ CUSTOM_FIELD : has
    PROJECT ||--o{ PROJECT_ITEM : contains
    PROJECT_ITEM }o--|| ISSUE : references
    PROJECT_ITEM }o--|| PULL_REQUEST : references
    PROJECT_ITEM }o--|| DRAFT_ISSUE : references
    REPOSITORY ||--o{ ISSUE : contains
    REPOSITORY ||--o{ PULL_REQUEST : contains
    ISSUE }o--o{ LABEL : tagged_with
    ISSUE }o--o{ MILESTONE : grouped_in
    ISSUE }o--o{ USER : assigned_to
    PULL_REQUEST }o--|| ISSUE : closes
    CUSTOM_FIELD ||--o{ FIELD_VALUE : has_values
    PROJECT_ITEM ||--o{ FIELD_VALUE : has
```

---

## 6. Branch Protection / Ruleset Enforcement Flow

```mermaid
flowchart LR
    DEV([Developer]) -->|git push\nor PR merge| CHECK{Branch\nprotection\ncheck}

    CHECK --> R1{Required\nreviews met?}
    R1 -->|No| BLOCK1[❌ Blocked:\nNeed N approvals]
    R1 -->|Yes| R2{Status checks\npass?}

    R2 -->|No / Pending| BLOCK2[❌ Blocked:\nCI not green]
    R2 -->|Yes| R3{Code owners\napproved?}

    R3 -->|No| BLOCK3[❌ Blocked:\nCODEOWNERS review needed]
    R3 -->|Yes| R4{Branch\nup to date?}

    R4 -->|No| UPDATE[Update branch\nwith base]
    UPDATE --> CHECK
    R4 -->|Yes| R5{Signed\ncommits?}

    R5 -->|No| BLOCK4[❌ Blocked:\nCommit not signed]
    R5 -->|Yes| MERGE([✅ Merge allowed])

    style MERGE fill:#238636,color:#fff
    style BLOCK1 fill:#da3633,color:#fff
    style BLOCK2 fill:#da3633,color:#fff
    style BLOCK3 fill:#da3633,color:#fff
    style BLOCK4 fill:#da3633,color:#fff
```

---

## 7. GitHub Actions Secret Precedence

```mermaid
flowchart TB
    JOB[Actions Job Step\naccesses secret MY_SECRET] --> ENV_JOB{Job uses\nenvironment?}

    ENV_JOB -->|Yes — env: production| ENV_SECRET{Environment secret\nMY_SECRET exists?}
    ENV_SECRET -->|Yes| USE_ENV[✅ Use environment secret\nhighest precedence]
    ENV_SECRET -->|No| REPO_CHECK

    ENV_JOB -->|No| REPO_CHECK{Repository secret\nMY_SECRET exists?}
    REPO_CHECK -->|Yes| USE_REPO[✅ Use repository secret]
    REPO_CHECK -->|No| ORG_CHECK{Organisation secret\nMY_SECRET exists\nand allowed for this repo?}
    ORG_CHECK -->|Yes| USE_ORG[✅ Use organisation secret]
    ORG_CHECK -->|No| EMPTY[Value is empty string]

    style USE_ENV fill:#238636,color:#fff
    style USE_REPO fill:#1f6feb,color:#fff
    style USE_ORG fill:#6e40c9,color:#fff
    style EMPTY fill:#da3633,color:#fff
```

---

## 8. GitHub App Authentication Flow

```mermaid
sequenceDiagram
    participant App as Your App/Bot
    participant GH as GitHub API

    Note over App: Has: App ID + Private Key (PEM)

    App->>App: Generate JWT<br/>{iss: APP_ID, exp: now+600}
    App->>App: Sign JWT with private key

    App->>GH: GET /app/installations<br/>Authorization: Bearer JWT
    GH-->>App: [{installation_id: 12345, ...}]

    App->>GH: POST /app/installations/12345/access_tokens<br/>Authorization: Bearer JWT
    GH-->>App: {token: "ghs_xxx", expires_at: "..."}

    Note over App,GH: Token valid for 1 hour

    App->>GH: API calls with installation token<br/>Authorization: Bearer ghs_xxx
    GH-->>App: Response data

    App->>App: Token expired? Regenerate
```

---

## 9. Repository Branching Strategy (GitHub Flow)

```mermaid
gitGraph
    commit id: "Initial commit"
    commit id: "Add CI workflow"

    branch feature/auth
    checkout feature/auth
    commit id: "Add login form"
    commit id: "Add JWT validation"
    commit id: "Add tests"

    checkout main
    merge feature/auth id: "Merge: feature/auth → main (PR #12)"

    branch hotfix/security-patch
    checkout hotfix/security-patch
    commit id: "Fix XSS vulnerability"

    checkout main
    merge hotfix/security-patch id: "Merge: hotfix → main (PR #13)"

    branch release/v2.0
    checkout release/v2.0
    commit id: "Bump version to 2.0.0"

    checkout main
    merge release/v2.0 id: "Release v2.0.0 tag"
```

---

## 10. Webhook Event Processing

```mermaid
flowchart LR
    GH[GitHub Event\ne.g. PR opened] -->|HTTP POST| WH[Your Webhook\nEndpoint]

    WH --> SIG{Verify\nX-Hub-Signature-256}
    SIG -->|Invalid| REJECT[❌ Return 401\nDrop event]
    SIG -->|Valid| PARSE[Parse payload\nX-GitHub-Event header]

    PARSE --> Q{Event type?}

    Q -->|pull_request\naction: opened| PR_HANDLER[PR Handler\nRun linter\nPost welcome comment]
    Q -->|issues\naction: opened| ISSUE_HANDLER[Issue Handler\nAuto-label\nAdd to project]
    Q -->|push\nbranch: main| PUSH_HANDLER[Push Handler\nTrigger deployment\nNotify Slack]
    Q -->|release\naction: published| REL_HANDLER[Release Handler\nBuild artifacts\nUpdate changelog]

    PR_HANDLER --> RESPOND[Return 200 OK\nWithin 10 seconds]
    ISSUE_HANDLER --> RESPOND
    PUSH_HANDLER --> RESPOND
    REL_HANDLER --> RESPOND

    style REJECT fill:#da3633,color:#fff
    style RESPOND fill:#238636,color:#fff
```
