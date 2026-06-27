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

## What you need

- A **coding agent** (this is a guide for one; e.g. Claude Code).
- A **Vercel** account + an **AI Gateway** credential (`AI_GATEWAY_API_KEY`). The app deploys here and
  the gateway runs both the model and the voice (speech in, speech out), keyless.
- **Node 24+** locally (the brain needs it).
- That is it for the foundation. Memory (Cognee) and any other integration are optional add-ons you
  bring in later.

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
- [`PRD.md`](./PRD.md) — what this product is and is not.
- [`docs/architecture.md`](./docs/architecture.md) — the diagrams (layers, request flow, deploy topology).
- `guide/00`–`07` — the build, in order. Start at `00-overview.md`.
- [`references.md`](./references.md) — platform docs and recommendations.
