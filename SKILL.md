---
name: devvit-expert
description: >
  Expert-level Devvit (Reddit developer platform) coding assistant. Use this skill whenever the user is building, debugging, or extending a Reddit app with Devvit — including questions about useWebView, message passing between server and web, Redis state management, scheduled jobs, the Reddit API, custom post types, menu items, triggers, or devvit.yaml config. Also triggers for: "how do I add a feature to my Devvit app", "my scheduler job isn't running", "how do I store data in Devvit", "write code for my Reddit app", "fix this Devvit bug", or anything involving the @devvit/public-api package. When in doubt, use this skill — it dramatically improves code accuracy for Devvit projects.
---

# Devvit Expert

You are a Devvit expert. Devvit is Reddit's platform for building custom post types, sidebar widgets, and automated bots that run natively inside Reddit. This skill gives you full platform knowledge so you can write correct Devvit code without guessing.

## Platform mental model

Devvit apps have two worlds that communicate via messages:

**Server world** (`src/main.tsx` and job files) — TypeScript running in Reddit's sandbox. Has access to Redis, the Reddit API, and the scheduler. No browser, no fetch to arbitrary URLs unless `http: true` is configured.

**Web world** (`webroot/index.html`) — A regular browser page served inside an iframe (the "web view"). Vanilla JS (or any bundled framework). Communicates with the server via `postMessage`.

The glue is `useWebView`. You almost always want Devvit Web (useWebView) for interactive apps, not the old Blocks-only approach.

## Quickstart skeleton

```typescript
// src/main.tsx
import { Devvit, useWebView } from '@devvit/public-api';

type WebToServer = { type: 'INIT_REQUEST' } | { type: 'MY_ACTION'; data: string };
type ServerToWeb = { type: 'INIT'; payload: any } | { type: 'RESULT'; ok: boolean };

Devvit.configure({ redis: true, redditAPI: true });

Devvit.addCustomPostType({
  name: 'MyApp',
  height: 'tall',
  render: (context) => {
    const webView = useWebView<WebToServer, ServerToWeb>({
      url: 'index.html',
      onMessage: async (msg, hook) => {
        if (msg.type === 'INIT_REQUEST') {
          const data = await context.redis.get('my:key');
          hook.postMessage({ type: 'INIT', payload: data });
        }
      },
    });
    return <webview id="myWebView" url="index.html" grow />;
  },
});
```

```html
<!-- webroot/index.html -->
<script>
  window.addEventListener('message', (event) => {
    if (event.data.type !== 'devvit-message') return;
    const msg = event.data.data.message;  // <-- two .data levels!
    if (msg.type === 'INIT') { /* render */ }
  });
  window.parent.postMessage({ type: 'INIT_REQUEST' }, '*');
</script>
```

## Key rules (memorize these)

1. **All state is Redis** — no SQL, no external DB unless you use HTTP. `context.redis` is your database.
2. **Messages nest weirdly on arrival**: web receives server messages as `event.data.data.message` (not `event.data`).
3. **Scheduler times are ET** — `'0 9 * * *'` is 9 AM America/New_York.
4. **`context.userId` can be undefined** — users browsing while logged out. Always null-check.
5. **`context.ui` is Blocks-only** — not available in scheduler jobs or triggers.
6. **`Devvit.addCustomPostType` is the modern way** — `useWebView` inside `render` is the current best practice.
7. **`hook.postMessage` vs `webView.postMessage`** — inside `onMessage`, use `hook.postMessage`. Outside (e.g., in render), use the ref returned by `useWebView`.
8. **Redis keys are scoped per installation** — use `context.redis.global` for cross-subreddit data.

## Reference files

For complete API signatures and copy-paste examples, read these files as needed:

- **`references/api.md`** — Full API reference: Context, Redis (strings/hashes/sorted sets/transactions), Scheduler, Reddit API, UIClient, Triggers, Settings, Forms
- **`references/patterns.md`** — Common patterns: leaderboard, user profiles, daily jobs, idempotent resolvers, web UI state management, error handling

## How to approach tasks

**Answering questions**: Explain the concept with a minimal working example. Reference the correct API — if in doubt, read `references/api.md`.

**Writing new features**: Think about (1) what data shape goes in Redis, (2) what messages flow between server and web, (3) what scheduled jobs are needed. Write in that order.

**Debugging**: The most common bugs are:
- Missing `event.data.data.message` unwrapping (using `event.data.message` instead)
- Not awaiting Redis calls
- Calling `context.ui` from a scheduler/trigger context
- Scheduler jobs not registered — must call `Devvit.addSchedulerJob()` before `scheduler.runJob()`
- `context.userId` is undefined for logged-out users

**Editing existing code**: Read the actual file first. Follow the existing key naming conventions (usually in `src/utils/redis.ts`). Keep type definitions in `src/types/index.ts`.

## Commands
```bash
npm install        # Install dependencies
npm run dev        # Playtest locally via Devvit CLI
npm run build      # Build for production
npm run upload     # Upload/deploy to Reddit
npx tsc --noEmit   # Type-check only
```
