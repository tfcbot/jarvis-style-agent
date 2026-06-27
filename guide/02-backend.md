# 02 — Backend: build the brain (EVE)

Build the **brain**: the agent that takes a message and replies. It lives in `brain/`, runs on
[EVE](https://eve.dev) (a filesystem-first agent framework), and exposes exactly **one door** the orb
knocks on: an OpenAI-compatible endpoint, `POST /v1/chat/completions`, guarded by a shared secret.

This is the Backend-for-Frontend for the model and tools: the shim holds the model key, runs the
agent loop, and hands back plain streamed text. The orb never sees a model; it just talks to this
door.

> Why OpenAI-compatible? Because it is the lingua franca. The orb speaks it, and later so can any
> other client (an MCP host, a voice avatar, a phone bridge). One door, many callers, no bespoke glue.

## Why EVE

EVE is **filesystem-first**: files under `agent/` *are* the agent. Drop a file in `agent/tools/` and
it is a tool; drop one in `agent/channels/` and it is an HTTP surface. There is no wiring to maintain.
That is exactly what makes the foundation easy to extend later (`07-extensibility.md`): adding
capability is adding a file. EVE requires **Node 24+**.

(If the person would rather not run a stateful agent framework, `07-extensibility.md` shows the same
door built as a minimal Hono API. For the foundation, use EVE.)

## File tree

```
brain/
  package.json
  tsconfig.json
  .node-version                     # 24
  .env                              # not committed
  .env.example
  agent/
    agent.ts                        # the model the brain runs                          (PIN)
    instructions.md                 # who the agent is (persona + voice rules)
    channels/
      openai-compat.ts              # the door: POST /v1/chat/completions               (PIN — verbatim)
```

## Project config

`brain/package.json`:

```json
{
  "name": "brain",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.10",
  "imports": { "#*": "./agent/*" },
  "scripts": {
    "dev": "eve dev",
    "build": "eve build",
    "start": "eve start",
    "typecheck": "tsgo"
  },
  "dependencies": {
    "ai": "7.0.0-canary.171",
    "eve": "latest",
    "zod": "4.4.3"
  },
  "devDependencies": {
    "@types/node": "^25",
    "@typescript/native-preview": "latest"
  },
  "engines": { "node": ">=24" }
}
```

`brain/tsconfig.json` (EVE-friendly; the `#*` import maps to `agent/`):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "types": ["node"]
  },
  "include": ["agent/**/*.ts"]
}
```

## The agent

`brain/agent/agent.ts` — **PIN**. The whole brain is one `defineAgent`. The model is env-driven so a
voice deployment can run a fast model:

```ts
import { defineAgent } from "eve";

export default defineAgent({
  // The brain. For a real-time voice loop, a fast model (claude-haiku-4.5) keeps
  // latency low. Env-driven so a voice deployment can run a faster model than a text one.
  model: process.env.AGENT_MODEL || "anthropic/claude-haiku-4.5",
});
```

> **Model id must be dotted, not dashed.** Use `anthropic/claude-haiku-4.5`. A dashed id
> (`claude-haiku-4-5`) is **not** a gateway id and the build fails with a "no gateway context window
> metadata" error. This is the single most common backend mistake.

## The instructions (the agent's identity)

`brain/agent/instructions.md`. For the foundation, keep it short: a capable assistant tuned for being
heard, not read. `03-context.md` goes deeper on persona; this is enough for "hello."

```markdown
# Identity

You are the person's own voice agent. You live behind a glowing orb: the person talks to you out loud
and hears your replies spoken back. You are capable, calm, and to the point. You are not a chatbot
reading a script.

# Voice

Your replies are heard, not read. So:

- Keep it short. Lead with the answer. One clear point, at most two or three sentences.
- Plain spoken text only. No markdown, no asterisks, no bullet lists, no URLs read aloud.
- If you are about to do something that takes a moment, say a brief filler line first ("one sec")
  so the orb is never silent during the wait.

# Behavior

- Be honest about what you can and cannot do. Right now you can talk. As the person adds tools, you
  will be able to do more — use them when they help, and never invent facts you would need a tool to
  look up.
- When the person greets you, greet them back warmly and briefly, and invite them to tell you what
  they want to do.
```

## The door: the OpenAI-compatible shim

`brain/agent/channels/openai-compat.ts` — **PIN — write verbatim.** This is the most important file in
the backend and the easiest to get subtly wrong. It exposes `POST /v1/chat/completions`, guards it
with a bearer (`BRAIN_SECRET`), runs the full EVE agent loop, and translates EVE's event stream
into OpenAI SSE. Reproduce it exactly.

Two non-obvious things it gets right, do not "simplify" them away:

1. **Each turn is a fresh stateless run built from the transcript.** It does *not* resume one EVE
   session, because a resumed session's `getEventStream` replays from the start and would return the
   first turn's reply forever. The whole conversation is fed in as text each turn.
2. **Tool calls resolve inside EVE and are never surfaced as OpenAI `tool_calls`.** The caller (the
   orb) just hears the natural-language result. `tool-calls` is the only non-terminal finish reason,
   so the stream ends on any other completed message.

```ts
import { defineChannel, POST, type Session } from "eve/channels";
import { createUnauthorizedResponse, extractBearerToken } from "eve/channels/auth";

