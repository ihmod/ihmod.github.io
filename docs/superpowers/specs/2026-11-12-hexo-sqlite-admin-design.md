# Hexo + SQLite 在线后台管理系统 设计文档

- 日期：2026-11-12
- 项目：iHMOD 博客（Hexo 7.3.0 + Butterfly 4.10.0）
- 状态：已与用户确认，待实现

## 1. 背景与目标

现有博客只能在本机通过 Hexo CLI 构建，再用 `hexo-deployer-git` 推送到 GitHub Pages。用户希望脱离本地环境，在浏览器里完成写作、编辑、发布。

目标：

1. 保留现有 Hexo + Butterfly 主题和站点 URL 规则（`abbrlink`），渲染层不改。
2. 新增一个自建的在线后台管理系统，运行在用户的 VPS 上。
3. 使用 SQLite 作为文章数据的唯一数据源。
4. 发布时把 SQLite 内容导出为 Markdown，推送到 GitHub；由 GitHub Actions 构建并发布到 GitHub Pages。
5. 现有 13 篇文章无损导入后台。

## 2. 非目标（v1 明确不做）

- 多用户、角色权限、审计日志。
- 在线编辑主题、`_config.yml`、`_config.butterfly.yml`、友链等站点配置。
- 评论、访问统计、全文搜索后台。
- 双向同步（SQLite 与 Markdown 互相导入）。
- 文章版本历史（v1 依赖 SQLite 备份与 Git 历史；后续可加 `revisions` 表）。
- 公网 HTTPS / 域名（按用户选择使用公网 IP + 防火墙白名单 + HTTP，风险见第 11 节）。

## 3. 关键决策

| 决策 | 结论 |
|---|---|
| 数据源 | SQLite 是唯一数据源；Markdown 是发布产物 |
| 渲染 | Hexo + Butterfly 原样保留，URL 仍由 `abbrlink` 决定 |
| 后台实现 | 完全自建（Node.js + Fastify + SQLite + EJS + Vditor） |
| 站点托管 | 继续 GitHub Pages，GitHub Actions 构建 |
| 后台代码 | 独立私有仓库 `ihmod/blog-admin`，不混入博客仓库 |
| 博客仓库 | `ihmod/ihmod.github.io`，`main` 分支放源码，Pages 源改为 GitHub Actions |
| 后台访问 | 公网 IP + `ufw` 白名单 + HTTP；仅监听指定端口，登录限流 |
| 部署 | 原生 Node + systemd（不用 Docker），单机 SQLite，无 Redis/MongoDB/消息队列 |

## 4. 总体架构

```
浏览器（写作 / 管理）
   │  http://<VPS_IP>:8080 （ufw 仅放行用户 IP）
   ▼
[blog-admin: Node 22 + Fastify + EJS + Vditor]
   │
   ├── SQLite: data/admin.db
   ├── 上传文件: data/uploads/
   ├── 博客 git 工作区: data/repo/（clone 自 ihmod/ihmod.github.io）
   └── 发布清单: data/publish-manifest.json
   │
   │ 点击“发布”：
   │   1. SQLite → source/_posts/**/*.md + source/img/uploads/*
   │   2. git add/commit/push main（部署密钥）
   ▼
GitHub: ihmod/ihmod.github.io
   │
   ▼
GitHub Actions: npm ci → hexo generate → deploy-pages
   │
   ▼
GitHub Pages: https://ihmod.github.io
```

数据流是单向的：**后台编辑 → SQLite → Markdown → Git → Actions → Pages**。直接修改仓库里的 Markdown 会在下次发布时被覆盖；`publish-manifest.json` 只允许后台删除自己生成过的文件。

## 5. 技术栈

