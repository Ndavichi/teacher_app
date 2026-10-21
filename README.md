# TeacherTools â€” Teaching Document Platform (React + Node/Express + MySQL)

A platform for Zambian teachers to generate teaching documents â€” **schemes of work, lesson plans, records of work, test papers and CBC grids** â€” aligned to the **2013 OBE** and **2024 CBC** curricula, with AI assistance and PDF/Word export.

This repository is mid-migration from the original PHP application to a modern React + Node/Express stack. The legacy PHP app now lives in [`legacy-php/`](./legacy-php) for reference and rollback, and the new stack runs from the repository root under `server/` (API) and `client/` (web).

> **Note on "MERN":** this stack is React + Node/Express + **MySQL** (not MongoDB). The existing 38-table MySQL schema is reused via Knex so no data migration is required. Everywhere else it follows the MERN pattern (SPA client, JSON API, JWT auth).

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18 + TypeScript, Vite, React Router, TanStack React Query, Zustand, Tailwind CSS |
| **Backend** | Node.js 18+ (tested on 24) + Express + TypeScript |
| **Database** | MySQL 8 / MariaDB (existing `my_file` schema) |
| **Query layer** | Knex.js + mysql2 |
| **Auth** | JWT access + refresh in httpOnly cookies; bcrypt verification against existing `$2y$` PHP hashes; **Google Sign-In** (GIS ID-token); role-based access control |
| **Cache / OTP store** | Redis (optional â€” degrades to in-memory) |
| **AI** | Multi-provider service behind one `callAi()` manager: free self-hosted **Kaggle Qwen3-30B-A3B** preferred when reachable, then **OpenRouter** (free + very cheap but effective models), then the rest of the pool (OpenCode, GitHub Models, Google, Groq, DeepSeek, Cloudflare) with automatic fallback, an env-configurable order (`AI_PROVIDER` / `AI_FALLBACK_PROVIDER`) and a central no-invention grounding rule |
| **Docs** | PDF/DOCX generation *(migration in progress â€” see status)* |
| **Payments** | Lenco (card + mobile money) |
| **Storage** | Local + IDrive E2 (S3-compatible) |

## Repository Layout

```
/
â”œâ”€â”€ server/               # Express + TypeScript API
â”‚   â”œâ”€â”€ src/
â”‚   â”‚   â”œâ”€â”€ config/       # config, database (Knex), settings, jwt
â”‚   â”‚   â”œâ”€â”€ middleware/   # auth, rbac, error, rate limiting
â”‚   â”‚   â”œâ”€â”€ modules/      # auth, public, portal, admin, payments
â”‚   â”‚   â”œâ”€â”€ services/     # ai, mail, redis, sessionStore, otp, audit
â”‚   â”‚   â”œâ”€â”€ scripts/      # migrate, seed
â”‚   â”‚   â””â”€â”€ utils/        # http, crypto, cookies, logger
â”‚   â””â”€â”€ .env.example
â”œâ”€â”€ client/               # React SPA (Vite)
â”‚   â””â”€â”€ src/
â”‚       â”œâ”€â”€ api / lib/    # axios client
â”‚       â”œâ”€â”€ components/   # layout, shared UI
â”‚       â”œâ”€â”€ pages/        # public, portal, admin pages
â”‚       â””â”€â”€ store/        # Zustand auth store
â”œâ”€â”€ legacy-php/           # Original PHP application (reference / rollback)
â”œâ”€â”€ database/             # SQL dumps for the shared schema
â””â”€â”€ package.json          # npm workspaces root
```

## Prerequisites

- **Node.js** 18+ (tested on 24)
- **npm** 9+
- **MySQL 8 / MariaDB** running (e.g. via WAMP/XAMPP) with the existing schema
- **Redis** *(optional)* â€” improves OTP/AI caching; without it the API uses an in-memory store

## Quick Start

All commands run from the repository root:

```bash
npm install
```

### 1. Environment

