# Blog Deploy Runbook

## Current flow

1. Edit Markdown under `source/_posts/` (later: the admin app writes these files).
2. Commit and push to `main`.
3. The GitHub Actions workflow `.github/workflows/deploy.yml` runs:
   - `npm ci`
   - `npm run build` (`hexo generate`)
   - uploads `public/` as a Pages artifact
   - deploys to GitHub Pages
4. Site: https://ihmod.github.io

## Required GitHub repository settings

- **Pages** → Build and deployment → Source: **GitHub Actions**
  (not "Deploy from a branch").
- **Environments** → `github-pages` → Deployment branches and tags:
  allow `main` (or select "All branches").
- **Default branch**: `main` (recommended).
- The old `master` branch (built site in `/docs`) is kept as the rollback path.
  Do not edit or delete it.

Without the environment rule, the deploy step fails with:

```
Branch "main" is not allowed to deploy to github-pages due to environment protection rules.
```

## Branches

- `main`: Hexo source + Butterfly theme + generated Markdown.
- `master`: legacy built site, served from `/docs` when Pages uses branch deployment.
  This is the rollback source, not the active source.

## Local build

```bash
npm ci
npm run build
```

Output: `public/`.

## Admin app integration

- Repo: `ihmod/blog-admin` (private).
- Deploy key: `/opt/ihmod-admin/secrets/deploy_key` on the admin server.
  Local generation copy: `~/ihmod-secrets/ihmod_blog_admin` (never commit).
- The admin app writes `source/_posts/**` and `source/img/uploads/**`,
  commits, and pushes `main`.
- GitHub Actions builds automatically; the admin server does not need Hexo.

## Rollback

If the Actions deployment is broken:

1. Go to Settings → Pages.
2. Change Source to **Deploy from a branch**.
3. Select branch `master`, folder `/docs`.
4. Save and wait for the legacy site to publish.

To restore the source-based flow: set Source back to **GitHub Actions**
and re-run the workflow.

## Troubleshooting (observed)

- **`Missing environment`** from `actions/deploy-pages`:
  ensure the job declares `environment: { name: github-pages }`
  and the deploy step has `id: deployment`.
- **`Branch "main" is not allowed to deploy to github-pages`**:
  Settings → Environments → `github-pages` → allow `main`.
- **Node.js 20 deprecation warnings**: non-fatal; actions are forced to Node 24.
- **First deployment 404**: wait 1–2 minutes for Pages propagation, then retry.

## Secrets

- Never commit private keys, `.env`, or `secrets/`.
- The old project-root key was moved to `~/ihmod-secrets/github`;
  revoke it on GitHub after the admin deploy key is working.
- The admin deploy key is repo-scoped and has write access only to
  `ihmod/ihmod.github.io`.