| 层 | 选型 | 理由 |
|---|---|---|
| 运行时 | Node.js 22 LTS + TypeScript | 与本地/CI Node 版本一致 |
| Web 框架 | Fastify 5 | 性能好，插件生态完整 |
| 数据库 | SQLite + better-sqlite3 | 单机、同步 API、无外部服务 |
| ORM/迁移 | Drizzle ORM + drizzle-kit | 类型安全、迁移可追踪 |
| 模板 | EJS（服务端渲染） | 不引入 React 构建链，代码量小 |
| Markdown 编辑器 | Vditor（本地打包，不用 CDN） | 中文友好、分屏预览、上传钩子 |
| 校验 | Zod | 请求/表单参数校验 |
| 密码哈希 | `@node-rs/argon2`（argon2id） | 内存硬、抗 GPU |
| Git 操作 | simple-git | 封装 commit/push/fetch |
| Markdown 解析 | gray-matter + js-yaml + marked | 导入与预览 |
| 预览净化 | DOMPurify（客户端）/ sanitize-html（服务端） | 防 XSS |
| 日志 | pino（Fastify 内置） | JSON 日志、可脱敏 |
| 测试 | Vitest | 单元 + 集成 |
| 部署 | 原生 Node + systemd（不用 Docker） | 少一层容器/iptables，ufw 规则直接生效 |

## 6. 仓库结构

### 6.1 博客仓库 `ihmod/ihmod.github.io`

```
_config.yml                 # 保留，移除 deploy 段
_config.butterfly.yml       # 保留，后台不编辑
package.json                # Hexo 依赖
themes/butterfly/           # 保留（约 27MB，直接提交）
source/
  _posts/                   # 后台生成的 Markdown（manifest 管理）
  img/uploads/              # 后台生成的图片（manifest 管理）
  _data/link.yml            # 保留，手工维护
  categories/ tags/ link/   # 保留
.github/workflows/deploy.yml
.gitignore                  # 增加 .obsidian/、.env、私钥、node_modules/、public/、db.json 等
```

### 6.2 后台仓库 `ihmod/blog-admin`（私有）

```
package.json
tsconfig.json
src/
  server.ts                 # Fastify 启动、插件注册
  config.ts                 # 环境变量解析与校验
  db/
    schema.ts               # Drizzle 表定义
    client.ts               # better-sqlite3 连接、WAL、外键
    migrate.ts              # 迁移执行
  auth/
    password.ts             # argon2id
    session.ts              # DB 会话
    csrf.ts                 # CSRF token
  routes/
    auth.ts                 # 登录/登出
    posts.ts                # 文章 CRUD/发布
    categories.ts
    tags.ts
    media.ts
    publish.ts              # 全站发布
  services/
    abbrlink.ts
    markdown.ts             # frontmatter 渲染/解析
    importer.ts             # 现有 Markdown 导入
    exporter.ts             # DB → 文件 + manifest
    git.ts                  # fetch/rebase/commit/push
    media.ts                # 上传、校验、存储
  views/                    # EJS
  public/                   # admin.css / admin.js / vditor/
scripts/
  import-posts.ts           # CLI 导入
  publish-cli.ts            # 命令行发布（应急）
  reset-password.ts         # 重置管理员密码
deploy/
  ihmod-admin.service       # systemd 单元
  backup.sh                 # SQLite + media 备份脚本
  ufw-setup.md              # 防火墙规则说明
test/
data/                       # 运行时卷：admin.db、uploads、repo、manifest、backups
```

## 7. 数据模型

### 7.1 DDL（SQLite）

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

CREATE TABLE users (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  username      TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);

CREATE TABLE sessions (
  id         TEXT PRIMARY KEY,          -- 256-bit 随机
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
  id           TEXT PRIMARY KEY,        -- ULID
  title        TEXT NOT NULL,
  slug         TEXT NOT NULL,
  category_id  INTEGER REFERENCES categories(id) ON DELETE SET NULL,
  description  TEXT,
  top_img      TEXT NOT NULL DEFAULT 'false',  -- 'false' 或图片 URL
  content_md   TEXT NOT NULL DEFAULT '',
  status       TEXT NOT NULL DEFAULT 'draft'
               CHECK (status IN ('draft', 'published')),
  abbrlink     TEXT NOT NULL UNIQUE,    -- 8 位十六进制
  source_path  TEXT NOT NULL UNIQUE,    -- 相对 source/_posts，如 网安/信息收集.md
  post_date    TEXT NOT NULL,           -- YYYY-MM-DD HH:mm:ss
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL,
  published_at TEXT,
  deleted_at   TEXT,                    -- 软删除
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
  id            TEXT PRIMARY KEY,       -- ULID
  original_name TEXT NOT NULL,
  stored_name   TEXT NOT NULL UNIQUE,   -- <id>.<ext>，Markdown 中使用
  mime          TEXT NOT NULL,
  size          INTEGER NOT NULL,
  width         INTEGER,
  height        INTEGER,
  created_at    TEXT NOT NULL
);

