# Blog Admin Post Import Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Import the 13 existing Hexo Markdown posts from `/home/ihmod/cc_projects/IHMOD/source/_posts` into the admin SQLite database, with a dry-run mode and an idempotent `--update` mode.

**Architecture:** A synchronous importer service walks a posts directory, parses frontmatter with `gray-matter`, maps fields into the existing `posts`/`categories`/`tags` schema, and runs inside a SQLite transaction. Dry-run executes the same transaction and rolls it back, returning a report. A thin CLI applies migrations, opens `DATABASE_PATH`, and prints a JSON report. Existing posts are matched by `source_path`; without `--update` they are skipped, preserving admin edits.

**Tech Stack:** Existing Plan 2A/2B stack + `gray-matter`.

---

## Prerequisites

Plan 2B is complete in `/home/ihmod/cc_projects/IHMOD-admin`:

- Category/tag/post services and pages exist.
- 62 tests pass.
- `src/lib/ids.ts`, `slug.ts`, `dates.ts`, `tags.ts` exist.

Run:

```bash
cd /home/ihmod/cc_projects/IHMOD-admin
npm test
```

Expected: 15 files, 62 tests pass.

## Current source facts (verified)

- 13 Markdown files under `/home/ihmod/cc_projects/IHMOD/source/_posts`.
- All 13 already have `abbrlink`.
- Categories in frontmatter: `网安` (7), `基础` (2), `日记` (4).
- The two `编程/` folder posts use `categories: 基础`, not `编程`; the importer must preserve the frontmatter category and the `编程/...` source path.
- No post has a `date` field. Only `261110.md`, `261111.md`, `261112.md` encode dates in the filename; the rest fall back to file mtime (recorded as issues for manual review).

## File Structure

```
src/services/importer.ts     # importPosts(db, sqlite, dir, options)
src/cli/import-posts.ts      # CLI entry
test/importer.test.ts
test/fixtures/posts/网安/信息收集.md
test/fixtures/posts/日记/261112.md
test/fixtures/posts/编程/MyBatis-Plus.md
test/fixtures/posts/无分类/no-frontmatter.md
test/fixtures/posts/.obsidian/app.json
README.md                    # add import instructions
package.json                 # add "import" script
```

No database migration is required.

---

## Task 1: Add `gray-matter` and the import script

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json`

- [ ] **Step 1: Install dependencies**

```bash
npm install gray-matter
npm install -D @types/gray-matter
```

Expected: both commands exit 0; `package.json` gains `gray-matter` and `@types/gray-matter`.

- [ ] **Step 2: Add the npm script**

```bash
npm pkg set scripts.import="tsx src/cli/import-posts.ts"
node -e "const p=require('./package.json'); console.log(p.scripts.import)"
```

Expected: `tsx src/cli/import-posts.ts`.

- [ ] **Step 3: Commit**

```bash
git add package.json package-lock.json
git commit -m "chore: add gray-matter and import script"
```

---

## Task 2: Importer service, fixtures, and tests (TDD)

**Files:**
- Create: `src/services/importer.ts`
- Create: `test/fixtures/posts/网安/信息收集.md`
- Create: `test/fixtures/posts/日记/261112.md`
- Create: `test/fixtures/posts/编程/MyBatis-Plus.md`
- Create: `test/fixtures/posts/无分类/no-frontmatter.md`
- Create: `test/fixtures/posts/.obsidian/app.json`
- Test: `test/importer.test.ts`

- [ ] **Step 1: Create the fixtures**

`test/fixtures/posts/网安/信息收集.md`:

```markdown
---
title: 信息收集
tags:
  - 信息收集
categories: 网安
top_img: false
description: 信息收集
abbrlink: 312e3d12
date: 2026-11-12
---

## 域名信息收集
```

`test/fixtures/posts/日记/261112.md`:

```markdown
---
title: 普通的一天
tags:
  - 日记
categories: 日记
top_img: false
description: 2026年11月12日
---

今天整理后台。
```

`test/fixtures/posts/编程/MyBatis-Plus.md`:

```markdown
---
title: MyBatis-Plus学习
tags:
  - MyBatis-Plus
categories: 基础
top_img: false
description: 后端开发工具
abbrlink: cc2af2d6
---

