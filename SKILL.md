---
name: vercel-github-actions-deploy
description: "Set up GitHub Actions to deploy any Vercel project using the Git Author Override method, enabling teammates to deploy on the free Hobby plan. Use when the user asks about Vercel deployment via GitHub Actions, CI/CD for Vercel, letting teammates deploy on Vercel free plan, bypassing Vercel's Hobby plan deploy restrictions, or automating Vercel production deploys. Covers workflow setup, GitHub Secrets configuration, and package manager variants (bun, npm, pnpm)."
license: MIT
metadata:
  author: "itsOmSarraf"
  version: "1.0"
  tags: "vercel, github-actions, ci-cd, deployment, free-plan"
---

# Vercel GitHub Actions Deploy (Git Author Override)

Deploy Vercel projects from GitHub Actions on the **free Hobby plan** — letting any teammate trigger production deploys.

## The Problem

Vercel's free plan ties deployments to the **account owner**. When a teammate pushes to `main`, Vercel checks the git commit author and rejects it. This normally requires the Pro plan ($20/mo per member).

## How It Works

```
Teammate pushes to main
        ↓
GitHub Actions triggers
        ↓
Rewrites commit author to account owner (on disposable CI runner only)
        ↓
Vercel CLI builds and deploys to production
```

- Runs on every push to `main` — by **anyone**, plus manual dispatch
- Actual repo history stays **untouched** (rewrite only on disposable runner)

## Prerequisites (User Action Required)

The assistant creates the workflow YAML, but the user **must provide these 5 secrets** — the assistant cannot obtain them.

**Step 1 — Link project and create token:**

```bash
npm install -g vercel
npx vercel link          # creates .vercel/project.json with orgId + projectId
echo ".vercel" >> .gitignore
```

Then create a deploy token at [vercel.com/account/tokens](https://vercel.com/account/tokens).

**Step 2 — Add all 5 as GitHub repository secrets** (Settings → Secrets and variables → Actions):

| Secret Name | Source |
|-------------|--------|
| `VERCEL_TOKEN` | Token from vercel.com/account/tokens |
| `VERCEL_ORG_ID` | `orgId` in `.vercel/project.json` |
| `VERCEL_PROJECT_ID` | `projectId` in `.vercel/project.json` |
| `DEPLOY_EMAIL` | Email of the Vercel account owner |
| `DEPLOY_NAME` | Display name of the Vercel account owner |

## Setup

### 1. Pick the workflow for the project's package manager

- **Bun** (has `bun.lock`) → `examples/deploy-bun.yml`
- **npm** (has `package-lock.json`) → `examples/deploy-npm.yml`
- **pnpm** (has `pnpm-lock.yaml`) → `examples/deploy-pnpm.yml`

Copy the chosen file to `.github/workflows/deploy.yml`.

### 2. Commit and push

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add Vercel deploy workflow"
git push origin main
```

### 3. Verify the deployment

1. Open the repo's **Actions** tab — confirm the workflow run shows a green check
2. Click the run → check the "Deploy to Vercel" step logged a production URL
3. Visit the logged URL and confirm the site is live

**If the run fails**, check these common causes:
- `VERCEL_TOKEN is not set` → GitHub Secrets are case-sensitive; verify the name is exactly `VERCEL_TOKEN`
- Build fails in CI but works locally → ensure all env vars exist in the Vercel dashboard (Settings → Environment Variables); `vercel pull` fetches them automatically but they must be configured first
- `Error: No commits found` → ensure the checkout step uses default `fetch-depth` (1 is fine)

## Workflow Template (Bun)

```yaml
name: Deploy to Vercel

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Override git author to repo owner
        run: |
          git config user.email "${{ secrets.DEPLOY_EMAIL }}"
          git config user.name "${{ secrets.DEPLOY_NAME }}"
          git commit --amend --reset-author --no-edit

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - uses: oven-sh/setup-bun@v2

      - name: Install Vercel CLI
        run: npm install -g vercel

      - name: Pull Vercel environment
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}

      - name: Build project
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}

      - name: Deploy to Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

**Using npm?** Remove the Bun step. **Using pnpm?** Replace it with `uses: pnpm/action-setup@v4`. See `examples/` for ready-to-use files.

## Preventing Double Deploys

If the account owner pushes, both Vercel's Git integration and GitHub Actions deploy simultaneously. To prevent this, set **Ignored Build Step** to `exit 0` in Vercel Dashboard → Project Settings → Git. This is a manual step in the Vercel dashboard.

## Preview Deploys for PRs

Change the trigger to `pull_request` and remove `--prod` from build/deploy steps. See [templates/deploy-workflow-template.md](templates/deploy-workflow-template.md) for a full preview workflow with PR comment integration.

## FAQ

**Does this work with monorepos?**
Yes. Ensure `vercel link` points to the correct project. See [templates/deploy-workflow-template.md](templates/deploy-workflow-template.md#monorepo-support) for multi-project setup.

## Additional Resources

- Full step-by-step setup guide: [templates/deploy-workflow-template.md](templates/deploy-workflow-template.md)
- Ready-to-use workflow files: [examples/](examples/)
- Vercel CLI docs: https://vercel.com/docs/cli
- GitHub Actions docs: https://docs.github.com/en/actions