CREATE TABLE publish_state (
  id          INTEGER PRIMARY KEY CHECK (id = 1),
  status      TEXT NOT NULL DEFAULT 'idle',  -- idle|running|success|error
  message     TEXT,
  commit_sha  TEXT,
  started_at  TEXT,
  finished_at TEXT
);
```

### 7.2 字段约定

- 时间统一存 `YYYY-MM-DD HH:mm:ss`（本地时区按 VPS 时区，部署时固定为 `Asia/Shanghai`），便于直接写入 Hexo frontmatter。
- `posts.id` 使用 ULID，便于排序且不暴露自增数量。
- `source_path` 是文件位置的唯一依据；**文章的 `slug` 或分类字段被编辑时**会改变它，导出时先删旧路径再写新路径。重命名分类本身不批量改动已有文章（见 9.2）。
- `content_hash` = SHA-256(标题 + 正文 + 分类 + 标签等表单字段)，用于自动保存时判断“无变化不写入”；发布清单另存导出文件哈希，二者用途不同。
- `abbrlink`：已有文章保留原值；新文章用 `crypto.randomBytes(4).toString('hex')` 生成 8 位十六进制，插入前查重。
- 分类：一篇文章一个分类（与现有 frontmatter 一致），存为字符串。
- 标签：多对多，导出为 YAML 列表。
- `top_img`：默认字符串 `false`，与现有文章一致。

## 8. 后台功能与路由

### 8.1 页面

| 路径 | 功能 |
|---|---|
| `GET /login` / `POST /login` / `POST /logout` | 登录、登出 |
| `GET /` | 仪表盘：文章数、草稿数、最近发布状态 |
| `GET /posts` | 文章列表：搜索、按分类/状态过滤、排序 |
| `GET /posts/new` / `POST /posts` | 新建 |
| `GET /posts/:id` / `POST /posts/:id` | 编辑并保存 |
| `POST /posts/:id/delete` | 软删除（下次发布移除文件；列表可筛选“回收站”） |
| `POST /posts/:id/restore` / `POST /posts/:id/purge` | 从回收站恢复 / 彻底删除记录 |
| `POST /posts/:id/unpublish` | 取消发布（改为草稿，下次发布移除文件） |
| `GET /categories` / `POST /categories` / `POST /categories/:id` / `POST /categories/:id/delete` | 分类管理（删除前检查引用） |
| `GET /tags` / 同上 | 标签管理 |
| `GET /media` / `POST /media` / `POST /media/:id/delete` | 媒体库（删除前检查是否被未删除文章引用；被引用则阻止并列出文章） |
| `POST /publish` | 全站发布（导出 + git push） |
| `GET /publish` | 发布状态与历史 |
| `GET /healthz` | 健康检查（DB 可读） |

### 8.2 JSON API（编辑器异步调用）

| 路径 | 功能 |
|---|---|
| `POST /api/posts/:id/autosave` | 自动保存表单字段（不触发发布），返回 `updated_at` |
| `POST /api/media` | multipart 上传图片，返回 `{ name, url }` |
| `GET /api/publish/state` | 轮询发布状态 |
| `POST /api/publish` | 触发发布（与 `POST /publish` 等价） |

### 8.3 编辑器

- 字段：标题、slug、分类、标签（逗号/回车分词）、摘要、顶部图、日期、状态、正文。
- Vditor 使用分屏预览模式，图片上传走 `/api/media`，插入 `/img/uploads/<stored_name>`。
- 自动保存：输入停止 2 秒后 debounce 调 `/api/posts/:id/autosave`；每 30 秒兜底保存；离开页面有未保存提醒。
- **无变化不写入**：自动保存先比较标题、正文、分类、标签等字段的哈希；没有实际变化时直接返回，不更新 `updated_at`，避免已发布文章因定时保存产生无意义的 `updated` 变化和 Git 提交。
- 保存草稿与发布分离：保存只写 SQLite；发布按钮先保存再调用 `/api/publish`。
- 预览：Vditor 客户端预览 + DOMPurify 净化；服务端可用 marked 做一致性校验（可选）。
- 移动端：基本可用（列表/编辑单列），不专门优化。

## 9. 内容映射（SQLite → Hexo Markdown）

### 9.1 frontmatter 格式

固定字段顺序，使用 js-yaml dump，保证稳定 diff：

```yaml
---
title: 信息收集
date: 2026-11-12 10:00:00
updated: 2026-11-12 10:00:00
tags:
  - 信息收集
