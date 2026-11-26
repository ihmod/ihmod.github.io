# Blog Admin Publish & Deploy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Export published SQLite posts to Markdown + media in a local clone of `ihmod/ihmod.github.io`, commit and push to `main`, and provide the native systemd/backup/ufw deployment for the admin on the VPS.

**Architecture:** A synchronous exporter renders frontmatter + body into `source/_posts/`, copies referenced media into `source/img/uploads/`, and maintains `data/publish-manifest.json`. An async publisher ensures a local Git clone, refuses to overwrite uncommitted changes outside the managed paths, commits only managed paths, rebases safely, and pushes `main`; GitHub Actions then builds and deploys Pages. A publish-state row and page make the result visible. Deployment is native Node + systemd (no Docker).

**Tech Stack:** Existing Plan 2A–2D stack + `js-yaml`, `simple-git`.

---

## Prerequisites

Plans 2A–2D are complete in `/home/ihmod/cc_projects/IHMOD-admin`:

- Login, posts/categories/tags, importer, Vditor, media, autosave.
- 82 tests pass.
- `data/admin.db` has 13 imported posts.
- The blog repo `ihmod/ihmod.github.io` has `main` (source) and GitHub Actions deployment.
- The admin deploy key exists locally at `~/ihmod-secrets/ihmod_blog_admin` and is registered as a write deploy key on the blog repo.

Run:

```bash
cd /home/ihmod/cc_projects/IHMOD-admin
npm test
```

Expected: 20 files, 82 tests pass.

## File Structure

```
src/lib/publish-format.ts      # safe paths, frontmatter render, manifest
src/services/exporter.ts       # DB -> files + media + manifest
src/services/publisher.ts      # git clone/check/commit/rebase/push + publish state
src/routes/publish.ts          # /publish page, POST /publish, /api/publish/state
src/cli/publish.ts             # emergency CLI publish
views/publish.ejs
views/partials/header.ejs      # publish nav
views/partials/notice.ejs      # publish notices/errors
src/app.ts                     # register publish routes
src/server.ts                  # recover interrupted publish state
src/config.ts                  # repo/manifest/author settings
.env.example
deploy/ihmod-admin.service
deploy/backup.sh
deploy/ufw-setup.md
deploy/DEPLOY.md
test/publish-format.test.ts
test/exporter.test.ts
test/publish.test.ts
test/publish-routes.test.ts
README.md
```

No database migration is needed; `publish_state` exists from Plan 2A.

---

## Task 1: Dependencies, config, and test defaults

**Files:**
- Modify: `package.json` / `package-lock.json`
- Modify: `src/config.ts`
- Modify: `.env.example`
- Modify: `test/helpers.ts`

- [ ] **Step 1: Install dependencies**

```bash
npm install js-yaml simple-git
npm install -D @types/js-yaml
```

Expected: packages added; `@types/js-yaml` exists (unlike gray-matter).

- [ ] **Step 2: Add publish config to `src/config.ts`**

Add after `UPLOADS_DIR`:

```ts
  REPO_DIR: z.string().default('./data/repo'),
  REPO_URL: z.string().default('git@github.com:ihmod/ihmod.github.io.git'),
  REPO_BRANCH: z.string().default('main'),
  GIT_SSH_KEY_PATH: z.string().optional(),
  PUBLISH_MANIFEST_PATH: z.string().default('./data/publish-manifest.json'),
  GIT_AUTHOR_NAME: z.string().default('iHMOD Admin'),
  GIT_AUTHOR_EMAIL: z.string().default('admin@ihmod.local'),
```

- [ ] **Step 3: Add the same keys to `.env.example`**

```dotenv
REPO_DIR=./data/repo
REPO_URL=git@github.com:ihmod/ihmod.github.io.git
REPO_BRANCH=main
GIT_SSH_KEY_PATH=./secrets/deploy_key
PUBLISH_MANIFEST_PATH=./data/publish-manifest.json
GIT_AUTHOR_NAME=iHMOD Admin
GIT_AUTHOR_EMAIL=admin@ihmod.local
```

- [ ] **Step 4: Give tests self-contained repo paths**

In `test/helpers.ts`, add to the `loadConfig` object:

```ts
    REPO_DIR: path.join(dir, 'repo'),
    PUBLISH_MANIFEST_PATH: path.join(dir, 'publish-manifest.json'),
```

These values are harmless for non-publish tests and keep any accidental publish from touching the project directory.

- [ ] **Step 5: Add the publish scripts**

```bash
npm pkg set scripts.publish="tsx src/cli/publish.ts"
```

- [ ] **Step 6: Verify and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add package.json package-lock.json src/config.ts .env.example test/helpers.ts
git commit -m "feat: add publish configuration"
```

---

## Task 2: Publish format utilities (TDD)

**Files:**
- Create: `src/lib/publish-format.ts`
- Test: `test/publish-format.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from 'vitest';
import {
  isManagedPath,
  isSafeRelativePath,
  renderPostMarkdown,
} from '../src/lib/publish-format';

