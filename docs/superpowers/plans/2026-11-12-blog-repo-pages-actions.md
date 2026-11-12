# Blog Repo Migration & GitHub Actions Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the Hexo source into `ihmod/ihmod.github.io` on `main`, deploy via GitHub Actions, remove the old `hexo-deployer-git` flow, and keep the site at `https://ihmod.github.io`.

**Architecture:** One repository holds the Hexo source and Butterfly theme on `main`. GitHub Pages is configured with "Source: GitHub Actions". A workflow runs `npm ci` + `hexo generate` and deploys the `public/` artifact. The old `master` branch (built site) is kept as a backup and is no longer the Pages source.

**Tech Stack:** Hexo 7.3.0, Node.js 22, GitHub Actions (`actions/checkout`, `actions/setup-node`, `actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages`).

---

## Prerequisites

- Node.js 22 and npm 9+ installed locally.
- Git installed.
- Access to the GitHub repository `ihmod/ihmod.github.io` (push permission).
- Ability to open GitHub repository Settings in a browser.
- No secrets are committed at any point.

## File Structure

| Path | Responsibility |
|---|---|
| `.gitignore` | Exclude build output, dependencies, IDE/Obsidian config, keys, env files |
| `.github/workflows/deploy.yml` | Build and deploy the site to GitHub Pages |
| `_config.yml` | Remove the old `deploy:` block |
| `package.json` / `package-lock.json` | Remove `hexo-deployer-git` |
| `docs/deploy.md` | Document the new deploy flow and rollback |
| `docs/superpowers/specs/2026-11-12-hexo-sqlite-admin-design.md` | Design reference (already written) |

No changes to `source/`, `themes/`, `_config.butterfly.yml`, or post content in this plan.

---

## Task 1: Back up the current project

**Files:** none (creates backups outside the repository).

- [ ] **Step 1: Create a backup directory and archive source/config/theme**

Run from `/home/ihmod/cc_projects/IHMOD`:

```bash
mkdir -p ~/ihmod-backups
STAMP=$(date +%Y%m%d-%H%M%S)
tar -czf ~/ihmod-backups/ihmod-source-$STAMP.tar.gz \
  --exclude=node_modules \
  _config.yml _config.butterfly.yml package.json package-lock.json \
  scaffolds source themes
tar -czf ~/ihmod-backups/ihmod-build-$STAMP.tar.gz public .deploy_git
ls -lh ~/ihmod-backups
```

Expected: two `.tar.gz` files appear with non-zero sizes. The source archive should be roughly 27–30 MB (theme included). The `github`/`github.pub` keys are **not** included in the plaintext archive; they are moved separately in Task 2.

- [ ] **Step 2: Verify the source archive lists key files**

```bash
tar -tzf ~/ihmod-backups/ihmod-source-$STAMP.tar.gz | grep -E '^(source/_posts/|themes/butterfly/|_config\.yml|package\.json)' | head -20
```

Expected: lines for `_config.yml`, `package.json`, `source/_posts/...`, `themes/butterfly/...`.

- [ ] **Step 3: Record the checksums**

```bash
sha256sum ~/ihmod-backups/*.tar.gz > ~/ihmod-backups/SHA256SUMS-$STAMP.txt
cat ~/ihmod-backups/SHA256SUMS-$STAMP.txt
```

Expected: two checksum lines; keep this file until the migration is verified.

---

## Task 2: Remove private keys from the project and initialize Git

**Files:**
- Modify: `.gitignore`
- Move: `github`, `github.pub` → `~/ihmod-secrets/`
- Create: `.git/` (local repository)

- [ ] **Step 1: Move the deploy keys out of the project directory**

```bash
cd /home/ihmod/cc_projects/IHMOD
mkdir -p ~/ihmod-secrets
mv github github.pub ~/ihmod-secrets/
chmod 600 ~/ihmod-secrets/github
ls -la ~/ihmod-secrets
```

Expected: `github` and `github.pub` are gone from the project root; `github` has mode `-rw-------`.

- [ ] **Step 2: Replace `.gitignore` with the complete content**

```gitignore
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
_multiconfig.yml
CNAME

# IDE / editor
.idea/
.vscode/
.obsidian/

# Secrets
.env
.env.*
secrets/
github
github.pub
*.key
*.pem

# Local backups
backups/
```

- [ ] **Step 3: Initialize the repository on `main`**

```bash
git init -b main
git config user.name "iHMOD"
git config user.email "ihmod@users.noreply.github.com"
```

Expected: `Initialized empty Git repository in .../IHMOD/.git/`.

- [ ] **Step 4: Stage the source, theme, config, and docs**

```bash
git add .gitignore _config.yml _config.butterfly.yml package.json package-lock.json \
  scaffolds source themes docs
git status --short
```