# MyBatis-Plus
```

`test/fixtures/posts/无分类/no-frontmatter.md`:

```markdown
没有 frontmatter 的正文。
```

`test/fixtures/posts/.obsidian/app.json`:

```json
{}
```

- [ ] **Step 2: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { categories, posts } from '../src/db/schema';
import { importPosts } from '../src/services/importer';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

const fixtures = path.resolve(process.cwd(), 'test/fixtures/posts');

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-import-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  return { db, sqlite };
}

describe('importPosts', () => {
  it('imports all fixtures and skips hidden directories', () => {
    const { db, sqlite } = setup();
    const report = importPosts(db, sqlite, fixtures);

    expect(report.imported).toBe(4);
    expect(report.updated).toBe(0);
    expect(report.skipped).toBe(0);

    const rows = db.select().from(posts).all();
    expect(rows.length).toBe(4);
    expect(rows.every((row) => row.status === 'published')).toBe(true);

    const info = rows.find((row) => row.sourcePath === '网安/信息收集.md')!;
    expect(info.abbrlink).toBe('312e3d12');
    expect(info.postDate).toBe('2026-11-12 00:00:00');

    const diary = rows.find((row) => row.sourcePath === '日记/261112.md')!;
    expect(diary.postDate).toBe('2026-11-12 00:00:00');
    expect(diary.abbrlink).toMatch(/^[0-9a-f]{8}$/);

    const mybatis = rows.find((row) => row.sourcePath === '编程/MyBatis-Plus.md')!;
    const base = db.select().from(categories).where(eq(categories.name, '基础')).get()!;
    expect(mybatis.categoryId).toBe(base.id);

    const noFrontmatter = rows.find(
      (row) => row.sourcePath === '无分类/no-frontmatter.md',
    )!;
    expect(noFrontmatter.title).toBe('no-frontmatter');
    expect(noFrontmatter.postDate).toMatch(/^\d{4}-\d{2}-\d{2} /);

    expect(report.issues.some((issue) => issue.includes('missing abbrlink'))).toBe(true);
    expect(report.issues.some((issue) => issue.includes('date fallback'))).toBe(true);
  });

  it('is idempotent by default and updates with the update flag', () => {
    const { db, sqlite } = setup();
    importPosts(db, sqlite, fixtures);

    const second = importPosts(db, sqlite, fixtures);
    expect(second.imported).toBe(0);
    expect(second.updated).toBe(0);
    expect(second.skipped).toBe(4);
    expect(db.select().from(posts).all().length).toBe(4);

    const third = importPosts(db, sqlite, fixtures, { update: true });
    expect(third.imported).toBe(0);
    expect(third.updated).toBe(4);
    expect(third.skipped).toBe(0);
    expect(db.select().from(posts).all().length).toBe(4);
  });

  it('rolls back in dry-run mode', () => {
    const { db, sqlite } = setup();
    const report = importPosts(db, sqlite, fixtures, { dryRun: true });
    expect(report.imported).toBe(4);
    expect(db.select().from(posts).all().length).toBe(0);
    expect(db.select().from(categories).all().length).toBe(0);
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

```bash
npm test -- test/importer.test.ts
```

Expected: FAIL because `src/services/importer.ts` does not exist.

- [ ] **Step 4: Implement `src/services/importer.ts`**

```ts
import fs from 'node:fs';
import path from 'node:path';
import matter from 'gray-matter';
import { eq } from 'drizzle-orm';
import type Database from 'better-sqlite3';
import type { Db } from '../db/client';
import { categories, posts } from '../db/schema';
import { contentHash, newAbbrlink, newId } from '../lib/ids';
import { slugify, uniqueSlug } from '../lib/slug';
import { formatStamp, nowStamp, parseDiaryDate } from '../lib/dates';
import { setPostTags } from '../lib/tags';

export interface ImportOptions {
  dryRun?: boolean;
  update?: boolean;
}

export interface ImportReport {
  imported: number;
  updated: number;
  skipped: number;
  issues: string[];
}

class DryRunRollback extends Error {
  constructor(public report: ImportReport) {
    super('dry-run');
  }
}

function walk(dir: string): string[] {
  const files: string[] = [];
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    if (entry.name.startsWith('.')) continue;
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) files.push(...walk(full));
    else if (entry.name.endsWith('.md')) files.push(full);
  }
  return files;
}

