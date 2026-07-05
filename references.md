# References

Curated platform docs behind this build, plus what to reach for and why. Links are starting points;
follow each platform's current docs for exact API shapes (endpoint names and payloads change).

> For the **vendor list + every environment variable key** (which goes on the brain vs the orb,
> required vs optional), see [`vendors.md`](./vendors.md). For the **repo + deploy workflow**, see
> [`docs/devops.md`](./docs/devops.md).

## The backend framework

- **EVE** — the filesystem-first agent framework the brain runs on. Files under `agent/` become the
  agent; tools and channels are just files. https://eve.dev
  - Recommendation: use EVE for the foundation and whenever the agent should *do* things (tools,
    skills, an agent loop). It is what makes "add a capability" mean "add a file."

## Hosting + model + voice

- **Vercel** — hosting for the brain, deploy-on-merge via git integration. The orb runs locally.
  https://vercel.com/docs
- **Vercel AI Gateway** — runs the model **and** the realtime voice from one credential. The
  `AI_GATEWAY_API_KEY` comes from here (Dashboard → AI Gateway); the orb's setup route uses it to
  mint short-lived realtime client tokens, so the browser never holds it.
  https://vercel.com/docs/ai-gateway
  - Use **dotted** model ids (`anthropic/claude-haiku-4.5`, `openai/gpt-realtime-2`). Dashes are not
    gateway ids and fail the build. `anthropic/claude-haiku-4.5` is the recommended brain default —
    the tool round-trip is spoken aloud, so latency is the feel.
- **Vercel AI SDK (`ai` + `@ai-sdk/gateway` + `@ai-sdk/react`)** — the realtime voice:
  `gateway.experimental_realtime.getToken()` (server-side token mint),
  `experimental_getRealtimeToolDefinitions` (tool defs for the session), and the
  `experimental_useRealtime` React hook (the browser session — a WebSocket with Web Audio playback).
  Stable v7/v4 packages, pinned in `01-frontend.md`. https://sdk.vercel.ai
  - The hook keys its session on the **reference identity** of `model` and `sessionConfig` — pass
    stable module-scope constants, never inline object literals.
- **Vercel CLI** — `vercel link`, `vercel git connect`, `vercel env add`. The person does one
  `vercel login` (or sets `VERCEL_TOKEN` for headless). https://vercel.com/docs/cli

## Frontend

- **Next.js** (App Router) — the orb app and its BFF routes. https://nextjs.org/docs
- **React** — https://react.dev
- **three.js** — the particle orb, bloom post-processing, shaders. https://threejs.org/docs

## Voice

- **`openai/gpt-realtime-2`** — the realtime speech model, served through the AI Gateway (no OpenAI
  key). Session behavior (voice id, persona instructions, server-VAD turn detection) is set in
  `sessionConfig`; see `07-extensibility.md` for tuning it.
  - For a managed talking-head/avatar instead of an orb, point a Custom-LLM avatar platform at the
    brain's OpenAI-compatible door (`/v1/chat/completions`); the brain does not change.

## Memory

- **Cognee** — the recommended durable-memory product: a knowledge graph with relevance retrieval.
  https://www.cognee.ai / https://docs.cognee.ai
  - **Use Cognee Cloud, bring your own key, integrate over REST or MCP.** Do **not** use the embedded
    `@cognee/cognee-ts` engine; that is the self-host path and the wrong tool for a serverless brain.
    Put `COGNEE_API_KEY` in the brain's env only. Follow the current docs for exact `add`/`search`
    endpoints; the wiring pattern is in `04-memory.md`.

## Protocols

- **OpenAI Chat Completions** — the contract the brain's door speaks (`POST /v1/chat/completions`,
  streamed SSE). The lingua franca that lets the orb (and any other client) drive the brain.
  https://platform.openai.com/docs/api-reference/chat
- **Model Context Protocol (MCP)** — to expose the agent's tools to other AI clients
  (`07-extensibility.md`). https://modelcontextprotocol.io
  - Vercel MCP handler (`mcp-handler`) for serving MCP from a Vercel app. https://vercel.com/docs

## Alternative backend

- **Hono** — a minimal, stateless backend that exposes the same `/v1/chat/completions` door without an
  agent framework. Use when the agent only needs to *talk*, not *do*. https://hono.dev

## Quick recommendation table

| Want to...                          | Reach for                                         |
| ----------------------------------- | ------------------------------------------------- |
| Build the agent brain               | EVE                                               |
| Run the model + realtime voice, one key | Vercel AI Gateway (dotted model ids)          |
| Host it, always on                  | Vercel (the brain, deploy-on-merge); the orb runs locally |
| Tune the voice                      | the realtime `sessionConfig` (voice id, persona, turn detection) |
| Durable, intelligent memory         | Cognee Cloud (BYO key, REST/MCP)                  |
| Expose tools to other AI apps       | MCP                                               |
| A minimal brain that only talks     | Hono (same door, no framework)                    |
