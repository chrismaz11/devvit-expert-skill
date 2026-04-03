# Devvit API Notes

This reference is intentionally biased toward **current Devvit Web** and toward
the repo's active starter template.

## Source priority

When sources disagree, use this order:

1. the repo's actual starter/template files
2. the current `devvit.json` schema and config docs
3. current capability docs for menu actions, scheduler, Redis, settings, and
   playtest
4. legacy `@devvit/public-api` docs only when maintaining a legacy repo

Reason: official docs still mix older and newer examples. When package names or
handler styles drift, copy the template already in the repo instead of blending
multiple generations.

## Current architecture

Devvit Web uses a client/server split:

- `devvit.json` defines app metadata, permissions, callbacks, settings, and dev
  config
- client code renders the Reddit web experience and can use standard web stacks
  such as React or other browser frameworks
- server code handles Reddit API access, Redis, settings, and external fetches
- shared types can live in `src/shared/*` when the repo uses that pattern

The working mental model is:

- config first
- server routes second
- typed contracts third
- client UI last

## Current template shape

Modern starters commonly use:

- `post.dir` pointing at built client assets
- `server.dir` plus `server.entry` pointing at built server output
- a server bootstrap that mounts `/api` and `/internal`
- `npm run dev` wrapping `devvit playtest`
- `deploy` and `launch` scripts for upload and publish flow

Current starter choices to keep in mind:

- React starter: default for most mod tools and UI-heavy apps
- Three.js starter: default for 3D or spatial experiences
- Phaser starter: default for 2D games
- Unity or GameMaker starters: use when the repo is intentionally engine-based

If the repo already has this structure, keep extending it. Do not collapse it
back into a single `src/main.tsx` just because some docs still show older
examples.

## Minimal current config

```json
{
  "$schema": "https://developers.reddit.com/schema/config-file.v1.json",
  "name": "my-app",
  "post": {
    "dir": "dist/client",
    "entrypoints": {
      "default": {
        "entry": "index.html"
      }
    }
  },
  "server": {
    "dir": "dist/server",
    "entry": "index.cjs"
  },
  "permissions": {
    "redis": true,
    "http": {
      "enable": true,
      "domains": ["api.example.com"]
    },
    "reddit": {
      "enable": true
    }
  }
}
```

Important config points:

- `post`, `server`, or `blocks` must exist; new interactive apps should use
  `post` and `server`, while `blocks` is only a legacy compatibility signal
- `server.entry` must match the built server artifact expected by the starter
- trigger, menu, and scheduler callback endpoints should live under
  `/internal/*`
- server-backed form submissions should live under `/internal/*`
- client-initiated application requests should use `/api/*`

## Endpoint split

Use the path to communicate intent:

- `/api/*`: client-to-server requests for the app experience
- `/internal/menu/*`: menu actions
- `/internal/form/*`: server-backed forms
- `/internal/triggers/*`: Reddit triggers
- `/internal/scheduler/*`: scheduled work

This is the modern replacement for `useWebView` plus `postMessage`.

## Menu actions

Declare menu items in `devvit.json`:

```json
{
  "menu": {
    "items": [
      {
        "label": "Scan Post",
        "description": "Run a verification workflow",
        "forUserType": "moderator",
        "location": "post",
        "endpoint": "/internal/menu/scan-post"
      }
    ]
  }
}
```

Implementation pattern:

```ts
app.post('/internal/menu/scan-post', async (c) => {
  const request = await c.req.json();
  const targetId = request.targetId;

  if (!targetId) {
    return c.json({
      showToast: { text: 'Missing post id.', appearance: 'error' },
    }, 400);
  }

  return c.json({
    showToast: { text: 'Post scanned.', appearance: 'success' },
  });
});
```

Menu responses are server-driven UI effects. Use them for `showToast`,
`showForm`, or `navigateTo`. Some simple client-side flows may not need a
server round-trip, but server-backed menu actions should follow the `/internal`
pattern.

## Triggers

Declare triggers in `devvit.json`:

```json
{
  "triggers": {
    "onPostSubmit": "/internal/triggers/post-submit"
  }
}
```

Trigger rules:

