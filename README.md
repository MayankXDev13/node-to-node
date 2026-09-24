# Node to Node

A visual workflow-automation builder (n8n / Zapier-style) built with Next.js. Design workflows on a canvas, then execute them as LangGraph state machines in Inngest background jobs — with AI steps, branching logic, and integrations for Telegram, GitHub, Notion, Google Calendar, and any HTTP API.

## How it works

1. **Build** — Drag trigger / transform / control / AI / action nodes on an `@xyflow/react` canvas (`modules/canvas`). Workflows autosave to Postgres as `nodes` + `edges` JSON.
2. **Trigger** — Run manually, via per-workflow webhook URL (`/api/webhooks/[workflowId]`), or via shared inbound webhooks for Telegram (`/api/webhooks/telegram`) and GitHub (`/api/webhooks/github`).
3. **Execute** — `startWorkflowExecution` creates an `Execution` + `ExecutionStep` rows, fires an Inngest `workflow/triggered` event, and `runWorkflow` compiles the canvas graph to LangGraph (`modules/engine/lib/compile-workflow.ts`) and invokes it node-by-node.
4. **Observe** — Node status streams back live via Inngest Realtime (`executionChannel`, `node-status` topic) and is overlaid on the canvas; full run history persists in `Execution` / `ExecutionStep`.

Templating uses `{{field}}` interpolation (`modules/engine/lib/template.ts`), so outputs of one node (e.g. `{{summary}}`, `{{chatId}}`) flow into the next node's config.

## Features