// The session event union isn't a public export, so recover it from the
// stream type that `getEventStream` resolves to.
type StreamEvent =
  Awaited<ReturnType<Session["getEventStream"]>> extends ReadableStream<infer E> ? E : never;

// OpenAI-compatible chat-completions shim.
//
// Exposes `POST /v1/chat/completions` so anything that speaks the OpenAI
// streaming API can drive this Eve agent as its brain. We run the full Eve agent
// loop (skills + tools) and translate Eve's NDJSON event stream into OpenAI SSE —
// streaming text only. Tool calls resolve entirely inside Eve and are never
// surfaced as OpenAI `tool_calls`; the caller just hears the natural-language result.
//
// Auth: `Authorization: Bearer <BRAIN_SECRET>`.

type ChatMessage = { role?: string; content?: unknown };
type ChatCompletionsBody = {
  messages?: ChatMessage[];
  stream?: boolean;
  model?: string;
  user?: string;
};

const MODEL_ID = "eve-jarvis";

// Stable hash for deriving a conversation key from text.
function hash(s: string): string {
  let h = 2166136261;
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return (h >>> 0).toString(36);
}

// OpenAI lets `content` be a string or an array of content parts; flatten the
// text parts so we always hand Eve a plain string.
function messageText(content: unknown): string {
  if (typeof content === "string") return content;
  if (Array.isArray(content)) {
    return content
      .map((part) =>
        typeof part === "string"
          ? part
          : part && typeof part === "object" && typeof (part as { text?: unknown }).text === "string"
            ? (part as { text: string }).text
            : "",
      )
      .join("");
  }
  return "";
}

function sseChunk(delta: { role?: string; content?: string }, finishReason: string | null, created: number): string {
  const payload = {
    id: `chatcmpl-${created}`,
    object: "chat.completion.chunk",
    created,
    model: MODEL_ID,
    choices: [{ index: 0, delta, finish_reason: finishReason }],
  };
  return `data: ${JSON.stringify(payload)}\n\n`;
}

// True once the stream has reached a terminal assistant reply for the turn.
// `tool-calls` is the only non-terminal assistant finish reason in Eve's loop,
// so any other finishReason on a completed message ends the turn.
function isTerminalEvent(event: StreamEvent): boolean {
  if (event.type === "session.completed" || event.type === "session.waiting") return true;
  if (event.type === "session.failed" || event.type === "turn.failed") return true;
  if (event.type === "message.completed" && event.data.finishReason !== "tool-calls") return true;
  return false;
}

export default defineChannel({
  routes: [
    POST("/v1/chat/completions", async (req, { send }) => {
      const secret = process.env.BRAIN_SECRET;
      if (!secret) {
        return new Response("BRAIN_SECRET not configured", { status: 500 });
      }
      const token = extractBearerToken(req.headers.get("authorization"));
      if (token !== secret) {
        return createUnauthorizedResponse({ message: "invalid api key", challenges: [{ scheme: "Bearer" }] });
      }

      const body = (await req.json().catch(() => ({}))) as ChatCompletionsBody;
      const messages = Array.isArray(body.messages) ? body.messages : [];
      // Clients resend the full transcript each turn (standard OpenAI chat).
      const convo = messages.filter((m) => m.role === "user" || m.role === "assistant");
      const userMsgs = messages.filter((m) => m.role === "user");
      const userMessage = messageText(userMsgs[userMsgs.length - 1]?.content).trim();
      if (!userMessage) {
        return new Response(JSON.stringify({ error: { message: "no user message", type: "invalid_request_error" } }), {
          status: 400,
          headers: { "content-type": "application/json" },
        });
      }

      // This endpoint drives a real-time voice loop, so steer EVE to spoken
      // style per turn. Short + no markdown/URLs = faster to generate and clean
      // for TTS.
      const SPOKEN =
        "[Voice mode: your reply is spoken aloud. Plain text only — no markdown, no asterisks, " +
        "no URLs. One clear point, at most two short sentences.]";
      // Memory: feed the whole conversation each turn so EVE has context. We do
      // NOT reuse one EVE session (a resumed session's getEventStream replays
      // from the start, returning the first turn's reply forever) — each request
      // is a fresh stateless run whose context lives in the transcript.
      const transcript = convo
        .map((m) => `${m.role === "assistant" ? "Jarvis" : "Customer"}: ${messageText(m.content).trim()}`)
        .join("\n");
      const turn =
        convo.length > 1
          ? `${SPOKEN}\n\nConversation so far:\n${transcript}\n\nReply to the Customer's latest message, in context. Don't repeat yourself or re-ask what they already told you.`
          : `${SPOKEN}\n\n${userMessage}`;
      // Unique per turn (messages grow) → a fresh EVE session each request.
      const continuationToken = `oa-${messages.length}-${hash(userMessage)}`;
      const session = await send(turn, { auth: null, continuationToken, mode: "conversation" });
      const events = await session.getEventStream();
      const created = Math.floor(Date.parse(new Date().toISOString()) / 1000);
      const wantStream = body.stream !== false;

      if (wantStream) {
        const encoder = new TextEncoder();
        const stream = new ReadableStream<Uint8Array>({
          async start(controller) {
            const reader = events.getReader();
            let sentRole = false;
            try {
              while (true) {
                const { done, value } = await reader.read();
                if (done) break;
                const event = value as StreamEvent;
                if (event.type === "message.appended") {
                  const delta = event.data.messageDelta;
                  if (delta) {
                    const head = sentRole ? {} : { role: "assistant" as const };
                    sentRole = true;
                    controller.enqueue(encoder.encode(sseChunk({ ...head, content: delta }, null, created)));
                  }
                } else if (isTerminalEvent(event)) {
                  break;
                }
              }
            } catch {
              // fall through to a clean close so the caller isn't left hanging
            } finally {
              controller.enqueue(encoder.encode(sseChunk({}, "stop", created)));
              controller.enqueue(encoder.encode("data: [DONE]\n\n"));
              controller.close();
              reader.releaseLock();
            }
          },
        });
        return new Response(stream, {
          headers: {
            "content-type": "text/event-stream; charset=utf-8",
            "cache-control": "no-cache",
            connection: "keep-alive",
          },
        });
      }

      // Non-streamed: collect to the terminal reply and return one JSON body.
      const reader = events.getReader();
      let text = "";
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        const event = value as StreamEvent;
        if (event.type === "message.appended") {
          text += event.data.messageDelta ?? "";
        } else if (event.type === "message.completed" && event.data.finishReason !== "tool-calls") {
          if (event.data.message) text = event.data.message;
          break;
        } else if (isTerminalEvent(event)) {
          break;
        }
      }
      reader.releaseLock();
      const completion = {
        id: `chatcmpl-${created}`,
        object: "chat.completion",
        created,
        model: MODEL_ID,
        choices: [{ index: 0, message: { role: "assistant", content: text }, finish_reason: "stop" }],
      };
      return new Response(JSON.stringify(completion), {
        status: 200,
        headers: { "content-type": "application/json" },
      });
    }),
  ],
});
```

## Environment

`brain/.env.example` (copy to `brain/.env`, fill in):

```bash
# Vercel AI Gateway credential that runs the agent's model.
AI_GATEWAY_API_KEY=

