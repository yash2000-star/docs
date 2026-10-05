# DocGen: project status report

**Date:** 2026-10-05
**Repo analyzed:** `yash2000-star/DocGen` (7 commits, 2026-03-18 to 2026-04-21, about 10,500 lines of JS/JSX)
**Goal:** An open-source, Mintlify-style docs SaaS where users **bring their own AI key (BYOK)**. The platform provides only the "premium, human-like" doc-generation layer.

> **Note:** `yash2000-star/docs` (the repo this session opened) is the **unmodified Mintlify starter kit**. It has one "Initial commit" and no custom code. The real project is **`yash2000-star/DocGen`**, and this report covers that repo.

---

## 1. Summary

| Question | Answer |
|---|---|
| Where did you stop? | Commit `7905b0c` (2026-04-21) is titled **"WIP: Landing page light-mode overhaul and components"**. It's a large commit: 81 files, +7.6k lines. It added all the dashboard **settings pages** (API keys, billing, auth, domain, agent skills, danger zone, profile), the `llms.txt`/`skill.md` generator, and a landing-page redesign. Many of those pages are **UI only**, with no backend behind them. |
| Does it work end to end today? | **No.** Several blockers stop the main flow: the backend won't boot without Razorpay keys, every docs page request throws on Express 5, the free-preview route crashes, and new projects never get a member row, so generation returns 403. Details are in section 4. |
| Is the BYOK vision built? | **0%.** All AI calls use the **platform's own** Gemini/Vertex credentials. No feature lets a user enter or store a key. |
| How close is it to an open-source production SaaS? | About **25–30%**. The UI shell is strong (about 70% of the screens exist). The generation prompts are a good start. The backend is still a single-machine prototype: no DB schema in the repo, no job queue, no tests, no license, and several critical security holes. |

### Readiness by area