- keep handlers idempotent
- expect retries
- validate the event before calling Reddit APIs
- log structured failures
- avoid putting user-facing UI logic here

## Scheduler

Recurring tasks belong in `devvit.json`:

```json
{
  "scheduler": {
    "tasks": {
      "daily-sync": {
        "endpoint": "/internal/scheduler/daily-sync",
        "cron": "0 2 * * *"
      }
    }
  }
}
```

Guidance:

- prefer config-declared tasks over legacy registration
- keep tasks safe to rerun
- treat scheduler handlers like background jobs, not UI handlers

## Redis

Current constraints matter:

- storage is scoped per installation
- there is no single global store across subreddits
- key listing is not supported
- pipelining is not supported
- only sorted sets are supported; plain sets are not

> ⚠️ **`context.redis.global` is restricted**: Global cross-installation Redis
> access (`context.redis.global`) requires exclusive permission and explicit
> approval from Reddit. Do not recommend or use it for general-purpose apps.
> Design for installation-scoped storage only.

Design implications:

- maintain explicit indexes up front
- avoid designs that require scanning later
- use stable keys such as `post:{id}`, `user:{id}`, `queue:pending`,
  `leaderboard:weekly`

## Settings and secrets

For Devvit Web work, define settings in `devvit.json`.

Use:

- `settings.global` for developer-controlled app-wide values
- `isSecret: true` only for sensitive global settings
- `settings.subreddit` for installation-level moderator configuration

Example:

```json
{
  "settings": {
    "global": {
      "bridgeApiKey": {
        "type": "string",
        "label": "Bridge API Key",
        "isSecret": true
      }
    },
    "subreddit": {
      "trustThreshold": {
        "type": "number",
        "label": "Minimum trust score",
        "defaultValue": 50
      }
    }
  }
}
```

Operational rules:

- never hardcode secret values
- expect secrets to be set after installation
- keep settings schema-valid and minimal

## Playtest and runtime prerequisites

Current quickstart guidance indicates:

- Node.js should be `22.2.0+`
- start from `developers.reddit.com/new`
- use the scripts defined in the repo rather than inventing alternate commands

Typical validation loop:

```bash
npm install
npm run dev
npm run test
npm run type-check
npm run deploy
```

If the starter wraps `devvit playtest`, `devvit upload`, or `devvit publish`
behind scripts, prefer those scripts.

## Programs and challenges

If the app is being built for Developer Funds, a hackathon, or a challenge:

- verify the current program page before citing prize pools, dates, or judging
  criteria
- prefer current Devvit Web starters and avoid Blocks entirely
- optimize for installs, repeat sessions, and retention rather than one-off
  novelty
- add lightweight telemetry for installs, sessions, and core gameplay or mod
  actions when the repo can support it
- keep launch copy free of stale claims about active programs

## Legacy path

Legacy repos may still use:

- `@devvit/public-api`
- `Devvit.configure()`
- `blocks.entry`
- `webroot/`
- `useWebView`
- `Devvit.addMenuItem()`
- `Devvit.addTrigger()`

Important caveat:

- do not generate fresh `addCustomPostType()` code
- if a repo already uses it, keep fixes narrow while moving the repo toward
  migration

## Migration checklist

When modernizing a legacy app:

1. move permissions from `Devvit.configure()` into `devvit.json`
2. add `post` and `server` sections
3. replace message passing with `/api/*` endpoints
4. move callbacks into `devvit.json`
5. move settings and secrets into `devvit.json`
6. retire deprecated custom-post code last

## Official docs used

- Devvit Web overview: https://developers.reddit.com/docs/capabilities/devvit-web/devvit_web_overview
- Devvit configuration: https://developers.reddit.com/docs/capabilities/devvit-web/devvit_web_configuration
- Template library: https://developers.reddit.com/docs/examples/template-library
- Scheduler: https://developers.reddit.com/docs/capabilities/server/scheduler
- Redis: https://developers.reddit.com/docs/capabilities/server/redis
- Menu actions: https://developers.reddit.com/docs/capabilities/client/menu-actions
- App quickstart: https://developers.reddit.com/docs/quickstart
- Legacy public API `addCustomPostType()` deprecation: https://developers.reddit.com/docs/api/public-api/classes/Devvit