function asRecord(value: unknown): Record<string, unknown> {
  return value && typeof value === 'object' && !Array.isArray(value)
    ? (value as Record<string, unknown>)
    : {};
}

function categoryFrom(data: Record<string, unknown>, relativePath: string): string {
  const raw = data.categories;
  if (Array.isArray(raw) && raw.length > 0) {
    const value = String(raw[0]).trim();
    if (value) return value;
  }
  if (typeof raw === 'string' && raw.trim()) return raw.trim();
  const parent = path.posix.dirname(relativePath);
  if (parent && parent !== '.') return parent.split('/')[0];
  return '未分类';
}

function tagsFrom(data: Record<string, unknown>): string[] {
  const raw = data.tags;
  const values = Array.isArray(raw)
    ? raw
    : typeof raw === 'string'
      ? raw.split(/[,，]/)
      : [];
  return [...new Set(values.map((value) => String(value).trim()).filter(Boolean))];
}

function topImgFrom(data: Record<string, unknown>): string {
  if (data.top_img === false) return 'false';
  if (typeof data.top_img === 'string' && data.top_img.trim()) return data.top_img.trim();
  return 'false';
}

function ensureCategory(db: Db, name: string): number {
  const existing = db
    .select({ id: categories.id })
    .from(categories)
    .where(eq(categories.name, name))
    .get();
  if (existing) return existing.id;
  db.insert(categories).values({ name, createdAt: nowStamp() }).run();
  return db
    .select({ id: categories.id })
    .from(categories)
    .where(eq(categories.name, name))
    .get()!.id;
}

export function importPosts(
  db: Db,
  sqlite: Database.Database,
  postsDir: string,
  options: ImportOptions = {},
): ImportReport {
  const files = walk(postsDir);
  const abbrlinks = new Set(
    db.select({ value: posts.abbrlink }).from(posts).all().map((row) => row.value),
  );
  let report: ImportReport | undefined;

  const run = (): ImportReport => {
    const result: ImportReport = { imported: 0, updated: 0, skipped: 0, issues: [] };

    for (const file of files) {
      const relativePath = path.relative(postsDir, file).split(path.sep).join('/');
      const parsed = matter(fs.readFileSync(file, 'utf8'));
      const data = asRecord(parsed.data);
      const body = parsed.content.trimStart();
      const title =
        typeof data.title === 'string' && data.title.trim()
          ? data.title.trim()
          : path.basename(file, '.md');
      const categoryName = categoryFrom(data, relativePath);
      const tags = tagsFrom(data);
      const topImg = topImgFrom(data);
      const description =
        typeof data.description === 'string' && data.description.trim()
          ? data.description.trim()
          : null;

      let postDate = formatStamp(data.date);
      if (!postDate) postDate = parseDiaryDate(path.basename(file));
      if (!postDate) {
        postDate = formatStamp(fs.statSync(file).mtime);
        result.issues.push(`date fallback to file mtime: ${relativePath}`);
      }
      if (!postDate) {
        postDate = nowStamp();
        result.issues.push(`date fallback to now: ${relativePath}`);
      }

      const existing = db
        .select()
        .from(posts)
        .where(eq(posts.sourcePath, relativePath))
        .get();

      if (existing && !options.update) {
        result.skipped += 1;
        continue;
      }

      const categoryId = ensureCategory(db, categoryName);
      const hash = contentHash({
        title,
        contentMd: body,
        categoryId,
        tags,
        description,
        topImg,
        postDate,
      });

      if (existing) {
        db.update(posts)
          .set({
            title,
            categoryId,
            description,
            topImg,
            contentMd: body,
            postDate,
            updatedAt: postDate,
            contentHash: hash,
          })
          .where(eq(posts.id, existing.id))
          .run();
        setPostTags(db, existing.id, tags);
        result.updated += 1;
        continue;
      }

      const rawAbbrlink = typeof data.abbrlink === 'string' ? data.abbrlink.trim() : '';
      let abbrlink: string;
      if (rawAbbrlink && !abbrlinks.has(rawAbbrlink)) {
        abbrlink = rawAbbrlink;
        abbrlinks.add(abbrlink);
      } else {
        if (rawAbbrlink) result.issues.push(`duplicate abbrlink ${rawAbbrlink}: ${relativePath}`);
        else result.issues.push(`missing abbrlink: ${relativePath}`);
        abbrlink = newAbbrlink(abbrlinks);
      }

      const takenSlugs = new Set(
        db
          .select({ slug: posts.slug })
          .from(posts)
          .where(eq(posts.categoryId, categoryId))
          .all()
          .map((row) => row.slug),
      );
      const slug = uniqueSlug(slugify(title), takenSlugs);

      db.insert(posts)
        .values({
          id: newId(),
          title,
          slug,
          categoryId,
          description,
          topImg,
          contentMd: body,
          status: 'published',
          abbrlink,
          sourcePath: relativePath,
          postDate,
          createdAt: postDate,
          updatedAt: postDate,
          publishedAt: postDate,
          contentHash: hash,
        })
        .run();

      const inserted = db
        .select({ id: posts.id })
        .from(posts)
        .where(eq(posts.sourcePath, relativePath))
        .get()!;
      setPostTags(db, inserted.id, tags);
      result.imported += 1;
    }

    return result;
  };

  try {
    const execute = sqlite.transaction(() => {
      report = run();
      if (options.dryRun) throw new DryRunRollback(report);
    });
    execute();
  } catch (error) {
    if (error instanceof DryRunRollback) return error.report;
    throw error;
  }

  return report!;
}
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
npm test -- test/importer.test.ts
```

Expected: 3 tests pass.

- [ ] **Step 6: Run the full suite and typecheck**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
```