- **Visual canvas**: drag-drop nodes, branching edges, autosave, run overlay, execution history.
- **14 node types**: `manual-trigger`, `webhook-trigger`, `telegram-trigger`, `github-trigger`, `set-fields`, `if`, `switch`, `ai`, `http-request`, `telegram-send`, `github-create-issue`, `notion-create-page`, `google-calendar-event`.
- **AI node**: OpenAI / Anthropic Claude / Gemini (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`) via LangChain.
- **Routing**: `IF` (true/false) and `Switch` (a/b/default) compiled to LangGraph conditional edges.
- **Auth**: Better Auth with Google (offline + Calendar scope) and GitHub OAuth; session-gated routes via `proxy.ts`.
- **Background runs**: Inngest `execute-workflow` function (`maxDuration: 300s`), per-node step tracking in Postgres.
- **10 starter templates** (`modules/workflows/lib/templates.ts`): Telegram→AI→Notion flagship, AI Telegram bot, feedback summarizer, GitHub→Notion logger, webhook AI alerts, meeting scheduler, priority router, request switchboard, manual path.

## Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router), React 19, Tailwind 4, shadcn |
| Canvas | `@xyflow/react` |
| Execution | `@langchain/langgraph` + LangChain providers (`openai`, `anthropic`, `google-genai`) |
| Jobs / realtime | `inngest` (events, serve route, Realtime channels) |
| DB / ORM | PostgreSQL, Prisma 7 (`@prisma/adapter-pg`, `pg`), custom output `lib/generated/prisma` |
| Auth | `better-auth` + Prisma adapter |
| Validation / UI | shadcn, Base UI, HugeIcons, Recharts |

## Project structure

```
app/
  (auth)/sign-in/          Sign-in page
  (root)/                  Authed app shell + workflows/[id] editor
  api/
    auth/[...all]/         Better Auth handler
    inngest/               Inngest serve endpoint
    realtime-token/        Inngest Realtime subscription token (per execution)
    webhooks/[workflowId]/ Per-workflow generic webhook
    webhooks/telegram|github/ Shared inbound webhooks (+ per-workflow variants)
    workflow/[workflowId]/ Run / execution-snapshot endpoints
modules/
  auth/        Better Auth actions, Google Calendar helpers/scopes
  canvas/      Editor UI, parse-graph, create-node, autosave + execution overlay hooks
  engine/      compile-workflow → LangGraph, execute-node, run-workflow, start-execution, template interpolation
  nodes/       Node catalogue (lib/index.ts) + per-type executors
  workflows/   CRUD actions + templates
  webhooks/    Dispatch to active workflows, Telegram/GitHub verify + register helpers
  inngest/     Client, workflow/triggered event, execute-workflow fn, realtime channels
lib/           db.ts (Prisma singleton), auth.ts, auth-client.ts
prisma/        schema.prisma (User, Session, Account, Verification, Workflow, Execution, ExecutionStep)
proxy.ts       Auth gate: redirects unauthed → /sign-in, authed away from /sign-in
```

### Data model

- `Workflow { id, name, active, nodes: Json, edges: Json, userId }` — canvas persisted as JSON; `active=true` enables webhook triggers.
- `Execution { workflowId, status: RUNNING|SUCCESS|ERROR|WAITING, trigger: MANUAL|WEBHOOK|TELEGRAM|GITHUB|SCHEDULE, data, steps }`
- `ExecutionStep { executionId, nodeId, nodeName, nodeType, status, input, output, error }`

## Getting started

### Prerequisites

- Node 20+, `pnpm@11.24.0`
- PostgreSQL (local or hosted, e.g. `npx create-db`)
- Optional: Inngest CLI, ngrok (for Telegram/GitHub webhooks), API keys for providers you want to use

### 1. Install

```bash
pnpm install
cp .env.example .env
```

### 2. Configure env

Fill in `.env` — see [`.env.example`](./.env.example) for full reference:

| Var | Required for |
|---|---|
| `DATABASE_URL` | Always (Prisma + pg adapter) |
| `BETTER_AUTH_URL`, `BETTER_AUTH_SECRET` | Always (auth + webhook base URL). Use ngrok HTTPS URL when testing webhooks |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google login + Calendar node (enable Calendar scope, callback `{BASE}/api/auth/callback/google`) |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | GitHub login (callback `{BASE}/api/auth/callback/github`) |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY` | AI node (only providers you use) |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET` | Telegram trigger/send |
| `GITHUB_TOKEN`, `GITHUB_WEBHOOK_SECRET` | Create-issue node / inbound GitHub webhook verification |
| `NOTION_TOKEN`, `NOTION_PARENT_PAGE_ID` | Notion create-page node |
| `WEBHOOK_BASE_URL` / `NEXT_PUBLIC_APP_URL` | Override public webhook base (defaults to `BETTER_AUTH_URL`) |
| `INNGEST_EVENT_KEY`, `INNGEST_SIGNING_KEY` | Only for Inngest Cloud (local dev needs no keys) |

### 3. Database

```bash
pnpm dlx prisma generate
pnpm dlx prisma migrate dev
```

### 4. Run

```bash
# Terminal 1 — Next.js
pnpm dev
# Terminal 2 — Inngest (background executions)
npx inngest-cli@latest dev -u http://localhost:3000/api/inngest
```

Open `http://localhost:3000`, sign in, create a workflow, toggle **Active** for webhook triggers, and press **Run**.

### Webhooks (Telegram / GitHub)

1. Expose dev server: `ngrok http 3000` (or similar), set `BETTER_AUTH_URL` / `WEBHOOK_BASE_URL` to the HTTPS URL, restart `pnpm dev`.
2. Telegram: set `TELEGRAM_BOT_TOKEN` (+ optional `TELEGRAM_WEBHOOK_SECRET`); the app calls `setWebhook` to `{BASE}/api/webhooks/telegram` automatically. Verify in BotFather / `getWebhookInfo`.
3. GitHub: point a repo webhook at `{BASE}/api/webhooks/github` with `GITHUB_WEBHOOK_SECRET`; activate a workflow containing a `github-trigger` node.
4. Per-workflow generic webhooks: `POST {BASE}/api/webhooks/[workflowId]` with any JSON body.

## Scripts

| Command | Purpose |
|---|---|
| `pnpm dev` | Start Next.js dev server |
| `pnpm build` / `pnpm start` | Production build / serve |
| `pnpm lint` | ESLint |

## Notes

- Google Calendar actions reuse the workflow owner's Better Auth Google account — the owner must sign in with Google and enable the Calendar scope from their profile before runs succeed.
- AI node writes both `summary` and `response` fields; downstream nodes reference them as `{{summary}}` / `{{response}}`.
- Webhook dispatch only fires for `active` workflows containing the matching trigger node type.