| Area | Built | Works | Production-ready |
|---|---|---|---|
| Landing page / marketing | ✅ | ✅ | 🟡 |
| Auth (GitHub OAuth, email OTP, TOTP 2FA via Supabase) | ✅ | ✅ (needs RLS check) | 🟡 |
| Dashboard shell, project list, navigation | ✅ | 🟡 | ❌ |
| AI generation pipeline (Architect → Ghostwriter → Critic) | ✅ | 🟡 (when the route isn't blocked) | ❌ |
| **BYOK (bring your own AI key)** | ❌ | ❌ | ❌ |
| Docs site renderer (MDX, sidebar, TOC, search, themes) | ✅ | ❌ (Express 5 route bug) | ❌ |
| AI chat over docs (RAG with pgvector) | ✅ | 🟡 | ❌ |
| Web editor | 🟡 | ❌ (reads a table nothing writes) | ❌ |
| Navigation editor | ✅ | 🟡 | ❌ |
| GitHub sync / webhook auto-update | 🟡 | 🟡 | ❌ |
| Custom domains | 🟡 (DB lookup only) | 🟡 | ❌ |
| Billing (Razorpay subscriptions) | 🟡 | 🟡 | ❌ |
| Analytics | 🟡 (UI only) | ❌ (nothing records views) | ❌ |
| Teams / roles (RBAC) | 🟡 (check only) | ❌ (no one creates members) | ❌ |
| API keys, Authentication gating | ❌ (mockups) | ❌ | ❌ |
| MCP / `llms.txt` / `skill.md` for agents | 🟡 | 🟡 (global, not per project) | ❌ |
| Self-hosting (schema, Docker, `.env.example`, docs) | ❌ | ❌ | ❌ |
| Tests / CI / license | ❌ | ❌ | ❌ |

---

## 2. Architecture (as built)

```
frontend/ (React 19 + Vite + Tailwind 4, SPA)
  ├─ Landing, Login (Supabase auth), Dashboard, Project pages, Settings
  └─ DocEngine → fetches raw MDX from backend → compiles in browser with @mdx-js/mdx evaluate()

backend/server.js (Express 5, single file, ~1000 lines, port 4000)
  ├─ POST /api/github        → clone repo → scan → Architect blueprint → Ghostwriter per page → Critic → write docs/*.mdx
  │                            → spawn build/build.js in background
  ├─ POST /api/public/github → anonymous "free preview" generation (PLG)
  ├─ POST /api/webhook       → GitHub push → regenerate changed files
  ├─ POST /api/ask           → RAG chat (Gemini embeddings + pgvector + Gemini 2.5 Pro)
  ├─ POST /api/settings, /api/docs, DELETE /api/projects, /api/github/save-config
  ├─ POST /api/billing/*     → Razorpay subscriptions + webhook
  ├─ POST /api/proxy         → API playground proxy
  └─ GET  /api/docs/..., /api/static/..., /llms.txt, /skill.md, /api/mcp/...

build/build.js (child process)
  └─ docs/<tenant>/<project>/*.mdx → dist/… (raw MDX, sidebar.json, search-index.json, toc, docs.json)
     + embeds every paragraph → Supabase doc_vectors

Storage: local filesystem (repos/, docs/, dist/) + Supabase Postgres (projects, users, usage, vectors…)
AI: Vertex AI Gemini 3 Flash (writer), Gemini 2.5 Flash-Lite (critic), Gemini 2.5 Pro (chat), gemini-embedding-001
```

**The best part of the codebase** is the three-stage generation design in `backend/aiWriter.js` and `backend/aiEditor.js`:

1. **Super-Architect:** reads README, `package.json` and source, then plans 5–10 conceptual pages (Introduction, Quickstart, Guides, Reference), not one page per file.
2. **Ghostwriter:** writes each page with Mintlify-style MDX components and site-wide cross-links.
3. **Critic:** cleans up MDX, removes made-up components, and checks params against the source.

This is the right foundation for "Mintlify-quality, human-like" docs. It needs to become provider-agnostic, budget tokens, and validate its own output (section 5).

---

## 3. What is complete

- **Landing page** and marketing UI.
- **Auth:** GitHub OAuth (asks for `repo` scope), email OTP, TOTP 2FA enroll/verify/unenroll, first-login user row, and claiming orphaned preview projects.
- **Dashboard UI:** project list, command palette, project overview with live status (Supabase Realtime plus polling fallback), new-project wizard with progress steps, settings layout with 8 sub-pages.
- **AI generation pipeline:** the Architect/Ghostwriter/Critic chain, a fallback blueprint if the Architect fails, and HTML-comment-to-MDX sanitizing.
- **Repo ingestion:** clones with `simple-git`, sorts code files from context files, skips files over 50 KB, and makes a rough folder classification (controllers, routes, models…).
- **Build worker:** heading IDs, search sections with deep links, TOC JSON, sidebar from frontmatter groups, `docs.json` deep-merged with defaults, scoped builds (`--scope`).
- **Docs renderer:** about 25 Mintlify-style components (Note, Tip, Warning, Steps, Cards, Tabs, CodeGroup, Accordion, ParamField, ResponseField, ApiPlayground, CodeGenerator…), Lucide icon resolver, theme presets, dark mode, logo, topbar links, CTA, footer socials, background image, breadcrumbs, Ctrl-K search modal, TOC.
- **RAG chat:** pgvector similarity search with a grounded answer prompt.
- **Navigation editor** (reorder groups/pages) and **branding settings**. Both save to `docgen.json`, trigger a rebuild, and optionally commit `docs.json` back to GitHub.
- **Billing plumbing:** Razorpay subscription creation, webhook HMAC check, plan states (active, cancelled, completed, halted), monthly usage limits.
- **Agent-readiness:** `llms.txt`, `skill.md`, `/.well-known/skills/index.json`, and a basic `/api/mcp/:tenant/:project` context endpoint.
- **Security started:** Helmet CSP, CSRF tokens, input escaping, HTTPS redirect, project roles check.

---

## 4. What is broken right now (verified)

I confirmed each item by running the code or tracing it line by line.

| # | Bug | Where | Effect |
|---|---|---|---|
| B1 | `new Razorpay({...})` runs at module load and throws when `RAZORPAY_KEY_ID` is missing | `backend/server.js:16` | **The backend won't start** unless the person has Razorpay keys. That blocks every open-source contributor. |
| B2 | In Express 5, `req.params.docPath` is an **array**, so `docPath.endsWith('.md')` throws | `backend/server.js:187-193` | **Every docs page request fails.** The viewer shows "Document not found". |
| B3 | `const generateDocs = require("../aiWriter")` gets an object, not a function. `scanCodeFiles()` returns `{codeFiles, contextFiles}`, not an array. | `backend/routes/public.js:11,112-121` | **The free-preview (PLG) flow always crashes.** |
| B4 | The public route inserts the project as `running` and then checks `status === 'running'`, so it always returns **409** | `backend/routes/public.js:39-92` | Same flow is blocked even after B3 is fixed. |
| B5 | Nothing ever inserts into `project_members`, but `/api/github` requires `admin`/`editor` there | `server.js:231-264`, `NewProject.jsx` | **Generating a new project returns 403** (unless your Supabase has a hidden trigger). The UI also ignores the 403 (`res.error` is never set on a fetch Response), so the spinner runs forever. |
| B6 | `/docs/:tenant/:version/*` is wrapped in `ProtectedRoute` | `frontend/src/App.jsx:167` | **Published docs need a login.** Anonymous readers and SEO crawlers get bounced. The PLG Explore flow also redirects visitors to `/`. |
| B7 | The editor reads/writes the Supabase `pages` table, but generation writes **files**. Nothing fills `pages`. | `Editor.jsx`, `routes/mcp.js` | The editor and MCP endpoint are always empty after generation. |
| B8 | Editor publish saves `<name>.md`, but the viewer only serves `.mdx`. The build is called without `--scope`. | `server.js:801,817` | Edited pages never show up. Each save **re-embeds every project on the server**. |
| B9 | The webhook uses the old one-file-per-page generator and writes `src/...` paths that aren't in navigation | `server.js:386-491` | Auto-updates create orphan pages outside the designed site. |
| B10 | Prompt/renderer contract mismatch: the AI is told `<Tooltip tip=…>` and `<Card icon="⚡">`, but the renderer expects `text` and **Lucide icon names** | `aiWriter.js:299,324` and `MdxComponents.jsx:245`, `Card.jsx` | Tooltips are blank and card icons fall back. |
| B11 | `build_events`, `page_views` and `user_onboarding` are read but **never written** | `ProjectOverview.jsx`, `Analytics.jsx` | Activity feed and analytics always show nothing. |
| B12 | Repos are cloned to `repos/<projectName>` with **no tenant in the path** | `backend/gitServices.js:8` | Two users with the same repo name overwrite each other mid-generation. |
| B13 | `GithubImporter.jsx` calls `setTenantName`, which is undefined. ESLint reports **31 errors**. | `frontend/src` | A dead component, plus lint debt. |

The frontend **builds** (`vite build` passes), but it ships as one 2.3 MB JS bundle with no code splitting.

---

## 5. What remains, by feature

### A. BYOK AI layer (core differentiator, not started)
- [ ] Provider abstraction: one `LLMClient` interface with adapters for **Anthropic, OpenAI, Google Gemini, OpenRouter, and any OpenAI-compatible endpoint (Ollama, LM Studio, Groq, Together…)**.
- [ ] **Encrypted key storage** per user/organization: AES-256-GCM envelope encryption with a server master key. Keys are never returned to the browser. Show only the last 4 characters.
- [ ] "Test connection" check and a model picker (writer model vs critic model).
- [ ] Remove all platform keys from generation, critic, chat and embeddings. Embeddings need a choice: use the user's key (OpenAI/Gemini/Voyage) or a **local open-source embedding model**, so search works with any provider.
- [ ] Token budgeting per model context window, cost estimate before a run, and per-run token/cost logging shown to the user.
- [ ] Retry/backoff on provider rate limits (replaces the fixed 3-second `setTimeout`).
- [ ] Clear errors for invalid key, out of quota, and model not found.

### B. Generation quality engine ("premium, human-like")
- [ ] Unify the **component contract**: one source file used by both the prompt and the renderer, so they can't drift apart (fixes B10).
- [ ] **Validate MDX:** compile each page with `@mdx-js/mdx` on the server and loop back to the Critic with the compile error if it fails.
- [ ] Smarter ingestion: respect `.gitignore`, skip binaries/lockfiles, rank files by importance (entry points, routes, public API), and summarize per file before the Architect step so big repos fit.
- [ ] Symbol extraction (tree-sitter) for accurate function signatures, and **OpenAPI/Swagger detection** for API reference pages.
- [ ] Mintlify-compatible frontmatter (`title`, `description`, `icon`) and `docs.json` `navigation` format.
- [ ] A small **quality eval set** (5–10 sample repos and a rubric) to compare prompt/model changes.
- [ ] Incremental regeneration: map changed files to affected pages through the blueprint (fixes B9).

### C. Job system and reliability
- [ ] Move generation out of the HTTP request into a **job queue**. `pg-boss` on your existing Postgres is the simplest; BullMQ with Redis is the alternative.
- [ ] Stream progress events (real steps, not a timer animation) and log builds to `build_events` (fixes B11).
- [ ] Per-job isolated workspace `repos/<tenant>/<project>/<jobId>` with cleanup (fixes B12).
- [ ] Move storage from local disk to object storage (Supabase Storage/S3) so the app can run on more than one server.

### D. Docs site (public viewer)
- [ ] Fix the Express 5 route (B2) and make docs **public** (B6).
- [ ] **Stop running untrusted MDX in the browser** (see S1). Compile on the server to static HTML/React, or use a safe MDX setup with no JS expressions.
- [ ] SSR/SSG for SEO (a docs product needs to be indexable), per-page meta/OG tags, sitemap.
- [ ] Mobile-responsive layout (the current 3-column layout uses fixed widths).

### E. Editor and Git sync
- [ ] Pick **one source of truth**. Recommended: **Git (like Mintlify)**, with the filesystem as a working copy and no `pages` table. That removes B7/B8.
- [ ] GitHub App instead of OAuth `repo` scope: install per repo, short-lived tokens, private repo cloning.
- [ ] Verify webhook signatures (`X-Hub-Signature-256`).
- [ ] "Docs autopilot": on push, regenerate the affected pages and **open a PR** instead of overwriting.

### F. Security hardening (required before going public)
See section 6.

### G. SaaS layer
- [ ] **Rethink pricing for BYOK.** You no longer pay for AI, so stop metering "generations". Charge for hosting, custom domains, team seats, private docs/auth gating, analytics and white-labeling. The open-source self-hosted version is free.
- [ ] Teams: organizations, invites, roles (fixes B5).
- [ ] Custom domains: DNS TXT verification and automatic SSL (Cloudflare for SaaS, Vercel Domains API, or Caddy on-demand TLS).
- [ ] Analytics ingestion: a lightweight page-view beacon and search-query logging.
- [ ] Real admin API keys (hashed), and the Authentication/private docs feature.
- [ ] A real MCP server per project (the current endpoint is a JSON dump), plus per-project `llms.txt` and `llms-full.txt`.
- [ ] Transactional email via a provider (not Gmail SMTP), with the domain taken from config instead of `localhost`.

### H. Open-source readiness
- [ ] **Add a LICENSE** (none exists). MIT/Apache-2.0 for full openness, or AGPL-3.0 to stop hosted clones. Many open-core SaaS projects use AGPL.
- [ ] Commit the **Supabase schema and migrations** (`supabase/migrations/*.sql`): tables, RLS policies, the `match_vectors` and `increment_usage` functions, and pgvector setup. Right now nobody else can run the project.
- [ ] `.env.example` for both apps, a config validator, and optional integrations (Razorpay and email off by default) (fixes B1).
- [ ] `docker-compose.yml` (app, worker, Postgres/Supabase), a one-command dev setup, and a self-hosting guide.
- [ ] Tests (unit tests for scanner/build, an end-to-end smoke test of generation with a mocked LLM), GitHub Actions CI, and fixing the lint errors.
- [ ] Consistent naming (`mintlify-clone`, DocGen and Prismify all appear in the code), plus README, CONTRIBUTING and a Code of Conduct.
- [ ] Split `server.js` into routes, services and middleware.

---

## 6. Security issues (fix before any public launch)

| # | Severity | Issue | Where |
|---|---|---|---|
| S1 | **Critical** | **Stored XSS through AI-generated MDX.** `evaluate()` runs any JS in the MDX inside the reader's browser. A malicious repo README can prompt-inject the AI into emitting `{fetch(...)}` or `export` code. That code runs on your origin, where the Supabase session sits in `localStorage`. The Critic is an LLM, not a sanitizer. | `frontend/src/pages/DocPage.jsx:62` |
| S2 | **Critical** | **Open SSRF proxy.** `/api/proxy` is unauthenticated and fetches any URL, including `169.254.169.254` cloud metadata and internal services. | `server.js:838` |
| S3 | **High** | **Unauthenticated, cost-burning endpoints.** `/api/ask`, `/api/webhook` (no signature check) and `/api/public/github` trigger clones and AI calls on *your* keys, with no rate limit. BYOK changes this, but each still needs auth and rate limits. | `server.js:386,904`, `routes/public.js` |
| S4 | High | **Anyone can webhook any tenant.** The tenant and repo come straight from the unsigned payload, so anyone can trigger regeneration for any project. | `server.js:398-400` |
| S5 | High | **Row-Level Security is unknown.** The frontend writes `projects`/`users` directly with the anon key (claiming orphans, deleting projects, setting custom domains). Without strict RLS, any user can take over or delete any project. The anon key is public in `frontend/.env` (it's designed to be, but only RLS keeps it safe). | `lib/auth.js`, `DangerZone.jsx`, `DomainSetup.jsx` |
| S6 | Medium | Path traversal: `tenant`/`version`/`docPath` are passed into `path.resolve`/`path.join` with no allowlist. `%2f` survives into a single param. | `server.js:197,719,775,795` |
| S7 | Medium | `csurf` is deprecated and unmaintained. CORS is hard-coded to `localhost:5173`. The CSP allows `unsafe-eval`. | `server.js` |
| S8 | Low | The GitHub OAuth `repo` scope gives full read/write to **all** of a user's repos, and the provider token is passed from the browser to the backend. A GitHub App fixes both. | `lib/auth.js:13` |

---

## 7. Proposed build order (one feature at a time)

Each step is small enough to build, test and merge on its own. This order makes the app runnable first, then builds the differentiator, then hardens it.

| Step | Feature | Why now | Size |
|---|---|---|---|
| **1** | **Make it run locally.** Fix B1–B6, add `.env.example`, commit the Supabase schema/migrations, update the README. | Nothing else can be tested until the core flow works end to end. | S–M |
| **2** | **BYOK provider layer.** `LLMClient` interface, Anthropic/OpenAI/Gemini/OpenAI-compatible adapters, encrypted key storage, settings UI with "test connection", model picker. | This is the product's core differentiator. | M |
| **3** | **Wire generation, critic, chat and embeddings to BYOK.** Remove platform keys and add token budgeting. | Finishes the BYOK promise. | M |
| **4** | **Safe MDX pipeline.** Shared component contract, server-side compile and validation with a retry loop, no browser `evaluate()` (fixes S1, B10). | Quality and security together. | M |
| **5** | **Job queue and real progress.** pg-boss worker, `build_events`, isolated workspaces (B11, B12). | Reliability, and large repos stop timing out. | M |
| **6** | **Generation quality v2.** Smarter ingestion and file ranking, OpenAPI detection, Mintlify frontmatter, eval set. | This is what makes the docs "premium". | L |
| **7** | **Security pass.** S2–S7, rate limiting, RLS policies, webhook signatures. | Needed before going public. | M |
| **8** | **Public docs site.** Public routes, SSR/SSG for SEO, mobile layout. | Needed before real users. | M–L |
| **9** | **Git as the source of truth.** GitHub App, private repos, editor on Git, auto-PR on push (B7–B9). | The Mintlify-style workflow. | L |
| **10** | **SaaS layer.** BYOK-appropriate pricing, teams/invites, custom domains with SSL, analytics, API keys, real MCP server. | Monetization. | L |
| **11** | **Open-source launch kit.** License, Docker Compose, CI and tests, contributing guide, self-hosting docs. | Launch. | M |

**Recommended next step: Step 1, "Make it run locally."** Without it, you can't see or test anything else. After that, Step 2 (BYOK) gives the project its identity.

---

## 8. Things to decide before Step 2

1. **License:** MIT/Apache (maximum adoption) or AGPL (protects against hosted forks)?
2. **Embeddings for search/chat:** require an embedding-capable key, or ship a local open-source embedding model so it works with any provider (including Anthropic-only users)?
3. **Key scope:** keys per user, per organization, or per project?
4. **Source of truth:** Git repo (Mintlify model, recommended) or the database?
5. **Payments:** keep Razorpay (India-first) or add Stripe for global users?