Expected: all tests pass; tsc exit 0.

- [ ] **Step 7: Commit**

```bash
git add src/services/importer.ts test/importer.test.ts test/fixtures
git commit -m "feat: add post importer"
```

---

## Task 3: Import CLI and real import verification

**Files:**
- Create: `src/cli/import-posts.ts`
- Modify: `README.md` (Task 4)

- [ ] **Step 1: Implement `src/cli/import-posts.ts`**

```ts
import 'dotenv/config';
import path from 'node:path';
import { createDb } from '../db/client';
import { applyMigrations } from '../db/migrate';
import { importPosts } from '../services/importer';

const args = process.argv.slice(2);
const flags = new Set(args.filter((arg) => arg.startsWith('--')));
const dirArg = args.find((arg) => !arg.startsWith('--'));

if (!dirArg) {
  console.error('usage: npm run import -- <posts-dir> [--dry-run] [--update]');
  process.exit(1);
}

const databasePath = process.env.DATABASE_PATH ?? './data/admin.db';
const resolvedDir = path.resolve(dirArg);
const { db, sqlite } = createDb(databasePath);
applyMigrations(sqlite, path.resolve(__dirname, '../../migrations'));

try {
  const report = importPosts(db, sqlite, resolvedDir, {
    dryRun: flags.has('--dry-run'),
    update: flags.has('--update'),
  });
  console.log(
    JSON.stringify(
      {
        database: databasePath,
        dir: resolvedDir,
        dryRun: flags.has('--dry-run'),
        update: flags.has('--update'),
        ...report,
      },
      null,
      2,
    ),
  );
} finally {
  sqlite.close();
}
```

- [ ] **Step 2: Run the dry-run against the real source**

```bash
DATABASE_PATH=./data/admin.db npm run import -- /home/ihmod/cc_projects/IHMOD/source/_posts --dry-run
```

Expected:
- `imported: 13`, `updated: 0`, `skipped: 0`.
- `issues` contains 10 `date fallback to file mtime` entries (the 3 diary filenames parse; `日记模板.md` and the other 9 posts fall back).
- No `missing abbrlink` entries (all 13 already have one).

Verify the dry-run did not persist posts:

```bash
node -e "const D=require('better-sqlite3'); const db=new D('data/admin.db'); console.log(db.prepare('select count(*) n from posts').get().n)"
```

Expected: `0`.

- [ ] **Step 3: Run the real import**

```bash
DATABASE_PATH=./data/admin.db npm run import -- /home/ihmod/cc_projects/IHMOD/source/_posts
```

Expected: `imported: 13`; same `issues` list.

- [ ] **Step 4: Verify the imported data**

```bash
node -e "const D=require('better-sqlite3'); const db=new D('data/admin.db'); console.log('posts', db.prepare('select count(*) n from posts').get().n); console.log('published', db.prepare(\"select count(*) n from posts where status='published'\").get().n); console.log('categories', db.prepare('select name from categories order by name').all().map(r=>r.name)); console.log('tags', db.prepare('select count(*) n from tags').get().n);"
```

