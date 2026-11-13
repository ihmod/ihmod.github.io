# Blog Admin Editor & Media Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the post textarea with a Vditor Markdown editor, add debounced autosave for existing posts, and add an image media library with upload, serving, and referential deletion checks.

**Architecture:** Vditor is served locally from `node_modules/vditor/dist` at `/static/vditor/`. The editor posts images to `POST /api/media`, which validates magic bytes, stores files in `UPLOADS_DIR` (default `./data/uploads`), records them in SQLite, and returns `/img/uploads/<uuid>.<ext>` URLs. Those URLs are served by a protected static route in the admin and will later be copied into the blog repo by the publisher. Autosave is a small JSON/form endpoint that writes only when title/content actually changed. The CSP is relaxed for inline styles (Vditor requires it) while keeping `script-src 'self'`.

**Tech Stack:** Existing Plan 2A/2B/2C stack + `vditor`, `@fastify/multipart`.

---

## Prerequisites

Plans 2A, 2B, and 2C are complete in `/home/ihmod/cc_projects/IHMOD-admin`:

- Login, posts/categories/tags, importer, CLI.
- 65 tests pass.
- `data/admin.db` has 13 imported posts (local runtime data, gitignored).

Run:

```bash
cd /home/ihmod/cc_projects/IHMOD-admin
npm test
```

Expected: 16 files, 65 tests pass.

## File Structure

```
src/services/media.ts          # magic-byte validation, save/list/delete
src/routes/media.ts            # /api/media, /media, delete
src/public/editor.js           # Vditor init + autosave
src/public/media.js            # media page upload
src/app.ts                     # multipart, static Vditor/uploads, CSP, media routes
src/config.ts                  # UPLOADS_DIR
src/lib/posts.ts               # autosavePost
src/routes/posts.ts            # POST /api/posts/:id/autosave
views/media.ejs
views/partials/header.ejs      # media nav + conditional Vditor CSS
views/posts/form.ejs           # Vditor container + hidden content input
public/admin.css               # media grid + editor bits
test/helpers.ts                # expose uploadsDir, temp UPLOADS_DIR
test/app.test.ts               # temp UPLOADS_DIR
test/routes.test.ts            # temp UPLOADS_DIR
test/media.test.ts
test/media-routes.test.ts
test/autosave.test.ts
test/editor-render.test.ts
.env.example                   # UPLOADS_DIR
README.md
```

No database migration is needed; the `media` table exists from Plan 2A.

---

## Task 1: Dependencies, uploads config, and temp test directories

**Files:**
- Modify: `package.json` / `package-lock.json`
- Modify: `src/config.ts`
- Modify: `.env.example`
- Modify: `test/helpers.ts`
- Modify: `test/app.test.ts`
- Modify: `test/routes.test.ts`

- [ ] **Step 1: Install dependencies**

```bash
npm install vditor @fastify/multipart
```

Expected: both packages appear in `package.json` dependencies.

- [ ] **Step 2: Add `UPLOADS_DIR` to `src/config.ts`**

Add after `DATABASE_PATH`:

```ts
  UPLOADS_DIR: z.string().default('./data/uploads'),
```

- [ ] **Step 3: Add `UPLOADS_DIR` to `.env.example`**

```dotenv
UPLOADS_DIR=./data/uploads
```

- [ ] **Step 4: Update `test/helpers.ts` to use a temp uploads directory**

In `TestApp`, add `uploadsDir: string;`.

Inside `makeTestApp`, after `const dbPath = ...`:

```ts
  const uploadsDir = path.join(dir, 'uploads');
```

Add to the `loadConfig` object:

```ts
    UPLOADS_DIR: uploadsDir,
```

Return it:

```ts
  return {
    app,
    db,
    sqlite,
    dir,
    uploadsDir,
    cleanup: async () => { ... },
  };
```

- [ ] **Step 5: Update the two inline test configs**

In `test/app.test.ts` and `test/routes.test.ts`, inside their `makeApp`/`makeTestApp` `loadConfig({...})` calls, add:

```ts
    UPLOADS_DIR: path.join(dir, 'uploads'),
```

(`dir` is already defined in both helpers.)

- [ ] **Step 6: Verify**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
```

Expected: all tests pass; tsc exit 0.

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json src/config.ts .env.example test/helpers.ts test/app.test.ts test/routes.test.ts
git commit -m "feat: add vditor, multipart, and uploads config"
```

---

## Task 2: Media service (TDD)