categories: 网安
top_img: false
description: 信息收集
abbrlink: 312e3d12
---
正文内容，保留原有 Markdown。
```

规则：

- `date` / `updated` 格式 `YYYY-MM-DD HH:mm:ss`；`date` 取 `post_date`，`updated` 取 `updated_at`。
- `tags` 为空时省略该字段；非空时按标签名排序输出，保证 diff 稳定。`categories` 为空时写 `未分类`。
- `top_img` 值为字符串 `false` 时渲染成 YAML 布尔 `top_img: false`（与现有文件一致）；否则按 URL 字符串输出；不写 `null`。
- `description` 为空时省略；非空时用 js-yaml 正确转义（含冒号、`#`、引号等）。
- `abbrlink` 必须存在；若值全为数字，加引号确保 YAML 解析为字符串。
- 正文末尾保证一个换行；文件使用 UTF-8、LF。

### 9.2 文件路径

- 新文章：`<分类名>/<slug>.md`；无分类用 `未分类/`。
- `slug` 默认由标题生成，允许中文、字母、数字、`-`、`_`；替换空格和路径分隔符；同分类内查重，冲突时追加 `-2`、`-3`。
- 已有文章：保留导入时的 `source_path`（如 `编程/MyBatis-Plus.md`），避免改动已有文件。
- 修改文章的分类或 slug：生成新路径，发布时删除旧路径（仅当旧路径在 manifest 中）。
- 重命名分类：**不批量改已有文章的 `source_path`**，只影响之后新建文章的路径；已有文章只有在编辑时**显式改变分类或 slug** 才移动文件。后台在分类改名时给出提示，避免一次改名导致大量文件移动和 Git 噪音。

## 10. 发布流水线

### 10.1 导出算法

发布由单实例进程内互斥锁 + SQLite `publish_state` 行锁保护，禁止并发。步骤：

1. 获取锁，`publish_state.status = running`。
2. 确保 `data/repo` 存在且为 `main`；`git fetch origin main`。
3. 检查工作区：`source/_posts` 与 `source/img/uploads` 之外的未提交改动 → 中止并报错（避免覆盖人工修改）。
4. 从 SQLite 构建期望文件集：
   - 所有 `status='published' AND deleted_at IS NULL` 的文章 → 渲染 Markdown。
   - 扫描已发布文章正文中的 `/img/uploads/<stored_name>` 引用，校验 `media` 记录和 `data/uploads/` 文件都存在；缺失则中止发布并指出文章与文件名，避免生成坏链。
   - 所有被已发布文章引用的媒体 → `source/img/uploads/<stored_name>`。
5. 与 `publish-manifest.json` 比较：
   - 新增/内容变化的文件写入（临时文件 + rename，原子替换）。
   - manifest 中存在但期望集合中不存在的文件删除（**只删除 manifest 管理的路径**）。
   - 媒体文件按哈希比较，变化才复制。
6. 原子更新 manifest。
7. `git add -A -- source/_posts source/img/uploads`。
8. 若无 staged 变化且 `origin/main..HEAD` 无待推送提交 → 记 success，结束。
9. `git commit -m "chore(publish): N posts"`（仅当有变化）。
10. `git fetch origin main`；若落后则 `git rebase origin/main`；冲突则 `git rebase --abort`，报错，保留本地状态，不 force push。
11. `git push origin main`。
12. 记 `commit_sha`、`finished_at`、`status=success`；失败记 `error` + 错误信息。
13. 释放锁。

