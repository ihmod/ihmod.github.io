# Blog Admin Content CRUD Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add category, tag, and post management to the admin app: list/filter/search posts, create and edit posts with a plain Markdown textarea, publish/unpublish, soft-delete/restore/purge, and manage categories/tags with referential checks.

**Architecture:** Fastify routes render EJS pages and call small synchronous service modules over the existing SQLite/Drizzle schema. Posts own slug, `abbrlink`, `source_path`, timestamps, and tag associations. All mutations are POST + CSRF and redirect to a list page with a fixed notice/error key. This plan intentionally uses a textarea; Vditor and media come later.

**Tech Stack:** Existing Plan 2A stack (Node.js 22, TypeScript 7/Node16, Fastify 5, EJS, better-sqlite3, Drizzle ORM, Zod, Vitest).

---

## Prerequisites

Plan 2A is complete and merged in `/home/ihmod/cc_projects/IHMOD-admin`:

- App boots, login works, 27 tests pass.
- Tables `posts`, `categories`, `tags`, `post_tags` exist.
- Existing files: `src/app.ts`, `src/plugins/session-auth.ts`, `src/db/*`, `src/routes/auth.ts`, `src/routes/dashboard.ts`, `views/partials/header.ejs`, `public/admin.css`.

Run the suite before starting:

```bash
cd /home/ihmod/cc_projects/IHMOD-admin
npm test
```

Expected: 8 test files, 27 tests pass.

## File Structure

New or modified files:

```
src/lib/ids.ts                 # UUID, abbrlink, content hash
src/lib/slug.ts                # slug generation + uniqueness
src/lib/dates.ts               # timestamp formatting (shared with importer)
src/lib/categories.ts          # category service
src/lib/tags.ts                # tag service + post tag replacement
src/lib/posts.ts               # post service (CRUD/list/filters)
src/routes/categories.ts       # category pages
src/routes/tags.ts             # tag pages
src/routes/posts.ts            # post pages
src/app.ts                     # register the three new route modules
views/categories.ejs
views/tags.ejs
views/posts/index.ejs
views/posts/form.ejs
views/partials/notice.ejs
views/partials/header.ejs      # add nav links
public/admin.css               # append table/form/toolbar styles
test/helpers.ts                # shared app/login helpers for new tests
test/ids.test.ts
test/slug.test.ts
test/dates.test.ts
test/categories-tags.test.ts
test/posts.test.ts
test/categories-tags-routes.test.ts
test/posts-routes.test.ts
```

No database migration is needed; the schema from Plan 2A already has every column used here.

---

## Task 1: Add shared test helpers

**Files:**
- Create: `test/helpers.ts`
- Modify: `package.json`? No.

- [ ] **Step 1: Create `test/helpers.ts`**

```ts
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import type { FastifyInstance } from 'fastify';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { seedAdmin } from '../src/auth/seed';
import { loadConfig } from '../src/config';
import { buildApp } from '../src/app';

export interface TestApp {
  app: FastifyInstance;
  db: ReturnType<typeof createDb>['db'];
  sqlite: ReturnType<typeof createDb>['sqlite'];
  dir: string;
  cleanup: () => Promise<void>;
}

export function setCookies(res: { headers: Record<string, unknown> }): string[] {
  const raw = res.headers['set-cookie'];
  if (!raw) return [];
  return Array.isArray(raw) ? raw : [raw as string];
}

export function cookieHeader(cookies: string[]): string {
  return cookies.map((cookie) => cookie.split(';')[0]).join('; ');
}

export function extractCsrf(html: string): string {
  const match = html.match(/name="_csrf" value="([^"]+)"/);
  if (!match) throw new Error('csrf token not found');
  return match[1];
}

export async function makeTestApp(overrides: Record<string, string> = {}): Promise<TestApp> {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-admin-'));
  const dbPath = path.join(dir, 'test.db');
  const { db, sqlite } = createDb(dbPath);
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  const config = loadConfig({
    NODE_ENV: 'test',
    SESSION_SECRET: 'x'.repeat(32),
    ADMIN_USERNAME: 'admin',
    ADMIN_PASSWORD: 'test-password-123',
    DATABASE_PATH: dbPath,
    LOGIN_RATE_LIMIT_MAX: '100',
    COOKIE_SECURE: 'false',
    ...overrides,
  });
  await seedAdmin(db, config);
  const app = await buildApp({ db, sqlite, config });
  return {
    app,
    db,
    sqlite,
    dir,
    cleanup: async () => {
      await app.close();
      sqlite.close();
      fs.rmSync(dir, { recursive: true, force: true });
    },
  };
}

export async function login(app: FastifyInstance): Promise<{ cookie: string; csrf: string }> {
  const page = await app.inject({ method: 'GET', url: '/login' });
  const cookies = setCookies(page);
  const csrf = extractCsrf(page.body);
  const res = await app.inject({
    method: 'POST',
    url: '/login',
    headers: {
      cookie: cookieHeader(cookies),
      'content-type': 'application/x-www-form-urlencoded',
    },
    payload: `_csrf=${encodeURIComponent(csrf)}&username=admin&password=test-password-123`,
  });
  if (res.statusCode !== 302) throw new Error(`login failed: ${res.statusCode}`);
  return { cookie: cookieHeader([...cookies, ...setCookies(res)]), csrf };
}
```

- [ ] **Step 2: Typecheck**

```bash
npx tsc -p tsconfig.json --noEmit
```

Expected: exit 0. `test/` is excluded from tsc, so this only checks source. If `tsc` complains, the issue is in source, not the helper.

- [ ] **Step 3: Commit**

```bash
git add test/helpers.ts
git commit -m "test: add shared admin test helpers"
```

---

## Task 2: Content utilities (ids, slug, dates) (TDD)

**Files:**
- Create: `src/lib/ids.ts`
- Create: `src/lib/slug.ts`
- Create: `src/lib/dates.ts`
- Test: `test/ids.test.ts`
- Test: `test/slug.test.ts`
- Test: `test/dates.test.ts`

- [ ] **Step 1: Write the failing tests**

`test/ids.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { contentHash, newAbbrlink, newId } from '../src/lib/ids';

describe('ids', () => {
  it('creates UUIDs', () => {
    expect(newId()).toMatch(/^[0-9a-f-]{36}$/);
    expect(newId()).not.toBe(newId());
  });

  it('creates unique 8-char hex abbrlinks', () => {
    const taken = new Set<string>();
    const values = new Set<string>();
    for (let i = 0; i < 50; i++) values.add(newAbbrlink(taken));
    expect(values.size).toBe(50);
    for (const value of values) expect(value).toMatch(/^[0-9a-f]{8}$/);
  });

  it('hashes content deterministically and ignores tag order', () => {
    const a = contentHash({
      title: 'T',
      contentMd: 'body',
      categoryId: 1,
      tags: ['a', 'b'],
      description: null,
      topImg: 'false',
      postDate: '2026-11-12 00:00:00',
    });
    const b = contentHash({
      title: 'T',
      contentMd: 'body',
      categoryId: 1,
      tags: ['b', 'a'],
      description: null,
      topImg: 'false',
      postDate: '2026-11-12 00:00:00',
    });
    expect(a).toBe(b);
    expect(a).toMatch(/^[0-9a-f]{64}$/);
    expect(contentHash({
      title: 'T2',
      contentMd: 'body',
      categoryId: 1,
      tags: ['a', 'b'],
      description: null,
      topImg: 'false',
      postDate: '2026-11-12 00:00:00',
    })).not.toBe(a);
  });
});
```

`test/slug.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { slugify, uniqueSlug } from '../src/lib/slug';

describe('slugify', () => {
  it('keeps Chinese and normalizes spaces', () => {
    expect(slugify('信息 收集')).toBe('信息-收集');
    expect(slugify('  Hello   World  ')).toBe('hello-world');
  });

  it('removes path separators and unsafe characters', () => {
    expect(slugify('网安/信息:收集')).toBe('网安-信息-收集');
    expect(slugify('../../etc/passwd')).toBe('etc-passwd');
  });

  it('falls back to post for empty input', () => {
    expect(slugify('   ')).toBe('post');
    expect(slugify('###')).toBe('post');
  });

  it('limits length', () => {
    expect(slugify('a'.repeat(200)).length).toBeLessThanOrEqual(80);
  });
});

describe('uniqueSlug', () => {
  it('adds -2, -3 suffixes', () => {
    const taken = new Set<string>();
    expect(uniqueSlug('post', taken)).toBe('post');
    expect(uniqueSlug('post', taken)).toBe('post-2');
    expect(uniqueSlug('post', taken)).toBe('post-3');
  });
});
```

`test/dates.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { formatStamp, nowStamp, parseDiaryDate } from '../src/lib/dates';

describe('dates', () => {
  it('formats date-only YAML values without timezone drift', () => {
    expect(formatStamp(new Date('2026-11-12T00:00:00.000Z'))).toBe('2026-11-12 00:00:00');
  });

  it('preserves string timestamps', () => {
    expect(formatStamp('2026-11-12 10:20:30')).toBe('2026-11-12 10:20:30');
    expect(formatStamp('2026-11-12')).toBe('2026-11-12 00:00:00');
  });

  it('parses diary filenames', () => {
    expect(parseDiaryDate('261112.md')).toBe('2026-11-12 00:00:00');
    expect(parseDiaryDate('not-a-date.md')).toBeNull();
  });

  it('rejects invalid values', () => {
    expect(formatStamp('nonsense')).toBeNull();
    expect(formatStamp(undefined)).toBeNull();
  });

  it('formats now as YYYY-MM-DD HH:mm:ss', () => {
    expect(nowStamp()).toMatch(/^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}$/);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
npm test -- test/ids.test.ts test/slug.test.ts test/dates.test.ts
```