The API reads `server/.env`. Copy the example (or reuse the existing one) and set your values:

```bash
cp server/.env.example server/.env
```

Key settings:

```env
NODE_ENV=development
PORT=4000
CLIENT_URL=http://localhost:5173

DB_HOST=localhost
DB_PORT=3306
DB_NAME=my_file
DB_USER=root
DB_PASS=yourpassword

JWT_SECRET=generate-a-strong-secret
JWT_REFRESH_SECRET=generate-another-strong-secret

REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# Google Sign-In (optional)
GOOGLE_OAUTH_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# AI provider layer (optional â€” defaults shown). See "The AI provider layer".
AI_PROVIDER=kaggle                    # tried first
AI_FALLBACK_PROVIDER=openrouter       # comma list; blank = built-in order
KAGGLE_ENABLED=1
KAGGLE_BASE_URL=https://kaggle.dtech.edu.eu.org
KAGGLE_MODEL_ID=qwen3-30b-a3b
```

The service reads configuration from `server/.env` first, then falls back to the `adm_system_settings` table (same dual-source behaviour as the PHP app).

### Google Sign-In setup (optional)

1. In the [Google Cloud Console](https://console.cloud.google.com/apis/credentials) create an **OAuth 2.0 Client ID** (Web application).
2. Add authorised JavaScript origins: `http://localhost:5173` and your production origin. **No redirect URI is required** â€” the app uses the Google Identity Services ID-token flow, which has no callback URL.
3. Put the client ID/secret in `server/.env` (`GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET`) and the **client ID only** in `client/.env` as `VITE_GOOGLE_CLIENT_ID`.
   - Alternatively, paste the whole downloaded credentials JSON as a single `GOOGLE_DETAILS={"web":{...}}` value in `server/.env` â€” the client ID and secret are read from it automatically. (`client/.env` still needs `VITE_GOOGLE_CLIENT_ID`.)
4. Run the additive schema migration once:

   ```bash
   npm run alter:google-auth
   ```

When configured, a **Continue with Google** button appears on the sign-in and sign-up pages. If unset, the button is hidden and email/password auth continues to work. Google accounts are created verified; existing emails are auto-linked, and new users are sent to `/onboarding` to add school details and subjects.

### Kaggle Qwen3-30B-A3B setup (optional â€” free primary AI provider)

The platform can use a free, self-hosted **Qwen3-30B-A3B** on Kaggle as its primary AI provider, falling back automatically to the paid database model pool when the Kaggle tunnel is down. Full instructions are in [`kaggle/README.md`](./kaggle/README.md).

Summary:

1. **Cloudflare Tunnel (once)** â€” in Zero Trust â†’ Networks â†’ Tunnels create a tunnel and add a **Public Hostname** `kaggle.dtech.edu.eu.org` â†’ Service `HTTP` â†’ `localhost:8000`. This auto-creates a proxied CNAME `kaggle.dtech.edu.eu.org â†’ <tunnel-id>.cfargotunnel.com`; do **not** add a manual A record. Copy the tunnel token. (Use a hostname that is not already serving another service â€” `ai.dtechzm.com` currently serves Ollama.)
2. **Kaggle** â€” open [`kaggle/qwen3_server.ipynb`](./kaggle/qwen3_server.ipynb) with **GPU T4 x2** and **Internet: On**, add Secrets `KAGGLE_API_KEY` and `CLOUDFLARE_TUNNEL_TOKEN`, then run all cells. The last cell keeps the session alive (~12h Kaggle limit).
3. **Register in the app** â€” run the additive setup once, then set the endpoint:

   ```bash
   npm run alter:kaggle   # adds editable KAGGLE_* rows to adm_system_settings (group `ai`)
   npm run seed:kaggle    # registers the `kaggle` provider + `Qwen3-30B-A3B` model
   ```

   ```env
   KAGGLE_ENABLED=1
   KAGGLE_BASE_URL=https://kaggle.dtech.edu.eu.org
   KAGGLE_API_KEY=<same token as the Kaggle secret>
   KAGGLE_MODEL_ID=qwen3-30b-a3b
   ```

   These can also be edited live at **`/admin/system-settings`** (group `ai`), and the provider/models at **`/admin/ai-providers`** and **`/admin/ai-models`**. If Kaggle is not set up, leave `KAGGLE_ENABLED=0` and the database models handle all generation.

**Quick tunnel (no Cloudflare dashboard setup).** If you skip the named tunnel entirely, the notebook can create an ephemeral `*.trycloudflare.com` tunnel and publish its URL to the API. Set a shared secret and add it as a Kaggle secret:

```env
# server/.env (and add the same value as the Kaggle secret KAGGLE_REGISTER_TOKEN)
KAGGLE_REGISTER_TOKEN=<long random string>
```

Then in the notebook, leave `CLOUDFLARE_TUNNEL_TOKEN` unset; it will start a quick tunnel and `POST /api/ai/kaggle/register` with `{ base_url }` (host restricted to `*.trycloudflare.com`). The API updates `KAGGLE_BASE_URL` and clears the Kaggle health cache automatically. The API must be reachable from the internet at `APP_URL` for this to work.

### 2. Verify the Database

Confirms connectivity and reports the schema (read-only):

```bash
npm run migrate
```

### 3. Develop

```bash
npm run dev
```

Starts both processes:

- **API** â†’ http://localhost:4000  (health: `GET /api/health`)
- **Web** â†’ http://localhost:5173  (proxies `/api` and `/assets` to the API)

### 4. Production Build

```bash
npm run build      # builds server (tsc) and client (vite)
npm run start      # runs the compiled API
```

## Available Scripts

| Script | Description |
|--------|-------------|
| `npm run dev` | Run API + client together |
| `npm run dev:server` | API only (tsx watch) |
| `npm run dev:client` | Web only (Vite) |
| `npm run build` | Build server + client |
| `npm run typecheck` | Type-check both workspaces |
| `npm run migrate` | Verify DB connectivity and list tables/row counts |
| `npm run seed` | Non-destructive baseline report |
| `npm run alter:google-auth` | Additive migration for Google sign-in columns |
| `npm run alter:cloud-storage` | Create cloud-storage tables + seed providers and `.env` connections |
| `npm run alter:kaggle` | Add editable `KAGGLE_*` settings rows (group `ai`) for the Kaggle provider |
| `npm run alter:subject-uniqueness` | Make `usr_subjects` unique per subject (drops the per-level assignment) |
| `npm run seed:kaggle` | Register the `kaggle` AI provider and the `Qwen3-30B-A3B` model |
| `npm run check:ai` | Show Kaggle health and the fallback pool order |
| `npm run ollama:tunnel` | Open the SSH tunnel to Contabo Ollama (dev; keep the window open) |
| `npm run ollama:tunnel:autostart` | Install the hidden logon auto-start (self-healing tunnel, no admin) |
| `npm run ollama:tunnel:autostart:remove` | Remove the tunnel auto-start |

## API Endpoints

Base path: `/api`. Auth uses httpOnly cookies (`tt_access`) or a `Bearer` token.

### Auth (`/api/auth`)
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/login` | Teacher login (username/email + password); may return `otp_required` |
| `POST` | `/signup` | Start registration (deferred insert â€” nothing is written until OTP) |
| `POST` | `/otp/verify` | Verify signup/login OTP, create account, set cookies |
| `POST` | `/otp/resend` | Resend OTP |
| `POST` | `/password/forgot` \| `/password/verify-otp` \| `/password/reset` | Password reset flow |
| `GET` | `/google/config` | Whether Google Sign-In is enabled |
| `POST` | `/google` | Google GIS ID-token sign-in / sign-up (optional `plan_code`) |
| `GET` | `/me` | Current user |
| `POST` | `/refresh` \| `/logout` | Refresh / clear session |

### Public (`/api/public`)
`/plans`, `/frameworks`, `/subjects`, `/academic-levels`, `/departments`, `/locations`, `/check/username`, `/check/email`, `/syllabus/topics`, `POST /contact`.

### Portal (`/api/portal`, authenticated)
`/subjects` (GET/POST/PUT/DELETE â€” one row per subject per curriculum, **no level**), `/subjects/coverage` (which subject/Grade-Form pairs have syllabus), `/schemes` (list/get/delete), `/schemes/generate` (AI), `/schemes/:id/ai-edit` (AI), `/schemes/entries/:id` (PATCH inline save), `/lessons` (list/get/delete), `/lessons/generate` (AI), `/documents/:kind/:id/print` (rendered HTML), `/usage`, `/subscription`, `/profile` (GET/PUT).

> **Subjects are not tied to levels.** A teacher adds *Geography* once and picks the grade/form inside the scheme, lesson, record and grid wizards â€” `adm_syllabus` and every generated document keep their own `level_id`, so content is still filtered by the chosen grade/form. `/subjects/coverage` powers the smart Grade/Form picker.

### Payments (`/api/payments`, authenticated)
`/checkout`, `POST /verify`, `/history`, `POST /webhook` (unauthenticated gateway callback).

### Admin (`/api/admin`, platform admin)
`/auth/login|logout|me`, `/overview`, and config-driven CRUD for: `frameworks`, `subjects`, `levels`, `departments`, `locations`, `methodologies`, `system-settings`, `plans`, `prompt-templates`, `reference-books`.

### Cloud storage (`/api/admin/cloud-storage`, platform admin)
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/providers` | Provider catalogue (S3/object + cloud storage) |
| `GET/POST` | `/connections` | List / create connections (secrets masked) |
| `PUT/DELETE` | `/connections/:id` | Update / remove a connection |
| `POST` | `/connections/test` | Live connection test (S3 SigV4, WebDAV) |
| `GET` | `/connections/:id/reveal/:field` | Reveal one saved secret |
| `GET` | `/assignments` | Which storage type stores which asset |
| `PUT` | `/assignments/:assetType` | Assign a storage to an asset type |

Managed from the admin console at **`/admin/cloud-storage`** (Connections, Providers, Assignments). Seeded from `.env`: **Backblaze B2** (`B2_*`), IDrive e2 (`IDRIVE_*`) and Google Drive (Google OAuth credentials). Asset types: `profile_pictures`, `documents`, `newsletters`, `general`.

## How AI is used to generate documents

AI is the content engine; the application owns the structure, the curriculum data and the final layout. The model is asked to return **structured JSON**, never HTML, and a database template renders the professional document.

```
Teacher wizard (subject, level, topic, selected syllabus fields)
        â”‚
        â–¼
Prompt built from adm_ai_prompt_templates (per generation type + curriculum)
        â”‚  â† syllabus content is injected as fixed source text
        â–¼
callAi()  â†’  provider fallback chain
        â”‚      1. Kaggle Qwen3-30B-A3B  (free, preferred only while reachable)
        â”‚      2. OpenRouter (free + very cheap but effective models)
        â”‚      3. remaining pool (GitHub Models, Google, Groq, DeepSeek, â€¦)
        â–¼
Structured JSON  â†’  validate + one repair retry
        â”‚
        â”œâ”€â”€ invalid / all providers down â†’ deterministic syllabus draft (no AI cost)
        â–¼
Saved to MySQL (raw AI JSON kept in `remarks`)  â†’  html_template  â†’  browser print / PDF
```

### The AI provider layer (concept)

The application never talks to a model directly. Every generation â€” scheme, lesson, record, grid, AI-edit â€” goes through **one internal service** (`callAi()` in `server/src/services/ai.service.ts`). Callers hand it messages and options; they don't know or care which provider answered. This keeps the rest of the codebase provider-agnostic, so swapping the model (e.g. moving from the Contabo 4B CPU model to a bigger GPU model later) is a configuration change, not a rewrite.

```
                    TEACHER AI
                        â”‚
                        â–¼
                 Node / Express
                        â”‚
                AI Provider Manager          â† callAi()
                  â”Œâ”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”
                  â”‚           â”‚
             Kaggle         OpenRouter
          (Qwen3-30B-A3B)    (free + cheap models)
                  â”‚           â”‚
                  â””â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”˜
                        â–¼
                 JSON Validation
                        â”‚
                        â–¼
              Lesson Plan Renderer
                        â”‚
             â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¼â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
             â–¼          â–¼          â–¼
           HTML        PDF      Database
```

Three ideas hold it together:

1. **Provider Manager â€” a priority-ordered pool.** Providers and models live in `adm_ai_providers` + `adm_ai_models` (editable at `/admin/ai-providers` and `/admin/ai-models`). The order is: `AI_PROVIDER` first, then `AI_FALLBACK_PROVIDER` (comma-separated), then a built-in default (`kaggle, openrouter, opencode, github-models, google, groq, deepseek, cloudflare`). A reachable self-hosted provider (Kaggle) is always tried first, and **free models are preferred over paid** within a provider. `callAi()` walks the list (up to 10 candidates) and returns the first usable answer.

2. **Grounding â€” no invented facts.** A single rule is prepended to the system prompt of *every* call: use only facts supplied in the prompt, and never invent dates, teacher/school names, class sizes, references or curriculum requirements â€” leave a field blank rather than guess. It lives in `GROUNDING_PREAMBLE` and can be overridden or disabled (`AI_GROUNDING_RULES=0`) via System Settings, so no individual template can bypass it.

3. **Resilience and cost.** Per-request timeouts, cached health checks and circuit breakers stop a dead provider from wasting time or money; blank or token-truncated responses are treated as misses and fall through to the next candidate; and if the whole pool fails, a deterministic syllabus draft is produced at zero cost.

The primary provider (Kaggle Qwen3-30B-A3B) is a free self-hosted endpoint reached over a private Cloudflare tunnel; no inference port is exposed publicly.

### 1. Curriculum-grounded prompts
- Prompts live in `adm_ai_prompt_templates` and are selected by generation type (`Lesson Plan`, `Scheme of Work`, â€¦) and curriculum framework (`2013-REV` OBE / `CBC-2024`). They are editable under **Admin â†’ AI Prompt Templates**.
- The teacher's selected **syllabus** is injected into the prompt as the authoritative source. For schemes, official syllabus text (learning activities, expected standard, references) always **overrides** AI output; for lessons, outcomes/activities are taken from the syllabus, not invented.
- On top of each template's own instructions, `callAi()` prepends the central grounding rule (see *The AI provider layer*), so even a template that omits the instruction cannot invent facts.
- Prompts are rendered with `{placeholders}` (school, teacher, subject, level, topic, outcomes, methodology, resources, duration, etc.) in `server/src/services/promptTemplates.ts`.

### 2. Provider selection and fallback
`callAi()` in `server/src/services/ai.service.ts` loads active models from `adm_ai_models` + `adm_ai_providers` and tries them in order:
1. **Kaggle Qwen3-30B-A3B** â€” the free self-hosted OpenAI-compatible primary, preferred only when its health check passes (the endpoint must advertise `qwen3-30b-a3b`).
2. **OpenRouter** â€” the fallback: free models first, then very cheap but effective ones (free router, Qwen, Llama, â€¦). Set with `AI_FALLBACK_PROVIDER=openrouter`.
3. **Remaining pool** â€” ordered by `AI_PROVIDER` then `AI_FALLBACK_PROVIDER` (both editable in System Settings, group `ai`), defaulting to `kaggle, openrouter, opencode, github-models, google, groq, deepseek, cloudflare`. Free models sort ahead of paid ones within a provider, then by quality rating. A blank or unset `AI_FALLBACK_PROVIDER` keeps the built-in order.

Lessons always use the rich curriculum template (OBE/CBC) for every provider, so all models produce the same detailed plan. The layout used is stored in `remarks._template_code` so reprints match.

Provider/model rows, the Kaggle endpoint and the circuit-breaker tuning are editable at **`/admin/ai-providers`**, **`/admin/ai-models`** and **`/admin/system-settings`** (group `ai`).

> **Ollama retired.** A local CPU model (Qwen3 4B) used to be tried first, but on the Contabo CPU it took several minutes per document and caused generations to time out and fall back to the deterministic draft, so it was removed (provider deactivated, model deleted). The `scripts/ollama-*.ps1` tunnel helpers remain for reference but are no longer used. To run a local model again, add its provider/model row in the admin UI and set `AI_PROVIDER` / `AI_FALLBACK_PROVIDER` accordingly.

Reliability safeguards:
- Every request has a timeout (`AbortController`), so a hung endpoint can't block generation.
- Kaggle is health-checked (cached ~60s) and skipped entirely when unreachable. The check retries a few times because the Cloudflare tunnel occasionally returns a transient `530`, and `KAGGLE_HEALTH_TIMEOUT_MS` is 8000 so the ~2â€“3s round-trip doesn't trip the timeout.
- A **circuit breaker** opens after repeated network failures for a cooldown, so a dead provider never wastes latency or cost â€” Kaggle has a dedicated breaker and every cloud provider is protected by a generic one (`AI_PROVIDER_CIRCUIT_FAILURES` / `AI_PROVIDER_CIRCUIT_COOLDOWN_MS`).
- The whole chain runs inside a time budget (`AI_CHAIN_BUDGET_MS`, default 5 min); fallback attempts after the first are capped at `AI_FALLBACK_TIMEOUT_MS` (default 60s), so a slow provider can't stall the chain for long.
- Up to 10 candidates are attempted per request; the first usable response wins. Free models are tried before paid ones within a provider.
- When the class-exercise or homework toggle is off, the model is instructed to leave those fields empty (no wasted generation), and the printable template's conditional blocks omit the section.

### 3. Structured output, validation and repair
- Responses are parsed with `parseAiJson()` (tolerates markdown fences and finds the first balanced JSON object).
- `aiValidation.ts` runs a light structural check. If the output is unusable, one **repair retry** re-asks the model with the concrete problems; then the deterministic fallback is used.
- Field aliases (`topic`/`topic_name`, `teaching_methods`/`methodology`, â€¦) are normalised in `schemeEntry.ts`, so editable templates don't silently produce empty columns.

### 4. Zero-cost deterministic fallback
If every provider fails (or returns unusable JSON):
- **Lesson plans** are built from the selected syllabus fields by `lessonDraft.ts`, producing a complete, printable plan with no AI cost.
- **Schemes of work** already compose methods/T/L aids and official syllabus text deterministically, so no week is ever blank.
- The saved plan is flagged `ai_generated = 0` and the API returns `draft_used: true`; usage is logged as `Fallback`.

### 5. Cost control and observability
- Plan limits are checked before generation (`checkAiUsage`), and each call is recorded in `adm_ai_usage_logs` (`model_used`, tokens, `status` = `Success`/`Fallback`/`Failed`).
- The `kaggle` provider/model is registered with `is_free = 1`, so free-first selection is visible in the logs.
- Admin UI: **AI Providers** (`/admin/ai-providers`), **AI Models** (`/admin/ai-models`), **System Settings** (`/admin/system-settings`) for the Kaggle URL/token â€” all editable without a redeploy.

### 6. Rendering
The AI JSON is stored in `usr_lesson_plans.remarks` / scheme entries and merged into the curriculum's `html_template` by `renderDocumentHtml()`. The document is returned as HTML and printed/exported by the browser (server-side PDF is used for subscription invoices only). `GET /api/portal/documents/:kind/:id/print` serves the rendered document.

> **Current coverage:** AI generation is implemented for **schemes of work** (`POST /api/portal/schemes/generate`, plus AI edit) and **lesson plans** (`POST /api/portal/lessons/generate`). Records of work and CBC grids are still on the legacy PHP flow.

## Authentication Model

1. `POST /api/auth/login` verifies the password with **bcryptjs**, which reads the legacy `$2y$` hashes directly (no reset needed).
2. If `email_verified = 0`, an OTP (default 6 digits, 3-minute expiry, configurable via `otp_length`) is emailed and the user must verify before a session is issued.
3. On success the API sets an **access token** (short-lived) and **refresh token** (long-lived) as httpOnly cookies.
4. Registration is **deferred**: data is held in a Redis/in-memory OTP record until the code is verified, then the account, tenant reference, school profile and subjects are inserted in one transaction.

## Migration Status

| Area | Status |
|------|--------|
| Repo restructure (`legacy-php/`, `server/`, `client/`) | âœ… Done |
| API foundation (config, Knex, settings, auth, RBAC, rate limiting, mail, audit) | âœ… Done |
| Public site (home, curriculum, pricing, contact, login, signup, OTP, forgot password, checkout, welcome) | âœ… Done |
| Dark theme redesign (mobile-first) â€” public + auth pages, portal/admin shell | âœ… Done |
| Google Sign-In (GIS ID-token) + `/onboarding` for new Google users | âœ… Done |
| Super-admin cloud storage (providers catalogue, connections, asset assignment) | âœ… Done |
| Teacher portal â€” subjects, schemes (list/new/view + AI generate + inline edit), usage, profile | âœ… Done |
| Admin â€” overview + config CRUD (frameworks, subjects, levels, departments, locations, methodologies, system settings, plans, prompt templates, reference books) | âœ… Done |
| Payments â€” Lenco verify + webhook + activation | âœ… Done |
| AI engine â€” single `callAi()` provider layer: Ollama Qwen3 4B primary (local, free) â†’ Kaggle Qwen3-30B-A3B â†’ free-first model pool fallback, with env-configurable order, central no-invention grounding, health checks, circuit breakers, timeouts, validation and deterministic fallback | âœ… Done |
| Lessons and Records of Work generation (OBE + CBC) | âœ… Done |
| CBC Grid modules | ðŸš§ In progress |
| PDF/DOCX generation parity | ðŸš§ In progress |
| Microsoft Graph / OneDrive (E-Library) | ðŸš§ In progress |
| Admin syllabus editor, broadcast, activity-log viewer | ðŸš§ In progress |

Unimplemented portal/admin routes render a clear "being migrated" placeholder rather than failing silently.

## Legacy PHP

The original PHP application is preserved in `legacy-php/` (`index.php`, `config/`, `public/`, `user/`, `admin/`, `vendor/`, `composer.*`). It remains runnable for reference and rollback while the modern stack is completed. Its `.env` is `legacy-php/.env`.

Shared branding assets are served by the API at `/assets` from `legacy-php/assets`, so both stacks render identical logos and fonts during the transition.

## Security

- JWT access/refresh tokens in httpOnly cookies (`SameSite` configurable), bcrypt password hashing.
- Role-based access control (`requireAdmin`, `requireMember`, admin roles).
- Rate limiting on authentication and API routes (`express-rate-limit`).
- Helmet security headers; parameterised Knex queries.
- Audit logging to `adm_activity_logs` plus file trails under `server/logs/`.

## Troubleshooting

- **`database connection failed`** â€” confirm MySQL is running and `DB_*` in `server/.env` are correct. The API still boots in development if the DB is down.
- **`redis unavailable (degraded mode)`** â€” expected when Redis is not installed; OTP and caching fall back to memory.
- **Client can't reach API** â€” ensure the API is on port `4000`; Vite proxies `/api` automatically in dev.
- **Blank page after login** â€” check that cookies aren't blocked; in production set `COOKIE_SECURE=true` and serve over HTTPS.

## License

Dtechzm â€” All rights reserved.
