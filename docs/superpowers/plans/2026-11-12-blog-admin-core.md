# Blog Admin Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first working slice of the SQLite-backed blog admin: project scaffold, config, SQLite schema/migrations, single-user authentication (argon2id + DB sessions + CSRF + rate limit), a login page, and a protected dashboard.

**Architecture:** A standalone Node.js/TypeScript app in a new directory, separate from the Hexo repository. Data lives in a local SQLite file managed by `better-sqlite3` + Drizzle ORM; schema changes are plain SQL migrations applied at startup. Fastify serves server-rendered EJS pages. Authentication is a signed session cookie backed by a DB session row; CSRF uses signed double-submit cookies. The app binds to a configurable host/port and is designed to run natively under systemd on the VPS.

**Tech Stack:** Node.js 22, TypeScript (CommonJS output), Fastify 5, EJS, better-sqlite3, Drizzle ORM, Zod, `@node-rs/argon2`, Vitest.

---

## Scope

This plan produces working, testable software: you can start the app, log in with a seeded admin account, see a protected dashboard, log out, and run the test suite.

Not in this plan (later plans):

- **2B:** categories, tags, posts CRUD, import of the 13 existing posts.
- **2C:** Vditor editor, autosave, media upload/library.
- **2D:** Markdown export, publish manifest, Git push, systemd/backup/ufw deployment, end-to-end publish.

## File Structure

New project directory: `/home/ihmod/cc_projects/IHMOD-admin` (sibling of the blog repo).

```
IHMOD-admin/
  package.json
  tsconfig.json
  vitest.config.ts
  .gitignore
  .env.example
  README.md
  migrations/
    001_init.sql
  public/
    admin.css
  views/
    partials/header.ejs
    partials/footer.ejs
    login.ejs
    dashboard.ejs
    error.ejs
  src/
    config.ts
    db/schema.ts
    db/client.ts
    db/migrate.ts
    auth/password.ts
    auth/session.ts
    auth/csrf.ts
    auth/seed.ts
    plugins/session-auth.ts
    routes/auth.ts
    routes/dashboard.ts
    app.ts
    server.ts
  test/
    config.test.ts
    db.test.ts
    schema.test.ts
    auth.test.ts
    csrf.test.ts
    seed.test.ts
    app.test.ts
    routes.test.ts
```

Runtime directories (gitignored): `data/`, `dist/`, `node_modules/`.

---

## Task 1: Scaffold the project

**Files:**
- Create: `/home/ihmod/cc_projects/IHMOD-admin/package.json`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Create: `.gitignore`
- Create: `.env.example`
- Create: `public/.gitkeep`
- Create: `views/.gitkeep`

- [ ] **Step 1: Create the directory and initialize Git**

```bash
mkdir -p /home/ihmod/cc_projects/IHMOD-admin
cd /home/ihmod/cc_projects/IHMOD-admin
git init -b main
git config user.name "iHMOD"
git config user.email "ihmod@users.noreply.github.com"
npm init -y
```

Expected: `Initialized empty Git repository ...` and `package.json` created.

- [ ] **Step 2: Install runtime and dev dependencies**

```bash
npm install fastify @fastify/cookie @fastify/formbody @fastify/static @fastify/view @fastify/helmet @fastify/rate-limit better-sqlite3 drizzle-orm zod @node-rs/argon2 ejs dotenv
npm install -D typescript tsx vitest @types/node @types/better-sqlite3 @types/ejs
```

Expected: both commands exit 0 and create `node_modules/` + `package-lock.json`.

- [ ] **Step 3: Set `package.json` metadata and scripts without touching dependency versions**

```bash
npm pkg set name=ihmod-blog-admin version=0.1.0 private=true main=dist/server.js
npm pkg set scripts.dev="tsx watch src/server.ts"
npm pkg set scripts.build="tsc -p tsconfig.json"
npm pkg set scripts.start="node dist/server.js"
npm pkg set scripts.test="vitest run"
npm pkg set scripts.test:watch="vitest"
```

Expected: `package.json` keeps the dependency versions that npm installed and gains the six scripts. Verify:

```bash
node -e "const p=require('./package.json'); console.log(Object.keys(p.scripts).sort().join(','))"
```

Expected: `build,dev,start,test,test:watch` (the `test:watch` key is listed after `test` when sorted).

- [ ] **Step 4: Create `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "types": ["node"]
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "test"]
}
```

- [ ] **Step 5: Create `vitest.config.ts`**

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['test/**/*.test.ts'],
    testTimeout: 15000,
  },
});
```

- [ ] **Step 6: Create `.gitignore`**

```gitignore
node_modules/
dist/
data/
coverage/
.env
.env.*
!.env.example
secrets/
*.log
```

- [ ] **Step 7: Create `.env.example`**

```dotenv
NODE_ENV=development
HOST=127.0.0.1
PORT=8080
DATABASE_PATH=./data/admin.db
SESSION_SECRET=replace-with-output-of-openssl-rand-hex-32
ADMIN_USERNAME=admin
ADMIN_PASSWORD=replace-with-a-strong-password
SESSION_TTL_HOURS=168
LOGIN_RATE_LIMIT_MAX=5
COOKIE_SECURE=false
```

- [ ] **Step 8: Create placeholder asset directories and commit**

```bash
mkdir -p public views/partials test src
touch public/.gitkeep views/.gitkeep
git add .
git commit -m "chore: scaffold admin app"
git log --oneline -1
```

Expected: a commit containing the scaffold (but not `node_modules/`, `.env`, or `data/`).

---

## Task 2: Config module (TDD)

**Files:**
- Create: `src/config.ts`
- Test: `test/config.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from 'vitest';
import { loadConfig } from '../src/config';

const base = { SESSION_SECRET: 'x'.repeat(32) };