Expected: FAIL with module-not-found errors.

- [ ] **Step 3: Implement `src/lib/ids.ts`**

```ts
import crypto from 'node:crypto';

export interface HashParts {
  title: string;
  contentMd: string;
  categoryId: number | null;
  tags: string[];
  description: string | null;
  topImg: string;
  postDate: string;
}

export function newId(): string {
  return crypto.randomUUID();
}

export function newAbbrlink(existing: Set<string>): string {
  for (;;) {
    const value = crypto.randomBytes(4).toString('hex');
    if (!existing.has(value)) {
      existing.add(value);
      return value;
    }
  }
}

export function contentHash(parts: HashParts): string {
  const canonical = JSON.stringify({
    ...parts,
    tags: [...parts.tags].sort(),
  });
  return crypto.createHash('sha256').update(canonical).digest('hex');
}
```

- [ ] **Step 4: Implement `src/lib/slug.ts`**

```ts
const UNSAFE = /[^\p{L}\p{N}._-]+/gu;

export function slugify(input: string): string {
  const value = input
    .normalize('NFKC')
    .trim()
    .toLowerCase()
    .replace(UNSAFE, '-')
    .replace(/-+/g, '-')
    .replace(/^[-.]+|[-.]+$/g, '')
    .slice(0, 80);
  return value || 'post';
}

export function uniqueSlug(base: string, taken: Set<string>): string {
  if (!taken.has(base)) {
    taken.add(base);
    return base;
  }
  let index = 2;
  while (taken.has(`${base}-${index}`)) index += 1;
  const value = `${base}-${index}`;
  taken.add(value);
  return value;
}
```

- [ ] **Step 5: Implement `src/lib/dates.ts`**

```ts
function pad(value: number): string {
  return String(value).padStart(2, '0');
}

export function nowStamp(date = new Date()): string {
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`;
}

export function formatStamp(value: unknown): string | null {
  if (value instanceof Date && !Number.isNaN(value.getTime())) {
    if (value.getUTCHours() === 0 && value.getUTCMinutes() === 0 && value.getUTCSeconds() === 0) {
      return `${value.getUTCFullYear()}-${pad(value.getUTCMonth() + 1)}-${pad(value.getUTCDate())} 00:00:00`;
    }
    return nowStamp(value);
  }

  if (typeof value === 'string') {
    const trimmed = value.trim();
    const match = trimmed.match(
      /^(\d{4})-(\d{2})-(\d{2})(?:[ T](\d{2}):(\d{2})(?::(\d{2}))?)?/,
    );
    if (match) {
      return `${match[1]}-${match[2]}-${match[3]} ${match[4] ?? '00'}:${match[5] ?? '00'}:${match[6] ?? '00'}`;
    }
    const parsed = new Date(trimmed);
    if (!Number.isNaN(parsed.getTime())) return nowStamp(parsed);
  }

  return null;
}

export function parseDiaryDate(filename: string): string | null {
  const match = filename.match(/^(\d{2})(\d{2})(\d{2})\.md$/);
  if (!match) return null;
  return `20${match[1]}-${match[2]}-${match[3]} 00:00:00`;
}
```

- [ ] **Step 6: Run the tests to verify they pass**

```bash
npm test -- test/ids.test.ts test/slug.test.ts test/dates.test.ts
```

Expected: 3 files, all tests pass.

- [ ] **Step 7: Typecheck and commit**

```bash
npx tsc -p tsconfig.json --noEmit
git add src/lib/ids.ts src/lib/slug.ts src/lib/dates.ts test/ids.test.ts test/slug.test.ts test/dates.test.ts
git commit -m "feat: add content id, slug, and date utilities"
```

Expected: tsc exit 0; one commit.

---

## Task 3: Category and tag services (TDD)

**Files:**
- Create: `src/lib/categories.ts`
- Create: `src/lib/tags.ts`
- Test: `test/categories-tags.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { categories, postTags, posts, tags } from '../src/db/schema';
import {
  CategoryInUseError,
  createCategory,
  deleteCategory,
  listCategories,
  renameCategory,
} from '../src/lib/categories';
import {
  TagInUseError,
  createTag,
  deleteTag,
  listTags,
  renameTag,
  setPostTags,
} from '../src/lib/tags';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-ct-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  const now = new Date().toISOString();
  const category = createCategory(db, '网安');
  db.insert(posts).values({
    id: 'p1',
    title: 'T',
    slug: 't',
    categoryId: category.id,
    topImg: 'false',
    contentMd: '',
    status: 'published',
    abbrlink: 'abc12345',
    sourcePath: '网安/t.md',
    postDate: now,
    createdAt: now,
    updatedAt: now,
  }).run();
  return { db };
}

describe('categories', () => {
  it('creates, lists, and renames', () => {
    const { db } = setup();
    expect(listCategories(db)[0]).toMatchObject({ name: '网安', postCount: 1 });
    renameCategory(db, listCategories(db)[0].id, '安全');
    expect(listCategories(db)[0].name).toBe('安全');
  });

  it('blocks deleting a category with posts', () => {
    const { db } = setup();
    const id = listCategories(db)[0].id;
    expect(() => deleteCategory(db, id)).toThrow(CategoryInUseError);
    expect(db.select().from(categories).where(eq(categories.id, id)).get()).toBeTruthy();
  });

  it('deletes an unused category', () => {
    const { db } = setup();
    const created = createCategory(db, '随笔');
    deleteCategory(db, created.id);
    expect(db.select().from(categories).where(eq(categories.id, created.id)).get()).toBeUndefined();
  });
});

