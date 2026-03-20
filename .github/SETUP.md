# CROW CI/CD Pipeline Setup

## Architecture

This CI/CD pipeline uses GitHub Actions with reusable workflows:

```
.github repo (CROW-B3/.github)
├── workflows/
│   ├── reusable-worker-ci.yml    # Reusable: lint + deploy for worker services
│   ├── reusable-client-ci.yml    # Reusable: lint + build + deploy for frontend services
│   ├── orchestrate-deploy.yml    # Full platform deploy in dependency order
│   ├── pr-ci.yml                 # Validates workflow YAML changes
│   └── SETUP.md                  # This file
│
Each service repo (e.g., CROW-B3/core-auth-service)
└── .github/workflows/ci.yml     # Calls the reusable workflow
```

## Deployment Tiers (Dependency Order)

| Tier | Services | Depends On |
|------|----------|------------|
| T1 | core-auth-service | - |
| T2 | core-user-service, core-billing-service | T1 |
| T3 | core-organization-service, core-product-service, core-notification-service, core-analytics-service | T2 |
| T4 | core-interaction-service, core-pattern-service, core-chat-service, core-social-collector, core-social-processor | T3 |
| T5 | bff-chat-service, bff-qna-service, a2a-service, mcp-service, web-ingest-service, cctv-ingest-service, infra-crawl-service, core-api-gateway | T4 |
| T6 | auth-client, dashboard-client, landing-client, rogue-store | T5 |

## Required Secrets

### Organization-level secrets (set at github.com/organizations/CROW-B3/settings/secrets/actions)

| Secret | Description |
|--------|-------------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with Workers deploy permissions |
| `CROW_DEPLOY_TOKEN` | GitHub PAT with `repo` and `actions:write` scope for cross-repo dispatch |

### Per-repo secrets (only if not using org-level secrets)

Each service repo needs `CLOUDFLARE_API_TOKEN`.

## Trigger Behavior

| Event | Environment | Action |
|-------|-------------|--------|
| Push to `main` | dev | Lint + typecheck + deploy to dev |
| Pull request to `main` | - | Lint + typecheck only (no deploy) |
| Manual dispatch (dev) | dev | Lint + typecheck + deploy to dev |
| Manual dispatch (production) | production | Lint + typecheck + deploy to production |
| Orchestrator dispatch | dev/production | Full platform deploy in dependency order |

## Cloudflare Account

- Account ID: `8f0203259905d8923687286c84921e6c`
- Deploy command (workers): `wrangler deploy --env dev --minify`
- Deploy command (clients): `bun run build:cloudflare && wrangler deploy --env dev`

## Setting Up a New Service

1. Create `.github/workflows/ci.yml` in the service repo
2. For worker services, call `CROW-B3/.github/.github/workflows/reusable-worker-ci.yml@main`
3. For client services, call `CROW-B3/.github/.github/workflows/reusable-client-ci.yml@main`
4. Ensure the repo has access to `CLOUDFLARE_API_TOKEN` secret
5. Add the service to `orchestrate-deploy.yml` if needed