**Files:**
- Create: `src/services/media.ts`
- Test: `test/media.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { media } from '../src/db/schema';
import { createPost } from '../src/lib/posts';
import {
  MediaInUseError,
  MediaValidationError,
  deleteMedia,
  detectImage,
  listMedia,
  saveUpload,
} from '../src/services/media';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-media-'));
  const uploadsDir = path.join(dir, 'uploads');
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  return { db, uploadsDir };
}

const png = Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0, 0, 0, 0]);
const jpeg = Buffer.from([0xff, 0xd8, 0xff, 0x00]);
const gif = Buffer.from('GIF89a');
const webp = Buffer.from('RIFF0000WEBP');

describe('detectImage', () => {
  it('detects supported formats', () => {
    expect(detectImage(png)?.mime).toBe('image/png');
    expect(detectImage(jpeg)?.mime).toBe('image/jpeg');
    expect(detectImage(gif)?.mime).toBe('image/gif');
    expect(detectImage(webp)?.mime).toBe('image/webp');
  });

  it('rejects unknown formats', () => {
    expect(detectImage(Buffer.from('not an image'))).toBeNull();
  });
});

describe('media service', () => {
  it('writes the file and records it', () => {
    const { db, uploadsDir } = setup();
    const saved = saveUpload(db, uploadsDir, { filename: 'x.png', buffer: png });
    expect(saved.storedName).toMatch(/\.png$/);
    expect(saved.url).toBe(`/img/uploads/${saved.storedName}`);
    expect(fs.existsSync(path.join(uploadsDir, saved.storedName))).toBe(true);

    const row = db.select().from(media).where(eq(media.id, saved.id)).get()!;
    expect(row.mime).toBe('image/png');
    expect(row.size).toBe(png.length);
    expect(listMedia(db).map((item) => item.id)).toContain(saved.id);
  });

  it('rejects unsupported data', () => {
    const { db, uploadsDir } = setup();
    expect(() =>
      saveUpload(db, uploadsDir, { filename: 'x.txt', buffer: Buffer.from('nope') }),
    ).toThrow(MediaValidationError);
  });

  it('deletes unused media and its file', () => {
    const { db, uploadsDir } = setup();
    const saved = saveUpload(db, uploadsDir, { filename: 'x.png', buffer: png });
    deleteMedia(db, uploadsDir, saved.id);
    expect(db.select().from(media).where(eq(media.id, saved.id)).get()).toBeUndefined();
    expect(fs.existsSync(path.join(uploadsDir, saved.storedName))).toBe(false);
  });

  it('blocks deletion while a post references the image', () => {
    const { db, uploadsDir } = setup();
    const saved = saveUpload(db, uploadsDir, { filename: 'x.png', buffer: png });
    createPost(db, {
      title: 'uses image',
      categoryId: null,
      contentMd: `![x](${saved.url})`,
      status: 'draft',
      postDate: '2026-11-12 10:00:00',
      tags: [],
    });
    expect(() => deleteMedia(db, uploadsDir, saved.id)).toThrow(MediaInUseError);
    expect(fs.existsSync(path.join(uploadsDir, saved.storedName))).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/media.test.ts
```

Expected: FAIL because `src/services/media.ts` does not exist.

- [ ] **Step 3: Implement `src/services/media.ts`**

```ts
import fs from 'node:fs';
import path from 'node:path';
import { desc, eq, like, sql } from 'drizzle-orm';
import type { Db } from '../db/client';
import { media, posts } from '../db/schema';
import { newId } from '../lib/ids';
import { nowStamp } from '../lib/dates';

export class MediaValidationError extends Error {}
export class MediaInUseError extends Error {}
export class MediaNotFoundError extends Error {}

interface ImageType {
  mime: string;
  ext: string;
  matches(buffer: Buffer): boolean;
}

const IMAGE_TYPES: ImageType[] = [
  {
    mime: 'image/png',
    ext: 'png',
    matches: (buffer) =>
      buffer.length >= 8 &&
      buffer.subarray(0, 8).equals(Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a])),
  },
  {
    mime: 'image/jpeg',
    ext: 'jpg',
    matches: (buffer) =>
      buffer.length >= 3 && buffer[0] === 0xff && buffer[1] === 0xd8 && buffer[2] === 0xff,
  },
  {
    mime: 'image/gif',
    ext: 'gif',
    matches: (buffer) =>
      buffer.length >= 6 &&
      ['GIF87a', 'GIF89a'].includes(buffer.subarray(0, 6).toString('ascii')),
  },
  {
    mime: 'image/webp',
    ext: 'webp',
    matches: (buffer) =>
      buffer.length >= 12 &&
      buffer.subarray(0, 4).toString('ascii') === 'RIFF' &&
      buffer.subarray(8, 12).toString('ascii') === 'WEBP',
  },
];

export function detectImage(buffer: Buffer): ImageType | null {
  return IMAGE_TYPES.find((type) => type.matches(buffer)) ?? null;
}

export function ensureUploadsDir(uploadsDir: string): void {
  fs.mkdirSync(uploadsDir, { recursive: true });
}

export interface SavedUpload {
  id: string;
  storedName: string;
  url: string;
}

export function saveUpload(
  db: Db,
  uploadsDir: string,
  file: { filename: string; buffer: Buffer },
): SavedUpload {
  if (file.buffer.length > 10 * 1024 * 1024) {
    throw new MediaValidationError('文件超过 10MB');
  }
  const type = detectImage(file.buffer);
  if (!type) throw new MediaValidationError('不支持的图片格式');

  ensureUploadsDir(uploadsDir);
  const id = newId();
  const storedName = `${id}.${type.ext}`;
  const target = path.join(uploadsDir, storedName);
  fs.writeFileSync(target, file.buffer);

  try {
    db.insert(media)
      .values({
        id,
        originalName: file.filename.slice(0, 255),
        storedName,
        mime: type.mime,
        size: file.buffer.length,
        createdAt: nowStamp(),
      })
      .run();
  } catch (error) {
    fs.rmSync(target, { force: true });
    throw error;
  }

  return { id, storedName, url: `/img/uploads/${storedName}` };
}

export function listMedia(db: Db) {
  return db.select().from(media).orderBy(desc(media.createdAt)).all();
}

export function deleteMedia(db: Db, uploadsDir: string, id: string): void {
  const row = db.select().from(media).where(eq(media.id, id)).get();
  if (!row) throw new MediaNotFoundError('媒体不存在');

  const used =
    db
      .select({ n: sql<number>`count(*)` })
      .from(posts)
      .where(like(posts.contentMd, `%/img/uploads/${row.storedName}%`))
      .get()?.n ?? 0;
  if (used > 0) throw new MediaInUseError('图片仍被文章引用');

  db.delete(media).where(eq(media.id, id)).run();
  fs.rmSync(path.join(uploadsDir, row.storedName), { force: true });
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/media.test.ts
```

