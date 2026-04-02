# Devvit Implementation Patterns

These patterns are for the coding agent using the skill. Bias toward current
Devvit Web and keep the workflow template-driven.

## 1. Classify the app before coding

Answer these first:

- Is the repo **Devvit Web** or **legacy public-api**?
- Is config owned by `devvit.json` or `Devvit.configure()`?
- Are callbacks declared in config or registered in code?
- Is the repo already publish-ready, or only playtest-ready?

Fast heuristics:

- `devvit.json` with `post` and `server` means current Devvit Web.
- `src/client`, `src/server`, and `src/shared` strongly suggest the modern
  template.
- `blocks.entry`, `src/main.tsx`, `@devvit/public-api`, `useWebView`, or
  `webroot/` means legacy.

Starter choice rules:

- mod tools: default to the React Devvit Web starter
- games: choose React for UI-heavy apps, Phaser for 2D games, Three.js for 3D
  experiences, and Unity or GameMaker only when an engine is truly required
- if the repo already uses a current starter cleanly, extend that starter
  instead of swapping frameworks mid-task

## 2. Follow the Devvit Web vibe-coding order

Implement in this order:

1. `devvit.json`
2. server bootstrap and routes
3. shared types and contracts
4. client requests and UI
5. Redis/state updates
6. playtest and publish checks

This avoids the most common failure mode: building UI before permissions,
endpoints, and runtime wiring exist.

## 3. Copy the repo's route taxonomy

Prefer the route structure already shown by the current starter:

- `/api/*` for client-initiated application requests
- `/internal/menu/*` for menu actions
- `/internal/form/*` for form submissions
- `/internal/triggers/*` for triggers
- `/internal/scheduler/*` for scheduled work

If the repo already composes separate routers under a server entrypoint, extend
that structure rather than flattening everything into one file.

## 4. Build menu actions as server-backed workflows

Menu action workflow:

1. declare the item in `devvit.json`
2. implement the handler under `/internal/menu/*`
3. validate the incoming target id and actor context
4. return a UI response immediately
5. log failures without leaking secrets or raw response bodies

Example:

```ts
app.post('/internal/menu/scan-post', async (c) => {
  const request = await c.req.json();
  const postId = request.targetId;

  if (!postId) {
    return c.json({ showToast: { text: 'Missing post id.', appearance: 'error' } }, 400);
  }

  return c.json({
    showToast: { text: 'Scan started.', appearance: 'success' },
  });
});
```

## 5. Keep trigger and scheduler handlers idempotent

Trigger and scheduler rules:

- validate the event payload before doing work
- load the exact Reddit object you need
- short-circuit if the resource was already processed
- write a single receipt or state update
- make reruns safe

Useful idempotency keys:

- `receipt:post:{postId}`
- `trigger:post-submit:{postId}`
- `job:daily-sync:{yyyy-mm-dd}`

## 6. Use shared types when the repo supports them

If the app already has `src/shared/*`, put request and response contracts there.

Good candidates:

- menu action response payloads
- client init and mutation responses
- external service request and result shapes

Do not bury API contracts inside unrelated UI files when the template already
has a shared layer.

## 7. External service pattern

For apps that call external services:

1. whitelist domains in `devvit.json`
2. keep the call on the server
3. authenticate with a secret from settings
4. validate the response before mutating Reddit state
5. return safe user-facing failures

Do not move external fetches into the browser layer just to make the UI code
look simpler.

## 8. Redis patterns that survive current limits

Current constraints mean you should model indexes up front.

Safe patterns:

- primary record: `post:{id}`
- queue surrogate: `queue:pending` as a sorted set
- user profile: `user:{id}`
- secondary index: `user-posts:{userId}`

Avoid:

- key scans
- plain Redis sets
- logic that assumes a global shared store across installations
- designs that depend on pipelining

## 9. Legacy migration pattern

When a repo uses legacy Blocks or `useWebView`, always plan migration off that
stack and migrate incrementally:

1. keep the existing app working
2. introduce `devvit.json` as the source of truth
3. move permissions out of `Devvit.configure()`
4. replace message passing with explicit server endpoints
5. move settings and secrets into `devvit.json`
6. remove deprecated custom-post code last

Do not choose a one-shot rewrite unless the user explicitly asks for it, but do
not leave Blocks as the intended end state.

## 10. Publish-readiness checklist

Before calling the app ready:

- `devvit.json` validates and matches current sections
- new code does not introduce `addCustomPostType()`
- secrets are not hardcoded
- required domains are whitelisted
- menu, trigger, form, and scheduler endpoints exist where declared
- build outputs match `post.dir` and `server.entry`
- `package.json` scripts still reflect the template's flow
- the app has been exercised with playtest, not just static inspection

If the app is targeting Developer Funds or a challenge, also verify:

- the current program page was checked for live rules and dates
- the app uses a current Devvit Web starter, not Blocks
- basic product telemetry exists for installs, sessions, and repeat engagement
- launch copy avoids stale prize amounts or expired challenge claims

## 11. Debugging order

Check in this order:

1. wrong architecture assumption
2. missing or incorrect `devvit.json` permissions
3. wrong endpoint path or wrong `/api` vs `/internal` split
4. build output mismatch
5. missing settings or secrets
6. Reddit object lookup failure
7. Redis schema or idempotency bug
8. business-logic bug

## 12. What to say when docs disagree

Make the conflict explicit and choose the safer path:

- current Devvit Web docs and templates outrank mixed-era examples
- some official pages still show older package names or older examples
- the repo's active template is usually the most reliable implementation source

Do not confidently synthesize a hybrid architecture when the repo already shows
the correct one.
