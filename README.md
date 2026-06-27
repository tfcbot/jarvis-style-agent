# Jarvis-Style Agent

A **build guide, not a codebase.** A pack of instructions you hand to your own coding agent; it builds
you a **Jarvis-style voice agent**: a glowing particle orb you talk to out loud, wired to a real agent
brain, deployed to the web so it is always on. You say "hello," it says "hello" back, in a voice.

You do not clone an app. Your agent recreates it from this guide, in your own private repo, on your
own infrastructure, with your own keys. This guide gets you the **foundation**: the orb, the brain,
deployed, talking. The hard part, done right. From there your agent builds the rest on top, as far as
you want to take it.

## The four concepts

Every system like this is the same four pieces. Name them and you can build anything.

| Concept      | What it is                                         | In this build                          |
| ------------ | -------------------------------------------------- | -------------------------------------- |
| **Frontend** | What you see and talk to                           | A particle orb that pulses when it talks |
| **Backend**  | The brain that thinks and replies                  | An [EVE](https://eve.dev) agent          |
| **Context**  | What it knows *during* a conversation              | Its instructions + the live transcript   |
| **Memory**   | What it knows *across* conversations               | Durable facts (Cognee), added when wanted |

They are tied together by one pattern: **Backend-for-Frontend (BFF)**. The frontend never calls a
model or a third-party service directly. It calls **one backend you own**, and that backend calls
everything else. Your secrets stay on the server, your frontend stays simple, and every integration
has exactly one home. This is the whole architecture. Learn it once.

## What you get

The guide walks your agent through building a two-part project:

```
orb/      a Next.js app: the particle orb + the BFF routes + the voice loop
brain/    the EVE agent: the model, its instructions, the door the orb talks to
```

- **Talk to it.** Tap the mic, speak, and hear a spoken reply. The orb pulses with the voice.
- **It is yours.** Your keys, your Vercel, your repo. Nothing of yours runs on anyone else's servers.
- **Always on.** Deployed to Vercel and redeploys when you merge. No laptop required.
- **Built to extend.** The foundation is unopinionated. Add tools, give it memory, expose it to other
  apps, swap the voice — your agent does it from `guide/07-extensibility.md` when you ask.

## Prerequisites

Have these ready before you start. The foundation needs only two vendors (GitHub + Vercel); the
gateway covers both the model and the voice, so there are no separate AI provider keys.

**Tools**

- A **coding agent** (this is a guide for one; e.g. Claude Code).
- **Node 24+** locally (the brain needs it).
- A **GitHub account** and the **GitHub CLI (`gh`)**, authenticated (`gh auth login`). Your agent
  builds in your own **private** repo and ships by merging to `main`.
- The **Vercel CLI** (`vercel`), and a `vercel login`. The app deploys to your Vercel.

**Accounts / keys**

- A **Vercel** account + an **AI Gateway** credential (`AI_GATEWAY_API_KEY`). The app deploys here and
  the gateway runs **both the model and the voice** (speech in, speech out), keyless.
- That is it for the foundation. **Cognee** (memory) and **ElevenLabs** (premium voice) are optional
  add-ons you bring in later.

See [`vendors.md`](./vendors.md) for the full vendor list and every environment variable (and which
ones are required vs optional). See [`docs/devops.md`](./docs/devops.md) for the repo + deploy workflow.

## How to use it

1. Drop this guide into your project (or point your agent at this repo).
2. Tell your agent: **"I want to build a Jarvis-style voice agent using this guide. Help me build it."**
3. It reads `AGENTS.md`, then builds the orb, the brain, deploys both to Vercel, and stops when you can
   say "hello" and hear a reply. That is the foundation.
4. Then tell it what to build next. It uses the extensibility guide to build your backend out cleanly.

## What this guide covers (and where it stops)

It covers the **foundation**: frontend + backend + deploy + "hello, you hear a reply." That clean base
is the product. Everything past it (your tools, your data, your integrations) is yours to build on top,
and the extensibility guide shows your agent how. The guide nails the part that is easy to get wrong so
the rest is easy.

## Map of this repo

- [`AGENTS.md`](./AGENTS.md) — the entry point your agent reads first.
- [`PRD.md`](./PRD.md) — the technical spec: architecture, folder structure, backend API (OpenAPI), acceptance.
- [`vendors.md`](./vendors.md) — vendors + every environment variable key (brain vs orb, required vs optional).
- [`docs/architecture.md`](./docs/architecture.md) — the diagrams (layers, request flow, deploy topology).
- [`docs/devops.md`](./docs/devops.md) — repo + deploy workflow (private repo, branch → merge-to-deploy, worktree cleanup).
- `guide/00`–`07` — the build, in order. Start at `00-overview.md`.
- [`references.md`](./references.md) — platform docs and recommendations.