Expected: 6 tests pass.

- [ ] **Step 5: Typecheck, run the full suite, and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/services/media.ts test/media.test.ts
git commit -m "feat: add media service"
```

---

## Task 3: Media routes, static serving, page, and upload script (TDD)

**Files:**
- Create: `src/routes/media.ts`
- Create: `src/public/media.js`
- Create: `views/media.ejs`
- Modify: `src/app.ts`
- Modify: `views/partials/header.ejs`
- Modify: `public/admin.css`
- Test: `test/media-routes.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import { media } from '../src/db/schema';
import { createPost } from '../src/lib/posts';
import { makeTestApp, login, type TestApp } from './helpers';

let current: TestApp | null = null;
afterEach(async () => {
  if (current) await current.cleanup();
  current = null;
});

const png = Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0, 0, 0, 0]);

function multipartBody(boundary: string, buffer: Buffer): Buffer {
  return Buffer.concat([
    Buffer.from(
      `--${boundary}\r\nContent-Disposition: form-data; name="file"; filename="x.png"\r\nContent-Type: image/png\r\n\r\n`,
    ),
    buffer,
    Buffer.from(`\r\n--${boundary}--\r\n`),
  ]);
}

async function setup() {
  current = await makeTestApp();
  const { cookie, csrf } = await login(current.app);
  return { ...current, cookie, csrf };
}