Expected: staged files only; **no** `github`, `github.pub`, `node_modules/`, `public/`, `db.json`, `.deploy_git/`, `.idea/`, or `.obsidian/`.

- [ ] **Step 5: Verify no secrets are staged**

```bash
git ls-files --cached | grep -E '(^|/)(github|github\.pub|\.env|.*\.pem|.*\.key)$' || echo "OK: no secrets staged"
git ls-files --cached | grep -E 'node_modules/|^public/|^db\.json$|\.deploy_git/' || echo "OK: no build artifacts staged"
```

Expected:
```
OK: no secrets staged
OK: no build artifacts staged
```

- [ ] **Step 6: Commit the baseline**

```bash
git commit -m "chore: import Hexo source and Butterfly theme"
git log --oneline -1
```

Expected: one commit on `main` containing `source/`, `themes/butterfly/`, configs, and `docs/`.

---

## Task 3: Add the GitHub Actions deployment workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Create the workflow file**

```bash
mkdir -p .github/workflows
```

Write `.github/workflows/deploy.yml`:

```yaml
name: Deploy Hexo

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Generate site
        run: npm run build

      - name: Configure Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public

      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

- [ ] **Step 2: Verify the workflow YAML parses**

```bash
node -e "const fs=require('fs'); const s=fs.readFileSync('.github/workflows/deploy.yml','utf8'); if(!s.includes('deploy-pages')) process.exit(1); console.log('workflow file present')"
```

Expected: `workflow file present`.

- [ ] **Step 3: Commit the workflow**

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: deploy Hexo via GitHub Actions"
git log --oneline -1
```

Expected: a new commit containing only the workflow file.

---

## Task 4: Remove the old `hexo-deployer-git` deployment path

**Files:**
- Modify: `_config.yml` (remove the `deploy:` block)
- Modify: `package.json` / `package-lock.json` (remove `hexo-deployer-git`)

- [ ] **Step 1: Confirm the old block exists**

```bash
grep -n -A6 '^deploy:' _config.yml
```

Expected: output includes `type: git`, `repo: git@github.com:ihmod/ihmod.github.io.git`, `branch: master`.

- [ ] **Step 2: Remove the `deploy:` block from `_config.yml`**

Delete these lines (the exact block at the end of the file):

```yaml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: git@github.com:ihmod/ihmod.github.io.git
  branch: master
```

Verify:

```bash
grep -n '^deploy:' _config.yml || echo "OK: deploy block removed"
```

Expected: `OK: deploy block removed`.

- [ ] **Step 3: Uninstall `hexo-deployer-git`**

```bash
npm uninstall hexo-deployer-git
```

Expected: npm removes the package and updates `package.json` and `package-lock.json`.

- [ ] **Step 4: Verify no references remain**

```bash
grep -n "hexo-deployer-git" package.json package-lock.json || echo "OK: dependency removed"
```

Expected: `OK: dependency removed`.

- [ ] **Step 5: Build locally to prove nothing broke**

```bash
npm run clean
npm run build
```

Expected: `hexo generate` finishes with `INFO  Generated: ...` and exit code 0.

- [ ] **Step 6: Commit**

```bash
git add _config.yml package.json package-lock.json
git commit -m "chore: remove hexo-deployer-git and old deploy config"
git log --oneline -1
```

Expected: a new commit containing only those three files.

---

## Task 5: Verify the local build output

**Files:** none (verification only).

- [ ] **Step 1: Confirm the homepage exists**

```bash
test -f public/index.html && echo "OK: index.html"
grep -o '<title>[^<]*</title>' public/index.html | head -1
```

Expected: `OK: index.html` and a title containing `iHMOD`.

- [ ] **Step 2: Count generated post pages**

```bash
find public/posts -name '*.html' | wc -l
find public/posts -name '*.html' | head -5
```

Expected: 13 post HTML files (matching the 13 source posts). The file names are `abbrlink` values, for example `312e3d12.html`.

- [ ] **Step 3: Confirm search index and assets exist**

```bash
test -f public/search.xml && echo "OK: search.xml"
find public -maxdepth 2 -type d | sort | head -20
```

Expected: `OK: search.xml`; directories include `public/posts`, `public/tags`, `public/categories`, `public/archives`, `public/link`, `public/css`, `public/js`, `public/img`.

- [ ] **Step 4: Confirm the build is ignored by Git**

```bash
git status --short
```

Expected: no output (or only ignored files; `public/` must not appear).

---

## Task 6: Push the source to GitHub `main`

**Files:** none (remote operation).

**Manual prerequisite:** authenticate to GitHub with an account that can push to `ihmod/ihmod.github.io`. Do not use the old project-root key; use your normal SSH key or an HTTPS personal access token.

