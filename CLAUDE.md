# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Secrets

Never read, display, or reference the contents of `functions/local.settings.json` or `infra/*.tfvars` — they contain secrets.

Do not print Yoco API keys or GitHub/Azure secrets. The Yoco key must only be referenced by variable name.

## Project Overview

Buffy Braveart Gallery — an art gallery e-commerce site built as an Azure Static Web App with a serverless API backend. Visitors browse artwork and purchase via Yoco Checkout. The artwork catalogue is temporarily managed in `functions/src/data/artworks.json` until a client-friendly product admin flow is selected.

The site previously used Stripe Products and Stripe Checkout. Stripe has been removed because the client is based in South Africa and Yoco is the selected payment gateway.

## Naming Convention

All Azure resources follow: `{type}-{project}-{env}-{region}-{instance}`

| Variable    | Value           |
| ----------- | --------------- |
| Project     | `braveart`      |
| Environment | `prd`           |
| Region      | `uks` (uksouth) |
| Instance    | `01`            |

| Resource       | Name                        |
| -------------- | --------------------------- |
| Resource group | `rg-braveart-prd-uks-01`    |
| Static Web App | `stapp-braveart-prd-uks-01` |

## Architecture

**Frontend** (`frontend/`): Vanilla HTML/CSS/JS with no framework and no build step. Each page has its own JS file loaded directly by the browser. CSS uses BEM naming where practical. Prices are stored in cents and formatted client-side for display as South African Rand.

**Gallery styling**: `frontend/gallery.html` loads `frontend/js/gallery.js`. Gallery cache-busting uses query-string versions on CSS/JS, and `frontend/staticwebapp.config.json` sets `Cache-Control: no-cache` for `/gallery` and `/js/*` to avoid stale checkout scripts.

**Artwork images**: Current static images live in `frontend/images/artworks/` and are referenced from catalogue records using paths like `/images/artworks/artwork-01.jpg`.

**API** (`functions/`): Azure Functions v4 (Node.js, programming model v4). Each function is a standalone file in `functions/src/functions/`. Functions are registered via `app.http()` — no `function.json` files.

**Data flow**:

- Artwork catalogue → `functions/src/data/artworks.json` records with `status: "available"`
- Artwork images → static files in `frontend/images/artworks/` or public URLs referenced by `imageUrl`
- Purchases → Yoco Checkout sessions, currency `ZAR`

**Checkout security**: The browser sends only `artworkId` to `/api/checkout`. The API must look up the artwork server-side and use the stored server-side `price` and `currency`. Never trust a browser-supplied amount.

**Yoco amount format**: Yoco expects amounts in cents. `1000000` means `R10,000.00`.

**Secrets**: Stored in GitHub Actions environments (`dev`, `prd`). OIDC is used for Azure auth. `YOCO_SECRET_KEY` and `FRONTEND_URL` are set as SWA app settings via Terraform. Local dev reads from `functions/local.settings.json`.

**Auth & routing** (`frontend/staticwebapp.config.json`): No role-based restrictions. `/gallery` rewrites to `gallery.html`. Global security headers are applied (CSP, X-Frame-Options, X-Content-Type-Options).

**Infrastructure** (`infra/`): Terraform (azurerm ~> 4.0). Custom domain: `buffybraveart.com`. Yoco secret key is passed as a Terraform variable named `yoco_secret_key`.

**CI/CD** (`.github/workflows/`): GitHub Actions with OIDC authentication. Pipelines consume the shared reusable workflows and composite actions from `skyhaven-ltd/pipeline-engineering-github-actions`, SHA-pinned to a released tag.

- `swa.yml` — deploys frontend + API to SWA. Auto-triggers on `frontend/**` or `functions/**` pushes to feature branches; manual trigger selects `dev`/`prd` environment.
- `terraform.yml` — runs Terraform plan/apply/destroy. Auto-triggers on `infra/**` pushes to feature branches (always targets `prd`). Manual trigger selects environment and action. State plumbing and Key Vault secret fetching are delegated to the shared composite actions.
- `lint.yml` — shared `reusable-lint.yml` (MegaLinter, `cupcake` flavour) on PRs to `main`. Config in `.github/validation/.mega-linter.yml`.
- `pr-validation.yml` — shared `reusable-terraform.yml` on PRs to `main`: Terraform hygiene, zizmor, then real `dev`+`prd` plans with Checkov plan-aware scanning. Plan-time secrets come from the platform Key Vault via `tf_var_secrets`.
- `tag.yml` — shared `reusable-tag.yml` creates a semver tag on PR merge. Branch prefix determines bump: `major/`, `minor/`, `patch/`.

**Branch naming**: Use `major/`, `minor/`, or `patch/` prefixes. This is required — it controls which CI workflows trigger and what version tag is created on merge.

**Custom domain**: `dns_delegated` Terraform variable is `false` by default. Set to `true` in the tfvars only after GoDaddy nameservers have been pointed to Azure DNS, otherwise the custom domain resources will fail to provision.

## Development

### Local API

```bash
cd functions && npm install
# Create functions/local.settings.json with required env vars (see below)
func start
```

The frontend is static files — serve `frontend/` with any HTTP server, or use `swa start` from Azure Static Web Apps CLI.

### Required Environment Variables

| Variable          | Used by                                |
| ----------------- | -------------------------------------- |
| `YOCO_SECRET_KEY` | `checkout`                             |
| `FRONTEND_URL`    | `checkout` success/cancel/failure URLs |

### API Endpoints

| Method | Route           | Auth      | Function      |
| ------ | --------------- | --------- | ------------- |
| GET    | `/api/artworks` | anonymous | `getArtworks` |
| POST   | `/api/checkout` | anonymous | `checkout`    |

## Artwork catalogue editing

Temporary catalogue file:

```text
functions/src/data/artworks.json
```

Record shape:

```json
{
  "id": "artwork-01",
  "name": "Artwork 1",
  "description": "",
  "price": 1000000,
  "currency": "ZAR",
  "imageUrl": "/images/artworks/artwork-01.jpg",
  "status": "available"
}
```

Only `available` records are returned by `/api/artworks`. Use `draft` or `sold` to hide records.

## Validation

Before committing payment or catalogue changes, run at least:

```bash
cd functions
npm ci
node --check src/functions/checkout.js
node --check src/functions/artworkCache.js
node --check src/functions/getArtworks.js
node -e "JSON.parse(require('fs').readFileSync('src/data/artworks.json','utf8')); console.log('artworks json ok')"
npm audit --audit-level=high
```

For infrastructure changes, run:

```bash
terraform -chdir=infra fmt -check
terraform -chdir=infra validate
```

Run `terraform init -backend=false` first if providers are not installed locally.