describe('media routes', () => {
  it('requires auth for the media page', async () => {
    current = await makeTestApp();
    const res = await current.app.inject({ method: 'GET', url: '/media' });
    expect(res.statusCode).toBe(302);
    expect(res.headers.location).toBe('/login');
  });

  it('uploads, serves, and deletes an image', async () => {
    const { app, db, uploadsDir, cookie, csrf } = await setup();
    const boundary = '----ihmodtest';
    const upload = await app.inject({
      method: 'POST',
      url: '/api/media',
      headers: {
        cookie,
        'x-csrf-token': csrf,
        'content-type': `multipart/form-data; boundary=${boundary}`,
      },
      payload: multipartBody(boundary, png),
    });
    expect(upload.statusCode).toBe(200);
    const body = upload.json() as { name: string; url: string };
    expect(body.url).toBe(`/img/uploads/${body.name}`);
    expect(fs.existsSync(path.join(uploadsDir, body.name))).toBe(true);

    const served = await app.inject({
      method: 'GET',
      url: `/img/uploads/${body.name}`,
      headers: { cookie },
    });
    expect(served.statusCode).toBe(200);
    expect(served.headers['content-type']).toContain('image/png');

    const anonymous = await app.inject({
      method: 'GET',
      url: `/img/uploads/${body.name}`,
    });
    expect(anonymous.statusCode).toBe(302);

    const row = db.select().from(media).where(eq(media.storedName, body.name)).get()!;
    const del = await app.inject({
      method: 'POST',
      url: `/media/${row.id}/delete`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(del.statusCode).toBe(302);
    expect(fs.existsSync(path.join(uploadsDir, body.name))).toBe(false);
  });

  it('blocks deletion when a post references the image', async () => {
    const { app, db, uploadsDir, cookie, csrf } = await setup();
    const boundary = '----ihmodtest';
    const upload = await app.inject({
      method: 'POST',
      url: '/api/media',
      headers: {
        cookie,
        'x-csrf-token': csrf,
        'content-type': `multipart/form-data; boundary=${boundary}`,
      },
      payload: multipartBody(boundary, png),
    });
    const body = upload.json() as { name: string; url: string };
    createPost(db, {
      title: 'uses image',
      categoryId: null,
      contentMd: `![x](${body.url})`,
      status: 'draft',
      postDate: '2026-11-12 10:00:00',
      tags: [],
    });
    const row = db.select().from(media).where(eq(media.storedName, body.name)).get()!;
    const del = await app.inject({
      method: 'POST',
      url: `/media/${row.id}/delete`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(del.statusCode).toBe(302);
    expect(del.headers.location).toContain('error=media-in-use');
    expect(fs.existsSync(path.join(uploadsDir, body.name))).toBe(true);
  });

  it('rejects an upload without CSRF', async () => {
    const { app, cookie } = await setup();
    const boundary = '----ihmodtest';
    const res = await app.inject({
      method: 'POST',
      url: '/api/media',
      headers: {
        cookie,
        'content-type': `multipart/form-data; boundary=${boundary}`,
      },
      payload: multipartBody(boundary, png),
    });
    expect(res.statusCode).toBe(403);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/media-routes.test.ts
```

Expected: FAIL because the routes/views do not exist.

- [ ] **Step 3: Implement `src/routes/media.ts`**

```ts
import type { FastifyInstance } from 'fastify';
import type { Db } from '../db/client';
import {
  MediaInUseError,
  MediaNotFoundError,
  MediaValidationError,
  deleteMedia,
  listMedia,
  saveUpload,
} from '../services/media';

export interface MediaRouteOptions {
  db: Db;
  uploadsDir: string;
}

export async function registerMediaRoutes(
  app: FastifyInstance,
  { db, uploadsDir }: MediaRouteOptions,
): Promise<void> {
  app.post('/api/media', async (request, reply) => {
    let file;
    try {
      file = await request.file();
    } catch {
      return reply.code(400).send({ error: 'invalid-upload' });
    }
    if (!file) return reply.code(400).send({ error: 'no-file' });

    try {
      const buffer = await file.toBuffer();
      const saved = saveUpload(db, uploadsDir, { filename: file.filename, buffer });
      return reply.send({
        name: saved.storedName,
        originalName: file.filename,
        url: saved.url,
      });
    } catch (error) {
      if (error instanceof MediaValidationError) {
        return reply.code(400).send({ error: error.message });
      }
      throw error;
    }
  });

  app.get('/media', async (request, reply) => {
    const query = request.query as Record<string, string | undefined>;
    return reply.view('media.ejs', {
      title: '媒体',
      user: request.user,
      csrfToken: request.csrfToken,
      items: listMedia(db),
      notice: query.notice,
      error: query.error,
    });
  });

  app.post('/media/:id/delete', async (request, reply) => {
    const { id } = request.params as { id: string };
    try {
      deleteMedia(db, uploadsDir, id);
    } catch (error) {
      if (error instanceof MediaInUseError) {
        return reply.redirect('/media?error=media-in-use');
      }
      if (error instanceof MediaNotFoundError) {
        return reply.redirect('/media?error=invalid');
      }
      throw error;
    }
    return reply.redirect('/media?notice=deleted');
  });
}
```

**Note:** `request.file()` requires `@fastify/multipart` registered before routes. Do that in Step 6. Only multipart/validation errors are mapped to 400; other errors propagate to the app error handler so server faults are not hidden.

- [ ] **Step 4: Create `views/media.ejs`**

```ejs
<%- include('partials/header', { title: '媒体', user, csrfToken }) %>
<%- include('partials/notice', { notice, error }) %>
<h1>媒体</h1>
<form id="media-upload" data-csrf="<%= csrfToken %>" class="toolbar">
  <label class="grow">上传图片
    <input type="file" name="file" accept="image/png,image/jpeg,image/gif,image/webp" required>
  </label>
  <button type="submit">上传</button>
  <span id="media-status" class="muted"></span>
</form>
<div class="media-grid">
  <% for (const item of items) { %>
    <figure class="media-item">
      <img src="/img/uploads/<%= item.storedName %>" alt="<%= item.originalName %>">
      <figcaption>
        <span class="muted"><%= item.originalName %></span>
        <input readonly value="/img/uploads/<%= item.storedName %>">
        <form method="post" action="/media/<%= item.id %>/delete">
          <input type="hidden" name="_csrf" value="<%= csrfToken %>">
          <button class="danger" type="submit">删除</button>
        </form>
      </figcaption>
    </figure>
  <% } %>
  <% if (items.length === 0) { %><p class="muted">还没有图片。</p><% } %>
</div>
<script src="/static/media.js"></script>
<%- include('partials/footer') %>
```

- [ ] **Step 5: Implement `src/public/media.js`**

```js
(function () {
  const form = document.getElementById('media-upload');
  if (!form) return;
  const input = form.querySelector('input[type=file]');
  const status = document.getElementById('media-status');

  form.addEventListener('submit', async (event) => {
    event.preventDefault();
    if (!input.files || input.files.length === 0) return;
    const data = new FormData();
    data.append('file', input.files[0]);
    if (status) status.textContent = '上传中…';
    try {
      const res = await fetch('/api/media', {
        method: 'POST',
        headers: { 'x-csrf-token': form.dataset.csrf },
        body: data,
      });
      if (!res.ok) throw new Error('upload failed');
      if (status) status.textContent = '上传成功';
      setTimeout(() => window.location.reload(), 400);
    } catch (error) {
      if (status) status.textContent = '上传失败';
    }
  });
})();
```

- [ ] **Step 6: Update `src/app.ts`**

Add imports:

```ts
import fs from 'node:fs';
import multipart from '@fastify/multipart';
import { registerMediaRoutes } from './routes/media';
```

After the cookie/formbody registration:

```ts
  await app.register(multipart, {
    limits: { fileSize: 10 * 1024 * 1024, files: 1 },
  });
```

**Move** the existing `registerSessionAuth(app, { db, config });` call so it runs immediately after the rate-limit registration and **before** the static registrations. The resulting order is: cookie, formbody, multipart, helmet, rate-limit, **registerSessionAuth**, static public/Vditor/uploads, view, health/routes. This ensures the new `/img/uploads/` static route is covered by the auth hook (it is not under `/static/`, so it must require a session).

Before the static registrations, ensure the uploads dir exists:

```ts
  fs.mkdirSync(config.UPLOADS_DIR, { recursive: true });
```

After the existing `fastifyStatic` registration for `public`, add two more static registrations:

```ts
  await app.register(fastifyStatic, {
    root: path.resolve(__dirname, '../node_modules/vditor/dist'),
    prefix: '/static/vditor/',
    decorateReply: false,
  });
  await app.register(fastifyStatic, {
    root: config.UPLOADS_DIR,
    prefix: '/img/uploads/',
    decorateReply: false,
  });
```

Register media routes after the post routes:

```ts
  await registerMediaRoutes(app, { db, uploadsDir: config.UPLOADS_DIR });
```

- [ ] **Step 7: Update the nav and styles**

In `views/partials/header.ejs`, add a media link before the username:

```ejs
    <a href="/media">媒体</a>
```

Append to `public/admin.css`:

```css
.media-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 1rem;
  margin-top: 1rem;
}
.media-item {
  margin: 0;
  background: var(--card);
  border: 1px solid var(--line);
  border-radius: 10px;
  overflow: hidden;
}
.media-item img {
  display: block;
  width: 100%;
  height: 150px;
  object-fit: cover;
  background: #f3f4f6;
}
.media-item figcaption {
  display: grid;
  gap: 0.4rem;
  padding: 0.6rem;
  font-size: 0.85rem;
}
.media-item input {
  width: 100%;
  padding: 0.25rem 0.35rem;
  border: 1px solid var(--line);
  border-radius: 6px;
  font: inherit;
}
```

- [ ] **Step 8: Run the test to verify it passes**

```bash
npm test -- test/media-routes.test.ts
```

Expected: 4 tests pass.

If the static file test returns 403/404, verify the uploads static registration uses `decorateReply: false` and that `UPLOADS_DIR` exists before registration.

- [ ] **Step 9: Full suite, typecheck, and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/routes/media.ts src/public/media.js views/media.ejs src/app.ts views/partials/header.ejs public/admin.css test/media-routes.test.ts
git commit -m "feat: add media library and upload endpoint"
```

---

## Task 4: Autosave service and endpoint (TDD)

**Files:**
- Modify: `src/lib/posts.ts`
- Modify: `src/routes/posts.ts`
- Test: `test/autosave.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import { eq } from 'drizzle-orm';
import { posts } from '../src/db/schema';
import { createPost } from '../src/lib/posts';
import { makeTestApp, login, type TestApp } from './helpers';

let current: TestApp | null = null;
afterEach(async () => {
  if (current) await current.cleanup();
  current = null;
});

async function setup() {
  current = await makeTestApp();
  const { cookie, csrf } = await login(current.app);
  const id = createPost(current.db, {
    title: 'A',
    categoryId: null,
    contentMd: 'one',
    status: 'draft',
    postDate: '2026-11-12 10:00:00',
    tags: [],
  });
  return { ...current, cookie, csrf, id };
}

function autosave(app: any, cookie: string, csrf: string, id: string, fields: Record<string, string>) {
  const body = new URLSearchParams(fields).toString();
  return app.inject({
    method: 'POST',
    url: `/api/posts/${id}/autosave`,
    headers: {
      cookie,
      'x-csrf-token': csrf,
      'content-type': 'application/x-www-form-urlencoded',
    },
    payload: body,
  });
}

describe('autosave', () => {
  it('saves changed content and no-ops on the same content', async () => {
    const { app, db, cookie, csrf, id } = await setup();
    const before = db.select().from(posts).where(eq(posts.id, id)).get()!;

    const first = await autosave(app, cookie, csrf, id, {
      title: 'A',
      content_md: 'two',
    });
    expect(first.statusCode).toBe(200);
    expect(first.json()).toMatchObject({ saved: true });
    const after = db.select().from(posts).where(eq(posts.id, id)).get()!;
    expect(after.contentMd).toBe('two');
    expect(after.contentHash).not.toBe(before.contentHash);

    const second = await autosave(app, cookie, csrf, id, {
      title: 'A',
      content_md: 'two',
    });
    expect(second.statusCode).toBe(200);
    expect(second.json()).toMatchObject({ saved: false });
    expect(db.select().from(posts).where(eq(posts.id, id)).get()!.updatedAt).toBe(after.updatedAt);
  });

  it('keeps the existing title when the title is empty', async () => {
    const { app, db, cookie, csrf, id } = await setup();
    const res = await autosave(app, cookie, csrf, id, {
      title: '',
      content_md: 'changed',
    });
    expect(res.statusCode).toBe(200);
    expect(db.select().from(posts).where(eq(posts.id, id)).get()!.title).toBe('A');
  });

  it('rejects autosave without CSRF', async () => {
    const { app, cookie, id } = await setup();
    const res = await app.inject({
      method: 'POST',
      url: `/api/posts/${id}/autosave`,
      headers: { cookie, 'content-type': 'application/x-www-form-urlencoded' },
      payload: 'content_md=two',
    });
    expect(res.statusCode).toBe(403);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/autosave.test.ts
```

Expected: FAIL because `/api/posts/:id/autosave` does not exist / `autosavePost` is missing.

- [ ] **Step 3: Add `autosavePost` to `src/lib/posts.ts`**

Add before the export of `softDeletePost`:

```ts
export interface AutosaveInput {
  title: string;
  contentMd: string;
}

export interface AutosaveResult {
  saved: boolean;
  updatedAt: string;
}

export function autosavePost(db: Db, id: string, input: AutosaveInput): AutosaveResult {
  const post = db.select().from(posts).where(eq(posts.id, id)).get();
  if (!post) throw new PostNotFoundError('文章不存在');

  const title = input.title.trim() || post.title;
  if (post.title === title && post.contentMd === input.contentMd) {
    return { saved: false, updatedAt: post.updatedAt };
  }

  const tagRows = db
    .select({ name: tags.name })
    .from(postTags)
    .innerJoin(tags, eq(postTags.tagId, tags.id))
    .where(eq(postTags.postId, id))
    .all();
  const now = nowStamp();
  const hash = contentHash({
    title,
    contentMd: input.contentMd,
    categoryId: post.categoryId,
    tags: tagRows.map((row) => row.name),
    description: post.description,
    topImg: post.topImg,
    postDate: post.postDate,
  });

  db.update(posts)
    .set({ title, contentMd: input.contentMd, updatedAt: now, contentHash: hash })
    .where(eq(posts.id, id))
    .run();

  return { saved: true, updatedAt: now };
}
```

- [ ] **Step 4: Add the autosave route to `src/routes/posts.ts`**

Import `autosavePost` in the existing named import from `../lib/posts`.

Add the route before the `GET /posts/:id` route (or anywhere before the 404 handler; the exact order is not important):

```ts
  app.post('/api/posts/:id/autosave', async (request, reply) => {
    const { id } = request.params as { id: string };
    const body = request.body as { title?: string; content_md?: string } | undefined;
    try {
      const result = autosavePost(db, id, {
        title: body?.title ?? '',
        contentMd: body?.content_md ?? '',
      });
      return reply.send(result);
    } catch (error) {
      if (error instanceof PostNotFoundError) {
        return reply.code(404).send({ error: 'not-found' });
      }
      throw error;
    }
  });
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
npm test -- test/autosave.test.ts
```

Expected: 3 tests pass.

- [ ] **Step 6: Full suite, typecheck, and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/lib/posts.ts src/routes/posts.ts test/autosave.test.ts
git commit -m "feat: add post autosave endpoint"
```

---

## Task 5: Vditor editor integration (TDD for rendering)

**Files:**
- Modify: `src/app.ts` (CSP)
- Modify: `views/partials/header.ejs` (conditional Vditor CSS)
- Modify: `views/posts/form.ejs`
- Create: `src/public/editor.js`
- Modify: `public/admin.css`
- Test: `test/editor-render.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import { createPost } from '../src/lib/posts';
import { makeTestApp, login, type TestApp } from './helpers';

let current: TestApp | null = null;
afterEach(async () => {
  if (current) await current.cleanup();
  current = null;
});

describe('editor rendering', () => {
  it('renders the Vditor container, hidden content field, and assets', async () => {
    current = await makeTestApp();
    const { app, db } = current;
    const { cookie } = await login(app);
    const id = createPost(db, {
      title: 'editor post',
      categoryId: null,
      contentMd: '# hello',
      status: 'draft',
      postDate: '2026-11-12 10:00:00',
      tags: [],
    });

    const res = await app.inject({
      method: 'GET',
      url: `/posts/${id}`,
      headers: { cookie },
    });
    expect(res.statusCode).toBe(200);
    expect(res.body).toContain('id="vditor"');
    expect(res.body).toContain('name="content_md"');
    expect(res.body).toContain('/static/editor.js');
    expect(res.body).toContain('/static/vditor/index.min.js');
    expect(res.body).toContain('/static/vditor/index.css');
  });

  it('serves the Vditor assets', async () => {
    current = await makeTestApp();
    const res = await current.app.inject({
      method: 'GET',
      url: '/static/vditor/index.min.js',
    });
    expect(res.statusCode).toBe(200);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/editor-render.test.ts
```

Expected: FAIL because the form still has a textarea and no Vditor assets.

- [ ] **Step 3: Update the CSP in `src/app.ts`**

Replace the `contentSecurityPolicy.directives` object with:

```ts
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'blob:', 'https:'],
        fontSrc: ["'self'", 'data:'],
        connectSrc: ["'self'"],
        workerSrc: ["'self'", 'blob:'],
        scriptSrc: ["'self'"],
        frameAncestors: ["'none'"],
        baseUri: ["'none'"],
        formAction: ["'self'"],
        upgradeInsecureRequests: null,
      },
```

- [ ] **Step 4: Update `views/partials/header.ejs`**

Inside `<head>`, after the admin.css link, add:

```ejs
<% if (typeof vditor !== 'undefined' && vditor) { %>
  <link rel="stylesheet" href="/static/vditor/index.css">
<% } %>
```

- [ ] **Step 5: Update `views/posts/form.ejs`**

Change the header include to pass `vditor: true`:

```ejs
<%- include('../partials/header', { title: isNew ? '新建文章' : '编辑文章', user, csrfToken, vditor: true }) %>
```

Replace the textarea block:

```ejs
    <label class="full">正文（Markdown）
      <textarea name="content_md"><%= post.contentMd %></textarea>
    </label>
```

with:

```ejs
    <label class="full">正文（Markdown）
      <input type="hidden" name="content_md" id="content_md" value="<%= post.contentMd %>">
      <div id="vditor" data-csrf="<%= csrfToken %>" data-post-id="<%= post.id ?? '' %>"></div>
      <span id="autosave-status" class="muted"></span>
    </label>
```

Change the main form tag to have an id:

```ejs
<form method="post" action="<%= isNew ? '/posts' : '/posts/' + post.id %>" id="post-form">
```

After the closing `</form>`, before the footer include, add:

```ejs
<script src="/static/vditor/index.min.js"></script>
<script src="/static/editor.js"></script>
```

- [ ] **Step 6: Create `src/public/editor.js`**

```js
(function () {
  const container = document.getElementById('vditor');
  const input = document.getElementById('content_md');
  if (!container || !input || typeof Vditor === 'undefined') return;

  const csrf = container.dataset.csrf;
  const postId = container.dataset.postId || '';
  const status = document.getElementById('autosave-status');
  const form = document.getElementById('post-form');
  const titleInput = document.querySelector('[name=title]');
  let timer = null;
  let saving = false;

  const vditor = new Vditor(container.id, {
    height: 560,
    mode: 'sv',
    theme: 'classic',
    icon: 'material',
    cache: { enable: false },
    value: input.value,
    toolbar: [
      'headings', 'bold', 'italic', 'strike', '|',
      'list', 'ordered-list', 'check', '|',
      'quote', 'line', 'code', 'inline-code', '|',
      'upload', 'link', 'table', '|',
      'undo', 'redo',
    ],
    upload: {
      url: '/api/media',
      fieldName: 'file',
      headers: { 'x-csrf-token': csrf },
      accept: 'image/png,image/jpeg,image/gif,image/webp',
      format: function (files, responseText) {
        const res = JSON.parse(responseText);
        const succMap = {};
        succMap[res.originalName || res.name] = res.url;
        return JSON.stringify({
          msg: '',
          code: 0,
          data: { errFiles: [], succMap: succMap },
        });
      },
    },
    input: function (value) {
      input.value = value;
      schedule();
    },
  });

  function setStatus(text) {
    if (status) status.textContent = text;
  }

  async function save() {
    if (!postId || saving) return;
    const value = vditor.getValue();
    input.value = value;
    saving = true;
    try {
      const body = new URLSearchParams();
      body.set('title', titleInput ? titleInput.value : '');
      body.set('content_md', value);
      const res = await fetch('/api/posts/' + encodeURIComponent(postId) + '/autosave', {
        method: 'POST',
        headers: {
          'content-type': 'application/x-www-form-urlencoded',
          'x-csrf-token': csrf,
        },
        body: body.toString(),
      });
      if (!res.ok) throw new Error('autosave failed');
      const data = await res.json();
      if (data.saved) {
        setStatus('已自动保存 ' + new Date().toLocaleTimeString());
      }
    } catch (error) {
      setStatus('自动保存失败');
    } finally {
      saving = false;
    }
  }

  function schedule() {
    if (!postId) return;
    clearTimeout(timer);
    timer = setTimeout(save, 2000);
  }

  if (form) {
    form.addEventListener('submit', function () {
      input.value = vditor.getValue();
    });
  }

  window.setInterval(save, 30000);
})();
```

- [ ] **Step 7: Add editor spacing to `public/admin.css`**

```css
#vditor { margin-top: 0.35rem; }
#autosave-status { display: block; margin-top: 0.35rem; font-size: 0.85rem; }
```

- [ ] **Step 8: Run the editor render test**

```bash
npm test -- test/editor-render.test.ts
```

Expected: 2 tests pass.

- [ ] **Step 9: Full suite, typecheck, and commit**

```bash
npm test
npx tsc -p tsconfig.json --noEmit
git add src/app.ts views/partials/header.ejs views/posts/form.ejs src/public/editor.js public/admin.css test/editor-render.test.ts
git commit -m "feat: integrate vditor editor and autosave client"
```

---

## Task 6: Final verification and README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Run the full suite and build**

```bash
npm test
npm run build
```

Expected: all tests pass; build exits 0.

- [ ] **Step 2: Smoke test the editor and media pages**

Start the compiled server with a temporary DB (port 8096):

```bash
export DATABASE_PATH=/tmp/ihmod-editor-smoke.db
export UPLOADS_DIR=/tmp/ihmod-editor-uploads
export SESSION_SECRET=$(openssl rand -hex 32)
export ADMIN_USERNAME=admin
export ADMIN_PASSWORD=smoke-password-123
export HOST=127.0.0.1
export PORT=8096
export COOKIE_SECURE=false
rm -rf "$DATABASE_PATH" "$UPLOADS_DIR"
node dist/server.js > /tmp/ihmod-editor-smoke.log 2>&1 &
PID=$!
sleep 2
curl -s http://127.0.0.1:8096/healthz
curl -s -c /tmp/ihmod-editor-cookies.txt http://127.0.0.1:8096/login -o /tmp/ihmod-editor-login.html
TOKEN=$(grep -o 'name="_csrf" value="[^"]*"' /tmp/ihmod-editor-login.html | sed 's/.*value="//;s/"$//')
curl -s -o /dev/null -w '%{http_code}\n' -b /tmp/ihmod-editor-cookies.txt -c /tmp/ihmod-editor-cookies.txt \
  -d "_csrf=$TOKEN&username=admin&password=smoke-password-123" http://127.0.0.1:8096/login
for p in posts/new media; do
  code=$(curl -s -o /dev/null -w '%{http_code}' -b /tmp/ihmod-editor-cookies.txt "http://127.0.0.1:8096/$p")
  echo "$p: $code"
done
kill $PID
wait $PID 2>/dev/null || true
rm -rf "$DATABASE_PATH" "$DATABASE_PATH"-wal "$DATABASE_PATH"-shm "$UPLOADS_DIR" \
  /tmp/ihmod-editor-cookies.txt /tmp/ihmod-editor-login.html /tmp/ihmod-editor-smoke.log
```

Expected:
- `/healthz` → `{"ok":true}`
- login → `302`
- `posts/new: 200`
- `media: 200`

**Manual browser check (required before calling this plan done):** start the server locally, log in, open `/posts/new`, confirm Vditor renders the toolbar and split preview, type text, upload a PNG from the toolbar, confirm the image preview loads from `/img/uploads/...`, save the post, reopen it, and confirm the autosave status appears after editing. If Vditor's JavaScript is blocked by CSP, inspect the browser console and adjust only the specific directive needed (do not add `'unsafe-eval'` unless Vditor actually requires it).

- [ ] **Step 3: Add the editor/media section to `README.md`**

```markdown
## Editor and media

- Posts use Vditor (served locally from `/static/vditor/`) with Markdown split preview.
- Images are uploaded to `POST /api/media`, stored in `UPLOADS_DIR`
  (default `./data/uploads`), and referenced as `/img/uploads/<uuid>.<ext>`.
- The admin serves `/img/uploads/` only to authenticated sessions; the blog
  publisher will copy these files into `source/img/uploads/` on publish.
- Existing posts autosave title/content 2 seconds after typing and every 30
  seconds; unchanged content is a no-op on the server.
- `/media` lists uploaded images, shows their URLs, and blocks deletion while a
  post still references the file.
```

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document editor and media"
```

---

## Acceptance Checklist

- [ ] `npm test` passes all Plan 2A/2B/2C/2D tests.
- [ ] `npm run build` exits 0.
- [ ] Vditor assets load from `/static/vditor/` and the editor page renders the container and hidden content field.
- [ ] Autosave writes changed content, no-ops on identical content, and rejects missing CSRF.
- [ ] Upload accepts PNG/JPEG/GIF/WebP by magic bytes and rejects other data.
- [ ] Uploaded images are served at `/img/uploads/...` only to authenticated sessions.
- [ ] Media deletion is blocked while a post references the image.
- [ ] CSP allows Vditor's inline styles but keeps `script-src 'self'`.
- [ ] No inline styles or event handlers are added to views (Vditor injects its own at runtime).
- [ ] Manual browser check: Vditor loads, image upload inserts a preview, and autosave status appears.
- [ ] `data/uploads/` and `data/admin.db` are not tracked.

## Known behavior

- Vditor requires `style-src 'unsafe-inline'`; `script-src` remains `'self'`.
- The admin's `/img/uploads/` route is protected; the published site serves the
  same paths from `source/img/uploads/` after Plan 2E.
- Autosave is only enabled for existing posts; new posts save through the normal
  form submit.
- Media deletion checks all posts, including drafts and trash.

## Next plan

**2E:** Markdown export, publish manifest, Git commit/push, systemd deployment,
backups, ufw, and end-to-end publish verification.