describe('tags', () => {
  it('creates, lists, and replaces post tags', () => {
    const { db } = setup();
    setPostTags(db, 'p1', ['收集', '谷歌', '收集']);
    expect(listTags(db).map((tag) => tag.name).sort()).toEqual(['收集', '谷歌']);
    setPostTags(db, 'p1', ['仅一个']);
    const tagsAfter = listTags(db);
    expect(tagsAfter.find((tag) => tag.name === '仅一个')?.postCount).toBe(1);
    expect(tagsAfter.find((tag) => tag.name === '收集')?.postCount).toBe(0);
  });

  it('blocks deleting a tag used by a post', () => {
    const { db } = setup();
    setPostTags(db, 'p1', ['收集']);
    const id = listTags(db)[0].id;
    expect(() => deleteTag(db, id)).toThrow(TagInUseError);
    expect(db.select().from(tags).where(eq(tags.id, id)).get()).toBeTruthy();
  });

  it('deletes an unused tag and renames in place', () => {
    const { db } = setup();
    const tag = createTag(db, '旧名');
    renameTag(db, tag.id, '新名');
    expect(listTags(db).map((entry) => entry.name)).toContain('新名');
    deleteTag(db, tag.id);
    expect(db.select().from(tags).where(eq(tags.id, tag.id)).get()).toBeUndefined();
    expect(db.select().from(postTags).all()).toEqual([]);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/categories-tags.test.ts
```

Expected: FAIL with module-not-found errors.

- [ ] **Step 3: Implement `src/lib/categories.ts`**

```ts
import { and, eq, isNull, sql } from 'drizzle-orm';
import type { Db } from '../db/client';
import { categories, posts } from '../db/schema';
import { nowStamp } from './dates';

export class CategoryInUseError extends Error {}
export class DuplicateCategoryError extends Error {}

export interface CategoryRow {
  id: number;
  name: string;
  postCount: number;
}

export function listCategories(db: Db): CategoryRow[] {
  return db
    .select({
      id: categories.id,
      name: categories.name,
      postCount: sql<number>`count(${posts.id})`,
    })
    .from(categories)
    .leftJoin(posts, and(eq(posts.categoryId, categories.id), isNull(posts.deletedAt)))
    .groupBy(categories.id, categories.name)
    .orderBy(categories.name)
    .all();
}

export function createCategory(db: Db, name: string) {
  const clean = name.trim();
  if (!clean) throw new Error('分类名不能为空');
  if (db.select().from(categories).where(eq(categories.name, clean)).get()) {
    throw new DuplicateCategoryError('分类已存在');
  }
  db.insert(categories).values({ name: clean, createdAt: nowStamp() }).run();
  return db.select().from(categories).where(eq(categories.name, clean)).get()!;
}

export function renameCategory(db: Db, id: number, name: string): void {
  const clean = name.trim();
  if (!clean) throw new Error('分类名不能为空');
  const existing = db.select().from(categories).where(eq(categories.name, clean)).get();
  if (existing && existing.id !== id) throw new DuplicateCategoryError('分类已存在');
  db.update(categories).set({ name: clean }).where(eq(categories.id, id)).run();
}

export function deleteCategory(db: Db, id: number): void {
  const used =
    db.select({ n: sql<number>`count(*)` }).from(posts).where(eq(posts.categoryId, id)).get()?.n ?? 0;
  if (used > 0) throw new CategoryInUseError('分类下还有文章');
  db.delete(categories).where(eq(categories.id, id)).run();
}
```

- [ ] **Step 4: Implement `src/lib/tags.ts`**

```ts
import { and, eq, isNull, sql } from 'drizzle-orm';
import type { Db } from '../db/client';
import { postTags, posts, tags } from '../db/schema';
import { nowStamp } from './dates';

export class TagInUseError extends Error {}
export class DuplicateTagError extends Error {}

export interface TagRow {
  id: number;
  name: string;
  postCount: number;
}

export function listTags(db: Db): TagRow[] {
  return db
    .select({
      id: tags.id,
      name: tags.name,
      postCount: sql<number>`count(${posts.id})`,
    })
    .from(tags)
    .leftJoin(postTags, eq(postTags.tagId, tags.id))
    .leftJoin(posts, and(eq(posts.id, postTags.postId), isNull(posts.deletedAt)))
    .groupBy(tags.id, tags.name)
    .orderBy(tags.name)
    .all();
}

export function createTag(db: Db, name: string) {
  const clean = name.trim();
  if (!clean) throw new Error('标签名不能为空');
  if (db.select().from(tags).where(eq(tags.name, clean)).get()) {
    throw new DuplicateTagError('标签已存在');
  }
  db.insert(tags).values({ name: clean, createdAt: nowStamp() }).run();
  return db.select().from(tags).where(eq(tags.name, clean)).get()!;
}

export function renameTag(db: Db, id: number, name: string): void {
  const clean = name.trim();
  if (!clean) throw new Error('标签名不能为空');
  const existing = db.select().from(tags).where(eq(tags.name, clean)).get();
  if (existing && existing.id !== id) throw new DuplicateTagError('标签已存在');
  db.update(tags).set({ name: clean }).where(eq(tags.id, id)).run();
}

export function deleteTag(db: Db, id: number): void {
  const used =
    db.select({ n: sql<number>`count(*)` }).from(postTags).where(eq(postTags.tagId, id)).get()?.n ?? 0;
  if (used > 0) throw new TagInUseError('标签仍被文章使用');
  db.delete(tags).where(eq(tags.id, id)).run();
}

export function setPostTags(db: Db, postId: string, names: string[]): void {
  const clean = [...new Set(names.map((name) => name.trim()).filter(Boolean))];
  db.delete(postTags).where(eq(postTags.postId, postId)).run();
  for (const name of clean) {
    let tag = db.select({ id: tags.id }).from(tags).where(eq(tags.name, name)).get();
    if (!tag) {
      db.insert(tags).values({ name, createdAt: nowStamp() }).run();
      tag = db.select({ id: tags.id }).from(tags).where(eq(tags.name, name)).get()!;
    }
    db.insert(postTags).values({ postId, tagId: tag.id }).run();
  }
}
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
npm test -- test/categories-tags.test.ts
```

Expected: 2 files? No — one file with 6 tests, all pass.

- [ ] **Step 6: Typecheck and commit**

```bash
npx tsc -p tsconfig.json --noEmit
git add src/lib/categories.ts src/lib/tags.ts test/categories-tags.test.ts
git commit -m "feat: add category and tag services"
```

Expected: tsc exit 0; commit created.

---

## Task 4: Post service (TDD)

**Files:**
- Create: `src/lib/posts.ts`
- Test: `test/posts.test.ts`

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
import {
  createPost,
  getPost,
  listPosts,
  PostNotFoundError,
  publishPost,
  purgePost,
  restorePost,
  softDeletePost,
  unpublishPost,
  updatePost,
} from '../src/lib/posts';
import { listTags } from '../src/lib/tags';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-posts-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  return { db };
}

const base = {
  title: '信息收集',
  categoryId: null,
  description: 'desc',
  topImg: 'false',
  contentMd: '# hello',
  status: 'draft' as const,
  postDate: '2026-11-12 10:00:00',
  tags: ['收集', '谷歌'],
};

describe('posts service', () => {
  it('creates a post with abbrlink, path, tags, and hash', () => {
    const { db } = setup();
    const id = createPost(db, base);
    const post = getPost(db, id)!;
    expect(post.title).toBe('信息收集');
    expect(post.slug).toBe('信息收集');
    expect(post.abbrlink).toMatch(/^[0-9a-f]{8}$/);
    expect(post.sourcePath).toBe('未分类/信息收集.md');
    expect(post.tags.sort()).toEqual(['收集', '谷歌']);
    expect(post.contentHash).toMatch(/^[0-9a-f]{64}$/);
    expect(post.publishedAt).toBeNull();
  });

  it('uses the category folder and uniquifies slugs', () => {
    const { db } = setup();
    const category = createCategory(db, '网安');
    const first = createPost(db, { ...base, categoryId: category.id });
    const second = createPost(db, { ...base, categoryId: category.id });
    expect(getPost(db, first)!.sourcePath).toBe('网安/信息收集.md');
    expect(getPost(db, second)!.slug).toBe('信息收集-2');
    expect(getPost(db, second)!.sourcePath).toBe('网安/信息收集-2.md');
  });

  it('updates content while preserving abbrlink', () => {
    const { db } = setup();
    const id = createPost(db, base);
    const before = getPost(db, id)!;
    updatePost(db, id, { ...base, title: '信息收集 2', contentMd: 'changed', tags: ['新'] });
    const after = getPost(db, id)!;
    expect(after.abbrlink).toBe(before.abbrlink);
    expect(after.title).toBe('信息收集 2');
    expect(after.contentMd).toBe('changed');
    expect(after.tags).toEqual(['新']);
    expect(after.contentHash).not.toBe(before.contentHash);
    expect(listTags(db).map((tag) => tag.name)).toContain('新');
  });

  it('moves the file path when the category changes', () => {
    const { db } = setup();
    const security = createCategory(db, '网安');
    const id = createPost(db, base);
    updatePost(db, id, { ...base, categoryId: security.id });
    expect(getPost(db, id)!.sourcePath).toBe('网安/信息收集.md');
  });

  it('publishes and unpublishes', () => {
    const { db } = setup();
    const id = createPost(db, base);
    publishPost(db, id);
    expect(getPost(db, id)!.status).toBe('published');
    expect(getPost(db, id)!.publishedAt).not.toBeNull();
    unpublishPost(db, id);
    expect(getPost(db, id)!.status).toBe('draft');
    expect(getPost(db, id)!.publishedAt).not.toBeNull();
  });

  it('soft deletes, restores, and purges', () => {
    const { db } = setup();
    const id = createPost(db, base);
    softDeletePost(db, id);
    expect(getPost(db, id)!.deletedAt).not.toBeNull();
    expect(listPosts(db, { status: 'trash' }).total).toBe(1);
    restorePost(db, id);
    expect(getPost(db, id)!.deletedAt).toBeNull();
    purgePost(db, id);
    expect(getPost(db, id)).toBeNull();
    expect(() => updatePost(db, id, base)).toThrow(PostNotFoundError);
  });

  it('filters, searches, and paginates', () => {
    const { db } = setup();
    const category = createCategory(db, '网安');
    createPost(db, { ...base, title: 'A', categoryId: category.id });
    const published = createPost(db, { ...base, title: 'B', status: 'published' });
    createPost(db, { ...base, title: 'C' });
    expect(listPosts(db, { status: 'published' }).total).toBe(1);
    expect(listPosts(db, { status: 'draft' }).total).toBe(2);
    expect(listPosts(db, { status: 'all', categoryId: category.id }).total).toBe(1);
    expect(listPosts(db, { status: 'all', q: 'B' }).items[0].id).toBe(published);
    const page1 = listPosts(db, { status: 'all', page: 1, pageSize: 2 });
    expect(page1.items.length).toBe(2);
    expect(page1.totalPages).toBe(2);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/posts.test.ts
```

Expected: FAIL with module-not-found errors.

- [ ] **Step 3: Implement `src/lib/posts.ts`**

```ts
import path from 'node:path';
import { and, desc, eq, inArray, isNotNull, isNull, like, sql, type SQL } from 'drizzle-orm';
import type { Db } from '../db/client';
import { categories, postTags, posts, tags } from '../db/schema';
import { contentHash, newAbbrlink, newId } from './ids';
import { slugify, uniqueSlug } from './slug';
import { formatStamp, nowStamp } from './dates';
import { setPostTags } from './tags';

export type PostStatus = 'draft' | 'published';

export interface PostInput {
  title: string;
  slug?: string | null;
  categoryId: number | null;
  description?: string | null;
  topImg?: string;
  contentMd: string;
  status: PostStatus;
  postDate?: string | null;
  tags: string[];
}

export interface PostListFilters {
  status: 'all' | 'draft' | 'published' | 'trash';
  categoryId?: number | null;
  q?: string;
  page?: number;
  pageSize?: number;
}

export class PostNotFoundError extends Error {}
export class PostValidationError extends Error {}

function categoryName(db: Db, categoryId: number | null): string {
  if (categoryId == null) return '未分类';
  return (
    db.select({ name: categories.name }).from(categories).where(eq(categories.id, categoryId)).get()
      ?.name ?? '未分类'
  );
}

function safeFolder(name: string): string {
  return name.replace(/[\\/]/g, '-').trim() || '未分类';
}

function slugsInCategory(db: Db, categoryId: number | null, excludeId?: string): Set<string> {
  const rows = categoryId == null
    ? db.select({ slug: posts.slug }).from(posts).where(
        excludeId ? and(isNull(posts.categoryId), sql`${posts.id} <> ${excludeId}`) : isNull(posts.categoryId),
      ).all()
    : db.select({ slug: posts.slug }).from(posts).where(
        excludeId
          ? and(eq(posts.categoryId, categoryId), sql`${posts.id} <> ${excludeId}`)
          : eq(posts.categoryId, categoryId),
      ).all();
  return new Set(rows.map((row) => row.slug));
}

function allSourcePaths(db: Db, excludeId?: string): Set<string> {
  const rows = excludeId
    ? db.select({ sourcePath: posts.sourcePath }).from(posts).where(sql`${posts.id} <> ${excludeId}`).all()
    : db.select({ sourcePath: posts.sourcePath }).from(posts).all();
  return new Set(rows.map((row) => row.sourcePath));
}

function uniquePath(base: string, taken: Set<string>): string {
  const ext = path.posix.extname(base);
  const stem = base.slice(0, -ext.length);
  let candidate = base;
  let index = 2;
  while (taken.has(candidate)) {
    candidate = `${stem}-${index}${ext}`;
    index += 1;
  }
  taken.add(candidate);
  return candidate;
}

function abbrlinkSet(db: Db): Set<string> {
  return new Set(db.select({ abbrlink: posts.abbrlink }).from(posts).all().map((row) => row.abbrlink));
}

function normalize(raw: PostInput): PostInput {
  if (!raw.title.trim()) throw new PostValidationError('标题不能为空');
  return {
    ...raw,
    title: raw.title.trim(),
    slug: raw.slug?.trim() || null,
    description: raw.description?.trim() || null,
    topImg: raw.topImg?.trim() || 'false',
    status: raw.status === 'published' ? 'published' : 'draft',
    tags: [...new Set(raw.tags.map((tag) => tag.trim()).filter(Boolean))],
  };
}

function hashFor(input: PostInput, categoryId: number | null, postDate: string): string {
  return contentHash({
    title: input.title,
    contentMd: input.contentMd,
    categoryId,
    tags: input.tags,
    description: input.description ?? null,
    topImg: input.topImg ?? 'false',
    postDate,
  });
}

export function createPost(db: Db, raw: PostInput): string {
  const input = normalize(raw);
  const categoryId = input.categoryId ?? null;
  const slug = uniqueSlug(slugify(input.slug ?? input.title), slugsInCategory(db, categoryId));
  const folder = safeFolder(categoryName(db, categoryId));
  const sourcePath = uniquePath(`${folder}/${slug}.md`, allSourcePaths(db));
  const abbrlink = newAbbrlink(abbrlinkSet(db));
  const now = nowStamp();
  const postDate = formatStamp(input.postDate) ?? now;
  const publishedAt = input.status === 'published' ? now : null;
  const id = newId();

  db.insert(posts)
    .values({
      id,
      title: input.title,
      slug,
      categoryId,
      description: input.description ?? null,
      topImg: input.topImg ?? 'false',
      contentMd: input.contentMd,
      status: input.status,
      abbrlink,
      sourcePath,
      postDate,
      createdAt: now,
      updatedAt: now,
      publishedAt,
      contentHash: hashFor(input, categoryId, postDate),
    })
    .run();
  setPostTags(db, id, input.tags);
  return id;
}

export function updatePost(db: Db, id: string, raw: PostInput): void {
  const existing = db.select().from(posts).where(eq(posts.id, id)).get();
  if (!existing) throw new PostNotFoundError('文章不存在');

  const input = normalize(raw);
  const categoryId = input.categoryId ?? null;
  const categoryChanged = existing.categoryId !== categoryId;
  const requestedSlug = slugify(input.slug ?? existing.slug);
  const slug = uniqueSlug(requestedSlug, slugsInCategory(db, categoryId, id));

  let sourcePath = existing.sourcePath;
  if (categoryChanged || slug !== existing.slug) {
    const folder = safeFolder(categoryName(db, categoryId));
    sourcePath = uniquePath(`${folder}/${slug}.md`, allSourcePaths(db, id));
  }

  const now = nowStamp();
  const postDate = formatStamp(input.postDate) ?? existing.postDate;
  const publishedAt =
    existing.publishedAt ?? (input.status === 'published' ? now : null);

  db.update(posts)
    .set({
      title: input.title,
      slug,
      categoryId,
      description: input.description ?? null,
      topImg: input.topImg ?? 'false',
      contentMd: input.contentMd,
      status: input.status,
      sourcePath,
      postDate,
      updatedAt: now,
      publishedAt,
      contentHash: hashFor(input, categoryId, postDate),
    })
    .where(eq(posts.id, id))
    .run();
  setPostTags(db, id, input.tags);
}

export function getPost(db: Db, id: string) {
  const row = db
    .select({
      id: posts.id,
      title: posts.title,
      slug: posts.slug,
      categoryId: posts.categoryId,
      categoryName: categories.name,
      description: posts.description,
      topImg: posts.topImg,
      contentMd: posts.contentMd,
      status: posts.status,
      abbrlink: posts.abbrlink,
      sourcePath: posts.sourcePath,
      postDate: posts.postDate,
      createdAt: posts.createdAt,
      updatedAt: posts.updatedAt,
      publishedAt: posts.publishedAt,
      deletedAt: posts.deletedAt,
      contentHash: posts.contentHash,
    })
    .from(posts)
    .leftJoin(categories, eq(posts.categoryId, categories.id))
    .where(eq(posts.id, id))
    .get();

  if (!row) return null;
  const tagRows = db
    .select({ name: tags.name })
    .from(postTags)
    .innerJoin(tags, eq(postTags.tagId, tags.id))
    .where(eq(postTags.postId, id))
    .orderBy(tags.name)
    .all();
  return { ...row, categoryName: row.categoryName ?? '未分类', tags: tagRows.map((tag) => tag.name) };
}

export function listPosts(db: Db, filters: PostListFilters) {
  const page = Math.max(1, filters.page ?? 1);
  const pageSize = Math.min(100, Math.max(1, filters.pageSize ?? 20));
  const conditions: SQL[] = [];

  if (filters.status === 'trash') {
    conditions.push(isNotNull(posts.deletedAt));
  } else if (filters.status === 'all') {
    conditions.push(isNull(posts.deletedAt));
  } else {
    conditions.push(isNull(posts.deletedAt), eq(posts.status, filters.status));
  }
  if (filters.categoryId != null) conditions.push(eq(posts.categoryId, filters.categoryId));
  if (filters.q?.trim()) conditions.push(like(posts.title, `%${filters.q.trim()}%`));

  const where = and(...conditions);
  const total =
    db.select({ n: sql<number>`count(*)` }).from(posts).where(where).get()?.n ?? 0;

  const rows = db
    .select({
      id: posts.id,
      title: posts.title,
      slug: posts.slug,
      categoryName: categories.name,
      status: posts.status,
      sourcePath: posts.sourcePath,
      abbrlink: posts.abbrlink,
      updatedAt: posts.updatedAt,
      deletedAt: posts.deletedAt,
    })
    .from(posts)
    .leftJoin(categories, eq(posts.categoryId, categories.id))
    .where(where)
    .orderBy(desc(posts.updatedAt))
    .limit(pageSize)
    .offset((page - 1) * pageSize)
    .all();

  const tagMap = new Map<string, string[]>();
  if (rows.length > 0) {
    const tagRows = db
      .select({ postId: postTags.postId, name: tags.name })
      .from(postTags)
      .innerJoin(tags, eq(postTags.tagId, tags.id))
      .where(inArray(postTags.postId, rows.map((row) => row.id)))
      .orderBy(tags.name)
      .all();
    for (const row of tagRows) {
      const list = tagMap.get(row.postId) ?? [];
      list.push(row.name);
      tagMap.set(row.postId, list);
    }
  }

  return {
    items: rows.map((row) => ({
      ...row,
      categoryName: row.categoryName ?? '未分类',
      tags: tagMap.get(row.id) ?? [],
    })),
    total,
    page,
    pageSize,
    totalPages: Math.max(1, Math.ceil(total / pageSize)),
  };
}

export function softDeletePost(db: Db, id: string): void {
  const now = nowStamp();
  const result = db
    .update(posts)
    .set({ deletedAt: now, updatedAt: now })
    .where(eq(posts.id, id))
    .run();
  if (result.changes === 0) throw new PostNotFoundError('文章不存在');
}

export function restorePost(db: Db, id: string): void {
  const result = db
    .update(posts)
    .set({ deletedAt: null, updatedAt: nowStamp() })
    .where(eq(posts.id, id))
    .run();
  if (result.changes === 0) throw new PostNotFoundError('文章不存在');
}

export function purgePost(db: Db, id: string): void {
  const result = db.delete(posts).where(eq(posts.id, id)).run();
  if (result.changes === 0) throw new PostNotFoundError('文章不存在');
}

export function publishPost(db: Db, id: string): void {
  const post = db.select().from(posts).where(eq(posts.id, id)).get();
  if (!post) throw new PostNotFoundError('文章不存在');
  db.update(posts)
    .set({
      status: 'published',
      publishedAt: post.publishedAt ?? nowStamp(),
      updatedAt: nowStamp(),
    })
    .where(eq(posts.id, id))
    .run();
}

export function unpublishPost(db: Db, id: string): void {
  const result = db
    .update(posts)
    .set({ status: 'draft', updatedAt: nowStamp() })
    .where(eq(posts.id, id))
    .run();
  if (result.changes === 0) throw new PostNotFoundError('文章不存在');
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/posts.test.ts
```

Expected: 7 tests pass.

- [ ] **Step 5: Typecheck and commit**

```bash
npx tsc -p tsconfig.json --noEmit
git add src/lib/posts.ts test/posts.test.ts
git commit -m "feat: add post service"
```

Expected: tsc exit 0; commit created.

`conditions` is explicitly typed as `SQL[]`; do not weaken strictness if TypeScript still complains. Fix the query typing instead.

---

## Task 5: Navigation, notices, and table styles

**Files:**
- Modify: `views/partials/header.ejs`
- Create: `views/partials/notice.ejs`
- Modify: `views/dashboard.ejs`
- Modify: `public/admin.css`

- [ ] **Step 1: Add nav links to `views/partials/header.ejs`**

Inside the `<div class="nav-links">`, before the username, add:

```ejs
    <a href="/posts">文章</a>
    <a href="/categories">分类</a>
    <a href="/tags">标签</a>
```

The final block should read:

```ejs
  <div class="nav-links">
    <a href="/">仪表盘</a>
    <a href="/posts">文章</a>
    <a href="/categories">分类</a>
    <a href="/tags">标签</a>
    <span class="who"><%= user.username %></span>
    <form method="post" action="/logout" class="inline">
      <input type="hidden" name="_csrf" value="<%= csrfToken %>">
      <button class="link-button" type="submit">退出</button>
    </form>
  </div>
```

- [ ] **Step 2: Create `views/partials/notice.ejs`**

```ejs
<% if (typeof notice !== 'undefined' && notice) { %>
  <p class="notice"><%= ({
    created: '已创建',
    saved: '已保存',
    deleted: '已移入回收站',
    restored: '已恢复',
    purged: '已彻底删除',
    published: '已发布',
    unpublished: '已取消发布'
  })[notice] || notice %></p>
<% } %>
<% if (typeof error !== 'undefined' && error) { %>
  <p class="error"><%= ({
    'category-in-use': '该分类下还有文章，无法删除',
    'tag-in-use': '该标签仍被文章使用，无法删除',
    duplicate: '名称已存在',
    invalid: '请检查输入内容',
    'not-found': '文章不存在'
  })[error] || error %></p>
<% } %>
```

Also update `views/dashboard.ejs`: replace

```ejs
<p class="muted">内容管理将在下一步接入。</p>
```

with

```ejs
<p class="muted"><a href="/posts">前往文章管理</a></p>
```

- [ ] **Step 3: Append table/form styles to `public/admin.css`**

Append this block to the end of `public/admin.css`:

```css
.toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: end;
  gap: 0.75rem;
  margin: 1rem 0 1.25rem;
}
.toolbar .grow { flex: 1 1 220px; }
.toolbar label { margin: 0; }
.toolbar input,
.toolbar select {
  display: block;
  width: 100%;
  margin-top: 0.25rem;
  padding: 0.5rem 0.6rem;
  border: 1px solid var(--line);
  border-radius: 8px;
  font: inherit;
  background: #fff;
}
.table {
  width: 100%;
  margin-top: 1rem;
  border-collapse: collapse;
  background: var(--card);
  border: 1px solid var(--line);
  border-radius: 10px;
  overflow: hidden;
}
.table th,
.table td {
  padding: 0.65rem 0.75rem;
  border-bottom: 1px solid var(--line);
  text-align: left;
  vertical-align: top;
}
.table th { font-size: 0.85rem; color: var(--muted); font-weight: 600; }
.table tr:last-child td { border-bottom: 0; }
.table .actions { display: flex; flex-wrap: wrap; gap: 0.5rem; }
.table .actions form { display: inline; }
.badge {
  display: inline-block;
  padding: 0.1rem 0.45rem;
  border-radius: 999px;
  font-size: 0.78rem;
  background: #eef2ff;
  color: #3730a3;
}
.badge.draft { background: #fef3c7; color: #92400e; }
.badge.trash { background: #fee2e2; color: #991b1b; }
.notice {
  color: #065f46;
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 0.5rem 0.75rem;
}
.form-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0 1rem;
}
.form-grid .full { grid-column: 1 / -1; }
textarea {
  display: block;
  width: 100%;
  min-height: 320px;
  margin-top: 0.35rem;
  padding: 0.75rem;
  border: 1px solid var(--line);
  border-radius: 8px;
  font: 0.95rem/1.5 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  resize: vertical;
}
.pagination {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 1rem;
  color: var(--muted);
}
.inline-form { display: flex; gap: 0.4rem; align-items: center; }
.inline-form input {
  padding: 0.35rem 0.5rem;
  border: 1px solid var(--line);
  border-radius: 6px;
  font: inherit;
}
.danger { background: var(--danger); }
.danger:hover { background: #b91c1c; }
.danger-text { color: var(--danger); }
.button {
  display: inline-block;
  padding: 0.55rem 1.1rem;
  border-radius: 8px;
  background: var(--accent);
  color: #fff;
  text-decoration: none;
}
.button:hover { background: #1d4ed8; text-decoration: none; }
@media (max-width: 720px) {
  .form-grid { grid-template-columns: 1fr; }
  .table { display: block; overflow-x: auto; }
}
```

- [ ] **Step 4: Commit**

```bash
git add views/partials/header.ejs views/partials/notice.ejs views/dashboard.ejs public/admin.css
git commit -m "feat: add content navigation and table styles"
```

---

## Task 6: Category and tag routes/pages (TDD)

**Files:**
- Create: `src/routes/categories.ts`
- Create: `src/routes/tags.ts`
- Create: `views/categories.ejs`
- Create: `views/tags.ejs`
- Modify: `src/app.ts`
- Test: `test/categories-tags-routes.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import { eq } from 'drizzle-orm';
import { categories, posts, postTags, tags } from '../src/db/schema';
import { makeTestApp, login, type TestApp } from './helpers';

let current: TestApp | null = null;
afterEach(async () => {
  if (current) await current.cleanup();
  current = null;
});

async function setup() {
  current = await makeTestApp();
  const { cookie, csrf } = await login(current.app);
  return { ...current, cookie, csrf };
}

describe('category and tag routes', () => {
  it('requires auth', async () => {
    current = await makeTestApp();
    const res = await current.app.inject({ method: 'GET', url: '/categories' });
    expect(res.statusCode).toBe(302);
    expect(res.headers.location).toBe('/login');
  });

  it('creates, renames, and blocks deletion of a used category', async () => {
    const { app, db, cookie, csrf } = await setup();
    const create = await app.inject({
      method: 'POST', url: '/categories',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}&name=${encodeURIComponent('网安')}`,
    });
    expect(create.statusCode).toBe(302);

    const category = db.select().from(categories).where(eq(categories.name, '网安')).get()!;
    const now = new Date().toISOString();
    db.insert(posts).values({
      id: 'p1', title: 'T', slug: 't', categoryId: category.id, topImg: 'false',
      contentMd: '', status: 'published', abbrlink: 'abc12345', sourcePath: '网安/t.md',
      postDate: now, createdAt: now, updatedAt: now,
    }).run();

    const del = await app.inject({
      method: 'POST', url: `/categories/${category.id}/delete`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(del.statusCode).toBe(302);
    expect(del.headers.location).toContain('error=category-in-use');
    expect(db.select().from(categories).where(eq(categories.id, category.id)).get()).toBeTruthy();

    const rename = await app.inject({
      method: 'POST', url: `/categories/${category.id}`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}&name=${encodeURIComponent('安全')}`,
    });
    expect(rename.statusCode).toBe(302);
    expect(db.select().from(categories).where(eq(categories.id, category.id)).get()!.name).toBe('安全');
  });

  it('creates a tag and blocks deleting it when used', async () => {
    const { app, db, cookie, csrf } = await setup();
    await app.inject({
      method: 'POST', url: '/tags',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}&name=${encodeURIComponent('收集')}`,
    });
    const tag = db.select().from(tags).where(eq(tags.name, '收集')).get()!;
    const now = new Date().toISOString();
    db.insert(posts).values({
      id: 'p1', title: 'T', slug: 't', topImg: 'false', contentMd: '',
      status: 'published', abbrlink: 'abc12345', sourcePath: 't.md',
      postDate: now, createdAt: now, updatedAt: now,
    }).run();
    db.insert(postTags).values({ postId: 'p1', tagId: tag.id }).run();

    const del = await app.inject({
      method: 'POST', url: `/tags/${tag.id}/delete`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(del.statusCode).toBe(302);
    expect(del.headers.location).toContain('error=tag-in-use');
    expect(db.select().from(tags).where(eq(tags.id, tag.id)).get()).toBeTruthy();
  });

  it('rejects mutations without CSRF', async () => {
    const { app, cookie } = await setup();
    const res = await app.inject({
      method: 'POST', url: '/categories',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: 'name=test',
    });
    expect(res.statusCode).toBe(403);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/categories-tags-routes.test.ts
```

Expected: FAIL because `src/routes/categories.ts` and `tags.ts` do not exist / app does not register them.

- [ ] **Step 3: Implement `src/routes/categories.ts`**

```ts
import type { FastifyInstance } from 'fastify';
import type { Db } from '../db/client';
import {
  CategoryInUseError,
  DuplicateCategoryError,
  createCategory,
  deleteCategory,
  listCategories,
  renameCategory,
} from '../lib/categories';

export interface CategoryRouteOptions { db: Db; }

function parseId(value: string): number | null {
  const id = Number(value);
  return Number.isInteger(id) && id > 0 ? id : null;
}

export async function registerCategoryRoutes(
  app: FastifyInstance,
  { db }: CategoryRouteOptions,
): Promise<void> {
  app.get('/categories', async (request, reply) => {
    const query = request.query as Record<string, string | undefined>;
    return reply.view('categories.ejs', {
      title: '分类',
      user: request.user,
      csrfToken: request.csrfToken,
      categories: listCategories(db),
      notice: query.notice,
      error: query.error,
    });
  });

  app.post('/categories', async (request, reply) => {
    const body = request.body as { name?: string } | undefined;
    try {
      createCategory(db, body?.name ?? '');
    } catch (error) {
      if (error instanceof DuplicateCategoryError) return reply.redirect('/categories?error=duplicate');
      if (error instanceof Error) return reply.redirect('/categories?error=invalid');
      throw error;
    }
    return reply.redirect('/categories?notice=created');
  });

  app.post('/categories/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    const categoryId = parseId(id);
    if (!categoryId) return reply.redirect('/categories?error=invalid');
    const body = request.body as { name?: string } | undefined;
    try {
      renameCategory(db, categoryId, body?.name ?? '');
    } catch (error) {
      if (error instanceof DuplicateCategoryError) return reply.redirect('/categories?error=duplicate');
      if (error instanceof Error) return reply.redirect('/categories?error=invalid');
      throw error;
    }
    return reply.redirect('/categories?notice=saved');
  });

  app.post('/categories/:id/delete', async (request, reply) => {
    const { id } = request.params as { id: string };
    const categoryId = parseId(id);
    if (!categoryId) return reply.redirect('/categories?error=invalid');
    try {
      deleteCategory(db, categoryId);
    } catch (error) {
      if (error instanceof CategoryInUseError) {
        return reply.redirect('/categories?error=category-in-use');
      }
      throw error;
    }
    return reply.redirect('/categories?notice=deleted');
  });
}
```

- [ ] **Step 4: Implement `src/routes/tags.ts`**

```ts
import type { FastifyInstance } from 'fastify';
import type { Db } from '../db/client';
import {
  DuplicateTagError,
  TagInUseError,
  createTag,
  deleteTag,
  listTags,
  renameTag,
} from '../lib/tags';

export interface TagRouteOptions { db: Db; }

function parseId(value: string): number | null {
  const id = Number(value);
  return Number.isInteger(id) && id > 0 ? id : null;
}

export async function registerTagRoutes(
  app: FastifyInstance,
  { db }: TagRouteOptions,
): Promise<void> {
  app.get('/tags', async (request, reply) => {
    const query = request.query as Record<string, string | undefined>;
    return reply.view('tags.ejs', {
      title: '标签',
      user: request.user,
      csrfToken: request.csrfToken,
      tags: listTags(db),
      notice: query.notice,
      error: query.error,
    });
  });

  app.post('/tags', async (request, reply) => {
    const body = request.body as { name?: string } | undefined;
    try {
      createTag(db, body?.name ?? '');
    } catch (error) {
      if (error instanceof DuplicateTagError) return reply.redirect('/tags?error=duplicate');
      if (error instanceof Error) return reply.redirect('/tags?error=invalid');
      throw error;
    }
    return reply.redirect('/tags?notice=created');
  });

  app.post('/tags/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    const tagId = parseId(id);
    if (!tagId) return reply.redirect('/tags?error=invalid');
    const body = request.body as { name?: string } | undefined;
    try {
      renameTag(db, tagId, body?.name ?? '');
    } catch (error) {
      if (error instanceof DuplicateTagError) return reply.redirect('/tags?error=duplicate');
      if (error instanceof Error) return reply.redirect('/tags?error=invalid');
      throw error;
    }
    return reply.redirect('/tags?notice=saved');
  });

  app.post('/tags/:id/delete', async (request, reply) => {
    const { id } = request.params as { id: string };
    const tagId = parseId(id);
    if (!tagId) return reply.redirect('/tags?error=invalid');
    try {
      deleteTag(db, tagId);
    } catch (error) {
      if (error instanceof TagInUseError) return reply.redirect('/tags?error=tag-in-use');
      throw error;
    }
    return reply.redirect('/tags?notice=deleted');
  });
}
```

- [ ] **Step 5: Create `views/categories.ejs`**

```ejs
<%- include('partials/header', { title: '分类', user, csrfToken }) %>
<%- include('partials/notice', { notice, error }) %>
<h1>分类</h1>
<section class="card">
  <form method="post" action="/categories" class="inline-form">
    <input type="hidden" name="_csrf" value="<%= csrfToken %>">
    <input name="name" placeholder="新分类名称" required>
    <button type="submit">新建分类</button>
  </form>
</section>
<table class="table">
  <thead>
    <tr><th>名称</th><th>文章数</th><th>操作</th></tr>
  </thead>
  <tbody>
    <% for (const category of categories) { %>
      <tr>
        <td>
          <form method="post" action="/categories/<%= category.id %>" class="inline-form">
            <input type="hidden" name="_csrf" value="<%= csrfToken %>">
            <input name="name" value="<%= category.name %>" required>
            <button type="submit">保存</button>
          </form>
        </td>
        <td><%= category.postCount %></td>
        <td>
          <form method="post" action="/categories/<%= category.id %>/delete">
            <input type="hidden" name="_csrf" value="<%= csrfToken %>">
            <button type="submit" class="danger">删除</button>
          </form>
        </td>
      </tr>
    <% } %>
    <% if (categories.length === 0) { %>
      <tr><td colspan="3" class="muted">还没有分类。</td></tr>
    <% } %>
  </tbody>
</table>
<%- include('partials/footer') %>
```

- [ ] **Step 6: Create `views/tags.ejs`**

```ejs
<%- include('partials/header', { title: '标签', user, csrfToken }) %>
<%- include('partials/notice', { notice, error }) %>
<h1>标签</h1>
<section class="card">
  <form method="post" action="/tags" class="inline-form">
    <input type="hidden" name="_csrf" value="<%= csrfToken %>">
    <input name="name" placeholder="新标签名称" required>
    <button type="submit">新建标签</button>
  </form>
</section>
<table class="table">
  <thead>
    <tr><th>名称</th><th>文章数</th><th>操作</th></tr>
  </thead>
  <tbody>
    <% for (const tag of tags) { %>
      <tr>
        <td>
          <form method="post" action="/tags/<%= tag.id %>" class="inline-form">
            <input type="hidden" name="_csrf" value="<%= csrfToken %>">
            <input name="name" value="<%= tag.name %>" required>
            <button type="submit">保存</button>
          </form>
        </td>
        <td><%= tag.postCount %></td>
        <td>
          <form method="post" action="/tags/<%= tag.id %>/delete">
            <input type="hidden" name="_csrf" value="<%= csrfToken %>">
            <button type="submit" class="danger">删除</button>
          </form>
        </td>
      </tr>
    <% } %>
    <% if (tags.length === 0) { %>
      <tr><td colspan="3" class="muted">还没有标签。</td></tr>
    <% } %>
  </tbody>
</table>
<%- include('partials/footer') %>
```

Note: the views intentionally use no inline handlers or inline styles, so they remain valid under the strict CSP (`script-src 'self'`, `style-src 'self'`). Deletion is a normal POST.

- [ ] **Step 7: Register the routes in `src/app.ts`**

Add imports after the dashboard import:

```ts
import { registerCategoryRoutes } from './routes/categories';
import { registerTagRoutes } from './routes/tags';
```

And after the dashboard registration:

```ts
  await registerCategoryRoutes(app, { db });
  await registerTagRoutes(app, { db });
```

- [ ] **Step 8: Run the test to verify it passes**

```bash
npm test -- test/categories-tags-routes.test.ts
```

Expected: 4 tests pass.

- [ ] **Step 9: Run the full suite, typecheck, and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/routes/categories.ts src/routes/tags.ts src/app.ts views/categories.ejs views/tags.ejs test/categories-tags-routes.test.ts
git commit -m "feat: add category and tag management pages"
```

Expected: all tests pass, tsc exit 0, commit created.

---

## Task 7: Post list, editor, and actions (TDD)

**Files:**
- Create: `src/routes/posts.ts`
- Create: `views/posts/index.ejs`
- Create: `views/posts/form.ejs`
- Modify: `src/app.ts`
- Test: `test/posts-routes.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import { eq } from 'drizzle-orm';
import { posts } from '../src/db/schema';
import { createCategory } from '../src/lib/categories';
import { makeTestApp, login, type TestApp } from './helpers';

let current: TestApp | null = null;
afterEach(async () => {
  if (current) await current.cleanup();
  current = null;
});

async function setup() {
  current = await makeTestApp();
  const { cookie, csrf } = await login(current.app);
  return { ...current, cookie, csrf };
}

function form(csrf: string, fields: Record<string, string>): string {
  const params = new URLSearchParams({ _csrf: csrf, ...fields });
  return params.toString();
}

describe('post routes', () => {
  it('requires auth for the list', async () => {
    current = await makeTestApp();
    const res = await current.app.inject({ method: 'GET', url: '/posts' });
    expect(res.statusCode).toBe(302);
    expect(res.headers.location).toBe('/login');
  });

  it('creates a post and shows it in the list', async () => {
    const { app, db, cookie, csrf } = await setup();
    const category = createCategory(db, '网安');
    const res = await app.inject({
      method: 'POST', url: '/posts',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: form(csrf, {
        title: '信息收集',
        category_id: String(category.id),
        tags: '收集, 谷歌',
        content_md: '# hello',
        status: 'draft',
        post_date: '2026-11-12 10:00:00',
      }),
    });
    expect(res.statusCode).toBe(302);
    const post = db.select().from(posts).where(eq(posts.title, '信息收集')).get()!;
    expect(post.sourcePath).toBe('网安/信息收集.md');

    const list = await app.inject({ method: 'GET', url: '/posts', headers: { cookie } });
    expect(list.statusCode).toBe(200);
    expect(list.body).toContain('信息收集');
  });

  it('edits a post and changes its status', async () => {
    const { app, db, cookie, csrf } = await setup();
    const create = await app.inject({
      method: 'POST', url: '/posts',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: form(csrf, { title: 'A', content_md: 'one', status: 'draft', post_date: '2026-11-12 10:00:00' }),
    });
    expect(create.statusCode).toBe(302);
    const post = db.select().from(posts).where(eq(posts.title, 'A')).get()!;

    const edit = await app.inject({
      method: 'POST', url: `/posts/${post.id}`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: form(csrf, { title: 'B', content_md: 'two', status: 'published', post_date: '2026-11-12 10:00:00' }),
    });
    expect(edit.statusCode).toBe(302);
    const updated = db.select().from(posts).where(eq(posts.id, post.id)).get()!;
    expect(updated.title).toBe('B');
    expect(updated.contentMd).toBe('two');
    expect(updated.status).toBe('published');
    expect(updated.publishedAt).not.toBeNull();
  });

  it('soft deletes, restores, and purges', async () => {
    const { app, db, cookie, csrf } = await setup();
    await app.inject({
      method: 'POST', url: '/posts',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: form(csrf, { title: 'A', content_md: '', status: 'draft', post_date: '2026-11-12 10:00:00' }),
    });
    const post = db.select().from(posts).where(eq(posts.title, 'A')).get()!;

    const del = await app.inject({
      method: 'POST', url: `/posts/${post.id}/delete`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(del.statusCode).toBe(302);
    expect(db.select().from(posts).where(eq(posts.id, post.id)).get()!.deletedAt).not.toBeNull();

    const restore = await app.inject({
      method: 'POST', url: `/posts/${post.id}/restore`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(restore.statusCode).toBe(302);
    expect(db.select().from(posts).where(eq(posts.id, post.id)).get()!.deletedAt).toBeNull();

    const purge = await app.inject({
      method: 'POST', url: `/posts/${post.id}/purge`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(purge.statusCode).toBe(302);
    expect(db.select().from(posts).where(eq(posts.id, post.id)).get()).toBeUndefined();
  });

  it('rejects a post without a title', async () => {
    const { app, cookie, csrf } = await setup();
    const res = await app.inject({
      method: 'POST', url: '/posts',
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: form(csrf, { title: '   ', content_md: '', status: 'draft', post_date: '2026-11-12 10:00:00' }),
    });
    expect(res.statusCode).toBe(400);
    expect(res.body).toContain('标题不能为空');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/posts-routes.test.ts
```

Expected: FAIL because `src/routes/posts.ts` does not exist / app does not register it.

- [ ] **Step 3: Implement `src/routes/posts.ts`**

```ts
import type { FastifyInstance, FastifyReply, FastifyRequest } from 'fastify';
import type { Db } from '../db/client';
import { listCategories } from '../lib/categories';
import {
  PostNotFoundError,
  PostValidationError,
  createPost,
  getPost,
  listPosts,
  publishPost,
  purgePost,
  restorePost,
  softDeletePost,
  unpublishPost,
  updatePost,
  type PostInput,
  type PostListFilters,
} from '../lib/posts';
import { nowStamp } from '../lib/dates';

export interface PostRouteOptions { db: Db; }

type Query = Record<string, string | undefined>;

function parseId(value: string): string | null {
  return value && value.length > 0 ? value : null;
}

function parseForm(request: FastifyRequest): PostInput {
  const body = (request.body ?? {}) as Record<string, string | undefined>;
  const categoryRaw = body.category_id;
  const parsedCategoryId = categoryRaw ? Number(categoryRaw) : Number.NaN;
  return {
    title: (body.title ?? '').trim(),
    slug: (body.slug ?? '').trim() || null,
    categoryId: Number.isInteger(parsedCategoryId) && parsedCategoryId > 0 ? parsedCategoryId : null,
    description: (body.description ?? '').trim() || null,
    topImg: (body.top_img ?? '').trim() || 'false',
    contentMd: body.content_md ?? '',
    status: body.status === 'published' ? 'published' : 'draft',
    postDate: (body.post_date ?? '').trim() || null,
    tags: (body.tags ?? '').split(/[,，]/).map((tag) => tag.trim()).filter(Boolean),
  };
}

function parseFilters(query: Query): PostListFilters {
  const status = query.status;
  return {
    status:
      status === 'draft' || status === 'published' || status === 'trash' ? status : 'all',
    categoryId: query.category ? Number(query.category) || null : null,
    q: query.q?.trim() || undefined,
    page: Math.max(1, Number(query.page) || 1),
    pageSize: 20,
  };
}

function noticeFrom(query: Query): { notice?: string; error?: string } {
  return { notice: query.notice, error: query.error };
}

async function renderList(
  db: Db,
  request: FastifyRequest,
  reply: FastifyReply,
  extra: { error?: string; notice?: string } = {},
) {
  const query = request.query as Query;
  const filters = parseFilters(query);
  const result = listPosts(db, filters);
  const categories = listCategories(db);
  return reply.view('posts/index.ejs', {
    title: '文章',
    user: request.user,
    csrfToken: request.csrfToken,
    ...result,
    filters,
    categories,
    ...noticeFrom(query),
    ...extra,
  });
}

async function renderForm(
  db: Db,
  request: FastifyRequest,
  reply: FastifyReply,
  data: {
    post: PostInput & { id?: string; abbrlink?: string; tagsText?: string };
    isNew: boolean;
    error?: string;
  },
) {
  return reply.view('posts/form.ejs', {
    title: data.isNew ? '新建文章' : '编辑文章',
    user: request.user,
    csrfToken: request.csrfToken,
    categories: listCategories(db),
    ...data,
  });
}

export async function registerPostRoutes(
  app: FastifyInstance,
  { db }: PostRouteOptions,
): Promise<void> {
  app.get('/posts', async (request, reply) => renderList(db, request, reply));

  app.get('/posts/new', async (request, reply) =>
    renderForm(db, request, reply, {
      isNew: true,
      post: {
        title: '',
        slug: null,
        categoryId: null,
        description: null,
        topImg: 'false',
        contentMd: '',
        status: 'draft',
        postDate: nowStamp(),
        tags: [],
        tagsText: '',
      },
    }),
  );

  app.post('/posts', async (request, reply) => {
    const input = parseForm(request);
    try {
      createPost(db, input);
    } catch (error) {
      if (error instanceof PostValidationError) {
        reply.code(400);
        return renderForm(db, request, reply, {
          isNew: true,
          post: { ...input, tagsText: input.tags.join(', ') },
          error: error.message,
        });
      }
      throw error;
    }
    return reply.redirect('/posts?notice=created');
  });

  app.get('/posts/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    const post = getPost(db, parseId(id) ?? '');
    if (!post) return reply.callNotFound();
    return renderForm(db, request, reply, {
      isNew: false,
      post: {
        id: post.id,
        title: post.title,
        slug: post.slug,
        categoryId: post.categoryId,
        description: post.description,
        topImg: post.topImg,
        contentMd: post.contentMd,
        status: post.status,
        postDate: post.postDate,
        tags: post.tags,
        tagsText: post.tags.join(', '),
        abbrlink: post.abbrlink,
      },
    });
  });

  app.post('/posts/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    try {
      updatePost(db, parseId(id) ?? '', parseForm(request));
    } catch (error) {
      if (error instanceof PostNotFoundError) return reply.callNotFound();
      if (error instanceof PostValidationError) {
        const post = getPost(db, parseId(id) ?? '');
        if (!post) return reply.callNotFound();
        reply.code(400);
        return renderForm(db, request, reply, {
          isNew: false,
          post: { ...parseForm(request), id: post.id, abbrlink: post.abbrlink, tagsText: parseForm(request).tags.join(', ') },
          error: error.message,
        });
      }
      throw error;
    }
    return reply.redirect('/posts?notice=saved');
  });

  app.post('/posts/:id/delete', async (request, reply) => {
    try {
      softDeletePost(db, (request.params as { id: string }).id);
    } catch (error) {
      if (error instanceof PostNotFoundError) return reply.callNotFound();
      throw error;
    }
    return reply.redirect('/posts?notice=deleted');
  });

  app.post('/posts/:id/restore', async (request, reply) => {
    try {
      restorePost(db, (request.params as { id: string }).id);
    } catch (error) {
      if (error instanceof PostNotFoundError) return reply.callNotFound();
      throw error;
    }
    return reply.redirect('/posts?status=trash&notice=restored');
  });

  app.post('/posts/:id/purge', async (request, reply) => {
    try {
      purgePost(db, (request.params as { id: string }).id);
    } catch (error) {
      if (error instanceof PostNotFoundError) return reply.callNotFound();
      throw error;
    }
    return reply.redirect('/posts?status=trash&notice=purged');
  });

  app.post('/posts/:id/publish', async (request, reply) => {
    try {
      publishPost(db, (request.params as { id: string }).id);
    } catch (error) {
      if (error instanceof PostNotFoundError) return reply.callNotFound();
      throw error;
    }
    return reply.redirect('/posts?notice=published');
  });

  app.post('/posts/:id/unpublish', async (request, reply) => {
    try {
      unpublishPost(db, (request.params as { id: string }).id);
    } catch (error) {
      if (error instanceof PostNotFoundError) return reply.callNotFound();
      throw error;
    }
    return reply.redirect('/posts?notice=unpublished');
  });
}
```

- [ ] **Step 4: Create `views/posts/index.ejs`**

```ejs
<%- include('../partials/header', { title: '文章', user, csrfToken }) %>
<%- include('../partials/notice', { notice, error }) %>
<div class="toolbar">
  <div class="grow">
    <h1>文章</h1>
  </div>
  <a class="button" href="/posts/new">新建文章</a>
</div>
<form method="get" action="/posts" class="toolbar">
  <label>状态
    <select name="status">
      <option value="all" <%= filters.status === 'all' ? 'selected' : '' %>>全部</option>
      <option value="draft" <%= filters.status === 'draft' ? 'selected' : '' %>>草稿</option>
      <option value="published" <%= filters.status === 'published' ? 'selected' : '' %>>已发布</option>
      <option value="trash" <%= filters.status === 'trash' ? 'selected' : '' %>>回收站</option>
    </select>
  </label>
  <label>分类
    <select name="category">
      <option value="">全部分类</option>
      <% for (const category of categories) { %>
        <option value="<%= category.id %>" <%= filters.categoryId === category.id ? 'selected' : '' %>><%= category.name %></option>
      <% } %>
    </select>
  </label>
  <label class="grow">搜索
    <input name="q" value="<%= filters.q ?? '' %>" placeholder="按标题搜索">
  </label>
  <button type="submit">筛选</button>
</form>
<table class="table">
  <thead>
    <tr><th>标题</th><th>分类</th><th>标签</th><th>状态</th><th>更新时间</th><th>操作</th></tr>
  </thead>
  <tbody>
    <% for (const post of items) { %>
      <tr>
        <td><a href="/posts/<%= post.id %>"><%= post.title %></a></td>
        <td><%= post.categoryName %></td>
        <td><%= post.tags.join('、') %></td>
        <td>
          <% if (post.deletedAt) { %>
            <span class="badge trash">回收站</span>
          <% } else if (post.status === 'published') { %>
            <span class="badge">已发布</span>
          <% } else { %>
            <span class="badge draft">草稿</span>
          <% } %>
        </td>
        <td><%= post.updatedAt %></td>
        <td class="actions">
          <a href="/posts/<%= post.id %>">编辑</a>
          <% if (post.deletedAt) { %>
            <form method="post" action="/posts/<%= post.id %>/restore"><input type="hidden" name="_csrf" value="<%= csrfToken %>"><button class="link-button" type="submit">恢复</button></form>
            <form method="post" action="/posts/<%= post.id %>/purge"><input type="hidden" name="_csrf" value="<%= csrfToken %>"><button class="link-button danger-text" type="submit">彻底删除</button></form>
          <% } else { %>
            <% if (post.status === 'published') { %>
              <form method="post" action="/posts/<%= post.id %>/unpublish"><input type="hidden" name="_csrf" value="<%= csrfToken %>"><button class="link-button" type="submit">取消发布</button></form>
            <% } else { %>
              <form method="post" action="/posts/<%= post.id %>/publish"><input type="hidden" name="_csrf" value="<%= csrfToken %>"><button class="link-button" type="submit">发布</button></form>
            <% } %>
            <form method="post" action="/posts/<%= post.id %>/delete"><input type="hidden" name="_csrf" value="<%= csrfToken %>"><button class="link-button danger-text" type="submit">删除</button></form>
          <% } %>
        </td>
      </tr>
    <% } %>
    <% if (items.length === 0) { %>
      <tr><td colspan="6" class="muted">没有文章。</td></tr>
    <% } %>
  </tbody>
</table>
<div class="pagination">
  <span>共 <%= total %> 篇，第 <%= page %> / <%= totalPages %> 页</span>
  <% if (page > 1) { %>
    <a href="/posts?page=<%= page - 1 %>&status=<%= filters.status %>&category=<%= filters.categoryId ?? '' %>&q=<%= encodeURIComponent(filters.q ?? '') %>">上一页</a>
  <% } %>
  <% if (page < totalPages) { %>
    <a href="/posts?page=<%= page + 1 %>&status=<%= filters.status %>&category=<%= filters.categoryId ?? '' %>&q=<%= encodeURIComponent(filters.q ?? '') %>">下一页</a>
  <% } %>
</div>
<%- include('../partials/footer') %>
```

Add a `.danger-text` style to CSS:

```css
.danger-text { color: var(--danger); }
```

- [ ] **Step 5: Create `views/posts/form.ejs`**

```ejs
<%- include('../partials/header', { title: isNew ? '新建文章' : '编辑文章', user, csrfToken }) %>
<%- include('../partials/notice', { notice: undefined, error }) %>
<h1><%= isNew ? '新建文章' : '编辑文章' %></h1>
<% if (!isNew && post.abbrlink) { %>
  <p class="muted">永久链接：/posts/<%= post.abbrlink %>.html</p>
<% } %>
<form method="post" action="<%= isNew ? '/posts' : '/posts/' + post.id %>">
  <input type="hidden" name="_csrf" value="<%= csrfToken %>">
  <div class="form-grid">
    <label class="full">标题
      <input name="title" value="<%= post.title %>" required>
    </label>
    <label>Slug（留空自动生成）
      <input name="slug" value="<%= post.slug ?? '' %>">
    </label>
    <label>分类
      <select name="category_id">
        <option value="">未分类</option>
        <% for (const category of categories) { %>
          <option value="<%= category.id %>" <%= post.categoryId === category.id ? 'selected' : '' %>><%= category.name %></option>
        <% } %>
      </select>
    </label>
    <label>标签（逗号分隔）
      <input name="tags" value="<%= post.tagsText ?? post.tags.join(', ') %>">
    </label>
    <label>日期（YYYY-MM-DD HH:mm:ss）
      <input name="post_date" value="<%= post.postDate ?? '' %>">
    </label>
    <label>状态
      <select name="status">
        <option value="draft" <%= post.status === 'draft' ? 'selected' : '' %>>草稿</option>
        <option value="published" <%= post.status === 'published' ? 'selected' : '' %>>已发布</option>
      </select>
    </label>
    <label>顶部图（false 或 URL）
      <input name="top_img" value="<%= post.topImg ?? 'false' %>">
    </label>
    <label class="full">摘要
      <input name="description" value="<%= post.description ?? '' %>">
    </label>
    <label class="full">正文（Markdown）
      <textarea name="content_md"><%= post.contentMd %></textarea>
    </label>
  </div>
  <div class="toolbar">
    <button type="submit">保存</button>
    <a href="/posts">返回列表</a>
  </div>
</form>
<%- include('../partials/footer') %>
```

Note: the `form.ejs` include path is `../partials/header` because the view is under `views/posts/`. The plan's other views are under `views/` and use `partials/...`. EJS resolves relative to the view file. The paths above are correct.

Also add to CSS:

```css
select {
  display: block;
  width: 100%;
  margin-top: 0.35rem;
  padding: 0.55rem 0.7rem;
  border: 1px solid var(--line);
  border-radius: 8px;
  font: inherit;
  background: #fff;
}
```

- [ ] **Step 6: Register post routes in `src/app.ts`**

Add the import:

```ts
import { registerPostRoutes } from './routes/posts';
```

And after the tag registration:

```ts
  await registerPostRoutes(app, { db });
```

- [ ] **Step 7: Run the test to verify it passes**

```bash
npm test -- test/posts-routes.test.ts
```

Expected: 5 tests pass.

- [ ] **Step 8: Run the full suite, typecheck, and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/routes/posts.ts src/app.ts views/posts/index.ejs views/posts/form.ejs public/admin.css test/posts-routes.test.ts
git commit -m "feat: add post list, editor, and actions"
```

Expected: all tests pass, tsc exit 0, commit created.

If TypeScript complains about `conditions` in `src/lib/posts.ts`, apply the annotation from Task 4. If it complains about `reply.callNotFound()` returning `void` vs a reply, use `return reply.callNotFound();` as written; Fastify types accept it.

---

## Task 8: Final verification and README update

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Run the full suite and build**

```bash
npm test
npm run build
```

Expected: all tests pass; build exits 0.

- [ ] **Step 2: Smoke test the content pages**

Start the compiled server with a temporary DB exactly like Plan 2A (use a different port and DB path):

```bash
export DATABASE_PATH=/tmp/ihmod-content-smoke.db
export SESSION_SECRET=$(openssl rand -hex 32)
export ADMIN_USERNAME=admin
export ADMIN_PASSWORD=smoke-password-123
export HOST=127.0.0.1
export PORT=8097
export COOKIE_SECURE=false
rm -f "$DATABASE_PATH"
node dist/server.js > /tmp/ihmod-content-smoke.log 2>&1 &
PID=$!
sleep 2
curl -s http://127.0.0.1:8097/healthz
curl -s -c /tmp/ihmod-content-cookies.txt http://127.0.0.1:8097/login -o /tmp/ihmod-content-login.html
TOKEN=$(grep -o 'name="_csrf" value="[^"]*"' /tmp/ihmod-content-login.html | sed 's/.*value="//;s/"$//')
curl -s -o /dev/null -w '%{http_code}\n' -b /tmp/ihmod-content-cookies.txt -c /tmp/ihmod-content-cookies.txt \
  -d "_csrf=$TOKEN&username=admin&password=smoke-password-123" http://127.0.0.1:8097/login
for p in posts categories tags; do
  code=$(curl -s -o /dev/null -w '%{http_code}' -b /tmp/ihmod-content-cookies.txt "http://127.0.0.1:8097/$p")
  echo "$p: $code"
done
kill $PID
wait $PID 2>/dev/null || true
rm -f "$DATABASE_PATH" "$DATABASE_PATH"-wal "$DATABASE_PATH"-shm /tmp/ihmod-content-cookies.txt /tmp/ihmod-content-login.html /tmp/ihmod-content-smoke.log
```

Expected:
- `/healthz` → `{"ok":true}`
- login → `302`
- `posts: 200`
- `categories: 200`
- `tags: 200`

- [ ] **Step 3: Update `README.md`**

Add a "Content management" section:

```markdown
## Content management

- `/posts` — list, filter, search; create/edit with a Markdown textarea.
- `/categories` — create, rename, delete (blocked while posts reference it).
- `/tags` — create, rename, delete (blocked while posts reference it).
- Post states: draft, published, trash (soft delete with restore/purge).
- Slugs and `abbrlink` values are generated automatically; existing values are preserved.
- Posts store a stable `source_path` under the post's category; changing category/slug moves it.
```

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document content management"
```

---

## Acceptance Checklist

- [ ] `npm test` passes all Plan 2A + Plan 2B tests.
- [ ] `npm run build` exits 0.
- [ ] `/posts`, `/categories`, `/tags` require login and return 200 after login.
- [ ] Creating a post generates a slug, 8-hex `abbrlink`, `source_path`, content hash, and tags.
- [ ] Duplicate titles get `-2` slug suffixes; `source_path` is unique.
- [ ] Editing a post preserves `abbrlink`; changing category/slug moves `source_path`.
- [ ] Publish/unpublish updates status and `published_at`.
- [ ] Soft delete/restore/purge work; trash filter lists deleted posts.
- [ ] Deleting a category or tag that is in use is blocked with a friendly error.
- [ ] All mutations require CSRF; missing token returns 403.
- [ ] No secrets or runtime data are tracked.

## Next plans

- **2C:** importer for the 13 existing Markdown posts (fixtures, CLI, dry-run, real import).
- **2D:** Vditor editor, autosave, media upload/library.
- **2E:** Markdown export, publish manifest, Git push, systemd/backup/ufw deployment.