### 10.2 失败与重试

- 推送失败（网络/密钥/冲突）：SQLite 数据不受影响；`publish_state` 记录错误；页面可重试。
- 重试时若本地已有未推送 commit，第 8 步会检测到并继续走 fetch/rebase/push。
- 永不 `git push --force`；永不 `git reset --hard` 用户仓库。
- 发布中进程崩溃：下次启动把 `running` 改为 `error`（超时判定），并提示检查 `data/repo`。

### 10.3 Git 凭据

- 使用 **ed25519 部署密钥**（无口令），GitHub 仓库 Deploy key 勾选 write。
- 私钥路径由 `GIT_SSH_KEY_PATH` 指定，放在 `/opt/ihmod-admin/secrets/deploy_key`，权限 600，只对应用进程用户可读。
- `core.sshCommand = ssh -i <GIT_SSH_KEY_PATH> -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new`。
- 仓库级 `user.name = iHMOD Admin`、`user.email = admin@ihmod.local`。
- 现有根目录 `github` 私钥：移出项目、在 GitHub 删除旧 deploy key、加入 `.gitignore`。后台不使用它。

### 10.4 GitHub Actions

`.github/workflows/deploy.yml`：

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
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: public
      - uses: actions/deploy-pages@v4
```

仓库设置：

- Settings → Pages → Source: **GitHub Actions**（不再用分支部署）。
- 首次部署前，旧 `master` 分支的构建产物保留为备份，不删除。
- 上面的 Actions 版本号以实现时 GitHub 官方 Pages 模板为准（v3/v4/v5 会随官方更新）。
- 移除 `_config.yml` 的 `deploy:` 段和 `hexo-deployer-git` 依赖，避免双发布机制。

## 11. 安全设计

### 11.1 访问与传输

用户选择：**公网 IP + ufw 白名单 + HTTP**。

- 后台监听 `0.0.0.0:8080`；`ufw` 只允许用户固定 IP 访问 8080，其余入站拒绝（SSH 22 仍开放，建议仅密钥登录）。
- **明确风险**：HTTP 明文，登录密码与 Cookie 可能在网络路径被截获。仅建议在可信固定网络使用；公共 Wi-Fi 不要登录。
- 缓解：强密码、登录限流、短会话、CSRF、登录失败日志。
- 后续升级路径（不改变应用架构）：
  - 加 Caddy + 域名 → 自动 HTTPS。
  - 或改监听 `127.0.0.1` → SSH 隧道 / Tailscale / Cloudflare Tunnel。
- 动态公网 IP：IP 变化会被防火墙挡住；通过 SSH 登录 VPS 更新 `ufw` 规则（提供一条命令）。SSH 始终可达，避免锁死。

### 11.2 认证与会话

- 单管理员；首次启动由环境变量 `ADMIN_USERNAME` / `ADMIN_PASSWORD` 创建，创建后环境变量密码失效，改密码在后台完成。
- 忘记密码：VPS 上执行 `npm run reset-password`（CLI 提示输入新密码），不依赖邮件服务。
- 密码策略：至少 12 位；argon2id 默认参数。
- 登录限流：同一 IP + 用户名 5 次/15 分钟；失败日志记录 IP、UA、时间，不记录密码。
- 会话：随机 256-bit ID 存 DB；HttpOnly、SameSite=Lax；有效 7 天，滑动续期；可在“设置”里“退出所有设备”。
- 无 HTTPS 时 Cookie 不设 `Secure`（否则浏览器不发送）；有 HTTPS 后启用。

### 11.3 请求安全

- CSRF：双提交 Cookie + 状态变更请求校验 `Origin`/`Referer`。
- EJS 默认转义；禁止对用户内容使用 `<%- %>`。
- Markdown 预览用 DOMPurify 净化；禁止 `javascript:` 链接、事件属性。
- 安全响应头（Helmet）：CSP 仅允许 self + Vditor 所需的内联样式；`X-Content-Type-Options: nosniff`；`Referrer-Policy: no-referrer`；`frame-ancestors 'none'`。
- SQL：Drizzle + better-sqlite3 预编译语句，禁止字符串拼接 SQL。
- 上传：
  - 仅允许 `image/png`、`image/jpeg`、`image/gif`、`image/webp`；用 magic bytes 校验，不只信 Content-Type。
  - 拒绝 SVG（可含脚本）。
  - 单文件 ≤ 10MB；随机文件名 `<ulid>.<ext>`；原文件名仅存 DB。
  - 上传目录不在静态根目录，通过受控路由读取并设置正确 `Content-Type`。
- 部署密钥不进镜像、不进 Git、不进日志。

### 11.4 备份

- 每日 03:00（VPS 时区）执行 `sqlite3 data/admin.db ".backup 'backups/admin-YYYYMMDD.db'"`，保留 14 天。
- 同日打包 `data/uploads/`；`data/repo/` 可重新 clone，不必备份。
- 备份目录放在应用代码目录之外：`/var/backups/ihmod-admin/`，避免重装/升级时丢失；可选 `rclone` 同步到对象存储。
- 文档包含恢复步骤：停服务 → 还原 DB/media → 启动 → 重新发布。

## 12. 部署

### 12.1 进程与端口

采用**原生部署**：Node 直接跑在 VPS 上，由 systemd 管理，监听 `0.0.0.0:8080`。不使用 Docker，因此不存在 Docker 端口绕过 `ufw` 的问题，防火墙规则直接生效。用户后续自行在 VPS 上完成原生部署；仓库提供 systemd 单元、启动脚本和部署说明。

systemd 单元（完整文件放 `deploy/ihmod-admin.service`）：

```ini
# /etc/systemd/system/ihmod-admin.service
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