- [ ] **Step 1: Add the remote and inspect existing branches**

```bash
git remote add origin git@github.com:ihmod/ihmod.github.io.git
# If origin already exists, use:
# git remote set-url origin git@github.com:ihmod/ihmod.github.io.git
# HTTPS alternative:
# git remote set-url origin https://github.com/ihmod/ihmod.github.io.git
git ls-remote --heads origin
```

Expected: lists existing branches (likely `refs/heads/master`). If `main` already exists, stop and inspect it before pushing.

- [ ] **Step 2: Push `main` without force**

```bash
git push -u origin main
git ls-remote --heads origin | grep -E 'refs/heads/(main|master)'
```

Expected: `refs/heads/main` exists and points at your new commit; `refs/heads/master` is unchanged.

If the push is rejected, do **not** force-push. Inspect the remote with `git fetch origin` and `git log --oneline --all --graph`.

- [ ] **Step 3: Manual — set the default branch**

In GitHub: repository → Settings → General → Default branch → switch to `main`.

Expected: the repository opens on `main`; the `master` branch still exists as a backup.

---

## Task 7: Configure Pages to use GitHub Actions and verify the first deploy

**Files:** none (GitHub settings + verification).

- [ ] **Step 1: Manual — set Pages source**

GitHub repository → Settings → Pages → Build and deployment → Source: **GitHub Actions**.

Do not choose "Deploy from a branch".

- [ ] **Step 2: Trigger the workflow**

Push the current `main` (already pushed in Task 6) or run the workflow manually:

```bash
git commit --allow-empty -m "ci: trigger pages deploy"
git push origin main
```

Expected: a new workflow run appears under the repository's **Actions** tab.

- [ ] **Step 3: Wait for the workflow to succeed**

```bash
# Replace OWNER/REPO if needed; this URL is public.
curl -s "https://api.github.com/repos/ihmod/ihmod.github.io/actions/runs?per_page=1" \
  | grep -E '"status"|"conclusion"|"html_url"' | head -5
```

Expected: the latest run eventually has `"status": "completed"` and `"conclusion": "success"`. If the API is rate-limited, check the Actions tab in the browser.

- [ ] **Step 4: Verify the live site**

```bash
curl -sI https://ihmod.github.io | head -1
curl -s https://ihmod.github.io | grep -o '<title>[^<]*</title>' | head -1
```

Expected:
- `HTTP/2 200` (or `HTTP/1.1 200`)
- a title containing `iHMOD`

If you see 404, wait 1–2 minutes and retry; if it persists, check Pages settings and the workflow logs.

---

## Task 8: Verify content parity

**Files:** none (verification only).

- [ ] **Step 1: Verify a known post by abbrlink**

The file `source/_posts/网安/信息收集.md` has `abbrlink: 312e3d12`.

```bash
curl -sI https://ihmod.github.io/posts/312e3d12.html | head -1
curl -s https://ihmod.github.io/posts/312e3d12.html | grep -o '<title>[^<]*</title>' | head -1
```

Expected: HTTP 200 and a title containing `信息收集`.

- [ ] **Step 2: Verify the search index**

```bash
curl -sI https://ihmod.github.io/search.xml | head -1
curl -s https://ihmod.github.io/search.xml | grep -c '<entry>'
```

Expected: HTTP 200 and a positive entry count (13 posts).

- [ ] **Step 3: Verify navigation pages**

```bash
for p in archives tags categories link; do
  code=$(curl -s -o /dev/null -w '%{http_code}' "https://ihmod.github.io/$p/")
  echo "$p: $code"
done
```

Expected: each line ends with `200`.

- [ ] **Step 4: Record the live deployment commit**

```bash
git rev-parse HEAD
```

Expected: the commit SHA shown in the Actions run that deployed the site. Keep it for rollback reference.

---

## Task 9: Create a dedicated deploy key for the future admin app

**Files:** none in the repository (key stored outside).

**Context:** The admin app will push generated Markdown to this repository. It must use a repo-scoped write key, not a personal SSH key.

- [ ] **Step 1: Generate an ed25519 key**

```bash
mkdir -p ~/ihmod-secrets
ssh-keygen -t ed25519 -C "ihmod-blog-admin" -f ~/ihmod-secrets/ihmod_blog_admin -N ""
chmod 600 ~/ihmod-secrets/ihmod_blog_admin
cat ~/ihmod-secrets/ihmod_blog_admin.pub
```

Expected: a new public key line printed; the private key is not printed.

- [ ] **Step 2: Manual — add the public key as a deploy key**

GitHub repository → Settings → Deploy keys → Add deploy key:
- Title: `ihmod-blog-admin`
- Key: paste the public key
- **Allow write access: checked**

- [ ] **Step 3: Test clone and write access without committing changes**