Expected:
- `posts 13`
- `published 13`
- categories include `基础`, `日记`, `网安` (no `编程`; the two `编程/` posts use `categories: 基础`)
- tags > 0

Verify a known post and a diary date:

```bash
node -e "const D=require('better-sqlite3'); const db=new D('data/admin.db'); console.log(db.prepare('select title, abbrlink, source_path, post_date from posts where abbrlink=?').get('312e3d12')); console.log(db.prepare('select title, post_date from posts where source_path=?').get('日记/261112.md'));"
```

Expected:
- The `信息收集` post with `abbrlink 312e3d12` and `source_path 网安/信息收集.md`. Its `post_date` is a valid `YYYY-MM-DD HH:mm:ss` derived from file mtime (the real post has no `date` field), not a fixed date.
- The diary post with `post_date 2026-11-12 00:00:00` (parsed from the filename).

- [ ] **Step 5: Verify idempotency**

```bash
DATABASE_PATH=./data/admin.db npm run import -- /home/ihmod/cc_projects/IHMOD/source/_posts
DATABASE_PATH=./data/admin.db npm run import -- /home/ihmod/cc_projects/IHMOD/source/_posts --update
```

Expected:
- Second run: `imported: 0`, `updated: 0`, `skipped: 13`.
- Third run with `--update`: `imported: 0`, `updated: 13`, `skipped: 0`.
- Post count remains 13.

- [ ] **Step 6: Commit the CLI**

```bash
git add src/cli/import-posts.ts
git commit -m "feat: add post import CLI"
```

---

## Task 4: README and final verification

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add the import section to `README.md`**

```markdown
## Import existing Hexo posts

The importer reads Markdown files recursively and matches existing posts by
`source_path`. It never deletes posts. Hidden directories such as `.obsidian/`
are skipped.

Dry run (writes are rolled back):

```bash
DATABASE_PATH=./data/admin.db npm run import -- /path/to/source/_posts --dry-run
```

Real import:

```bash
DATABASE_PATH=./data/admin.db npm run import -- /path/to/source/_posts
```

Re-import and overwrite managed fields (title, body, date, category, tags,
description, top_img) while preserving IDs, `abbrlink`, slug, `source_path`,
status, and `published_at`:

```bash
DATABASE_PATH=./data/admin.db npm run import -- /path/to/source/_posts --update
```

Date handling: `frontmatter.date` wins; `YYMMDD.md` diary filenames are parsed;
otherwise the file mtime is used and reported as an issue. Review the reported
issues in the admin before publishing.
```

- [ ] **Step 2: Run the full suite and build**

```bash
npm test
npm run build
```

Expected: all tests pass; build exits 0.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: document post import"
```

---

## Acceptance Checklist

- [ ] `npm test` passes (all Plan 2A/2B tests + 3 importer tests).
- [ ] `npm run build` exits 0.
- [ ] Fixture dry-run imports 4 and leaves the DB empty.
- [ ] Fixture import creates 4 posts, skips `.obsidian/`, preserves `312e3d12`, parses the diary date, and reports missing `abbrlink`/date fallback issues.
- [ ] Second fixture import skips 4; `--update` updates 4.
- [ ] Real dry-run reports 13 imports and leaves `posts` empty.
- [ ] Real import creates 13 published posts in categories `基础`, `日记`, `网安`; all known `abbrlink` values preserved; source paths preserved.
- [ ] Diary `261112.md` gets `2026-11-12 00:00:00`.
- [ ] Second real run skips 13; `--update` updates 13; count remains 13.
- [ ] `data/admin.db` is not tracked or committed.
- [ ] The CLI prints a JSON report and exits 0.

## Known behavior

- `编程/MyBatis-Plus.md` and `编程/PHP学习.md` import under category `基础` because their frontmatter says `categories: 基础`; their `source_path` stays under `编程/`.
- Non-diary posts without a `date` field use file mtime; the report lists them for manual adjustment in the admin.
- The importer preserves existing admin status (`draft`/`published`) when `--update` is used; it does not force republish.

## Next plans

- **2D:** Vditor editor, autosave, media upload/library.
- **2E:** Markdown export, publish manifest, Git push, systemd/backup/ufw deployment.