# Shared bearer the shim validates. MUST equal BRAIN_SECRET in the orb's .env.local.
# Generate any long random string, e.g.  openssl rand -hex 32
BRAIN_SECRET=

# Model the agent runs (routed via the gateway). Use a valid *dotted* gateway id.
# anthropic/claude-haiku-4.5 is fast (good for voice). Dashes will NOT work.
AGENT_MODEL=anthropic/claude-haiku-4.5
```

`brain/.gitignore`:

```
node_modules
.env
.env*
!.env.example
.eve
.workflow-data
.vercel
.output
dist
```

The person provides `AI_GATEWAY_API_KEY` (same gateway as the orb's voice) and picks a value for
`BRAIN_SECRET`. **`BRAIN_SECRET` is one shared bearer used on both sides** — the exact same value goes
in the brain's `.env` (here) and in the orb's `orb/.env.local`. It is the only thing standing between
the public internet and the brain, so make it long and random (e.g. `openssl rand -hex 32`).

## Run + verify locally

```bash
cd brain
bun install

# Build must succeed and emit .output. If it complains about the model,
# the id is dashed. Fix it to a dotted gateway id.
AGENT_MODEL="anthropic/claude-haiku-4.5" bun run build

# Start the brain on port 8787 (match the orb's BRAIN_URL).
PORT=8787 bun run dev
```

Smoke-test the door directly with curl (replace the secret with your `BRAIN_SECRET`):

```bash
curl -N http://127.0.0.1:8787/v1/chat/completions \
  -H "authorization: Bearer $BRAIN_SECRET" \
  -H "content-type: application/json" \
  -d '{"stream":true,"messages":[{"role":"user","content":"hello"}]}'
```

You should see `data:` SSE frames stream a greeting, ending with `data: [DONE]`. A `401` means the
bearer does not match; a `500` mentioning `BRAIN_SECRET` means it is not set.

## Connect the orb to the brain

In `orb/.env.local`, point the orb at this brain:

```bash
BRAIN_MODE=sidecar
BRAIN_URL=http://127.0.0.1:8787
BRAIN_SECRET=        # the SAME value you set in brain/.env
```

Run both (brain on 8787, orb on 3000), open the orb, tap the mic, say "hello." You should hear a
spoken reply from the brain, and the orb should pulse with it. That is "hello" working locally. Next,
deploy the brain so it is always-on.

Checklist:

- [ ] `eve build` succeeds and emits `.output`.
- [ ] curl to `/v1/chat/completions` streams a reply and ends with `[DONE]`.
- [ ] Wrong bearer returns `401`.
- [ ] Orb (`BRAIN_MODE=sidecar`) talks to the brain end to end, with voice.

Next: `05-deploy.md`. Deploy the brain to Vercel; run the orb locally.