```bash
rm -rf /tmp/ihmod-admin-key-test
GIT_SSH_COMMAND="ssh -i ~/ihmod-secrets/ihmod_blog_admin -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new" \
  git clone -b main git@github.com:ihmod/ihmod.github.io.git /tmp/ihmod-admin-key-test
git -C /tmp/ihmod-admin-key-test push --dry-run origin main
```

Expected: clone succeeds; `push --dry-run` exits 0 and reports "Everything up-to-date" or a dry-run summary. No actual commit is pushed.

- [ ] **Step 4: Revoke the old project-root key**

If `~/ihmod-secrets/github` (the old key) is registered as a deploy key on GitHub, delete it in Settings → Deploy keys. Do not delete the new `ihmod-blog-admin` key.

Expected: old key is gone; the new key remains with write access.

- [ ] **Step 5: Do not commit the key**

```bash
git status --short
```

Expected: no output; `~/ihmod-secrets/` is outside the repository.

---

## Task 10: Document the deploy flow and rollback

**Files:**
- Create: `docs/deploy.md`

- [ ] **Step 1: Write `docs/deploy.md`**

```markdown
# Blog Deploy Runbook

## Current flow

1. Edit Markdown under `source/_posts/` (later: the admin app writes these files).
2. Commit and push to `main`.
3. GitHub Actions workflow `.github/workflows/deploy.yml` runs:
   - `npm ci`
   - `npm run build` (`hexo generate`)
   - uploads `public/` as a Pages artifact
   - deploys to GitHub Pages
4. Site: https://ihmod.github.io

## Repository layout

- Branch `main`: Hexo source + Butterfly theme + generated Markdown.
- Branch `master`: legacy built-site backup. Do not use it as the Pages source.
- Pages source: GitHub Actions.

## Local build

```bash
npm ci
npm run build
```

Output: `public/`.

## Admin app integration

- Repo: `ihmod/blog-admin` (private).
- Deploy key: `/opt/ihmod-admin/secrets/deploy_key` on the admin server (local generation copy: `~/ihmod-secrets/ihmod_blog_admin`; never commit).
- The admin app writes `source/_posts/**` and `source/img/uploads/**`, commits, and pushes `main`.
- GitHub Actions builds automatically; the admin app does not need Hexo on the VPS.

## Rollback

If the Actions deployment is broken:

1. Go to Settings → Pages.
2. Change Source to "Deploy from a branch".
3. Select branch `master`, folder `/ (root)`.
4. Save and wait for the legacy site to publish.

To restore the source-based flow, switch Source back to GitHub Actions and re-run the workflow.

## Secrets

- Never commit private keys, `.env`, or `secrets/`.
- The old project-root key was moved to `~/ihmod-secrets/github` and should be revoked on GitHub.
- The admin deploy key is repo-scoped and has write access only to `ihmod/ihmod.github.io`.
```

- [ ] **Step 2: Commit the runbook**

```bash
git add docs/deploy.md
git commit -m "docs: add deploy runbook and rollback steps"
git log --oneline -3
```

Expected: three recent commits including the runbook.

- [ ] **Step 3: Push and verify the final state**

```bash
git push origin main
git status --short
git ls-remote --heads origin | grep -E 'refs/heads/(main|master)'
```

Expected: push succeeds; working tree clean; `main` and `master` both exist.

---

## Acceptance Checklist

- [ ] `main` contains `source/`, `themes/butterfly/`, `_config.yml`, `_config.butterfly.yml`, `package.json`, `package-lock.json`, `.github/workflows/deploy.yml`, `docs/`.
- [ ] `github`, `github.pub`, `node_modules/`, `public/`, `db.json`, `.deploy_git/`, `.idea/`, `.obsidian/` are not tracked.
- [ ] `_config.yml` has no `deploy:` block.
- [ ] `package.json` has no `hexo-deployer-git`.
- [ ] GitHub Pages source is "GitHub Actions".
- [ ] The latest Actions run is green.
- [ ] `https://ihmod.github.io` returns 200 and shows the correct title.
- [ ] A known post (`/posts/312e3d12.html`) returns 200 and the correct title.
- [ ] `search.xml` returns 200 with 13 entries.
- [ ] Archives, tags, categories, and link pages return 200.
- [ ] The new admin deploy key can clone and `push --dry-run` to `main`.
- [ ] The old project-root deploy key is revoked.
- [ ] The design spec and this plan are committed under `docs/superpowers/`.

---

## Rollback Summary

- Immediate: switch Pages source back to branch `master` (legacy built site).
- Local: restore from `~/ihmod-backups/ihmod-source-*.tar.gz` and `~/ihmod-backups/ihmod-build-*.tar.gz`.
- Do not delete `master` until the new flow has been stable.
