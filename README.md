# devvit-expert-skill

A standalone LLM skill for Reddit Devvit developers.

This repo packages a Devvit-focused skill that pushes models toward the current
Devvit Web workflow instead of mixed legacy examples. The goal is simple:
improve code accuracy for Reddit Devvit apps by making the model classify the
repo correctly, follow `devvit.json`, respect the `/api` versus `/internal`
split, and keep old `@devvit/public-api` patterns in maintenance-only mode.

## What this skill is

- A reusable skill folder you can drop into Codex, Claude Code, or similar
  skill-based workflows
- A Devvit Web-first guide for building, debugging, migrating, and publishing
  Reddit apps
- A compact main skill file plus deeper references for architecture,
  configuration, patterns, and migration

## What it optimizes for

- Correct repo classification: Devvit Web versus legacy public-api
- Config-first implementation with `devvit.json` as the source of truth
- Clear separation between client UX routes and Reddit callback routes
- Safer handling of Redis, settings, secrets, triggers, scheduler tasks, and
  external HTTP
- Fewer hallucinated APIs and fewer mixed-generation Devvit examples

## Repo layout

```text
SKILL.md
references/
  api.md
  patterns.md
agents/
  openai.yaml
```

- `SKILL.md`: primary instructions and trigger metadata
- `references/api.md`: config, runtime, endpoint, settings, Redis, and docs
  guidance
- `references/patterns.md`: implementation order, route taxonomy, Redis usage,
  and migration tactics
- `agents/openai.yaml`: UI metadata for environments that surface skills in a
  picker

## Installation

### Codex

Copy this repo into your Codex skills directory as `devvit-expert`:

```text
~/.codex/skills/devvit-expert/
  SKILL.md
  references/
  agents/
```

### Claude Code

Copy it into your project or global Claude skills directory:

```text
.claude/skills/devvit-expert/
  SKILL.md
  references/
  agents/
```

### Manual use

If your environment does not support skill folders, load `SKILL.md` first and
read the reference files on demand.

## Skill stance

This repo intentionally prefers:

- modern Devvit Web
- `devvit.json`
- client/server split
- `/api/*` for app UX
- `/internal/*` for menu actions, forms, triggers, and scheduler callbacks

This repo intentionally de-emphasizes:

- teaching `useWebView` plus `postMessage` as the default for new work
- treating legacy `@devvit/public-api` examples as the primary source of truth
- recommending deprecated or older patterns for fresh apps

Legacy guidance is still included, but only for maintaining or migrating
existing repos.

## Typical use cases

- “Build a new Devvit app feature”
- “Migrate this repo from older Devvit patterns”
- “Why is my menu action not firing?”
- “What belongs in `devvit.json` versus server code?”
- “How should I structure routes, settings, Redis keys, and scheduler tasks?”
- “Check whether this app is publish-ready”

## Notes

- This is a standalone Devvit developer skill with no product-specific
  workflow.
- The source of truth in this repo is the text-based skill content, not a
  packaged binary artifact.

## Contributing

Issues and PRs are welcome if they improve Devvit accuracy, align the guidance
with current Reddit platform patterns, or tighten the skill’s default workflow.