describe('publish-format', () => {
  it('rejects unsafe relative paths', () => {
    expect(isSafeRelativePath('网安/信息收集.md')).toBe(true);
    expect(isSafeRelativePath('source/_posts/x.md')).toBe(true);
    expect(isSafeRelativePath('../evil.md')).toBe(false);
    expect(isSafeRelativePath('/abs.md')).toBe(false);
    expect(isSafeRelativePath('a\\b.md')).toBe(false);
    expect(isSafeRelativePath('a/../b.md')).toBe(false);
    expect(isSafeRelativePath('')).toBe(false);
  });

  it('recognizes managed prefixes', () => {
    expect(isManagedPath('source/_posts/a.md')).toBe(true);
    expect(isManagedPath('source/img/uploads/x.png')).toBe(true);
    expect(isManagedPath('themes/butterfly/_config.yml')).toBe(false);
    expect(isManagedPath('_config.yml')).toBe(false);
  });

  it('renders Hexo frontmatter and body', () => {
    const markdown = renderPostMarkdown(
      {
        title: '信息收集',
        contentMd: '## 域名\n\n内容\n',
        postDate: '2026-11-12 10:00:00',
        updatedAt: '2026-11-12 11:00:00',
        topImg: 'false',
        description: 'desc',
        abbrlink: '312e3d12',
      },
      '网安',
      ['信息收集', '谷歌'],
    );
    expect(markdown).toContain('title: 信息收集');
    expect(markdown).toMatch(/date: ['"]?2026-11-12 10:00:00['"]?/);
    expect(markdown).toMatch(/updated: ['"]?2026-11-12 11:00:00['"]?/);
    expect(markdown).toContain('categories: 网安');
    expect(markdown).toContain('top_img: false');
    expect(markdown).toContain('abbrlink: 312e3d12');
    expect(markdown).toContain('  - 信息收集');
    expect(markdown).toContain('  - 谷歌');
    expect(markdown).toMatch(/---\n\n## 域名/);
  });

  it('omits empty tags and description and quotes all-digit abbrlinks', () => {
    const markdown = renderPostMarkdown(
      {
        title: 'T',
        contentMd: 'body',
        postDate: '2026-11-12 00:00:00',
        updatedAt: '2026-11-12 00:00:00',
        topImg: 'https://example.com/a.png',
        description: null,
        abbrlink: '12345678',
      },
      '未分类',
      [],
    );
    expect(markdown).not.toContain('tags:');
    expect(markdown).not.toContain('description:');
    expect(markdown).toMatch(/top_img: ['"]?https:\/\/example\.com\/a\.png['"]?/);
    expect(markdown).toContain("abbrlink: '12345678'");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/publish-format.test.ts
```

Expected: FAIL because `src/lib/publish-format.ts` does not exist.

- [ ] **Step 3: Implement `src/lib/publish-format.ts`**

```ts
import yaml from 'js-yaml';

export interface RenderPost {
  title: string;
  contentMd: string;
  postDate: string;
  updatedAt: string;
  topImg: string;
  description: string | null;
  abbrlink: string;
}

export function isSafeRelativePath(value: string): boolean {
  if (!value || value.startsWith('/') || value.includes('\\') || value.includes('\0')) {
    return false;
  }
  return value
    .split('/')
    .every((segment) => segment && segment !== '.' && segment !== '..');
}

export function isManagedPath(value: string): boolean {
  return value.startsWith('source/_posts/') || value.startsWith('source/img/uploads/');
}

export function renderPostMarkdown(post: RenderPost, categoryName: string, tags: string[]): string {
  const data: Record<string, unknown> = {
    title: post.title,
    date: post.postDate,
    updated: post.updatedAt,
  };
  if (tags.length > 0) data.tags = tags;
  data.categories = categoryName;
  data.top_img = post.topImg === 'false' ? false : post.topImg;
  if (post.description) data.description = post.description;
  data.abbrlink = post.abbrlink;

  const front = yaml.dump(data, { lineWidth: -1, noRefs: true, sortKeys: false });
  const body = post.contentMd.replace(/\r\n/g, '\n').trimEnd();
  return `---\n${front}---\n\n${body}\n`;
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/publish-format.test.ts
```

Expected: 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/lib/publish-format.ts test/publish-format.test.ts
git commit -m "feat: add publish format utilities"
```

If `js-yaml` quotes the `date`/`updated` values, the tests will reveal the exact output; adjust the assertions to the safe quoted form only if Hexo still parses it (quoted dates are valid YAML strings). Do not remove the fields.

---

## Task 3: Exporter service (TDD)

**Files:**
- Create: `src/services/exporter.ts`
- Test: `test/exporter.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { posts } from '../src/db/schema';
import { createCategory } from '../src/lib/categories';
import { createPost, updatePost, unpublishPost } from '../src/lib/posts';
import { saveUpload } from '../src/services/media';
import { exportContent } from '../src/services/exporter';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-export-'));
  const repoDir = path.join(dir, 'repo');
  const uploadsDir = path.join(dir, 'uploads');
  const manifestPath = path.join(dir, 'publish-manifest.json');
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  return { db, repoDir, uploadsDir, manifestPath };
}

const base = {
  title: '信息收集',
  categoryId: null,
  description: 'desc',
  topImg: 'false',
  contentMd: '## 域名\n\n内容\n',
  status: 'draft' as const,
  postDate: '2026-11-12 10:00:00',
  tags: ['信息收集', '谷歌'],
};

describe('exportContent', () => {
  it('writes published posts and a manifest, and omits drafts', () => {
    const { db, repoDir, uploadsDir, manifestPath } = setup();
    const category = createCategory(db, '网安');
    const published = createPost(db, { ...base, categoryId: category.id, status: 'published' });
    createPost(db, { ...base, title: '草稿', status: 'draft' });

    const report = exportContent(db, { repoDir, uploadsDir, manifestPath });
    expect(report.posts).toBe(1);
    expect(report.files).toBe(1);
    expect(report.changed).toBe(1);

    const file = path.join(repoDir, 'source/_posts/网安/信息收集.md');
    expect(fs.existsSync(file)).toBe(true);
    const content = fs.readFileSync(file, 'utf8');
    expect(content).toContain('categories: 网安');
    expect(content).toContain('abbrlink:');
    expect(content).toContain('  - 谷歌');
    expect(content).not.toContain('草稿');

    const manifest = JSON.parse(fs.readFileSync(manifestPath, 'utf8'));
    expect(manifest.version).toBe(1);
    expect(Object.keys(manifest.files)).toContain('source/_posts/网安/信息收集.md');

    const second = exportContent(db, { repoDir, uploadsDir, manifestPath });
    expect(second.changed).toBe(0);

    updatePost(db, published, { ...base, categoryId: category.id, status: 'published', contentMd: 'changed' });
    const third = exportContent(db, { repoDir, uploadsDir, manifestPath });
    expect(third.changed).toBe(1);

    unpublishPost(db, published);
    const fourth = exportContent(db, { repoDir, uploadsDir, manifestPath });
    expect(fourth.deleted).toBe(1);
    expect(fs.existsSync(file)).toBe(false);
  });

  it('copies referenced media and rejects missing media', () => {
    const { db, repoDir, uploadsDir, manifestPath } = setup();
    const png = Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0, 0, 0, 0]);
    const saved = saveUpload(db, uploadsDir, { filename: 'x.png', buffer: png });
    createPost(db, {
      ...base,
      contentMd: `![x](${saved.url})`,
      status: 'published',
    });
    exportContent(db, { repoDir, uploadsDir, manifestPath });
    expect(fs.existsSync(path.join(repoDir, 'source/img/uploads', saved.storedName))).toBe(true);

    createPost(db, {
      ...base,
      title: 'missing',
      contentMd: '![x](/img/uploads/missing.png)',
      status: 'published',
    });
    expect(() => exportContent(db, { repoDir, uploadsDir, manifestPath })).toThrow(/missing media/);
  });

  it('does not delete files outside the manifest or managed prefixes', () => {
    const { db, repoDir, uploadsDir, manifestPath } = setup();
    fs.mkdirSync(path.join(repoDir, 'themes'), { recursive: true });
    fs.writeFileSync(path.join(repoDir, '_config.yml'), 'title: test\n');
    fs.writeFileSync(path.join(repoDir, 'themes', 'keep.txt'), 'keep');
    exportContent(db, { repoDir, uploadsDir, manifestPath });
    expect(fs.existsSync(path.join(repoDir, '_config.yml'))).toBe(true);
    expect(fs.existsSync(path.join(repoDir, 'themes', 'keep.txt'))).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/exporter.test.ts
```

Expected: FAIL because `src/services/exporter.ts` does not exist.

- [ ] **Step 3: Implement `src/services/exporter.ts`**

```ts
import crypto from 'node:crypto';
import fs from 'node:fs';
import path from 'node:path';
import { and, eq, isNull } from 'drizzle-orm';
import type { Db } from '../db/client';
import { categories, media, postTags, posts, tags } from '../db/schema';
import { isManagedPath, isSafeRelativePath, renderPostMarkdown } from '../lib/publish-format';

export class ExportError extends Error {}

export interface ExportOptions {
  repoDir: string;
  uploadsDir: string;
  manifestPath: string;
}

export interface ExportReport {
  posts: number;
  files: number;
  changed: number;
  deleted: number;
}

export interface PublishManifest {
  version: 1;
  files: Record<string, string>;
}

function sha256(data: Buffer | string): string {
  return crypto.createHash('sha256').update(data).digest('hex');
}

function loadManifest(manifestPath: string): PublishManifest {
  if (!fs.existsSync(manifestPath)) return { version: 1, files: {} };
  const parsed = JSON.parse(fs.readFileSync(manifestPath, 'utf8')) as PublishManifest;
  if (parsed.version !== 1 || typeof parsed.files !== 'object' || parsed.files === null) {
    throw new ExportError('invalid publish manifest');
  }
  return parsed;
}

function saveManifest(manifestPath: string, manifest: PublishManifest): void {
  fs.mkdirSync(path.dirname(manifestPath), { recursive: true });
  const temp = `${manifestPath}.tmp`;
  fs.writeFileSync(temp, `${JSON.stringify(manifest, null, 2)}\n`);
  fs.renameSync(temp, manifestPath);
}

function writeAtomic(target: string, data: Buffer): void {
  fs.mkdirSync(path.dirname(target), { recursive: true });
  const temp = `${target}.tmp`;
  fs.writeFileSync(temp, data);
  fs.renameSync(temp, target);
}

interface DesiredFile {
  hash: string;
  data: Buffer;
}

export function exportContent(db: Db, options: ExportOptions): ExportReport {
  const postRows = db
    .select({
      id: posts.id,
      title: posts.title,
      contentMd: posts.contentMd,
      topImg: posts.topImg,
      description: posts.description,
      abbrlink: posts.abbrlink,
      sourcePath: posts.sourcePath,
      postDate: posts.postDate,
      updatedAt: posts.updatedAt,
      categoryName: categories.name,
    })
    .from(posts)
    .leftJoin(categories, eq(posts.categoryId, categories.id))
    .where(and(eq(posts.status, 'published'), isNull(posts.deletedAt)))
    .all();

  const tagRows = db
    .select({ postId: postTags.postId, name: tags.name })
    .from(postTags)
    .innerJoin(tags, eq(postTags.tagId, tags.id))
    .orderBy(tags.name)
    .all();
  const tagMap = new Map<string, string[]>();
  for (const row of tagRows) {
    const list = tagMap.get(row.postId) ?? [];
    list.push(row.name);
    tagMap.set(row.postId, list);
  }

  const desired = new Map<string, DesiredFile>();
  const mediaNames = new Set<string>();

  for (const post of postRows) {
    if (!isSafeRelativePath(post.sourcePath)) {
      throw new ExportError(`unsafe source path: ${post.sourcePath}`);
    }
    const markdown = renderPostMarkdown(
      post,
      post.categoryName ?? '未分类',
      tagMap.get(post.id) ?? [],
    );
    const data = Buffer.from(markdown, 'utf8');
    desired.set(`source/_posts/${post.sourcePath}`, { hash: sha256(data), data });

    for (const match of post.contentMd.matchAll(/\/img\/uploads\/([A-Za-z0-9._-]+)/g)) {
      const name = match[1];
      if (!isSafeRelativePath(name)) throw new ExportError(`unsafe media name: ${name}`);
      const row = db.select().from(media).where(eq(media.storedName, name)).get();
      if (!row) throw new ExportError(`missing media record: ${name}`);
      const file = path.join(options.uploadsDir, name);
      if (!fs.existsSync(file)) throw new ExportError(`missing media file: ${name}`);
      mediaNames.add(name);
    }
  }

  for (const name of mediaNames) {
    const data = fs.readFileSync(path.join(options.uploadsDir, name));
    desired.set(`source/img/uploads/${name}`, { hash: sha256(data), data });
  }

  const previous = loadManifest(options.manifestPath);
  let changed = 0;
  let deleted = 0;

  for (const [relative, file] of desired) {
    const target = path.join(options.repoDir, relative);
    if (!fs.existsSync(target) || sha256(fs.readFileSync(target)) !== file.hash) {
      writeAtomic(target, file.data);
      changed += 1;
    }
  }

  for (const relative of Object.keys(previous.files)) {
    if (desired.has(relative) || !isManagedPath(relative)) continue;
    const target = path.join(options.repoDir, relative);
    if (fs.existsSync(target)) {
      fs.rmSync(target, { force: true });
      deleted += 1;
    }
  }

  const files: Record<string, string> = {};
  for (const [relative, file] of desired) files[relative] = file.hash;
  saveManifest(options.manifestPath, { version: 1, files });

  return { posts: postRows.length, files: desired.size, changed, deleted };
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/exporter.test.ts
```

Expected: 3 tests pass.

- [ ] **Step 5: Full suite, typecheck, commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/services/exporter.ts test/exporter.test.ts
git commit -m "feat: add markdown exporter"
```

---

## Task 4: Publisher service and publish state (TDD)

**Files:**
- Create: `src/services/publisher.ts`
- Test: `test/publish.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { execFileSync } from 'node:child_process';
import { eq } from 'drizzle-orm';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { publishState } from '../src/db/schema';
import { createPost, unpublishPost, updatePost } from '../src/lib/posts';
import { getPublishState, publish, recoverPublishState } from '../src/services/publisher';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function git(cwd: string, ...args: string[]): string {
  return execFileSync('git', args, { cwd, encoding: 'utf8' });
}

function setupRepo() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-publish-'));
  const bare = path.join(dir, 'origin.git');
  const seed = path.join(dir, 'seed');
  git(dir, 'init', '--bare', bare);
  git(dir, 'clone', bare, seed);
  git(seed, 'config', 'user.name', 'Seed');
  git(seed, 'config', 'user.email', 'seed@example.com');
  fs.mkdirSync(path.join(seed, 'source', '_posts'), { recursive: true });
  fs.writeFileSync(path.join(seed, '_config.yml'), 'title: seed\n');
  fs.writeFileSync(path.join(seed, 'source', '_posts', 'keep.md'), '---\ntitle: keep\n---\n');
  git(seed, 'add', '-A');
  git(seed, 'commit', '-m', 'seed');
  git(seed, 'branch', '-M', 'main');
  git(seed, 'push', 'origin', 'main');

  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  const repoDir = path.join(dir, 'repo');
  const uploadsDir = path.join(dir, 'uploads');
  const manifestPath = path.join(dir, 'publish-manifest.json');
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  return {
    db, sqlite, dir, bare, seed, repoDir, uploadsDir, manifestPath,
  };
}

const options = (s: ReturnType<typeof setupRepo>) => ({
  repoDir: s.repoDir,
  repoUrl: s.bare,
  branch: 'main',
  uploadsDir: s.uploadsDir,
  manifestPath: s.manifestPath,
  authorName: 'Admin',
  authorEmail: 'admin@example.com',
});

const base = {
  title: '信息收集',
  categoryId: null,
  description: 'desc',
  topImg: 'false',
  contentMd: 'body',
  status: 'draft' as const,
  postDate: '2026-11-12 10:00:00',
  tags: [],
};

describe('publish', () => {
  it('clones, commits, pushes, and records success', async () => {
    const s = setupRepo();
    const id = createPost(s.db, { ...base, status: 'published' });
    const report = await publish(s.db, options(s));
    expect(report.posts).toBe(1);
    expect(report.pushed).toBe(true);
    expect(report.commit).toMatch(/^[0-9a-f]{40}$/);

    const state = getPublishState(s.db);
    expect(state.status).toBe('success');
    expect(state.commitSha).toBe(report.commit);

    const log = git(s.bare, 'log', '--oneline', 'main');
    expect(log).toContain('chore(publish)');

    const second = await publish(s.db, options(s));
    expect(second.pushed).toBe(false);
    const count = Number(git(s.bare, 'rev-list', '--count', 'main').trim());
    expect(count).toBe(2); // seed + one publish

    updatePost(s.db, id, { ...base, status: 'published', contentMd: 'changed' });
    const third = await publish(s.db, options(s));
    expect(third.pushed).toBe(true);

    unpublishPost(s.db, id);
    const fourth = await publish(s.db, options(s));
    expect(fourth.pushed).toBe(true);
  });

  it('aborts on uncommitted changes outside managed paths', async () => {
    const s = setupRepo();
    createPost(s.db, { ...base, status: 'published' });
    await publish(s.db, options(s));

    fs.writeFileSync(path.join(s.repoDir, '_config.yml'), 'title: dirty\n');
    await expect(publish(s.db, options(s))).rejects.toThrow(/uncommitted changes/);
    expect(getPublishState(s.db).status).toBe('error');
  });

  it('recovers an interrupted running state', () => {
    const s = setupRepo();
    s.db.update(publishState).set({ status: 'running', startedAt: '2026-11-12 00:00:00' }).where(eq(publishState.id, 1)).run();
    recoverPublishState(s.db);
    expect(getPublishState(s.db).status).toBe('error');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/publish.test.ts
```

Expected: FAIL because `src/services/publisher.ts` does not exist.

- [ ] **Step 3: Implement `src/services/publisher.ts`**

```ts
import fs from 'node:fs';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import simpleGit, { type SimpleGit } from 'simple-git';
import type { Db } from '../db/client';
import { publishState } from '../db/schema';
import { nowStamp } from '../lib/dates';
import { isManagedPath } from '../lib/publish-format';
import { exportContent } from './exporter';

export class PublishError extends Error {}
export class PublishBusyError extends Error {}

export interface PublishOptions {
  repoDir: string;
  repoUrl: string;
  branch: string;
  keyPath?: string;
  authorName: string;
  authorEmail: string;
  uploadsDir: string;
  manifestPath: string;
}

export interface PublishReport {
  posts: number;
  files: number;
  commit: string;
  pushed: boolean;
}

export interface PublishStateRow {
  id: number;
  status: 'idle' | 'running' | 'success' | 'error';
  message: string | null;
  commitSha: string | null;
  startedAt: string | null;
  finishedAt: string | null;
}

let inFlight: Promise<PublishReport> | null = null;

function sshCommand(keyPath: string): string {
  return `ssh -i ${keyPath} -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new`;
}

function setState(
  db: Db,
  status: PublishStateRow['status'],
  message: string | null = null,
  commitSha: string | null = null,
): void {
  const now = nowStamp();
  db.update(publishState)
    .set({
      status,
      message,
      commitSha,
      startedAt: status === 'running' ? now : undefined,
      finishedAt: status === 'running' ? null : now,
    })
    .where(eq(publishState.id, 1))
    .run();
}

export function getPublishState(db: Db): PublishStateRow {
  const row = db.select().from(publishState).where(eq(publishState.id, 1)).get();
  if (!row) throw new PublishError('publish_state row missing');
  return row as PublishStateRow;
}

export function recoverPublishState(db: Db): void {
  const state = getPublishState(db);
  if (state.status === 'running') {
    setState(db, 'error', 'publish was interrupted; check the repo and retry');
  }
}

async function openRepo(options: PublishOptions): Promise<SimpleGit> {
  if (!fs.existsSync(path.join(options.repoDir, '.git'))) {
    fs.mkdirSync(path.dirname(options.repoDir), { recursive: true });
    const clone = simpleGit();
    if (options.keyPath) clone.env('GIT_SSH_COMMAND', sshCommand(options.keyPath));
    await clone.clone(options.repoUrl, options.repoDir, ['--branch', options.branch]);
  }

  const git = simpleGit(options.repoDir);
  if (options.keyPath) git.env('GIT_SSH_COMMAND', sshCommand(options.keyPath));
  await git.addConfig('user.name', options.authorName);
  await git.addConfig('user.email', options.authorEmail);

  const current = (await git.revparse(['--abbrev-ref', 'HEAD'])).trim();
  if (current !== options.branch) await git.checkout(options.branch);
  await git.fetch('origin', options.branch);
  return git;
}

async function assertCleanOutsideManaged(git: SimpleGit): Promise<void> {
  const status = await git.status();
  const dirty = status.files
    .map((file) => file.path)
    .filter((filePath) => !isManagedPath(filePath));
  if (dirty.length > 0) {
    throw new PublishError(`uncommitted changes outside managed paths: ${dirty.join(', ')}`);
  }
}

async function runPublish(db: Db, options: PublishOptions): Promise<PublishReport> {
  setState(db, 'running');
  try {
    const git = await openRepo(options);
    await assertCleanOutsideManaged(git);

    const exported = exportContent(db, {
      repoDir: options.repoDir,
      uploadsDir: options.uploadsDir,
      manifestPath: options.manifestPath,
    });

    fs.mkdirSync(path.join(options.repoDir, 'source/_posts'), { recursive: true });
    fs.mkdirSync(path.join(options.repoDir, 'source/img/uploads'), { recursive: true });
    await git.add(['-A', '--', 'source/_posts', 'source/img/uploads']);
    const status = await git.status();
    if (status.staged.length > 0) {
      await git.commit(`chore(publish): sync ${exported.posts} posts`);
    }

    await git.fetch('origin', options.branch);
    const behind = Number(
      (await git.raw(['rev-list', '--count', `HEAD..origin/${options.branch}`])).trim(),
    );
    if (behind > 0) {
      try {
        await git.rebase([`origin/${options.branch}`]);
      } catch (error) {
        try {
          await git.rebase(['--abort']);
        } catch {
          // ignore abort failure; the original error is more useful
        }
        throw new PublishError(
          `rebase conflict: ${error instanceof Error ? error.message : String(error)}`,
        );
      }
    }

    const ahead = Number(
      (await git.raw(['rev-list', '--count', `origin/${options.branch}..HEAD`])).trim(),
    );
    if (ahead > 0) await git.push('origin', options.branch);

    const commit = (await git.revparse(['HEAD'])).trim();
    setState(db, 'success', null, commit);
    return {
      posts: exported.posts,
      files: exported.files,
      commit,
      pushed: ahead > 0,
    };
  } catch (error) {
    setState(db, 'error', error instanceof Error ? error.message : String(error));
    throw error;
  }
}

export async function publish(db: Db, options: PublishOptions): Promise<PublishReport> {
  if (inFlight) throw new PublishBusyError('publish already running');
  inFlight = runPublish(db, options).finally(() => {
    inFlight = null;
  });
  return inFlight;
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/publish.test.ts
```

Expected: 3 tests pass.

If `simple-git`'s `env` signature differs, use `git.env('GIT_SSH_COMMAND', sshCommand(...))` (as written) or `git.customBinary`/`execFileSync` for clone; keep behavior.

- [ ] **Step 5: Full suite, typecheck, commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/services/publisher.ts test/publish.test.ts
git commit -m "feat: add git publisher"
```

---

## Task 5: Publish routes/UI (TDD)

**Files:**
- Create: `src/routes/publish.ts`
- Create: `views/publish.ejs`
- Modify: `src/app.ts`
- Modify: `src/server.ts`
- Modify: `views/partials/header.ejs`
- Modify: `views/partials/notice.ejs`
- Test: `test/publish-routes.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { execFileSync } from 'node:child_process';
import { createPost } from '../src/lib/posts';
import { makeTestApp, login, type TestApp } from './helpers';

let current: TestApp | null = null;
let repoRoot: string | null = null;
afterEach(async () => {
  if (current) await current.cleanup();
  if (repoRoot) fs.rmSync(repoRoot, { recursive: true, force: true });
  current = null;
  repoRoot = null;
});

function git(cwd: string, ...args: string[]): string {
  return execFileSync('git', args, { cwd, encoding: 'utf8' });
}

function seedBareRepo(): { bare: string; root: string } {
  const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-pubroutes-'));
  const bare = path.join(root, 'origin.git');
  const seed = path.join(root, 'seed');
  git(root, 'init', '--bare', bare);
  git(root, 'clone', bare, seed);
  git(seed, 'config', 'user.name', 'Seed');
  git(seed, 'config', 'user.email', 'seed@example.com');
  fs.writeFileSync(path.join(seed, '_config.yml'), 'title: seed\n');
  git(seed, 'add', '-A');
  git(seed, 'commit', '-m', 'seed');
  git(seed, 'branch', '-M', 'main');
  git(seed, 'push', 'origin', 'main');
  return { bare, root };
}

async function setup() {
  const seeded = seedBareRepo();
  repoRoot = seeded.root;
  current = await makeTestApp({
    REPO_DIR: path.join(seeded.root, 'repo'),
    REPO_URL: seeded.bare,
    REPO_BRANCH: 'main',
    PUBLISH_MANIFEST_PATH: path.join(seeded.root, 'manifest.json'),
  });
  const { cookie, csrf } = await login(current.app);
  return { ...current, cookie, csrf, bare: seeded.bare };
}

describe('publish routes', () => {
  it('requires auth', async () => {
    current = await makeTestApp();
    const res = await current.app.inject({ method: 'GET', url: '/publish' });
    expect(res.statusCode).toBe(302);
  });

  it('renders the publish page and triggers a publish', async () => {
    const { app, db, cookie, csrf, bare } = await setup();
    createPost(db, {
      title: 'P',
      categoryId: null,
      contentMd: 'body',
      status: 'published',
      postDate: '2026-11-12 10:00:00',
      tags: [],
    });

    const page = await app.inject({ method: 'GET', url: '/publish', headers: { cookie } });
    expect(page.statusCode).toBe(200);
    expect(page.body).toContain('发布');

    const post = await app.inject({
      method: 'POST',
      url: '/publish',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(post.statusCode).toBe(302);
    expect(post.headers.location).toContain('notice=pushed');

    const state = await app.inject({
      method: 'GET',
      url: '/api/publish/state',
      headers: { cookie },
    });
    expect(state.statusCode).toBe(200);
    expect(state.json()).toMatchObject({ status: 'success' });
    expect(git(bare, 'log', '--oneline', 'main')).toContain('chore(publish)');
  });

  it('rejects publish without CSRF', async () => {
    const { app, cookie } = await setup();
    const res = await app.inject({
      method: 'POST',
      url: '/publish',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: '',
    });
    expect(res.statusCode).toBe(403);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/publish-routes.test.ts
```

Expected: FAIL because the route/view do not exist.

- [ ] **Step 3: Implement `src/routes/publish.ts`**

```ts
import type { FastifyInstance } from 'fastify';
import type { Config } from '../config';
import type { Db } from '../db/client';
import {
  PublishBusyError,
  getPublishState,
  publish,
  type PublishOptions,
} from '../services/publisher';

export interface PublishRouteOptions {
  db: Db;
  config: Config;
}

export function publishOptionsFromConfig(config: Config): PublishOptions {
  return {
    repoDir: config.REPO_DIR,
    repoUrl: config.REPO_URL,
    branch: config.REPO_BRANCH,
    keyPath: config.GIT_SSH_KEY_PATH,
    authorName: config.GIT_AUTHOR_NAME,
    authorEmail: config.GIT_AUTHOR_EMAIL,
    uploadsDir: config.UPLOADS_DIR,
    manifestPath: config.PUBLISH_MANIFEST_PATH,
  };
}

export async function registerPublishRoutes(
  app: FastifyInstance,
  { db, config }: PublishRouteOptions,
): Promise<void> {
  app.get('/publish', async (request, reply) => {
    const query = request.query as Record<string, string | undefined>;
    return reply.view('publish.ejs', {
      title: '发布',
      user: request.user,
      csrfToken: request.csrfToken,
      state: getPublishState(db),
      notice: query.notice,
      error: query.error,
    });
  });

  app.post('/publish', async (request, reply) => {
    try {
      await publish(db, publishOptionsFromConfig(config));
    } catch (error) {
      if (error instanceof PublishBusyError) return reply.redirect('/publish?error=busy');
      return reply.redirect('/publish?error=failed');
    }
    return reply.redirect('/publish?notice=pushed');
  });

  app.get('/api/publish/state', async (request, reply) => {
    return reply.send(getPublishState(db));
  });
}
```

- [ ] **Step 4: Create `views/publish.ejs`**

```ejs
<%- include('partials/header', { title: '发布', user, csrfToken }) %>
<%- include('partials/notice', { notice, error }) %>
<h1>发布</h1>
<section class="card">
  <p>状态：
    <% if (state.status === 'running') { %>
      <span class="badge draft">发布中</span>
    <% } else if (state.status === 'success') { %>
      <span class="badge">成功</span>
    <% } else if (state.status === 'error') { %>
      <span class="badge trash">失败</span>
    <% } else { %>
      <span class="badge">空闲</span>
    <% } %>
  </p>
  <% if (state.message) { %><p class="error"><%= state.message %></p><% } %>
  <% if (state.commitSha) { %>
    <p class="muted">最近提交：<code><%= state.commitSha.slice(0, 12) %></code></p>
  <% } %>
  <% if (state.finishedAt) { %>
    <p class="muted">完成时间：<%= state.finishedAt %></p>
  <% } %>
  <form method="post" action="/publish">
    <input type="hidden" name="_csrf" value="<%= csrfToken %>">
    <button type="submit" <%= state.status === 'running' ? 'disabled' : '' %>>发布到 GitHub</button>
  </form>
  <p class="muted">
    发布后由 GitHub Actions 构建并部署：
    <a href="https://github.com/ihmod/ihmod.github.io/actions" target="_blank" rel="noreferrer">
      https://github.com/ihmod/ihmod.github.io/actions
    </a>
  </p>
</section>
<%- include('partials/footer') %>
```

- [ ] **Step 5: Add notices and nav**

In `views/partials/notice.ejs`, add to the notice map:

```js
    pushed: '已发布到 GitHub',
```

and to the error map:

```js
    busy: '发布正在进行中，请稍后重试',
    failed: '发布失败，请查看发布状态或服务器日志'
```

In `views/partials/header.ejs`, add before the username:

```ejs
    <a href="/publish">发布</a>
```

- [ ] **Step 6: Register routes and recover state**

In `src/app.ts`, import:

```ts
import { registerPublishRoutes } from './routes/publish';
```

Register after the media routes:

```ts
  await registerPublishRoutes(app, { db, config });
```

In `src/server.ts`, import:

```ts
import { recoverPublishState } from './services/publisher';
```

After migrations and seeding:

```ts
  recoverPublishState(db);
```

- [ ] **Step 7: Run the test to verify it passes**

```bash
npm test -- test/publish-routes.test.ts
```

Expected: 3 tests pass.

- [ ] **Step 8: Full suite, typecheck, commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/routes/publish.ts views/publish.ejs src/app.ts src/server.ts views/partials/header.ejs views/partials/notice.ejs test/publish-routes.test.ts
git commit -m "feat: add publish page and endpoint"
```

---

## Task 6: Publish CLI

**Files:**
- Create: `src/cli/publish.ts`

- [ ] **Step 1: Implement the CLI**

```ts
import 'dotenv/config';
import path from 'node:path';
import { loadConfig } from '../config';
import { createDb } from '../db/client';
import { applyMigrations } from '../db/migrate';
import { publishOptionsFromConfig } from '../routes/publish';
import { publish } from '../services/publisher';

async function main(): Promise<void> {
  const config = loadConfig();
  const { db, sqlite } = createDb(config.DATABASE_PATH);
  applyMigrations(sqlite, path.resolve(__dirname, '../../migrations'));
  try {
    const report = await publish(db, publishOptionsFromConfig(config));
    console.log(JSON.stringify(report, null, 2));
  } finally {
    sqlite.close();
  }
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

- [ ] **Step 2: Typecheck the CLI and verify the script**

```bash
npx tsc -p tsconfig.json --noEmit
node -e "const p=require('./package.json'); console.log(p.scripts.publish)"
```

Expected: tsc exit 0; `tsx src/cli/publish.ts`.

The full publish flow is covered by `test/publish.test.ts`; the CLI is a thin wrapper and will be exercised on the VPS during first deploy.

- [ ] **Step 3: Commit**

```bash
git add src/cli/publish.ts
git commit -m "feat: add publish CLI"
```

---

## Task 7: Native deployment files

**Files:**
- Create: `deploy/ihmod-admin.service`
- Create: `deploy/backup.sh`
- Create: `deploy/ufw-setup.md`
- Create: `deploy/DEPLOY.md`

- [ ] **Step 1: Create `deploy/ihmod-admin.service`**

```ini
[Unit]
Description=iHMOD Blog Admin
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ihmod
Group=ihmod
WorkingDirectory=/opt/ihmod-admin
EnvironmentFile=/etc/ihmod-admin.env
ExecStart=/usr/bin/node dist/server.js
Restart=always
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/ihmod-admin/data /var/backups/ihmod-admin
ReadOnlyPaths=/opt/ihmod-admin/secrets

[Install]
WantedBy=multi-user.target
```

- [ ] **Step 2: Create `deploy/backup.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE=/opt/ihmod-admin
OUT=/var/backups/ihmod-admin
STAMP=$(date +%Y%m%d-%H%M%S)

mkdir -p "$OUT"
sqlite3 "$BASE/data/admin.db" ".backup '$OUT/admin-$STAMP.db'"
tar -czf "$OUT/uploads-$STAMP.tar.gz" -C "$BASE/data" uploads
find "$OUT" -type f -mtime +14 -delete
```

Make it executable:

```bash
chmod +x deploy/backup.sh
```

- [ ] **Step 3: Create `deploy/ufw-setup.md`**

````markdown
# UFW setup

Replace `<YOUR_IP>` with the public IP you will use to open the admin.

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow from <YOUR_IP> to any port 8080 proto tcp
ufw enable
ufw status verbose
```

- SSH (22) must stay reachable so you can recover if your public IP changes.
- If the IP changes, update the rule over SSH:
  `ufw delete allow from <OLD_IP> to any port 8080 proto tcp`
  `ufw allow from <NEW_IP> to any port 8080 proto tcp`
- The admin uses HTTP; only use it from a trusted network. See the design doc for
  the documented risk and the HTTPS/Tailscale upgrade paths.
````

- [ ] **Step 4: Create `deploy/DEPLOY.md`**

````markdown
# Deploy iHMOD Blog Admin (native, no Docker)

Target: Debian/Ubuntu VPS with Node.js 22, git, sqlite3, and systemd.

## 1. System user and directories

```bash
sudo useradd --system --create-home --shell /usr/sbin/nologin ihmod
sudo mkdir -p /opt/ihmod-admin /var/backups/ihmod-admin
sudo chown -R ihmod:ihmod /opt/ihmod-admin /var/backups/ihmod-admin
```

## 2. Code and build

```bash
sudo -u ihmod -H bash
cd /opt/ihmod-admin
# copy or git clone the private ihmod/blog-admin repo here
npm ci
npm run build
```

## 3. Secrets

```bash
mkdir -p /opt/ihmod-admin/secrets
chmod 700 /opt/ihmod-admin/secrets
# copy the deploy key generated in Plan 1 to the VPS:
#   ~/ihmod-secrets/ihmod_blog_admin -> /opt/ihmod-admin/secrets/deploy_key
chmod 600 /opt/ihmod-admin/secrets/deploy_key
```

Create `/etc/ihmod-admin.env` (mode 600, owned by root):

```dotenv
NODE_ENV=production
HOST=0.0.0.0
PORT=8080
DATABASE_PATH=/opt/ihmod-admin/data/admin.db
UPLOADS_DIR=/opt/ihmod-admin/data/uploads
SESSION_SECRET=<openssl rand -hex 32>
ADMIN_USERNAME=admin
ADMIN_PASSWORD=<first-run strong password>
SESSION_TTL_HOURS=168
LOGIN_RATE_LIMIT_MAX=5
COOKIE_SECURE=false
REPO_DIR=/opt/ihmod-admin/data/repo
REPO_URL=git@github.com:ihmod/ihmod.github.io.git
REPO_BRANCH=main
GIT_SSH_KEY_PATH=/opt/ihmod-admin/secrets/deploy_key
PUBLISH_MANIFEST_PATH=/opt/ihmod-admin/data/publish-manifest.json
GIT_AUTHOR_NAME=iHMOD Admin
GIT_AUTHOR_EMAIL=admin@ihmod.local
TZ=Asia/Shanghai
```

## 4. Import existing posts

Clone the blog repo (the publisher also clones if missing) and import:

```bash
GIT_SSH_COMMAND="ssh -i /opt/ihmod-admin/secrets/deploy_key -o IdentitiesOnly=yes" \
  git clone -b main git@github.com:ihmod/ihmod.github.io.git /opt/ihmod-admin/data/repo
DATABASE_PATH=/opt/ihmod-admin/data/admin.db \
  npm run import -- /opt/ihmod-admin/data/repo/source/_posts
```

## 5. systemd

```bash
sudo cp deploy/ihmod-admin.service /etc/systemd/system/ihmod-admin.service
sudo systemctl daemon-reload
sudo systemctl enable --now ihmod-admin
sudo systemctl status ihmod-admin
curl -s http://127.0.0.1:8080/healthz
```

## 6. Firewall

Follow `deploy/ufw-setup.md`.

## 7. Backups

```bash
sudo crontab -e
# add: 0 3 * * * /opt/ihmod-admin/deploy/backup.sh
```

## 8. First publish

Log in at `http://<VPS_IP>:8080`, open 发布, click 发布到 GitHub.
Then check https://github.com/ihmod/ihmod.github.io/actions and the live site.

## Rollback

If a publish breaks the site, switch GitHub Pages back to
"Deploy from a branch: master /docs" (see the blog repo `docs/deploy.md`).
The admin's SQLite data is unaffected.
````

- [ ] **Step 5: Validate and commit**

```bash
bash -n deploy/backup.sh
systemd-analyze verify deploy/ihmod-admin.service 2>/dev/null || true
git add deploy
git commit -m "docs: add native deployment files"
```

Expected: `bash -n` exits 0. `systemd-analyze` may warn about the missing binary path on a non-systemd dev machine; that is acceptable.

---

## Task 8: Final verification and README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Run the full suite and build**

```bash
npm test
npm run build
```

Expected: all tests pass; build exits 0.

- [ ] **Step 2: Run the local end-to-end publish test**

```bash
npm test -- test/publish.test.ts test/publish-routes.test.ts
```

Expected: all publish tests pass (local bare repo, clone, commit, push, state, dirty abort).

- [ ] **Step 3: Update `README.md`**

Add a "Publish and deploy" section:

```markdown
## Publish and deploy

- `/publish` shows the publish state and pushes the current database to
  `ihmod/ihmod.github.io` `main`.
- The exporter writes `source/_posts/**` and referenced `source/img/uploads/**`,
  maintains `data/publish-manifest.json`, and never touches the theme/config.
- The publisher refuses to run with uncommitted changes outside the managed
  paths, commits only managed paths, rebases, and never force-pushes.
- GitHub Actions builds and deploys Pages from `main`.
- Deployment is native systemd (no Docker): see `deploy/DEPLOY.md`.
- Emergency CLI publish: `npm run publish`.
```

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document publish and deploy"
```

---

## Acceptance Checklist

- [ ] `npm test` passes all Plan 2A–2E tests.
- [ ] `npm run build` exits 0.
- [ ] The exporter writes published posts only, preserves abbrlink, renders `top_img: false`, and omits drafts.
- [ ] The exporter copies referenced media and fails on missing media records/files.
- [ ] The manifest prevents deleting non-managed files; theme/config survive.
- [ ] The publisher clones a repo, commits managed paths, pushes, and records `success`.
- [ ] A second publish with no changes creates no new commit.
- [ ] Uncommitted changes outside managed paths abort the publish and set `error`.
- [ ] `/publish` requires auth and CSRF; the state API returns the row.
- [ ] The publish CLI prints a JSON report.
- [ ] `deploy/ihmod-admin.service`, `deploy/backup.sh`, `deploy/ufw-setup.md`, and `deploy/DEPLOY.md` exist.
- [ ] `bash -n deploy/backup.sh` passes.
- [ ] No `data/`, `dist/`, `.env`, or secrets are tracked.
- [ ] Real GitHub publish is performed on the deployed VPS by the user, followed by an Actions/live-site check.

## Notes / risks

- The first publish rewrites the 13 imported posts' frontmatter (adds `date`/`updated`) and creates a commit; this is expected.
- The exporter treats SQLite as the source of truth and overwrites managed Markdown files; manual edits to `source/_posts/**` or `source/img/uploads/**` in the repo will be replaced on the next publish.
- The publisher never force-pushes or resets; conflicts abort and are surfaced in `publish_state`.
- The admin uses HTTP with a UFW IP whitelist; the design documents the risk and the HTTPS/Tailscale upgrade paths.
- The deploy key is repo-scoped and write-only to the blog repo; never commit it.
