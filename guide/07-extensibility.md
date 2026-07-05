# 07 — Extensibility: build your backend out cleanly

The foundation is done: the person can say "hello" and hear a reply. Now make it *do things*. This is
where the architecture pays off. Because everything routes through the backend (the BFF pattern),
every new capability has exactly one home, and adding it never touches the frontend.

Build these **on request**, one at a time, green before the next. Do not pre-build.

## The mental move

To extend the agent, you almost always add one of two things to the **backend**:

- A **tool** — a function the agent can call to read data or take an action (check the weather, read a
  spreadsheet, create a record, send a message).
- A **skill** — a packaged procedure the agent follows for a recurring job (a multi-step routine with
  its own instructions).

The frontend does not change. The orb keeps talking to the same door; the agent simply has more it can
do behind that door. That is the whole point of the foundation.

## Add a tool (EVE)

In EVE, a tool is a file in `agent/tools/`. Define its description (the agent reads this to decide when
to call it), its parameters (a zod schema), and what it does:

```ts
// agent/tools/get_weather.ts
import { defineTool } from "eve/tools";
import { z } from "zod";

export default defineTool({
  description: "Get the current weather for a city. Use when the person asks about weather.",
  inputSchema: z.object({ city: z.string().describe("City name, e.g. 'Austin'.") }),
  async execute({ city }) {
    const res = await fetch(`https://api.example.com/weather?q=${encodeURIComponent(city)}`, {
      headers: { authorization: `Bearer ${process.env.WEATHER_API_KEY}` },
    });
    if (!res.ok) return { error: "weather service unreachable" };
    const w = await res.json();
    return { city, tempF: w.tempF, summary: w.summary };
  },
});
```

The tool name is the filename (`get_weather`). Guard configuration the same way the store tools do:
if the env key is unset, return `{ configured: false, message: "…" }` instead of failing — the
agent then tells the person what to connect rather than guessing.

Then:

- **Tell the agent it exists** in `instructions.md` (one line: what it does, when to use it). Tools the
  agent is not told about go unused.
- **Put any secret in the brain's env** (`WEATHER_API_KEY`), never the orb. New integration → new
  backend env var. The frontend stays clean.
- **Gate anything that spends money or acts publicly.** For writes, sends, or purchases, have the agent
  read back what it is about to do and get a clear "yes" first (set this rule in `instructions.md`).
  Voice gets misheard; this is the guardrail.

That is the entire extension loop: add a file, name it in the instructions, set its env. Repeat for
every capability the person wants. Reading their store, posting to a channel, querying a database,
calling another API: all the same shape.

## Add a skill (EVE)

When a capability is a *procedure* rather than a single call (e.g. "run the weekly review": pull
numbers, summarize, draft a message, ask to send), package it as a skill in `agent/skills/`. A skill
bundles instructions and the tools it uses so the agent runs the routine consistently. Use tools for
verbs, skills for routines.

## Expose the agent to other apps (MCP)

The brain already speaks OpenAI-compatible HTTP. To let other AI clients (Claude, Claude Code, other
MCP hosts) drive its tools, expose an **MCP server** from the backend. The tools you already wrote are
reused as-is; MCP is just another door onto the same capabilities. Add an MCP channel/route in the
brain, protect it (OAuth or a bearer), and the agent's tools become available to any MCP client. The
frontend is unaffected. (See `references.md` for the MCP spec and the Vercel MCP handler.)

## Swap the backend: EVE → a minimal Hono API

EVE gives you a stateful agent loop, tools, and skills as files. If the person wants something
**minimal and stateless** instead (just relay a chat turn to a model, no framework), you can replace
the brain with a tiny **[Hono](https://hono.dev)** server that exposes the *same door*: `POST
/v1/chat/completions`, bearer-guarded, streaming OpenAI SSE. Because the orb only knows that contract,
**nothing on the frontend changes**.

Shape of it:

```ts
// A minimal stateless brain: same door, no agent framework. (Sketch.)
import { Hono } from "hono";
import { streamText } from "ai";

const app = new Hono();

app.post("/v1/chat/completions", async (c) => {
  const auth = c.req.header("authorization");
  if (auth !== `Bearer ${process.env.BRAIN_SECRET}`) return c.json({ error: "unauthorized" }, 401);

  const { messages } = await c.req.json();
  const result = streamText({ model: process.env.AGENT_MODEL ?? "anthropic/claude-haiku-4.5", messages });
  return result.toTextStreamResponse(); // emit OpenAI-style SSE deltas
});

export default app;
```

Trade-off: Hono is less to run, but you give up EVE's filesystem-first tools/skills and agent loop. Use
EVE when the agent should *do* things; use Hono when it only needs to *talk*. Either way the orb is the
same. That is the BFF contract earning its keep.

## Tune the voice: the realtime session config

The voice is the realtime speech model (`openai/gpt-realtime-2` through the gateway), and its
levers live in the session, not in a pipeline:

- **`sessionConfig`** (in `RealtimeVoice.tsx`): the spoken persona (`instructions`), the voice id
  (`voice: 'alloy'` — pick from the model's voice list), and turn-taking (`turnDetection:
  { type: 'server-vad' }`). Keep the config a stable module-scope constant when you change it.
- **`REALTIME_MODEL`** (orb env): swap the speech model as new realtime models ship — the token
  route and client read the same id.
- **The tool contract**: sharpen the `ask_jarvis_brain` description in `api/realtime/setup` as the
  brain gains capabilities, so the voice model knows exactly when to reach for it and answers
  everything else itself.

For a managed talking-head/avatar instead of an orb, point a Custom-LLM avatar platform at the
brain's OpenAI-compatible door; the brain does not change.

## Recommendations

- **Extend the backend, not the frontend.** If you find yourself adding an SDK or a key to the orb,
  stop: it belongs in the brain behind a tool. The orb only ever calls `/api/*`.
- **One capability per PR, green before the next.** Add the tool, name it in instructions, set the env,
  verify the agent uses it, ship. Then the next. (Mirrors how this guide itself was built.)
- **Name capabilities in `instructions.md`.** The agent cannot use what it is not told it has.
- **Gate writes and spends behind a read-back + "yes."** Always, for anything irreversible.
- **Keep the model fast for voice.** Latency is the whole feel. `anthropic/claude-haiku-4.5` is a good
  default; reserve heavier models for non-voice paths.
- **Persist memory before relying on it.** The Stage-1 notes store resets on cold starts; move to
  Cognee (`04-memory.md`) before the person depends on the agent remembering.

The foundation does not constrain what gets built on it. It just makes building clean: one home per
capability, secrets in the backend, the frontend untouched. Build out as far as the person wants.

See `references.md` for the platform docs behind everything here.
