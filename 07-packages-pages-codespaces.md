# GitHub Packages, Pages & Codespaces

---

## GitHub Packages

> Artifact registry integrated with GitHub authentication and Actions. Hosts: npm, Docker/OCI, Maven, Gradle, NuGet, RubyGems.

### Container Registry (GHCR)

```bash
# Authenticate
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag image
docker tag myapp:latest ghcr.io/OWNER/myapp:latest

# Push
docker push ghcr.io/OWNER/myapp:latest

# Pull
docker pull ghcr.io/OWNER/myapp:latest
```

**In GitHub Actions (preferred)**:
```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: docker/build-push-action@v5
  with:
    push: true
    tags: |
      ghcr.io/${{ github.repository }}:latest
      ghcr.io/${{ github.repository }}:${{ github.sha }}
    labels: |
      org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
```

### npm Packages

`.npmrc` in project root:
```
@OWNER:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

`package.json`:
```json
{
  "name": "@OWNER/my-package",
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

Actions publish:
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    registry-url: 'https://npm.pkg.github.com'
    scope: '@OWNER'
- run: npm publish
  env:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Package Visibility & Access

| Setting | Behaviour |
|---|---|
| Public package | Anyone can pull anonymously |
| Private package | Requires auth; linked to repo permissions |
| Org-level package | Managed via org member permissions |

Packages inherit repo visibility by default. Can be changed independently in package settings.

### Storage & Billing Limits

| Plan | Free storage | Free data transfer |
|---|---|---|
| Free | 500 MB | 1 GB/month |
| Pro | 2 GB | 10 GB/month |
| Team | 2 GB | 10 GB/month |
| Enterprise | 50 GB | 100 GB/month |

---

## GitHub Pages

> Free static site hosting from a GitHub repository. Supports Jekyll natively; any static site works.

### Source Options

| Source | URL | Use case |
|---|---|---|
| `main` branch `/` root | `https://username.github.io/repo` | Simplest setup |
| `main` branch `/docs` folder | Same URL | Docs alongside code |
| `gh-pages` branch | Same URL | Separate deploy branch |
| GitHub Actions (custom) | Same URL | Any SSG (Hugo, Next.js, Vite…) |

### User/Org Sites vs Project Sites

| Type | Repo name | URL |
|---|---|---|
| User site | `username.github.io` | `https://username.github.io` |
| Org site | `orgname.github.io` | `https://orgname.github.io` |
| Project site | Any repo | `https://username.github.io/reponame` |

### Custom Domain

1. Add `CNAME` file to Pages source with your domain: `www.example.com`
2. At DNS provider: add CNAME record `www` → `username.github.io`
3. For apex domain: add A records pointing to GitHub's IPs:
   ```
   185.199.108.153
   185.199.109.153
   185.199.110.153
   185.199.111.153
   ```
4. Enable HTTPS: Settings → Pages → Enforce HTTPS

### Actions Deploy Pattern (Any SSG)

```yaml
name: Deploy Pages
on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist

  deploy:
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

### Jekyll Configuration (`./_config.yml`)

```yaml
title: My Project
description: Documentation site
theme: minima
baseurl: "/my-repo"    # For project sites (not user/org sites)
url: "https://username.github.io"
```

### Pages Limits

| Resource | Limit |
|---|---|
| Site size | 1 GB |
| Bandwidth/month | 100 GB (soft) |
| Build frequency | 10 builds/hour |
| Build time | 10 minutes |
| Custom domains | 1 per Pages site |

---

## GitHub Codespaces

> Cloud-hosted development environment. VS Code (browser or desktop) connected to a remote container pre-configured from your repo.

### How It Works

```
Repository
└── .devcontainer/
    ├── devcontainer.json    # Container config
    └── Dockerfile           # Custom image (optional)

→ GitHub spins up container on Azure
→ VS Code connects via browser or SSH tunnel
→ Port forwarding exposes local services
→ Persisted /workspaces mount
```

### devcontainer.json

```json
{
  "name": "My Project Dev",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "features": {
    "ghcr.io/devcontainers/features/azure-cli:1": {},
    "ghcr.io/devcontainers/features/terraform:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "ms-azuretools.vscode-azureappservice"
      ],
      "settings": {
        "editor.formatOnSave": true
      }
    }
  },
  "postCreateCommand": "npm install",
  "postStartCommand": "npm run dev &",
  "forwardPorts": [3000, 5432],
  "portsAttributes": {
    "3000": { "label": "App", "onAutoForward": "openBrowser" }
  },
  "remoteEnv": {
    "NODE_ENV": "development"
  }
}
```

### Machine Types

| Type | vCPU | RAM | Storage | Cost/hr |
|---|---|---|---|---|
| 2-core | 2 | 8 GB | 32 GB | $0.18 |
| 4-core | 4 | 16 GB | 32 GB | $0.36 |
| 8-core | 8 | 32 GB | 64 GB | $0.72 |
| 16-core | 16 | 64 GB | 128 GB | $1.44 |
| 32-core | 32 | 128 GB | 128 GB | $2.88 |

### Free Tier (personal accounts)
- 120 core-hours/month free (= 60 hrs on 2-core)
- 15 GB storage free

### Codespace Lifecycle

```
Created → Running (active, billed) → Stopped (inactive, storage only)
                                   → Deleted (no cost)

Auto-stop: after 30 min idle (configurable)
Auto-delete: after 30 days stopped (configurable at org level)
```

### Secrets in Codespaces

- Set at: Settings → Codespaces → Secrets
- Available as environment variables in the container
- Separate from Actions secrets

### Prebuilds

Reduces startup time by pre-building the container image when code is pushed:
- Config: Org/Repo Settings → Codespaces → Prebuild configuration
- Triggers on: push to configured branch
- Stores pre-built image; new codespaces start instantly

---

## GitHub Copilot

> AI coding assistant. Available in VS Code, JetBrains, Neovim, GitHub.com, CLI.

### Plans
| Plan | Price | Features |
|---|---|---|
| Individual | $10/month | Code completions, chat |
| Business | $19/user/month | + policy controls, audit logs |
| Enterprise | $39/user/month | + Copilot customisation, fine-tuning on org code |

### Features

| Feature | Description |
|---|---|
| **Inline completions** | Ghost text suggestions as you type |
| **Copilot Chat** | Conversational AI in IDE sidebar |
| **Inline Chat** | `/fix`, `/explain`, `/test` on selected code |
| **Commit message** | Auto-generate from staged diff |
| **PR description** | Auto-generate from changed files |
| **Code review** | Copilot suggests review comments on PRs |
| **CLI** | `gh copilot suggest "how do I..."` |
| **Copilot Workspace** | Plan + implement features from issue (Enterprise) |

### Copilot in Actions

```yaml
- name: Copilot Auto-fix
  uses: github/copilot-autofix-action@v1
```

Automatically fixes code scanning alerts using Copilot suggestions.
