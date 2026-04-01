---
name: devvit-expert
description: >
  Current Devvit coding assistant for Reddit apps. Use this skill whenever the
  user is building, debugging, migrating, or publishing a Devvit app. Default
  to modern Devvit Web with `devvit.json`, `post` + `server`, typed shared
  contracts, and `/api` and `/internal` endpoints. Treat legacy
  `@devvit/public-api`, `useWebView`, `webroot`, and `addCustomPostType()` as
  compatibility-only paths for existing apps.
---

# Devvit Expert

Default to **Devvit Web**. Treat legacy `@devvit/public-api` apps as maintenance
work unless the repo is already legacy or the user explicitly asks for older
patterns.

## First move

Read the actual project before writing code:

1. `package.json`
2. `devvit.json`
3. the app entry layout such as `src/client/*`, `src/server/*`, `src/shared/*`,
   or legacy `src/main.tsx`

Then classify the repo:

- **Devvit Web** if it uses `devvit.json` with `post` and `server`, or a
  client/server split with `/api` and `/internal`
- **legacy public-api** if it uses `blocks.entry`, `@devvit/public-api`,
  `Devvit.configure()`, `useWebView`, or `webroot/`

## Default working model

For new work and most fixes, use the current Devvit Web template mentality:

- `devvit.json` is the source of truth
- client UI talks to server endpoints, not `postMessage`
- `/api/*` is for app UX
- `/internal/*` is for Reddit callbacks such as menu items, triggers, forms,
  and scheduler tasks
- external HTTP, Reddit API calls, Redis, and secrets stay on the server

If the docs conflict with the repo, prefer:

1. the repo's current starter/template files
2. `devvit.json` schema and current config docs
3. current Devvit Web capability docs
4. legacy docs only for legacy maintenance

## Devvit Web vibe-coding workflow

Implement in this order:

1. lock down `devvit.json`
2. add server routes and handlers
3. define shared request and response types
4. wire client fetches and UI
5. validate playtest, build output, permissions, and publishability

Use the repo's existing template as the coding scaffold. Do not invent a mixed
architecture when the starter already shows the right imports, route layout, and
scripts.

Practical defaults:

- interactive UI: `post.dir` for client assets plus `server.entry` for server
  code
- menu items: declare in `devvit.json`, implement under `/internal/menu/*`
- triggers: declare in `devvit.json`, keep handlers idempotent under
  `/internal/triggers/*`
- forms: define in `devvit.json`, submit to `/internal/form/*`
- client requests: use `/api/*`
- shared contracts: keep typed payloads in `src/shared/*` when the repo uses
  that pattern

## Guardrails

- Keep secrets out of client code and source control
- Keep external HTTP calls on the server and whitelist domains in `devvit.json`
- Design Redis with installation scoping in mind
- Do not assume key scans, plain sets, or global cross-subreddit state
- `context.redis.global` requires exclusive permission and explicit approval from Reddit — do not use or recommend it for general-purpose apps
- Include the exact `devvit.json` changes when giving code
- When debugging, check architecture, permissions, endpoint paths, settings, and
  build output before blaming business logic

## Legacy compatibility

Legacy repos may still use:

- `@devvit/public-api`
- `Devvit.configure()`
- `Devvit.addMenuItem()`
- `Devvit.addTrigger()`
- `Devvit.addSchedulerJob()`
- `useWebView`
- `webroot/`

When editing a legacy repo:

- keep fixes narrow unless the user asked to migrate
- do not teach legacy patterns as the preferred architecture
- do not recommend `addCustomPostType()` for new work
- prefer gradual migration to `devvit.json` plus Devvit Web

## References

- Read `references/patterns.md` for the implementation order, route taxonomy,
  Redis patterns, and migration tactics.
- Read `references/api.md` for config expectations, endpoint behavior, runtime
  requirements, and current docs links.