- `/etc/ihmod-admin.env`（600）：`DATABASE_PATH`、`UPLOADS_DIR`、`REPO_DIR`、`REPO_URL`、`REPO_BRANCH`、`GIT_SSH_KEY_PATH`、`SESSION_SECRET`、`ADMIN_USERNAME`、`ADMIN_PASSWORD`（仅首次）、`HOST=0.0.0.0`、`PORT=8080`、`TZ=Asia/Shanghai`。
- `data/` 保存 `admin.db`、`uploads/`、`repo/`、`publish-manifest.json`；备份放 `/var/backups/ihmod-admin/`（见 11.4）。
- Node 使用系统级 `/usr/bin/node`（若用 nvm，`ProtectHome=true` 会挡住，需改 `ExecStart` 和 `ReadOnlyPaths`）。
- 首次启动：执行迁移 → 克隆博客仓库（若无）→ 创建管理员 → 启动服务。
- 健康检查：`/healthz` 检查 DB 与仓库目录。

### 12.2 防火墙

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp                 # 建议改成仅允许管理 IP + 密钥登录
ufw allow from <你的公网IP> to any port 8080 proto tcp
ufw enable
```

- 不开放 80/443。
- 用 `ss -lntp` 确认 8080 只被 Node 进程监听，且没有其他服务意外暴露。
- 公网 IP 变化时在 SSH 里更新规则：
  ```bash
  ufw delete allow from <旧IP> to any port 8080 proto tcp
  ufw allow from <新IP> to any port 8080 proto tcp
  ```

## 13. 初始导入

CLI：`npm run import -- --posts <博客仓库>/source/_posts`

流程：

1. 递归读取 `**/*.md`，跳过 `.obsidian/` 等隐藏目录；路径分隔符统一为 `/`；gray-matter 解析 frontmatter。
2. 映射：
   - `title` ← frontmatter.title（缺失时用文件名）
   - `categories` ← frontmatter.categories（字符串取原值；数组取第一项；缺失用父目录名；再无则 `未分类`）
   - `tags` ← frontmatter.tags（数组或逗号分隔字符串）
   - `top_img` ← frontmatter.top_img（缺省 `false`）
   - `description` ← frontmatter.description（可空）
   - `abbrlink` ← 原值；缺失则生成并保证唯一
   - `post_date` ← frontmatter.date；否则日记文件名 `YYMMDD.md` 解析为 `20YY-MM-DD 00:00:00`；否则文件 mtime
   - `source_path` ← 相对路径
   - `content_md` ← 正文
   - `status` ← published
   - `updated_at` ← 与 `post_date` 相同（避免首次发布把所有文件都改成导入当天）
3. 按 `source_path` upsert，重复执行幂等。
4. 自动创建缺失分类/标签。
5. 输出报告：导入/更新/跳过数量，以及缺 abbrlink、日期回退等需人工确认的项。
6. 导入后执行一次 dry-run 导出，逐文件对比原文件（允许 `date`/`updated` 新增），人工确认后正式发布。

已知需人工确认：`编程/MyBatis-Plus.md` 的 `categories: 基础` 与目录 `编程` 不一致，按 frontmatter 的 `基础` 导入，`source_path` 仍保留 `编程/MyBatis-Plus.md`。

## 14. 错误处理与可观测性

- 统一错误页/JSON 错误结构；生产环境不返回堆栈。
- 关键操作日志：登录失败、发布开始/成功/失败、git 命令退出码、上传拒绝原因。
- 日志脱敏：Cookie、Authorization、密码、密钥路径内容不落日志。
- 发布过程记录每步耗时，便于定位网络或 git 问题。
- 健康检查失败返回 503。

## 15. 测试策略

- 单元测试（Vitest）：
  - abbrlink 生成/查重。
  - frontmatter 渲染（特殊字符、中文、空字段、多标签）。
  - slug 生成与路径映射。
  - import 解析（无 frontmatter、数组分类、日记文件名）。
  - manifest diff（新增/修改/删除，不误删非托管文件）。
- 集成测试：
  - 临时 SQLite + 临时裸 Git 仓库，验证发布：新建/修改/取消发布/删除/无变化时不产生 commit。
  - git 冲突/网络失败时返回错误且不 force push。
  - 认证：登录、会话过期、CSRF、限流。
- 端到端（可选）：Playwright 登录 → 新建文章 → 发布 → 检查仓库文件。
- 上线前：先在测试仓库/测试分支跑通完整 Actions 部署，再切正式仓库。

## 16. 实施阶段

1. **仓库与 CI 准备**：备份现有产物；改造 `ihmod.github.io`（源码 `main` + Pages Actions）；跑通一次纯 Hexo 构建部署。
2. **后台骨架**：Fastify + 配置 + Drizzle 迁移 + 登录/会话 + 布局。
3. **内容管理**：分类、标签、文章 CRUD。
4. **导入**：导入现有 13 篇，dry-run 对比。
5. **编辑器**：Vditor + 自动保存 + 图片上传/媒体库。
6. **发布**：导出器 + manifest + git push + 发布状态页。
7. **部署与加固**：原生 Node + systemd、ufw、备份、健康检查、日志。
8. **验收**：新文章发布、修改、取消发布、失败重试、手机访问、恢复演练。

## 17. 风险与应对

| 风险 | 应对 |
|---|---|
| HTTP 明文凭据 | 已告知；强密码/限流/短会话；后续可加 HTTPS 或隧道 |
| 动态 IP 被防火墙锁死 | SSH 始终可达；一条命令更新白名单 |
| 直接改仓库被覆盖 | manifest 只删自己管理的文件；文档明确“后台是唯一编辑入口” |
| SQLite 单点 | 每日备份 + 恢复演练；单机个人博客足够 |
| Git 冲突/推送失败 | 不 force push；错误显式呈现；可重试 |
| GitHub Actions 首次配置失败 | 先测试仓库验证；保留旧 `master` 产物 |
| 现有私钥泄露 | 移出项目、吊销旧 key、加入 .gitignore |
| 主题 27MB 克隆慢 | 仅首次克隆；之后增量 fetch |
| 日记日期不准确 | 导入时按文件名解析并报告；后台可改 |

## 18. 待实现时确认的细节

以下不阻塞设计，进入实现计划时再定：

- VPS 的 Linux 发行版、Node 版本、8080 是否被占用、是否有 systemd 权限。
- 用户公网 IP（用于 ufw 规则，不写入文档）。
- 后台仓库名最终用 `ihmod/blog-admin` 还是其他。
- 是否需要“关于/友链”等页面的后台编辑（v1 不做）。
- 是否需要文章版本历史（v1 不做）。