describe('loadConfig', () => {
  it('applies defaults', () => {
    const config = loadConfig({ ...base });
    expect(config.NODE_ENV).toBe('development');
    expect(config.HOST).toBe('127.0.0.1');
    expect(config.PORT).toBe(8080);
    expect(config.SESSION_TTL_HOURS).toBe(168);
    expect(config.LOGIN_RATE_LIMIT_MAX).toBe(5);
    expect(config.COOKIE_SECURE).toBe(false);
  });

  it('rejects a short session secret', () => {
    expect(() => loadConfig({ SESSION_SECRET: 'short' })).toThrow();
  });

  it('coerces numeric values', () => {
    const config = loadConfig({ ...base, PORT: '9000', SESSION_TTL_HOURS: '24' });
    expect(config.PORT).toBe(9000);
    expect(config.SESSION_TTL_HOURS).toBe(24);
  });

  it('parses COOKIE_SECURE as a boolean', () => {
    expect(loadConfig({ ...base, COOKIE_SECURE: 'true' }).COOKIE_SECURE).toBe(true);
    expect(loadConfig({ ...base, COOKIE_SECURE: 'false' }).COOKIE_SECURE).toBe(false);
  });

  it('rejects a bad NODE_ENV', () => {
    expect(() => loadConfig({ ...base, NODE_ENV: 'staging' })).toThrow();
  });

  it('rejects a short ADMIN_PASSWORD when provided', () => {
    expect(() => loadConfig({ ...base, ADMIN_PASSWORD: 'short' })).toThrow();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/config.test.ts
```

Expected: FAIL because `src/config.ts` does not exist.

- [ ] **Step 3: Implement `src/config.ts`**

```ts
import { z } from 'zod';

const booleanFromString = z
  .string()
  .default('false')
  .transform((value) => value === 'true');

const schema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  HOST: z.string().default('127.0.0.1'),
  PORT: z.coerce.number().int().positive().default(8080),
  DATABASE_PATH: z.string().default('./data/admin.db'),
  SESSION_SECRET: z.string().min(32, 'SESSION_SECRET must be at least 32 characters'),
  ADMIN_USERNAME: z.string().min(1).default('admin'),
  ADMIN_PASSWORD: z.string().min(12).optional(),
  SESSION_TTL_HOURS: z.coerce.number().int().positive().default(168),
  LOGIN_RATE_LIMIT_MAX: z.coerce.number().int().positive().default(5),
  COOKIE_SECURE: booleanFromString,
});

export type Config = z.infer<typeof schema>;

export function loadConfig(
  env: NodeJS.ProcessEnv | Record<string, string | undefined> = process.env,
): Config {
  return schema.parse(env);
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/config.test.ts
```

Expected: 6 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/config.ts test/config.test.ts
git commit -m "feat: add config validation"
```

---

## Task 3: SQLite client, migrations, and schema SQL (TDD)

**Files:**
- Create: `migrations/001_init.sql`
- Create: `src/db/client.ts`
- Create: `src/db/migrate.ts`
- Test: `test/db.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

describe('migrations', () => {
  it('creates all tables and seeds publish_state', () => {
    const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-db-'));
    const { sqlite } = createDb(path.join(dir, 'test.db'));
    cleanups.push(() => {
      sqlite.close();
      fs.rmSync(dir, { recursive: true, force: true });
    });

    applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));

    const tables = sqlite
      .prepare("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
      .all()
      .map((row: any) => row.name);

    expect(tables).toEqual(
      expect.arrayContaining([
        '_migrations',
        'users',
        'sessions',
        'categories',
        'tags',
        'posts',
        'post_tags',
        'media',
        'publish_state',
      ]),
    );

    const state = sqlite.prepare('SELECT status FROM publish_state WHERE id = 1').get() as any;
    expect(state.status).toBe('idle');
  });

  it('is idempotent', () => {
    const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-db-'));
    const { sqlite } = createDb(path.join(dir, 'test.db'));
    cleanups.push(() => {
      sqlite.close();
      fs.rmSync(dir, { recursive: true, force: true });
    });

    applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
    applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));

    const count = sqlite.prepare('SELECT COUNT(*) AS n FROM _migrations').get() as any;
    expect(count.n).toBe(1);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/db.test.ts
```

Expected: FAIL because `src/db/client.ts` does not exist.

- [ ] **Step 3: Create `migrations/001_init.sql`**

```sql
CREATE TABLE users (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  username      TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);

CREATE TABLE sessions (
  id         TEXT PRIMARY KEY,
  user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  expires_at TEXT NOT NULL,
  created_at TEXT NOT NULL,
  ip         TEXT,
  user_agent TEXT
);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);

CREATE TABLE categories (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  name       TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL
);

CREATE TABLE tags (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  name       TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL
);

CREATE TABLE posts (
  id           TEXT PRIMARY KEY,
  title        TEXT NOT NULL,
  slug         TEXT NOT NULL,
  category_id  INTEGER REFERENCES categories(id) ON DELETE SET NULL,
  description  TEXT,
  top_img      TEXT NOT NULL DEFAULT 'false',
  content_md   TEXT NOT NULL DEFAULT '',
  status       TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'published')),
  abbrlink     TEXT NOT NULL UNIQUE,
  source_path  TEXT NOT NULL UNIQUE,
  post_date    TEXT NOT NULL,
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL,
  published_at TEXT,
  deleted_at   TEXT,
  content_hash TEXT NOT NULL DEFAULT ''
);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_category ON posts(category_id);
CREATE INDEX idx_posts_updated ON posts(updated_at);

CREATE TABLE post_tags (
  post_id TEXT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

CREATE TABLE media (
  id            TEXT PRIMARY KEY,
  original_name TEXT NOT NULL,
  stored_name   TEXT NOT NULL UNIQUE,
  mime          TEXT NOT NULL,
  size          INTEGER NOT NULL,
  width         INTEGER,
  height        INTEGER,
  created_at    TEXT NOT NULL
);

CREATE TABLE publish_state (
  id          INTEGER PRIMARY KEY CHECK (id = 1),
  status      TEXT NOT NULL DEFAULT 'idle',
  message     TEXT,
  commit_sha  TEXT,
  started_at  TEXT,
  finished_at TEXT
);

INSERT INTO publish_state (id, status) VALUES (1, 'idle');
```

- [ ] **Step 4: Implement `src/db/client.ts`**

```ts
import fs from 'node:fs';
import path from 'node:path';
import Database from 'better-sqlite3';
import { drizzle } from 'drizzle-orm/better-sqlite3';
import type { BetterSQLite3Database } from 'drizzle-orm/better-sqlite3';
import * as schema from './schema';

export type Db = BetterSQLite3Database<typeof schema>;

export interface DbHandle {
  db: Db;
  sqlite: Database.Database;
}

export function createDb(databasePath: string): DbHandle {
  if (databasePath !== ':memory:') {
    fs.mkdirSync(path.dirname(databasePath), { recursive: true });
  }
  const sqlite = new Database(databasePath);
  sqlite.pragma('journal_mode = WAL');
  sqlite.pragma('foreign_keys = ON');
  sqlite.pragma('busy_timeout = 5000');
  const db = drizzle(sqlite, { schema });
  return { db, sqlite };
}
```

- [ ] **Step 5: Implement `src/db/migrate.ts`**

```ts
import fs from 'node:fs';
import path from 'node:path';
import type Database from 'better-sqlite3';

interface AppliedRow {
  name: string;
}

export function applyMigrations(sqlite: Database.Database, migrationsDir: string): void {
  sqlite.exec(
    'CREATE TABLE IF NOT EXISTS _migrations (name TEXT PRIMARY KEY, applied_at TEXT NOT NULL)',
  );

  const applied = new Set(
    (sqlite.prepare('SELECT name FROM _migrations').all() as AppliedRow[]).map((row) => row.name),
  );

  const files = fs
    .readdirSync(migrationsDir)
    .filter((file) => file.endsWith('.sql'))
    .sort();

  for (const file of files) {
    if (applied.has(file)) continue;
    const sql = fs.readFileSync(path.join(migrationsDir, file), 'utf8');
    const apply = sqlite.transaction(() => {
      sqlite.exec(sql);
      sqlite
        .prepare('INSERT INTO _migrations (name, applied_at) VALUES (?, ?)')
        .run(file, new Date().toISOString());
    });
    apply();
  }
}
```

- [ ] **Step 6: Run the test to verify it passes**

```bash
npm test -- test/db.test.ts
```

Expected: 2 tests pass.

- [ ] **Step 7: Commit**

```bash
git add migrations/001_init.sql src/db/client.ts src/db/migrate.ts test/db.test.ts
git commit -m "feat: add sqlite client and migrations"
```

---

## Task 4: Drizzle schema and a typed CRUD test (TDD)

**Files:**
- Create: `src/db/schema.ts`
- Test: `test/schema.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { eq } from 'drizzle-orm';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { categories, posts, users } from '../src/db/schema';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

describe('drizzle schema', () => {
  it('inserts and reads a post', () => {
    const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-schema-'));
    const { db, sqlite } = createDb(path.join(dir, 'test.db'));
    cleanups.push(() => {
      sqlite.close();
      fs.rmSync(dir, { recursive: true, force: true });
    });
    applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));

    const now = new Date().toISOString();
    db.insert(users)
      .values({ username: 'admin', passwordHash: 'hash', createdAt: now, updatedAt: now })
      .run();
    db.insert(categories).values({ name: '网安', createdAt: now }).run();
    db.insert(posts)
      .values({
        id: 'post-1',
        title: '信息收集',
        slug: 'info-gathering',
        categoryId: 1,
        description: 'test',
        topImg: 'false',
        contentMd: '# hello',
        status: 'draft',
        abbrlink: '312e3d12',
        sourcePath: '网安/信息收集.md',
        postDate: now,
        createdAt: now,
        updatedAt: now,
      })
      .run();

    const row = db.select().from(posts).where(eq(posts.id, 'post-1')).get();
    expect(row?.title).toBe('信息收集');
    expect(row?.status).toBe('draft');
    expect(row?.topImg).toBe('false');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/schema.test.ts
```

Expected: FAIL because `src/db/schema.ts` does not exist.

- [ ] **Step 3: Implement `src/db/schema.ts`**

```ts
import {
  index,
  integer,
  primaryKey,
  sqliteTable,
  text,
} from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  username: text('username').notNull().unique(),
  passwordHash: text('password_hash').notNull(),
  createdAt: text('created_at').notNull(),
  updatedAt: text('updated_at').notNull(),
});

export const sessions = sqliteTable(
  'sessions',
  {
    id: text('id').primaryKey(),
    userId: integer('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    expiresAt: text('expires_at').notNull(),
    createdAt: text('created_at').notNull(),
    ip: text('ip'),
    userAgent: text('user_agent'),
  },
  (table) => ({
    expiresIdx: index('idx_sessions_expires').on(table.expiresAt),
  }),
);

export const categories = sqliteTable('categories', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull().unique(),
  createdAt: text('created_at').notNull(),
});

export const tags = sqliteTable('tags', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull().unique(),
  createdAt: text('created_at').notNull(),
});

export const posts = sqliteTable(
  'posts',
  {
    id: text('id').primaryKey(),
    title: text('title').notNull(),
    slug: text('slug').notNull(),
    categoryId: integer('category_id').references(() => categories.id, {
      onDelete: 'set null',
    }),
    description: text('description'),
    topImg: text('top_img').notNull().default('false'),
    contentMd: text('content_md').notNull().default(''),
    status: text('status').$type<'draft' | 'published'>().notNull().default('draft'),
    abbrlink: text('abbrlink').notNull().unique(),
    sourcePath: text('source_path').notNull().unique(),
    postDate: text('post_date').notNull(),
    createdAt: text('created_at').notNull(),
    updatedAt: text('updated_at').notNull(),
    publishedAt: text('published_at'),
    deletedAt: text('deleted_at'),
    contentHash: text('content_hash').notNull().default(''),
  },
  (table) => ({
    statusIdx: index('idx_posts_status').on(table.status),
    categoryIdx: index('idx_posts_category').on(table.categoryId),
    updatedIdx: index('idx_posts_updated').on(table.updatedAt),
  }),
);

export const postTags = sqliteTable(
  'post_tags',
  {
    postId: text('post_id')
      .notNull()
      .references(() => posts.id, { onDelete: 'cascade' }),
    tagId: integer('tag_id')
      .notNull()
      .references(() => tags.id, { onDelete: 'cascade' }),
  },
  (table) => ({
    pk: primaryKey({ columns: [table.postId, table.tagId] }),
  }),
);

export const media = sqliteTable('media', {
  id: text('id').primaryKey(),
  originalName: text('original_name').notNull(),
  storedName: text('stored_name').notNull().unique(),
  mime: text('mime').notNull(),
  size: integer('size').notNull(),
  width: integer('width'),
  height: integer('height'),
  createdAt: text('created_at').notNull(),
});

export const publishState = sqliteTable('publish_state', {
  id: integer('id').primaryKey(),
  status: text('status')
    .$type<'idle' | 'running' | 'success' | 'error'>()
    .notNull()
    .default('idle'),
  message: text('message'),
  commitSha: text('commit_sha'),
  startedAt: text('started_at'),
  finishedAt: text('finished_at'),
});
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/schema.test.ts
```

Expected: 1 test passes.

- [ ] **Step 5: Commit**

```bash
git add src/db/schema.ts test/schema.test.ts
git commit -m "feat: add drizzle schema"
```

---

## Task 5: Password hashing and DB sessions (TDD)

**Files:**
- Create: `src/auth/password.ts`
- Create: `src/auth/session.ts`
- Test: `test/auth.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { users } from '../src/db/schema';
import { hashPassword, verifyPassword } from '../src/auth/password';
import {
  createSession,
  deleteExpiredSessions,
  deleteSession,
  getSessionUser,
} from '../src/auth/session';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-auth-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  const now = new Date().toISOString();
  db.insert(users)
    .values({ username: 'admin', passwordHash: 'hash', createdAt: now, updatedAt: now })
    .run();
  const user = db.select().from(users).get()!;
  return { db, user };
}

describe('password hashing', () => {
  it('hashes and verifies a password', async () => {
    const hash = await hashPassword('correct horse battery staple');
    expect(hash).not.toBe('correct horse battery staple');
    expect(await verifyPassword(hash, 'correct horse battery staple')).toBe(true);
    expect(await verifyPassword(hash, 'wrong')).toBe(false);
  });
});

describe('sessions', () => {
  it('creates and loads a session', () => {
    const { db, user } = setup();
    const id = createSession(db, user.id, 1, { ip: '127.0.0.1', userAgent: 'test' });
    expect(id).toMatch(/^[0-9a-f]{64}$/);
    const loaded = getSessionUser(db, id);
    expect(loaded).toEqual({ id: user.id, username: 'admin' });
  });

  it('rejects an unknown session', () => {
    const { db } = setup();
    expect(getSessionUser(db, '0'.repeat(64))).toBeNull();
  });

  it('deletes a session', () => {
    const { db, user } = setup();
    const id = createSession(db, user.id, 1, { ip: null, userAgent: null });
    deleteSession(db, id);
    expect(getSessionUser(db, id)).toBeNull();
  });

  it('expires sessions and cleans them up', () => {
    const { db, user } = setup();
    const id = createSession(db, user.id, -1, { ip: null, userAgent: null });
    expect(getSessionUser(db, id)).toBeNull();
    deleteExpiredSessions(db);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/auth.test.ts
```

Expected: FAIL because the auth modules do not exist.

- [ ] **Step 3: Implement `src/auth/password.ts`**

```ts
import { Algorithm, hash, verify } from '@node-rs/argon2';

const options = {
  algorithm: Algorithm.Argon2id,
  memoryCost: 19456,
  timeCost: 2,
  parallelism: 1,
};

export function hashPassword(password: string): Promise<string> {
  return hash(password, options);
}

export function verifyPassword(passwordHash: string, password: string): Promise<boolean> {
  return verify(passwordHash, password);
}
```

- [ ] **Step 4: Implement `src/auth/session.ts`**

```ts
import crypto from 'node:crypto';
import { eq } from 'drizzle-orm';
import type { Db } from '../db/client';
import { sessions, users } from '../db/schema';

export const SESSION_COOKIE = 'ihmod_sid';

export interface SessionUser {
  id: number;
  username: string;
}

export interface SessionMeta {
  ip: string | null;
  userAgent: string | null;
}

export function newSessionId(): string {
  return crypto.randomBytes(32).toString('hex');
}

export function createSession(
  db: Db,
  userId: number,
  ttlHours: number,
  meta: SessionMeta,
): string {
  const id = newSessionId();
  const now = new Date();
  const expiresAt = new Date(now.getTime() + ttlHours * 60 * 60 * 1000);
  db.insert(sessions)
    .values({
      id,
      userId,
      expiresAt: expiresAt.toISOString(),
      createdAt: now.toISOString(),
      ip: meta.ip,
      userAgent: meta.userAgent,
    })
    .run();
  return id;
}

export function getSessionUser(db: Db, sessionId: string): SessionUser | null {
  const row = db
    .select({
      id: users.id,
      username: users.username,
      expiresAt: sessions.expiresAt,
    })
    .from(sessions)
    .innerJoin(users, eq(sessions.userId, users.id))
    .where(eq(sessions.id, sessionId))
    .get();

  if (!row) return null;
  if (row.expiresAt <= new Date().toISOString()) {
    deleteSession(db, sessionId);
    return null;
  }
  return { id: row.id, username: row.username };
}

export function deleteSession(db: Db, sessionId: string): void {
  db.delete(sessions).where(eq(sessions.id, sessionId)).run();
}

export function deleteExpiredSessions(db: Db): void {
  db.delete(sessions).where(eq(sessions.expiresAt, new Date().toISOString())).run();
}
```

Note: `deleteExpiredSessions` should delete rows with `expires_at <= now`, not `=`. Replace the final function with:

```ts
import { lte } from 'drizzle-orm';

export function deleteExpiredSessions(db: Db): void {
  db.delete(sessions).where(lte(sessions.expiresAt, new Date().toISOString())).run();
}
```

Add `lte` to the import from `drizzle-orm`.

- [ ] **Step 5: Run the test to verify it passes**

```bash
npm test -- test/auth.test.ts
```

Expected: 5 tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/auth/password.ts src/auth/session.ts test/auth.test.ts
git commit -m "feat: add password hashing and db sessions"
```

---

## Task 6: CSRF helpers (TDD)

**Files:**
- Create: `src/auth/csrf.ts`
- Test: `test/csrf.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from 'vitest';
import { CSRF_COOKIE, generateCsrfToken, safeEqual } from '../src/auth/csrf';

describe('csrf', () => {
  it('generates a 64-char hex token', () => {
    const token = generateCsrfToken();
    expect(token).toMatch(/^[0-9a-f]{64}$/);
    expect(generateCsrfToken()).not.toBe(token);
  });

  it('compares equal strings safely', () => {
    expect(safeEqual('abc', 'abc')).toBe(true);
    expect(safeEqual('abc', 'abd')).toBe(false);
    expect(safeEqual('abc', 'abcd')).toBe(false);
    expect(safeEqual('', '')).toBe(true);
  });

  it('exposes the cookie name', () => {
    expect(CSRF_COOKIE).toBe('ihmod_csrf');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/csrf.test.ts
```

Expected: FAIL because `src/auth/csrf.ts` does not exist.

- [ ] **Step 3: Implement `src/auth/csrf.ts`**

```ts
import crypto from 'node:crypto';

export const CSRF_COOKIE = 'ihmod_csrf';

export function generateCsrfToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

export function safeEqual(a: string, b: string): boolean {
  const left = Buffer.from(a);
  const right = Buffer.from(b);
  if (left.length !== right.length) return false;
  return crypto.timingSafeEqual(left, right);
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/csrf.test.ts
```

Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/auth/csrf.ts test/csrf.test.ts
git commit -m "feat: add csrf helpers"
```

---

## Task 7: Admin seeding (TDD)

**Files:**
- Create: `src/auth/seed.ts`
- Test: `test/seed.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { users } from '../src/db/schema';
import { seedAdmin } from '../src/auth/seed';
import { loadConfig } from '../src/config';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

function setup() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-seed-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  cleanups.push(() => {
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  return db;
}

const base = {
  NODE_ENV: 'test',
  SESSION_SECRET: 'x'.repeat(32),
  ADMIN_USERNAME: 'admin',
};

describe('seedAdmin', () => {
  it('requires ADMIN_PASSWORD on first run', async () => {
    const db = setup();
    await expect(seedAdmin(db, loadConfig(base))).rejects.toThrow(/ADMIN_PASSWORD/);
  });

  it('creates the first admin and is idempotent', async () => {
    const db = setup();
    const config = loadConfig({ ...base, ADMIN_PASSWORD: 'test-password-123' });
    expect(await seedAdmin(db, config)).toEqual({ created: true });
    expect((db.select().from(users).all()).length).toBe(1);
    expect(await seedAdmin(db, config)).toEqual({ created: false });
    expect((db.select().from(users).all()).length).toBe(1);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/seed.test.ts
```

Expected: FAIL because `src/auth/seed.ts` does not exist.

- [ ] **Step 3: Implement `src/auth/seed.ts`**

```ts
import type { Config } from '../config';
import type { Db } from '../db/client';
import { users } from '../db/schema';
import { hashPassword } from './password';

export interface SeedResult {
  created: boolean;
}

export async function seedAdmin(db: Db, config: Config): Promise<SeedResult> {
  const existing = db.select().from(users).limit(1).all();
  if (existing.length > 0) return { created: false };

  if (!config.ADMIN_PASSWORD) {
    throw new Error('ADMIN_PASSWORD is required on first run');
  }

  const now = new Date().toISOString();
  const passwordHash = await hashPassword(config.ADMIN_PASSWORD);
  db.insert(users)
    .values({
      username: config.ADMIN_USERNAME,
      passwordHash,
      createdAt: now,
      updatedAt: now,
    })
    .run();

  return { created: true };
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- test/seed.test.ts
```

Expected: 2 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/auth/seed.ts test/seed.test.ts
git commit -m "feat: seed the first admin"
```

---

## Task 8: Fastify app, security plugins, session/CSRF hooks (TDD)

**Files:**
- Create: `src/plugins/session-auth.ts`
- Create: `src/app.ts`
- Create: `src/server.ts`
- Create: `views/error.ejs` (error handler needs it)
- Create: `public/admin.css` (placeholder for static plugin)
- Test: `test/app.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { seedAdmin } from '../src/auth/seed';
import { loadConfig } from '../src/config';
import { buildApp } from '../src/app';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

async function makeApp() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-app-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  const config = loadConfig({
    NODE_ENV: 'test',
    SESSION_SECRET: 'x'.repeat(32),
    ADMIN_USERNAME: 'admin',
    ADMIN_PASSWORD: 'test-password-123',
    DATABASE_PATH: path.join(dir, 'test.db'),
    LOGIN_RATE_LIMIT_MAX: '100',
    COOKIE_SECURE: 'false',
  });
  await seedAdmin(db, config);
  const app = await buildApp({ db, sqlite, config });
  cleanups.push(async () => {
    await app.close();
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  return app;
}

describe('app security', () => {
  it('serves a public health check', async () => {
    const app = await makeApp();
    const res = await app.inject({ method: 'GET', url: '/healthz' });
    expect(res.statusCode).toBe(200);
    expect(res.json()).toMatchObject({ ok: true });
  });

  it('redirects unauthenticated page requests to /login', async () => {
    const app = await makeApp();
    const res = await app.inject({ method: 'GET', url: '/' });
    expect(res.statusCode).toBe(302);
    expect(res.headers.location).toBe('/login');
  });

  it('rejects a login POST without a CSRF token', async () => {
    const app = await makeApp();
    const res = await app.inject({
      method: 'POST',
      url: '/login',
      headers: { 'content-type': 'application/x-www-form-urlencoded' },
      payload: 'username=admin&password=test-password-123',
    });
    expect(res.statusCode).toBe(403);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/app.test.ts
```

Expected: FAIL because `src/app.ts` does not exist.

- [ ] **Step 3: Implement `src/plugins/session-auth.ts`**

```ts
import type { FastifyInstance, FastifyReply, FastifyRequest } from 'fastify';
import type { Config } from '../config';
import type { Db } from '../db/client';
import type { SessionUser } from '../auth/session';
import { getSessionUser, SESSION_COOKIE } from '../auth/session';
import { CSRF_COOKIE, generateCsrfToken, safeEqual } from '../auth/csrf';

declare module 'fastify' {
  interface FastifyRequest {
    user: SessionUser | null;
    csrfToken: string;
  }
}

interface Options {
  db: Db;
  config: Config;
}

const PUBLIC_PATHS = new Set(['/login', '/healthz']);

function cookieOptions(config: Config) {
  return {
    signed: true,
    httpOnly: true,
    sameSite: 'lax' as const,
    path: '/',
    secure: config.COOKIE_SECURE,
  };
}

export function registerSessionAuth(app: FastifyInstance, options: Options): void {
  const { db, config } = options;

  app.decorateRequest('user', null);
  app.decorateRequest('csrfToken', '');

  app.addHook('preHandler', async (request: FastifyRequest, reply: FastifyReply) => {
    const path = request.url.split('?')[0];

    if (path.startsWith('/static/') || path === '/favicon.ico' || path === '/healthz') {
      return;
    }

    // Establish or validate the CSRF token before checking auth.
    const rawCsrf = request.cookies[CSRF_COOKIE];
    let csrfToken: string | null = null;
    if (rawCsrf) {
      const unsigned = request.unsignCookie(rawCsrf);
      if (unsigned.valid && unsigned.value) csrfToken = unsigned.value;
    }
    if (!csrfToken) {
      csrfToken = generateCsrfToken();
      reply.setCookie(CSRF_COOKIE, csrfToken, cookieOptions(config));
    }
    request.csrfToken = csrfToken;

    if (request.method !== 'GET' && request.method !== 'HEAD' && request.method !== 'OPTIONS') {
      const submitted =
        (request.body as Record<string, unknown> | undefined)?._csrf ??
        request.headers['x-csrf-token'];
      if (!submitted || !safeEqual(csrfToken, String(submitted))) {
        return reply.code(403).send('Invalid CSRF token');
      }
      const origin = request.headers.origin;
      if (origin) {
        try {
          if (new URL(origin).host !== request.headers.host) {
            return reply.code(403).send('Invalid origin');
          }
        } catch {
          return reply.code(403).send('Invalid origin');
        }
      }
    }

    const rawSession = request.cookies[SESSION_COOKIE];
    if (rawSession) {
      const unsigned = request.unsignCookie(rawSession);
      if (unsigned.valid && unsigned.value) {
        request.user = getSessionUser(db, unsigned.value);
      }
    }

    if (!PUBLIC_PATHS.has(path) && !request.user) {
      if (request.method === 'GET') {
        return reply.redirect('/login');
      }
      return reply.code(401).send({ error: 'Unauthorized' });
    }
  });
}

export { cookieOptions };
```

- [ ] **Step 4: Create `views/error.ejs`**

```ejs
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>出错了 | iHMOD Admin</title>
  <link rel="stylesheet" href="/static/admin.css">
</head>
<body>
  <main class="container">
    <section class="card narrow">
      <h1>出错了</h1>
      <p><%= message %></p>
      <p><a href="/">返回仪表盘</a></p>
    </section>
  </main>
</body>
</html>
```

- [ ] **Step 5: Create a placeholder `public/admin.css`**

```css
body { font-family: system-ui, sans-serif; margin: 0; }
.container { max-width: 960px; margin: 0 auto; padding: 2rem 1rem; }
.card { background: #fff; border: 1px solid #ddd; border-radius: 8px; padding: 1.5rem; }
.narrow { max-width: 420px; margin: 2rem auto; }
```

- [ ] **Step 6: Implement `src/app.ts`**

```ts
import path from 'node:path';
import Fastify, { type FastifyInstance } from 'fastify';
import cookie from '@fastify/cookie';
import formbody from '@fastify/formbody';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';
import fastifyStatic from '@fastify/static';
import view from '@fastify/view';
import ejs from 'ejs';
import type Database from 'better-sqlite3';
import type { Config } from './config';
import type { Db } from './db/client';
import { registerSessionAuth } from './plugins/session-auth';

export interface AppDeps {
  db: Db;
  sqlite: Database.Database;
  config: Config;
}

export async function buildApp({ db, sqlite, config }: AppDeps): Promise<FastifyInstance> {
  const app = Fastify({
    logger: config.NODE_ENV === 'test' ? false : { level: 'info' },
  });

  await app.register(cookie, { secret: config.SESSION_SECRET });
  await app.register(formbody);
  await app.register(helmet, {
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        scriptSrc: ["'self'"],
        frameAncestors: ["'none'"],
        baseUri: ["'none'"],
        formAction: ["'self'"],
        upgradeInsecureRequests: null,
      },
    },
  });
  await app.register(rateLimit, { global: false });
  await app.register(fastifyStatic, {
    root: path.resolve(__dirname, '../public'),
    prefix: '/static/',
  });
  await app.register(view, {
    engine: { ejs },
    root: path.resolve(__dirname, '../views'),
  });

  registerSessionAuth(app, { db, config });

  app.get('/healthz', async () => {
    sqlite.prepare('SELECT 1').get();
    return { ok: true };
  });

  // Temporary stubs so the session/CSRF hooks can be tested in this task.
  // Task 9 replaces these with the real login/logout/dashboard routes.
  app.get('/', async () => ({ ok: true }));
  app.post('/login', async () => ({ ok: true }));

  app.setNotFoundHandler((request, reply) => {
    return reply.code(404).view('error.ejs', {
      message: '页面不存在',
      user: request.user ?? null,
      csrfToken: request.csrfToken ?? '',
    });
  });

  app.setErrorHandler((error, request, reply) => {
    request.log.error({ err: error }, 'request failed');
    if (reply.sent) return;
    const statusCode = error.statusCode ?? 500;
    const message = statusCode === 500 ? '服务器内部错误' : error.message;
    if ((request.headers.accept ?? '').includes('text/html')) {
      return reply.code(statusCode).view('error.ejs', {
        message,
        user: request.user ?? null,
        csrfToken: request.csrfToken ?? '',
      });
    }
    return reply.code(statusCode).send({ error: message });
  });

  return app;
}
```

- [ ] **Step 7: Implement `src/server.ts`**

```ts
import 'dotenv/config';
import path from 'node:path';
import { loadConfig } from './config';
import { createDb } from './db/client';
import { applyMigrations } from './db/migrate';
import { seedAdmin } from './auth/seed';
import { buildApp } from './app';

async function main(): Promise<void> {
  const config = loadConfig();
  const { db, sqlite } = createDb(config.DATABASE_PATH);
  applyMigrations(sqlite, path.resolve(__dirname, '../migrations'));
  const seed = await seedAdmin(db, config);
  if (seed.created) console.log(`created admin user: ${config.ADMIN_USERNAME}`);

  const app = await buildApp({ db, sqlite, config });
  await app.listen({ host: config.HOST, port: config.PORT });
  console.log(`admin listening on http://${config.HOST}:${config.PORT}`);

  const shutdown = async (): Promise<void> => {
    await app.close();
    sqlite.close();
    process.exit(0);
  };
  process.on('SIGINT', shutdown);
  process.on('SIGTERM', shutdown);
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

- [ ] **Step 8: Run the test to verify it passes**

```bash
npm test -- test/app.test.ts
```

Expected: 3 tests pass.

If TypeScript complains about `upgradeInsecureRequests` or `request.user` types, keep the declared module augmentation and check the `@fastify/helmet` version's directive names. The code above is intentionally minimal.

- [ ] **Step 9: Commit**

```bash
git add src/plugins/session-auth.ts src/app.ts src/server.ts views/error.ejs public/admin.css test/app.test.ts
git commit -m "feat: add fastify app with session and csrf hooks"
```

---

## Task 9: Login, logout, dashboard, and views (TDD)

**Files:**
- Create: `src/routes/auth.ts`
- Create: `src/routes/dashboard.ts`
- Modify: `src/app.ts`
- Create: `views/partials/header.ejs`
- Create: `views/partials/footer.ejs`
- Create: `views/login.ejs`
- Create: `views/dashboard.ejs`
- Modify: `public/admin.css`
- Test: `test/routes.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from 'vitest';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { createDb } from '../src/db/client';
import { applyMigrations } from '../src/db/migrate';
import { seedAdmin } from '../src/auth/seed';
import { loadConfig } from '../src/config';
import { buildApp } from '../src/app';

const cleanups: Array<() => void> = [];
afterEach(() => {
  while (cleanups.length) cleanups.pop()!();
});

async function makeApp(overrides: Record<string, string> = {}) {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ihmod-routes-'));
  const { db, sqlite } = createDb(path.join(dir, 'test.db'));
  applyMigrations(sqlite, path.resolve(process.cwd(), 'migrations'));
  const config = loadConfig({
    NODE_ENV: 'test',
    SESSION_SECRET: 'x'.repeat(32),
    ADMIN_USERNAME: 'admin',
    ADMIN_PASSWORD: 'test-password-123',
    DATABASE_PATH: path.join(dir, 'test.db'),
    LOGIN_RATE_LIMIT_MAX: '100',
    COOKIE_SECURE: 'false',
    ...overrides,
  });
  await seedAdmin(db, config);
  const app = await buildApp({ db, sqlite, config });
  cleanups.push(async () => {
    await app.close();
    sqlite.close();
    fs.rmSync(dir, { recursive: true, force: true });
  });
  return app;
}

function setCookies(res: { headers: Record<string, unknown> }): string[] {
  const raw = res.headers['set-cookie'];
  if (!raw) return [];
  return Array.isArray(raw) ? raw : [raw as string];
}

function cookieHeader(cookies: string[]): string {
  return cookies.map((cookie) => cookie.split(';')[0]).join('; ');
}

function extractCsrf(html: string): string {
  const match = html.match(/name="_csrf" value="([^"]+)"/);
  if (!match) throw new Error('csrf token not found');
  return match[1];
}

describe('login flow', () => {
  it('renders the login form with a CSRF token', async () => {
    const app = await makeApp();
    const res = await app.inject({ method: 'GET', url: '/login' });
    expect(res.statusCode).toBe(200);
    expect(res.body).toContain('name="_csrf"');
    expect(setCookies(res).length).toBeGreaterThan(0);
  });

  it('logs in with valid credentials and reaches the dashboard', async () => {
    const app = await makeApp();
    const loginPage = await app.inject({ method: 'GET', url: '/login' });
    const cookies = setCookies(loginPage);
    const csrf = extractCsrf(loginPage.body);

    const login = await app.inject({
      method: 'POST',
      url: '/login',
      headers: {
        cookie: cookieHeader(cookies),
        'content-type': 'application/x-www-form-urlencoded',
      },
      payload: `_csrf=${encodeURIComponent(csrf)}&username=admin&password=test-password-123`,
    });
    expect(login.statusCode).toBe(302);
    expect(login.headers.location).toBe('/');

    const allCookies = [...cookies, ...setCookies(login)];
    const dashboard = await app.inject({
      method: 'GET',
      url: '/',
      headers: { cookie: cookieHeader(allCookies) },
    });
    expect(dashboard.statusCode).toBe(200);
    expect(dashboard.body).toContain('仪表盘');
  });

  it('rejects a wrong password', async () => {
    const app = await makeApp();
    const loginPage = await app.inject({ method: 'GET', url: '/login' });
    const cookies = setCookies(loginPage);
    const csrf = extractCsrf(loginPage.body);

    const login = await app.inject({
      method: 'POST',
      url: '/login',
      headers: {
        cookie: cookieHeader(cookies),
        'content-type': 'application/x-www-form-urlencoded',
      },
      payload: `_csrf=${encodeURIComponent(csrf)}&username=admin&password=wrong-password`,
    });
    expect(login.statusCode).toBe(401);
    expect(login.body).toContain('用户名或密码错误');
  });

  it('logs out and invalidates the session', async () => {
    const app = await makeApp();
    const loginPage = await app.inject({ method: 'GET', url: '/login' });
    const cookies = setCookies(loginPage);
    const csrf = extractCsrf(loginPage.body);
    const login = await app.inject({
      method: 'POST',
      url: '/login',
      headers: {
        cookie: cookieHeader(cookies),
        'content-type': 'application/x-www-form-urlencoded',
      },
      payload: `_csrf=${encodeURIComponent(csrf)}&username=admin&password=test-password-123`,
    });
    const allCookies = [...cookies, ...setCookies(login)];

    const logout = await app.inject({
      method: 'POST',
      url: '/logout',
      headers: {
        cookie: cookieHeader(allCookies),
        'content-type': 'application/x-www-form-urlencoded',
      },
      payload: `_csrf=${encodeURIComponent(csrf)}`,
    });
    expect(logout.statusCode).toBe(302);
    expect(logout.headers.location).toBe('/login');
  });

  it('rate-limits repeated failed logins', async () => {
    const app = await makeApp({ LOGIN_RATE_LIMIT_MAX: '2' });
    const loginPage = await app.inject({ method: 'GET', url: '/login' });
    const cookies = setCookies(loginPage);
    const csrf = extractCsrf(loginPage.body);

    const attempt = () =>
      app.inject({
        method: 'POST',
        url: '/login',
        headers: {
          cookie: cookieHeader(cookies),
          'content-type': 'application/x-www-form-urlencoded',
        },
        payload: `_csrf=${encodeURIComponent(csrf)}&username=admin&password=wrong`,
      });

    expect((await attempt()).statusCode).toBe(401);
    expect((await attempt()).statusCode).toBe(401);
    expect((await attempt()).statusCode).toBe(429);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- test/routes.test.ts
```

Expected: FAIL because the real login/dashboard routes and views are not implemented yet (Task 8 only has temporary stubs).

- [ ] **Step 3: Implement `src/routes/auth.ts`**

```ts
import type { FastifyInstance } from 'fastify';
import { eq } from 'drizzle-orm';
import type { Config } from '../config';
import type { Db } from '../db/client';
import { users } from '../db/schema';
import { verifyPassword } from '../auth/password';
import { createSession, deleteSession, SESSION_COOKIE } from '../auth/session';
import { cookieOptions } from '../plugins/session-auth';

export interface AuthRouteOptions {
  db: Db;
  config: Config;
}

export async function registerAuthRoutes(
  app: FastifyInstance,
  { db, config }: AuthRouteOptions,
): Promise<void> {
  app.get('/login', async (request, reply) => {
    if (request.user) return reply.redirect('/');
    return reply.view('login.ejs', {
      title: '登录',
      user: null,
      csrfToken: request.csrfToken,
      error: null,
    });
  });

  app.post(
    '/login',
    {
      config: {
        rateLimit: {
          max: config.LOGIN_RATE_LIMIT_MAX,
          timeWindow: '15 minutes',
        },
      },
    },
    async (request, reply) => {
      const body = request.body as { username?: string; password?: string } | undefined;
      const username = (body?.username ?? '').trim();
      const password = body?.password ?? '';

      const user = db.select().from(users).where(eq(users.username, username)).get();
      const valid = user ? await verifyPassword(user.passwordHash, password) : false;

      if (!user || !valid) {
        return reply.code(401).view('login.ejs', {
          title: '登录',
          user: null,
          csrfToken: request.csrfToken,
          error: '用户名或密码错误',
        });
      }

      const sessionId = createSession(db, user.id, config.SESSION_TTL_HOURS, {
        ip: request.ip ?? null,
        userAgent: request.headers['user-agent'] ?? null,
      });
      reply.setCookie(SESSION_COOKIE, sessionId, {
        ...cookieOptions(config),
        maxAge: config.SESSION_TTL_HOURS * 60 * 60,
      });
      return reply.redirect('/');
    },
  );

  app.post('/logout', async (request, reply) => {
    const raw = request.cookies[SESSION_COOKIE];
    if (raw) {
      const unsigned = request.unsignCookie(raw);
      if (unsigned.valid && unsigned.value) {
        deleteSession(db, unsigned.value);
      }
    }
    reply.clearCookie(SESSION_COOKIE, { path: '/' });
    return reply.redirect('/login');
  });
}
```

- [ ] **Step 4: Implement `src/routes/dashboard.ts`, then wire both routes into `src/app.ts`**

```ts
import type { FastifyInstance } from 'fastify';
import { and, eq, isNull, sql } from 'drizzle-orm';
import type { Db } from '../db/client';
import { categories, posts, tags } from '../db/schema';

export interface DashboardRouteOptions {
  db: Db;
}

export async function registerDashboardRoutes(
  app: FastifyInstance,
  { db }: DashboardRouteOptions,
): Promise<void> {
  app.get('/', async (request, reply) => {
    const total =
      db.select({ n: sql<number>`count(*)` }).from(posts).where(isNull(posts.deletedAt)).get()?.n ?? 0;
    const published =
      db
        .select({ n: sql<number>`count(*)` })
        .from(posts)
        .where(and(eq(posts.status, 'published'), isNull(posts.deletedAt)))
        .get()?.n ?? 0;
    const drafts =
      db
        .select({ n: sql<number>`count(*)` })
        .from(posts)
        .where(and(eq(posts.status, 'draft'), isNull(posts.deletedAt)))
        .get()?.n ?? 0;
    const categoryCount =
      db.select({ n: sql<number>`count(*)` }).from(categories).get()?.n ?? 0;
    const tagCount = db.select({ n: sql<number>`count(*)` }).from(tags).get()?.n ?? 0;

    return reply.view('dashboard.ejs', {
      title: '仪表盘',
      user: request.user,
      csrfToken: request.csrfToken,
      counts: { total, published, drafts, categories: categoryCount, tags: tagCount },
    });
  });
}
```

Then update `src/app.ts`:

1. Add the imports after the `registerSessionAuth` import:

```ts
import { registerAuthRoutes } from './routes/auth';
import { registerDashboardRoutes } from './routes/dashboard';
```

2. Replace the two temporary stubs:

```ts
  // Temporary stubs so the session/CSRF hooks can be tested in this task.
  // Task 9 replaces these with the real login/logout/dashboard routes.
  app.get('/', async () => ({ ok: true }));
  app.post('/login', async () => ({ ok: true }));
```

with:

```ts
  await registerAuthRoutes(app, { db, config });
  await registerDashboardRoutes(app, { db });
```

- [ ] **Step 5: Create `views/partials/header.ejs`**

```ejs
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title><%= typeof title !== 'undefined' && title ? title + ' | ' : '' %>iHMOD Admin</title>
  <link rel="stylesheet" href="/static/admin.css">
</head>
<body>
<% if (typeof user !== 'undefined' && user) { %>
<nav class="nav">
  <a class="brand" href="/">iHMOD Admin</a>
  <div class="nav-links">
    <a href="/">仪表盘</a>
    <span class="who"><%= user.username %></span>
    <form method="post" action="/logout" class="inline">
      <input type="hidden" name="_csrf" value="<%= csrfToken %>">
      <button class="link-button" type="submit">退出</button>
    </form>
  </div>
</nav>
<% } %>
<main class="container">
```

- [ ] **Step 6: Create `views/partials/footer.ejs`**

```ejs
</main>
</body>
</html>
```

- [ ] **Step 7: Create `views/login.ejs`**

```ejs
<%- include('partials/header', { title: '登录', user: null, csrfToken }) %>
<section class="card narrow">
  <h1>登录</h1>
  <% if (error) { %>
    <p class="error"><%= error %></p>
  <% } %>
  <form method="post" action="/login">
    <input type="hidden" name="_csrf" value="<%= csrfToken %>">
    <label>用户名
      <input name="username" autocomplete="username" required autofocus>
    </label>
    <label>密码
      <input name="password" type="password" autocomplete="current-password" required>
    </label>
    <button type="submit">登录</button>
  </form>
</section>
<%- include('partials/footer') %>
```

- [ ] **Step 8: Create `views/dashboard.ejs`**

```ejs
<%- include('partials/header', { title: '仪表盘', user, csrfToken }) %>
<h1>仪表盘</h1>
<div class="grid">
  <div class="stat"><span class="num"><%= counts.total %></span><span>文章</span></div>
  <div class="stat"><span class="num"><%= counts.published %></span><span>已发布</span></div>
  <div class="stat"><span class="num"><%= counts.drafts %></span><span>草稿</span></div>
  <div class="stat"><span class="num"><%= counts.categories %></span><span>分类</span></div>
  <div class="stat"><span class="num"><%= counts.tags %></span><span>标签</span></div>
</div>
<p class="muted">内容管理将在下一步接入。</p>
<%- include('partials/footer') %>
```

- [ ] **Step 9: Replace `public/admin.css` with the full stylesheet**

```css
:root {
  --bg: #f5f6f8;
  --card: #ffffff;
  --text: #1f2933;
  --muted: #6b7280;
  --line: #e5e7eb;
  --accent: #2563eb;
  --danger: #dc2626;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--bg);
  color: var(--text);
  font-family: system-ui, -apple-system, 'Segoe UI', 'Noto Sans SC', sans-serif;
  line-height: 1.6;
}

a { color: var(--accent); text-decoration: none; }
a:hover { text-decoration: underline; }

.nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 0.75rem 1.25rem;
  background: var(--card);
  border-bottom: 1px solid var(--line);
}

.brand { font-weight: 700; color: var(--text); }
.nav-links { display: flex; align-items: center; gap: 1rem; }
.who { color: var(--muted); font-size: 0.9rem; }

.container { max-width: 960px; margin: 0 auto; padding: 2rem 1rem 4rem; }

.card {
  background: var(--card);
  border: 1px solid var(--line);
  border-radius: 10px;
  padding: 1.5rem;
}

.narrow { max-width: 420px; margin: 2rem auto; }

h1 { margin-top: 0; font-size: 1.5rem; }
.muted { color: var(--muted); }
.error {
  color: var(--danger);
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 0.5rem 0.75rem;
}

label { display: block; margin-bottom: 1rem; font-size: 0.95rem; }
input[type='text'],
input[type='password'],
input[name='username'],
input[name='password'] {
  display: block;
  width: 100%;
  margin-top: 0.35rem;
  padding: 0.55rem 0.7rem;
  border: 1px solid var(--line);
  border-radius: 8px;
  font: inherit;
}

button {
  padding: 0.55rem 1.1rem;
  border: 0;
  border-radius: 8px;
  background: var(--accent);
  color: #fff;
  font: inherit;
  cursor: pointer;
}

button:hover { background: #1d4ed8; }

.inline { display: inline; }
.link-button {
  background: none;
  color: var(--accent);
  padding: 0;
}
.link-button:hover { background: none; text-decoration: underline; }

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 1rem;
  margin: 1.5rem 0;
}

.stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 1rem;
  background: var(--card);
  border: 1px solid var(--line);
  border-radius: 10px;
  color: var(--muted);
  font-size: 0.9rem;
}

.num { font-size: 1.75rem; font-weight: 700; color: var(--text); }
```

- [ ] **Step 10: Run the test to verify it passes**

```bash
npm test -- test/routes.test.ts
```

Expected: 5 tests pass.

If the rate-limit test is flaky because `inject` shares an IP, run the file again; the plugin is per-app instance and each test creates a new app.

- [ ] **Step 11: Run the whole suite**

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 12: Commit**

```bash
git add src/routes/auth.ts src/routes/dashboard.ts src/app.ts views/partials/header.ejs views/partials/footer.ejs views/login.ejs views/dashboard.ejs public/admin.css test/routes.test.ts
git commit -m "feat: add login, logout, and dashboard"
```

---

## Task 10: Build, smoke test, and README

**Files:**
- Modify: `README.md`
- Modify: `package.json` (no code changes expected; verify scripts)

- [ ] **Step 1: Build the project**

```bash
npm run build
```

Expected: exit 0 and `dist/server.js`, `dist/app.js`, etc. are created.

- [ ] **Step 2: Start the compiled server against a temporary database**

```bash
export DATABASE_PATH=/tmp/ihmod-admin-smoke.db
export SESSION_SECRET=$(openssl rand -hex 32)
export ADMIN_USERNAME=admin
export ADMIN_PASSWORD=smoke-password-123
export HOST=127.0.0.1
export PORT=8099
export COOKIE_SECURE=false
rm -f "$DATABASE_PATH"
node dist/server.js > /tmp/ihmod-admin-smoke.log 2>&1 &
PID=$!
sleep 2
curl -s http://127.0.0.1:8099/healthz
```

Expected: `{"ok":true}` and the server log contains `admin listening`.

- [ ] **Step 3: Exercise the login flow with curl**

```bash
curl -s -c /tmp/ihmod-cookies.txt http://127.0.0.1:8099/login -o /tmp/ihmod-login.html
TOKEN=$(grep -o 'name="_csrf" value="[^"]*"' /tmp/ihmod-login.html | sed 's/.*value="//;s/"$//')
curl -s -o /dev/null -w '%{http_code}\n' \
  -b /tmp/ihmod-cookies.txt -c /tmp/ihmod-cookies.txt \
  -d "_csrf=$TOKEN&username=admin&password=smoke-password-123" \
  http://127.0.0.1:8099/login
curl -s -b /tmp/ihmod-cookies.txt http://127.0.0.1:8099/ | grep -o '<h1>[^<]*</h1>' | head -1
```

Expected: `302`, then `<h1>仪表盘</h1>`.

- [ ] **Step 4: Stop the server and clean up**

```bash
kill $PID
rm -f /tmp/ihmod-admin-smoke.db /tmp/ihmod-cookies.txt /tmp/ihmod-login.html /tmp/ihmod-admin-smoke.log
unset DATABASE_PATH SESSION_SECRET ADMIN_USERNAME ADMIN_PASSWORD HOST PORT COOKIE_SECURE
```

- [ ] **Step 5: Write `README.md`**

```markdown
# iHMOD Blog Admin

Self-hosted admin for the iHMOD Hexo blog. SQLite is the source of truth;
Markdown is generated and pushed to GitHub (later plans).

## Requirements

- Node.js 22
- npm 9+

## Setup

```bash
npm install
cp .env.example .env
# set SESSION_SECRET (`openssl rand -hex 32`) and ADMIN_PASSWORD in .env
```

## Develop

```bash
npm run dev
```

Open http://127.0.0.1:8080 and log in with the credentials from `.env`.
The first run creates the admin user, applies migrations, and writes `data/admin.db`.

## Test

```bash
npm test
```

## Build and run

```bash
npm run build
npm start
```

## Layout

- `src/` TypeScript source
- `migrations/` plain SQL migrations
- `views/` EJS templates
- `public/` static assets
- `data/` runtime SQLite + uploads (gitignored)
```

- [ ] **Step 6: Commit**

```bash
git add README.md
git commit -m "docs: add admin setup and run instructions"
```

---

## Task 11: Push the admin repo to GitHub

**Files:** none (remote operation).

**Manual prerequisite:** create a **private** GitHub repository named `ihmod/blog-admin` (empty, no README) at https://github.com/new.

- [ ] **Step 1: Add the remote and push**

```bash
git remote add origin git@github.com:ihmod/blog-admin.git
git push -u origin main
git remote -v
git log --oneline -1
```

Expected: `main` is pushed to the private repo; `git log` shows the latest commit.

- [ ] **Step 2: Verify no secrets were pushed**

```bash
git ls-files | grep -E '(^|/)(\.env|secrets/|data/|dist/|node_modules/)' || echo "OK: no secrets or build output tracked"
```

Expected: `OK: no secrets or build output tracked`.

---

## Acceptance Checklist

- [ ] `npm test` passes (config, DB, schema, auth, CSRF, seed, app, routes).
- [ ] `npm run build` exits 0.
- [ ] `npm start` applies migrations, seeds the admin, and serves `/healthz`.
- [ ] `/` redirects to `/login` when unauthenticated.
- [ ] Login with the seeded credentials reaches the dashboard.
- [ ] Wrong password returns 401 with a visible error.
- [ ] Repeated failed logins are rate-limited.
- [ ] `POST /login` without CSRF returns 403.
- [ ] Logout invalidates the DB session.
- [ ] `data/`, `dist/`, `node_modules/`, `.env` are not tracked.
- [ ] The private repo `ihmod/blog-admin` contains the code.

## Notes for the next plans

- **2B** adds categories/tags/posts CRUD and the importer. It will replace the dashboard note with real content navigation.
- **2C** adds Vditor and the media library; the CSP will need `img-src` for uploads and Vditor assets.
- **2D** adds the exporter/manifest/git publish and systemd/backup/ufw deployment. The deploy key is already available at `~/ihmod-secrets/ihmod_blog_admin` and will be copied to `/opt/ihmod-admin/secrets/deploy_key` on the VPS.
