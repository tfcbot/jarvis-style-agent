# 01 — Frontend: build the orb

Build the **orb**: a Next.js app that draws a glowing particle sphere, runs a realtime voice session
the person talks to, and surrounds the orb with a live-metrics HUD. It holds **no secrets** and
contains **no intelligence**. It talks only to its own server routes (the BFF), which do the talking
to everything else.

> Pinned code below is marked **PIN — write verbatim**. Reproduce it exactly. These are the pieces
> that are easy to get subtly wrong (the realtime token mint, the tool round-trip, the streaming SSE
> parse, the orb shader). Everything else you may adapt to the person's setup.

> **The layered UI further down (boot screen, live-metrics HUD, settings) is specified, not pinned.**
> Those sections give the *intent*, a **folder structure to follow**, **contracts to honor**, and a
> **suggested prompt** you can hand your building agent — the agent owns the implementation.
>
> **No "modes."** Do not add brain mode-switching (sidecar/remote/mock) or a mock brain. The orb
> knows where the brain is via `BRAIN_URL`; that is enough. (So the settings panel has no "brain
> mode" selector and there is no `/api/health` mode badge.)

## The voice architecture: two brains, one voice

The person talks to a **realtime speech model** (`openai/gpt-realtime-2`, over the Vercel AI
Gateway) that hears and speaks natively — no transcribe step, no TTS step, no per-sentence latency.
Ordinary conversation is handled by the realtime model itself. It has exactly **one tool**,
`ask_jarvis_brain`, which routes anything needing real data or an action to the EVE brain (the
backend you build in `02-backend.md`) and relays the answer back in its own voice.

The browser never holds `AI_GATEWAY_API_KEY`. A BFF route mints a **short-lived client token**
(`vcst_…`) per session; the browser connects its WebSocket with that. This is the BFF pattern doing
its job: the credential stays server-side, the browser gets a disposable pass.

```
Browser ──POST /api/realtime/setup──> BFF mints ephemeral token (holds AI_GATEWAY_API_KEY)
Browser <═══ WebSocket (audio both ways) ═══> AI Gateway ──> gpt-realtime-2
   model needs data → tool call ask_jarvis_brain
Browser ──POST /api/realtime/ask──> BFF ──bearer──> brain /v1/chat/completions ──> {text} back
```

Three facts about the transport that shape everything else — design around them, do not fight them:

1. **It is a WebSocket with Web Audio playback, not WebRTC.** The browser's acoustic echo canceller
   only operates inside the WebRTC pipeline, so it never sees the model's output. On open speakers
   the mic can re-capture the model's own voice, the server's voice-activity detection reads it as a
   new user turn, and the model answers itself. Request `echoCancellation`/`noiseSuppression`/
   `autoGainControl` on the mic anyway (they clean room noise), default to **full duplex** so
   interruption works, and keep a **half-duplex flag** (disable the mic track while the model is
   playing) as the documented fallback for pure-speaker setups.
2. **The hook keys its session on reference identity.** `experimental_useRealtime` compares `model`
   and `sessionConfig` with `!==` to decide whether to rebuild the session. Pass them as **stable
   module-scope constants**. An inline object literal makes the SDK tear down and rebuild the
   WebSocket on every render — and renders fire on every status change and audio-state toggle.
3. **Barge-in rides on server VAD.** When the server hears the person start talking, it emits
   `speech-started`; the SDK stops playback and truncates the interrupted reply. That means an
   audible mic is what makes interruption work — which is why full duplex is the default.

## Stack

- **Next.js 16**, **React 19** (App Router).
- **three.js** for the orb.
- **d3-shape** / **d3-scale** for the HUD charts (sparklines, bars — the live-metrics panels).
- The **Vercel AI SDK**: `ai` + `@ai-sdk/gateway` (realtime token minting, server-side) and
  `@ai-sdk/react` (the `experimental_useRealtime` hook, browser-side).

The realtime model speaks for itself; the EVE brain answers data questions through the one tool.
Build the brain in `02-backend.md`, then point the orb at it with `BRAIN_URL`. The orb runs locally;
only the brain deploys.

## File tree

```
orb/
  package.json
  tsconfig.json
  next.config.ts
  .env.local                      # not committed; see "Environment" below
  app/
    layout.tsx
    page.tsx
    globals.css
    api/
      realtime/
        setup/route.ts            # BFF: mint the ephemeral realtime token + tool defs (PIN)
        ask/route.ts              # BFF: the tool's landing pad -> brain -> {text}      (PIN)
      metrics/route.ts            # BFF: GET snapshot / POST refresh (live HUD)         (spec)
      events/route.ts             # BFF: SSE feed of live agent activity (optional)     (spec)
      config/route.ts             # BFF: read/write settings to .env.local              (spec)
    components/
      Orb.tsx                     # the particle orb                                    (PIN)
      RealtimeVoice.tsx           # realtime session + status pill + mic control        (spec)
      voice-signal.ts             # shared 0..1 "is speaking" amplitude                 (PIN)
      Dashboard.tsx               # live-metrics HUD panels around the orb              (spec)
      BootSequence.tsx            # boot / loading overlay                              (spec)
      Settings.tsx                # runtime config panel (gear)                         (spec)
  lib/
    domain/chat.ts                # turn shape + stream contract                        (PIN)
    openai-sse.ts                 # parse OpenAI SSE into content deltas                (PIN)
    ports/brain.ts                # the Brain interface                                 (PIN)
    adapters/http-brain.ts        # real brain over the OpenAI-compatible shim          (PIN)
    adapters/brain-select.ts      # pick the brain from env                             (PIN)
    domain/metrics.ts             # metrics snapshot contract for the HUD               (spec)
    metrics/                      # vendor read adapters (installed as skills)          (spec)
    domain/telemetry.ts           # HUD telemetry event contract + reducer (optional)   (spec)
    ports/telemetry-source.ts     # the telemetry feed interface (optional)             (spec)
    server/bus.ts                 # in-process pub/sub the brain publishes to (opt.)    (spec)
    adapters/bus-telemetry.ts     # bus -> telemetry-source adapter (optional)          (spec)
    server/env-file.ts            # read/write .env.local for the settings panel        (spec)
```

## Project config

`package.json`:

```json
{
  "name": "orb",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.10",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@ai-sdk/gateway": "^4.0.12",
    "@ai-sdk/react": "^4.0.16",
    "ai": "^7.0.15",
    "d3-scale": "^4.0.2",
    "d3-shape": "^3.2.0",
    "next": "16.2.9",
    "react": "19.2.4",
    "react-dom": "19.2.4",
    "three": "^0.184.0",
    "zod": "^4.4.3"
  },
  "devDependencies": {
    "@types/d3-scale": "^4.0.9",
    "@types/d3-shape": "^3.1.8",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "@types/three": "^0.184.1",
    "typescript": "^5"
  }
}
```

> Realtime voice (`experimental_useRealtime`, `gateway.experimental_realtime`) ships in the stable
> `ai` v7 / `@ai-sdk/gateway` v4 / `@ai-sdk/react` v4 packages above. Keep them in sync.

`tsconfig.json` — note the `"exclude": ["brain"]`. The brain lives in a sibling folder in the same
repo (you will add it in `02-backend.md`); this keeps it out of the orb's TypeScript build. This
matters for deploy; see `05-deploy.md`.

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules", "brain"]
}
```

`next.config.ts`:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;
```

## The BFF: domain contract

The shape the ask route and the brain adapter agree on. Keep it tiny.

`lib/domain/chat.ts` — **PIN — write verbatim:**

```ts
// Domain: chat turn shape and the brain stream contract.

export type ChatRole = 'user' | 'assistant' | 'system';
export type ChatMessage = { role: ChatRole; content: string };

export type ChatDelta =
  | { type: 'delta'; text: string }
  | { type: 'done' }
  | { type: 'error'; error: string };
```

## The BFF: the brain port + adapters

The ask route does not call the brain directly; it calls a small **`Brain` port** and an env switch
picks the adapter (the HTTP brain). This is the BFF pattern in miniature: the route depends on an
interface, not a provider.

`lib/ports/brain.ts` — **PIN:**

```ts
// Port: the brain. Runs a chat turn and streams reply deltas.

import type { ChatMessage, ChatDelta } from '@/lib/domain/chat';

export interface Brain {
  readonly mode: 'sidecar' | 'remote';
  chat(messages: ChatMessage[], signal: AbortSignal): AsyncIterable<ChatDelta>;
}
```

`lib/openai-sse.ts` — **PIN — write verbatim** (parses the brain's OpenAI-style stream):

```ts
// Parse an OpenAI-style chat-completions SSE stream into content deltas.
// Split on blank lines, read `data:` payloads, stop on [DONE], yield
// choices[0].delta.content.

export async function* parseOpenAISSE(
  body: ReadableStream<Uint8Array>,
  signal: AbortSignal,
): AsyncIterable<string> {
  const reader = body.getReader();
  const decoder = new TextDecoder();
  let buf = '';
  try {
    while (!signal.aborted) {
      const { value, done } = await reader.read();
      if (done) break;
      buf += decoder.decode(value, { stream: true });
      let idx: number;
      while ((idx = buf.indexOf('\n\n')) !== -1) {
        const frame = buf.slice(0, idx);
        buf = buf.slice(idx + 2);
        for (const line of frame.split('\n')) {
          const m = /^data:\s?(.*)$/.exec(line);
          if (!m) continue;
          const payload = m[1];
          if (payload === '[DONE]') return;
          try {
            const json = JSON.parse(payload);
            const content = json?.choices?.[0]?.delta?.content;
            if (typeof content === 'string' && content) yield content;
          } catch {
            // partial/non-JSON keepalive frame — skip
          }
        }
      }
    }
  } finally {
    reader.cancel().catch(() => {});
  }
}
```

`lib/adapters/http-brain.ts` — **PIN** (the real brain: proxy the turn to the brain's
OpenAI-compatible shim with a bearer, stream deltas back):

```ts
// Adapter: the real brain. Proxies a chat turn to the EVE backend over its
// OpenAI-compatible shim, bearer-guarded, and yields content deltas.

import type { Brain } from '@/lib/ports/brain';
import type { ChatMessage, ChatDelta } from '@/lib/domain/chat';
import { parseOpenAISSE } from '@/lib/openai-sse';

export type HttpBrainOpts = {
  mode: 'sidecar' | 'remote';
  url: string;
  secret: string;
  model?: string;
};

export function httpBrain(opts: HttpBrainOpts): Brain {
  return {
    mode: opts.mode,
    async *chat(messages: ChatMessage[], signal: AbortSignal): AsyncIterable<ChatDelta> {
      let res: Response;
      try {
        res = await fetch(`${opts.url.replace(/\/$/, '')}/v1/chat/completions`, {
          method: 'POST',
          headers: { 'content-type': 'application/json', authorization: `Bearer ${opts.secret}` },
          body: JSON.stringify({ model: opts.model, messages, stream: true }),
          signal,
        });
      } catch {
        yield { type: 'error', error: 'brain_unreachable' };
        return;
      }

      if (res.status === 401) { yield { type: 'error', error: 'brain_auth_failed' }; return; }
      if (!res.ok || !res.body) { yield { type: 'error', error: 'brain_unreachable' }; return; }

      for await (const text of parseOpenAISSE(res.body, signal)) {
        yield { type: 'delta', text };
      }
      yield { type: 'done' };
    },
  };
}
```

`lib/adapters/brain-select.ts` — **PIN** (env switch; always the real HTTP brain):

```ts
// Select the brain adapter from env. The orb runs locally and talks to the
// deployed EVE brain over its OpenAI-compatible shim.
//
//   BRAIN_MODE=remote   -> the deployed brain (default; BRAIN_URL = its Vercel URL)
//   BRAIN_MODE=sidecar  -> a brain you run locally (BRAIN_URL = http://127.0.0.1:8787)

import type { Brain } from '@/lib/ports/brain';
import { httpBrain } from '@/lib/adapters/http-brain';

export function selectBrain(): Brain {
  const url = process.env.BRAIN_URL;
  if (!url) throw new Error('BRAIN_URL is not set. Point it at the deployed brain (or a local one).');
  const mode = (process.env.BRAIN_MODE ?? 'remote').toLowerCase() === 'sidecar' ? 'sidecar' : 'remote';
  return httpBrain({ mode, url, secret: process.env.BRAIN_SECRET ?? '', model: process.env.AGENT_MODEL });
}
```

## Realtime voice: the two BFF routes

Two server routes carry the whole voice architecture. The browser calls only these; they hold the
secrets.

`app/api/realtime/setup/route.ts` — **PIN** (mint the ephemeral client token; define the ONE tool).
The tool's `description` is a working contract, not documentation — the realtime model reads it to
decide when to call. Keep the discipline: ordinary conversation is answered natively; the brain is
the **only** source of truth for real data and actions. Tailor the description's examples to what
the person's brain can actually do.

```ts
import { gateway } from '@ai-sdk/gateway';
import { experimental_getRealtimeToolDefinitions as getRealtimeToolDefinitions, tool } from 'ai';
import { z } from 'zod';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

// Two brains, one voice: the realtime model handles ordinary conversation
// itself — natively, no tool call, no added latency. The ONE tool it can
// reach for is this escape hatch to the EVE brain, used only for real data
// or actions, where the brain's confirm-before-write gate is expected to add
// a deliberate beat anyway.
const tools = {
  ask_jarvis_brain: tool({
    description:
      'Ask the operator brain a question or ask it to take an action — anything needing live data, ' +
      'the catalog, writes, or the tools only it has. Do NOT use this for ordinary conversation, ' +
      "small talk, or general knowledge — answer those yourself directly. Pass the person's request " +
      'in their own words; relay the answer back plainly, without adding commentary of your own.',
    inputSchema: z.object({
      message: z.string().describe("the person's request, in their own words"),
    }),
  }),
};

// Ephemeral, short-lived client secret — the browser only ever holds this,
// never AI_GATEWAY_API_KEY.
export async function POST() {
  try {
    const toolDefinitions = await getRealtimeToolDefinitions({ tools });
    const token = await gateway.experimental_realtime.getToken({
      model: process.env.REALTIME_MODEL ?? 'openai/gpt-realtime-2',
      sessionConfig: { tools: toolDefinitions },
    });
    return Response.json({ ...token, tools: toolDefinitions });
  } catch (e) {
    return Response.json(
      { error: e instanceof Error ? e.message : 'realtime_setup_failed' },
      { status: 502 },
    );
  }
}
```

`app/api/realtime/ask/route.ts` — **PIN** (the tool's landing pad: run the message through the
brain, accumulate the streamed reply into one string — a tool result returns whole, not streamed):

```ts
import type { ChatMessage } from '@/lib/domain/chat';
import { selectBrain } from '@/lib/adapters/brain-select';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

export async function POST(req: Request): Promise<Response> {
  let message = '';
  try {
    const body = await req.json();
    message = typeof body?.message === 'string' ? body.message : '';
  } catch {
    return Response.json({ error: 'bad_request' }, { status: 400 });
  }
  if (!message.trim()) return Response.json({ error: 'no_message' }, { status: 400 });

  const messages: ChatMessage[] = [{ role: 'user', content: message }];
  const brain = selectBrain();
  const ac = new AbortController();
  req.signal.addEventListener('abort', () => ac.abort());

  try {
    let text = '';
    for await (const delta of brain.chat(messages, ac.signal)) {
      if (delta.type === 'delta') text += delta.text;
      else if (delta.type === 'error') throw new Error(delta.error);
    }
    return Response.json({ text });
  } catch (e) {
    return Response.json(
      { error: e instanceof Error ? e.message : 'brain_failed' },
      { status: 502 },
    );
  }
}
```

## Realtime voice: the client

`app/components/voice-signal.ts` — **PIN — write verbatim** (a module singleton the voice component
writes and the orb reads, no prop drilling):

```ts
'use client';

// Shared 0..1 "agent is speaking" amplitude. The voice component writes it from
// the realtime session's playing state; the Orb reads it each frame to pulse.

let amplitude = 0;

export const speechAmplitude = {
  set(v: number) { amplitude = v < 0 ? 0 : v > 1 ? 1 : v; },
  get() { return amplitude; },
};
```

`app/components/RealtimeVoice.tsx` — **specified, not pinned.** The contracts it must honor:

- **Stable session references.** `const REALTIME_MODEL = gateway.experimental_realtime(MODEL_ID)`
  and `const SESSION_CONFIG = { instructions, voice: 'alloy', turnDetection: { type: 'server-vad' },
  inputAudioTranscription: {} }` live at **module scope**. Never inline them into the hook call
  (see the transport facts above — inline objects rebuild the session every render).
- **The hook**: `experimental_useRealtime({ model: REALTIME_MODEL, api: { token:
  '/api/realtime/setup' }, sessionConfig: SESSION_CONFIG, onToolCall, onError })`. `onToolCall`
  handles `ask_jarvis_brain` by POSTing `{ message }` to `/api/realtime/ask` and returning
  `{ text }` (or `{ error }`).
- **Connect flow**: `getUserMedia({ audio: { echoCancellation: true, noiseSuppression: true,
  autoGainControl: true } })` → `await realtime.connect()` → `realtime.startAudioCapture(stream)`.
  Separate the two failure modes so the pill can name them: mic permission denied vs. session
  connect failed.
- **Full teardown on unmount**: call `realtime.disconnect()` (via a ref to the latest disconnect),
  not just stopping mic tracks — an undisconnected session keeps listening and talking in the
  background.
- **The always-visible status pill.** One derived phase drives it, so a tap always produces visible
  feedback and a failure names itself: `○ TAP TO TALK` (idle) → `CONNECTING…` → `● MIC LIVE`
  (connected + capturing + not playing) → `JARVIS SPEAKING` (playing) → `⚠ MIC BLOCKED` /
  `⚠ CONNECT FAILED — <reason>` (errors). The pill and the mic button are both tap-to-connect /
  tap-to-end.
- **Orb pulse**: a `requestAnimationFrame` loop easing `speechAmplitude` toward ~0.55 while
  `realtime.isPlaying`, back to 0 when not. No audio analyser needed.
- **Duplex**: full duplex by default (barge-in works). Keep a module-scope `HALF_DUPLEX: boolean =
  false` flag; when true, disable the mic track (`track.enabled = false`) while `isPlaying` — the
  fallback for a pure-speaker setup that starts talking to itself, at the cost of voice interruption.

> **Suggested prompt**
>
> ```
> Build app/components/RealtimeVoice.tsx — the realtime voice control (client component). Use
> experimental_useRealtime from @ai-sdk/react with gateway.experimental_realtime('openai/gpt-realtime-2')
> as a module-scope constant and a module-scope SESSION_CONFIG (instructions persona, voice 'alloy',
> turnDetection server-vad, inputAudioTranscription {}) — never inline these into the hook call. Token
> endpoint: POST /api/realtime/setup. onToolCall: handle 'ask_jarvis_brain' by POSTing {message} to
> /api/realtime/ask and returning its {text} (or {error}). Render an always-visible tappable status
> pill cycling: "○ TAP TO TALK" idle → "CONNECTING…" → "● MIC LIVE" (red, glowing; connected +
> capturing + not playing) → "JARVIS SPEAKING" (playing) → "⚠ MIC BLOCKED" / "⚠ CONNECT FAILED —
> <reason>" on errors — plus a round mic button with the same tap-to-connect/tap-to-end behavior.
> Connect: getUserMedia with echoCancellation/noiseSuppression/autoGainControl, then connect(), then
> startAudioCapture(stream); distinguish mic-permission failure from connect failure. Disconnect stops
> capture, stops mic tracks, and calls realtime.disconnect(); also fully tear down on unmount via a
> ref to the latest disconnect. Drive the shared speechAmplitude signal toward 0.55 while isPlaying
> (rAF ease), 0 otherwise. Include a module-scope HALF_DUPLEX=false flag that, when true, sets the
> mic track's enabled=false while isPlaying (speaker echo fallback; trades away barge-in).
> The persona instructions: a capable, warm, concise voice assistant that answers ordinary
> conversation itself and calls ask_jarvis_brain only for live data or actions, never inventing
> those numbers.
> ```

## The orb

`app/components/Orb.tsx` — **PIN — write verbatim**. A three.js particle sphere (Fibonacci shell +
bright rings) with a bloom pass. It renders into whatever box its parent gives it (sizing from the
container via ResizeObserver, pointer math from the canvas rect). The voice wiring: a `uPulse`
shader uniform eases toward `speechAmplitude.get()` each frame — while the agent talks the orb
**stirs** (breathing expansion, lifted drift) with only a gentle glow lift; movement carries the
"talking" read, not brightness.

```tsx
'use client';

import { useEffect, useRef } from 'react';
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
import { speechAmplitude } from './voice-signal';

const COUNT = 42000;
const ORB_R = 1.75;

// Glowing particle orb: a Fibonacci-sphere shell (even coverage) plus a few
// bright great-circle rings, sampled into the particle system.
function buildOrbPoints(): { pos: Float32Array; bright: Float32Array; count: number } {
  const pos: number[] = [];
  const bright: number[] = [];
  const golden = Math.PI * (3 - Math.sqrt(5));

  const shell = Math.floor(COUNT * 0.86);
  for (let i = 0; i < shell; i++) {
    const y = 1 - (i / (shell - 1)) * 2;     // 1 -> -1
    const rad = Math.sqrt(Math.max(0, 1 - y * y));
    const th = golden * i;
    const jr = 1 + (Math.random() - 0.5) * 0.05; // thin shell jitter
    pos.push(Math.cos(th) * rad * jr, y * jr, Math.sin(th) * rad * jr);
    bright.push(Math.random() < 0.14 ? 0.85 + Math.random() * 0.15 : 0.32 + Math.random() * 0.3);
  }

  // bright great-circle rings for structure
  const rings = COUNT - shell;
  for (let i = 0; i < rings; i++) {
    const a = Math.random() * Math.PI * 2;
    const ringSel = Math.floor(Math.random() * 3);
    let x = Math.cos(a), y = Math.sin(a), z = 0;
    if (ringSel === 1) { z = y; y = 0; }
    else if (ringSel === 2) { z = x; x = 0; y = Math.sin(a); x = Math.cos(a) * 0.0; z = Math.cos(a); }
    const j = 1 + (Math.random() - 0.5) * 0.02;
    pos.push(x * j, y * j, z * j);
    bright.push(0.7 + Math.random() * 0.3);
  }

  return { pos: new Float32Array(pos), bright: new Float32Array(bright), count: bright.length };
}

// Renders into whatever box its parent gives it — sizing comes from the
// container via ResizeObserver, pointer math from the canvas rect, so the same
// living orb works at any scale.
export default function Orb() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const cursorRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current!;
    const container = canvas.parentElement ?? document.body;
    const cursorEl = cursorRef.current!;
    const STILL = new URLSearchParams(window.location.search).has('assembled');

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
    renderer.setClearColor(0x02030a, 1);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100);
    camera.position.set(0, 0, 8.4);

    const orb = buildOrbPoints();
    const C = orb.count;

    const positions  = new Float32Array(C * 3);
    const homes      = new Float32Array(C * 3);
    const velocities = new Float32Array(C * 3);
    const seeds      = new Float32Array(C);
    const sizes      = new Float32Array(C);
    const bright     = orb.bright;

    for (let i = 0; i < C; i++) {
      const hx = orb.pos[i * 3] * ORB_R, hy = orb.pos[i * 3 + 1] * ORB_R, hz = orb.pos[i * 3 + 2] * ORB_R;
      homes[i * 3] = hx; homes[i * 3 + 1] = hy; homes[i * 3 + 2] = hz;
      if (STILL) {
        positions[i * 3] = hx; positions[i * 3 + 1] = hy; positions[i * 3 + 2] = hz;
      } else {
        const r = 6 + Math.random() * 6, th = Math.random() * Math.PI * 2, ph = Math.acos(2 * Math.random() - 1);
        positions[i * 3]     = r * Math.sin(ph) * Math.cos(th);
        positions[i * 3 + 1] = r * Math.sin(ph) * Math.sin(th);
        positions[i * 3 + 2] = r * Math.cos(ph) - 4;
      }
      seeds[i] = Math.random();
      sizes[i] = 0.8 + Math.random() * 0.5 + bright[i] * 0.5;
    }

    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geo.setAttribute('aBright', new THREE.BufferAttribute(bright, 1));
    geo.setAttribute('aSeed', new THREE.BufferAttribute(seeds, 1));
    geo.setAttribute('aSize', new THREE.BufferAttribute(sizes, 1));

    const mat = new THREE.ShaderMaterial({
      transparent: true, depthTest: false, depthWrite: false, blending: THREE.AdditiveBlending,
      uniforms: {
        uTime: { value: 0 },
        uPixelRatio: { value: renderer.getPixelRatio() },
        uColorLo: { value: new THREE.Color('#1452ff') },
        uColorHi: { value: new THREE.Color('#bfe6ff') },
        uPulse: { value: 0 },
      },
      vertexShader: /* glsl */`
        attribute float aBright; attribute float aSeed; attribute float aSize;
        uniform float uTime; uniform float uPixelRatio; uniform float uPulse;
        varying float vBright;
        void main(){
          vBright = aBright;
          vec3 p = position;
          float t = uTime * 0.6 + aSeed * 6.2831;
          // Ambient float, lifted a touch while speaking so the orb visibly
          // stirs — movement carries the "talking" read, not brightness.
          p += (0.015 + uPulse * 0.02) * vec3(sin(t), cos(t * 1.2), sin(t * 0.8));
          // Speaking: a gentle breathing expansion (subtle motion, not a flare).
          p *= 1.0 + uPulse * (0.028 + 0.022 * sin(uTime * 2.1 + aSeed * 6.2831));
          vec4 mv = modelViewMatrix * vec4(p, 1.0);
          gl_PointSize = aSize * uPixelRatio * (7.0 / -mv.z) * (1.0 + uPulse * 0.18);
          gl_Position = projectionMatrix * mv;
        }
      `,
      fragmentShader: /* glsl */`
        uniform vec3 uColorLo; uniform vec3 uColorHi; uniform float uPulse;
        varying float vBright;
        void main(){
          vec2 uv = gl_PointCoord * 2.0 - 1.0;
          float r2 = dot(uv, uv);
          if (r2 > 1.0) discard;
          float core = 1.0 - smoothstep(0.0, 1.0, r2);
          vec3 col = mix(uColorLo, uColorHi, vBright);
          gl_FragColor = vec4(col, core * (0.25 + 0.8 * vBright) * (1.0 + uPulse * 0.18));
        }
      `,
    });

    const points = new THREE.Points(geo, mat);
    points.frustumCulled = false;
    scene.add(points);

    const composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));
    // strength 0.34, radius 0.55, threshold 0.2 — a threshold above 0 means only
    // the brighter ring/core particles bloom, so the orb reads as a structured
    // glowing sphere instead of one washed-out blown-out ball.
    const bloom = new UnrealBloomPass(new THREE.Vector2(1, 1), 0.34, 0.55, 0.2);
    const BLOOM_BASE = 0.34;
    composer.addPass(bloom);
    const finalPass = new ShaderPass({
      uniforms: { tDiffuse: { value: null } },
      vertexShader: `varying vec2 vUv; void main(){ vUv=uv; gl_Position=projectionMatrix*modelViewMatrix*vec4(position,1.0);} `,
      fragmentShader: /* glsl */`
        uniform sampler2D tDiffuse; varying vec2 vUv;
        void main(){
          vec2 uv=vUv; float d=distance(uv,vec2(0.5));
          vec3 c = texture2D(tDiffuse, uv).rgb;
          c *= smoothstep(1.15, 0.32, d);
          gl_FragColor = vec4(c, 1.0);
        }
      `,
    });
    finalPass.renderToScreen = true;
    composer.addPass(finalPass);

    // interaction
    const pointer = new THREE.Vector2(0, 0);
    const pointerWorld = new THREE.Vector3(0, 0, 999);
    const rawPointer = { active: false };
    const raycaster = new THREE.Raycaster();
    const facePlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);
    let cursorTimer: ReturnType<typeof setTimeout>;

    const clampNdc = (v: number) => Math.max(-1.2, Math.min(1.2, v));
    const updatePointer = (cx: number, cy: number) => {
      rawPointer.active = true;
      // NDC relative to the canvas box (not the window), clamped so a pointer
      // far outside a small canvas doesn't yank the rotation.
      const rect = canvas.getBoundingClientRect();
      pointer.x = clampNdc(((cx - rect.left) / rect.width) * 2 - 1);
      pointer.y = clampNdc(-((cy - rect.top) / rect.height) * 2 + 1);
      raycaster.setFromCamera(pointer, camera);
      raycaster.ray.intersectPlane(facePlane, pointerWorld);
      cursorEl.style.transform = `translate(${cx}px, ${cy}px)`;
    };
    const onMove = (e: PointerEvent) => {
      updatePointer(e.clientX, e.clientY);
      cursorEl.classList.add('active');
      clearTimeout(cursorTimer);
      cursorTimer = setTimeout(() => cursorEl.classList.remove('active'), 140);
    };
    const onLeave = () => { rawPointer.active = false; };
    const onTouch = (e: TouchEvent) => { if (e.touches[0]) updatePointer(e.touches[0].clientX, e.touches[0].clientY); };
    window.addEventListener('pointermove', onMove);
    window.addEventListener('pointerleave', onLeave);
    window.addEventListener('touchmove', onTouch, { passive: true });

    // loop
    const REPEL_RADIUS = 1.0, REPEL_FORCE = 0.07, SPRING = 0.014, DAMP = 0.86;
    const clock = new THREE.Clock();
    let intro = STILL ? 1 : 0;
    let spinY = 0;
    let frame = 0;
    let pulse = 0;
    const posAttr = geo.attributes.position as THREE.BufferAttribute;

    const tick = () => {
      const dt = Math.min(clock.getDelta(), 0.05), t = clock.elapsedTime;
      mat.uniforms.uTime.value = t;

      // Pulse the orb with the agent's live speech amplitude (smoothed).
      pulse += (speechAmplitude.get() - pulse) * 0.28;
      mat.uniforms.uPulse.value = pulse;
      // Subtle glow lift while speaking — a gentle bloom bump, not a flare.
      bloom.strength = BLOOM_BASE + pulse * 0.16;

      spinY += dt * 0.16;
      points.rotation.y = spinY + pointer.x * 0.25;
      points.rotation.x = 0.12 - pointer.y * 0.18;

      intro = Math.min(1, intro + dt * 0.5);
      const e = 1 - Math.pow(1 - intro, 3);
      const px = pointerWorld.x, py = pointerWorld.y;
      const active = rawPointer.active && isFinite(px);
      const r2 = REPEL_RADIUS * REPEL_RADIUS;
      const arr = posAttr.array as Float32Array;

      for (let i = 0; i < C; i++) {
        const ix = i * 3, iy = ix + 1, iz = ix + 2;
        const hx = homes[ix] * e, hy = homes[iy] * e, hz = homes[iz] * e;
        let vx = velocities[ix], vy = velocities[iy], vz = velocities[iz];
        vx += (hx - arr[ix]) * SPRING; vy += (hy - arr[iy]) * SPRING; vz += (hz - arr[iz]) * SPRING;

        if (active && intro > 0.85) {
          const dx = arr[ix] - px, dy = arr[iy] - py, d2 = dx * dx + dy * dy;
          if (d2 < r2) {
            const d = Math.sqrt(d2) + 1e-4, f = (1 - d / REPEL_RADIUS) * REPEL_FORCE;
            vx += (dx / d) * f; vy += (dy / d) * f; vz += (0.5 - Math.random()) * f * 1.5;
          }
        }
        vx *= DAMP; vy *= DAMP; vz *= DAMP;
        velocities[ix] = vx; velocities[iy] = vy; velocities[iz] = vz;
        arr[ix] += vx; arr[iy] += vy; arr[iz] += vz;
      }
      posAttr.needsUpdate = true;

      composer.render();
      frame = requestAnimationFrame(tick);
    };

    const resize = () => {
      const w = Math.max(1, container.clientWidth), h = Math.max(1, container.clientHeight);
      renderer.setSize(w, h, false); composer.setSize(w, h); bloom.setSize(w, h);
      camera.aspect = w / h;
      camera.position.z = w / h < 0.85 ? 11 : 8.4;
      camera.updateProjectionMatrix();
    };
    const ro = new ResizeObserver(resize);
    ro.observe(container);
    window.addEventListener('resize', resize);
    resize();
    tick();

    return () => {
      cancelAnimationFrame(frame);
      clearTimeout(cursorTimer);
      ro.disconnect();
      window.removeEventListener('pointermove', onMove);
      window.removeEventListener('pointerleave', onLeave);
      window.removeEventListener('touchmove', onTouch);
      window.removeEventListener('resize', resize);
      geo.dispose(); mat.dispose();
      composer.dispose(); renderer.dispose();
    };
  }, []);

  return (
    <>
      <canvas ref={canvasRef} className="scene" />
      <div ref={cursorRef} className="cursor" />
    </>
  );
}
```

## The layered UI: boot screen, live-metrics HUD, settings

The orb + realtime voice above is the whole *functional* voice product. This section is the
**"Jarvis feel"** layered around it: a boot/loading overlay, a live-metrics HUD, and a settings
panel. None of it holds secrets or intelligence — it is presentation plus two thin data feeds.

Each piece is specified, not pinned: **what it is**, the **folder structure to follow**, the
**contracts to honor**, and a **suggested prompt** to hand your building agent.

### The boot sequence

A full-screen overlay that plays on first load: a glitching `J.A.R.V.I.S.` title, a "CALIBRATING"
percentage ring, and a stream of boot-log lines (`PWR  arc reactor coil … ONLINE`) that reveal one by
one, then fade out to expose the orb. Re-triggerable with the `R` key, a `⟳ REBOOT` button, or
`window.dispatchEvent(new Event('jarvis:boot'))`. Auto-runs on load when `BOOT_ON_LOAD` is true.

Folder:

```
app/components/
  BootSequence.tsx        # the boot overlay (client component, no backend)
```

Example — the data and phase machine that drive it (trimmed):

```tsx
// Each line: [tag, label, status]. Streamed in one by one, ~210ms apart.
const BOOT_LINES: [string, string, string][] = [
  ['PWR', 'arc reactor coil', 'ONLINE'],
  ['CORE', 'neural lattice', 'SYNCED'],
  ['NET', 'uplink handshake', 'OK'],
  ['VOX', 'realtime voice session', 'READY'],
  ['AI', 'inference engine', 'HOT'],
];
type Phase = 'boot' | 'ready' | 'fading' | 'hidden';
// boot: reveal lines + ramp % → ready: "GOOD EVENING. ALL SYSTEMS ONLINE." → fading → hidden.
```

> **Suggested prompt**
>
> ```
> Build app/components/BootSequence.tsx — a full-screen JARVIS boot overlay (Next.js client
> component, no backend). On mount, stream ~8 boot-log lines one by one (each: a system tag like
> PWR/CORE/NET/VOX/AI, a label, and a status like ONLINE/OK), while a "CALIBRATING" percentage ring
> ramps 0→100. Then show a glitchy "J.A.R.V.I.S." title and a time-aware greeting
> ("GOOD MORNING/AFTERNOON/EVENING. ALL SYSTEMS ONLINE."), hold ~850ms, fade out and unmount to
> reveal the orb. Add a scanline overlay for CRT flavor. Re-trigger on the "R" key, a "⟳ REBOOT"
> button, and a window 'jarvis:boot' event. Only auto-run on load when the BOOT_ON_LOAD config flag
> is true. Dark background, cyan-on-black, monospace. Use the example BOOT_LINES + phase machine above.
> ```

### The live-metrics HUD

The orb is the centerpiece; the HUD arranges **around** it — real numbers from the person's own
vendors, not ambient decoration. The layout:

- A **top row of ~6 metric cards**: one headline number each (e.g. sales, orders, average order
  value, sessions, conversion, followers), a small sparkline, and a trend marker (▲/▼/▸) comparing
  the second half of the window against the first.
- A **left column** of store panels: a sales/day area chart with an orders/day bar strip, a
  traffic panel (sessions/day line + conversion %/day), a top-products list, and an **activity
  feed** (live agent events + vendor sync notes).
- A **right column** of social panels: followers per platform (bar + per-account rows with a
  total), recent post impressions (bar strip + per-post rows), a reach panel (per-platform meters +
  totals for impressions/likes/comments/clicks), and an ads panel (spend, impressions, clicks,
  spend/day) when an ad account is connected.
- A **gateway cost tile** (its own small panel): AI Gateway **credits remaining** as the headline
  with **lifetime spend** beneath — the running cost of the brain and the realtime voice, which both
  bill to the same Vercel team. This is *infrastructure* cost and is distinct from the ads panel's
  **ad spend** above; do not conflate the two.
- Small **data tags** orbiting the orb (e.g. `SESS 181`, `FLWR 392`, `CONV 1.1%`) tying the
  centerpiece to the data.

Charts use `d3-shape` + `d3-scale` (lines, areas, bars); keep them quiet — thin strokes, dim grids,
monospace numerals.

**The data model is pull, never poll.** Vendors are read **only** when the person triggers a
REFRESH (a button in the top bar): `POST /api/metrics` re-reads every configured vendor server-side,
persists the snapshot (e.g. `.data/metrics.json`), and returns it; `GET /api/metrics` serves the
last snapshot with its `syncedAt` timestamp on load. No timers, no background vendor calls. The SSE
`/api/events` feed (below) is separate and carries **live agent activity only**.

**The snapshot contract** (`lib/domain/metrics.ts`) is the piece to fix before building panels:
one object with a slice per vendor, where every slice carries `configured: boolean` and an optional
`error`, its headline numbers, and daily series as `{ date: string; value: number }[]` day-points.
Each vendor slice degrades **independently** — one vendor failing never blanks the others. Panels
for an unconfigured vendor render "not connected", never invented numbers.

**Vendor adapters are not built here.** The HUD knows only the snapshot contract; the adapters that
fill it (`lib/metrics/…` — a store platform, a social/ads platform) are installed as skills or built
per `07-extensibility.md`, with their keys in server env. Until one is installed, its panels read
"not connected" and the rest of the HUD works.

**The gateway slice is the one built-in.** Unlike the store/social adapters (installed as skills), a
`gateway` slice reads AI Gateway credits with the `AI_GATEWAY_API_KEY` the BFF already holds — no
extra key, no skill. Its adapter (`lib/metrics/gateway.ts`) makes **one** call, `GET
https://ai-gateway.vercel.sh/v1/credits` with `Authorization: Bearer ${AI_GATEWAY_API_KEY}`, and
maps the `{ balance, total_used }` response to the slice's `{ balance, totalUsed }` (USD); `configured`
is `false` when the key is absent, `error` carries any fetch/auth failure. This single call covers
**both** model and realtime-voice spend — both bill to the same Vercel account — and works on **every
Vercel plan**. Deliberately kept minimal: per-day / per-model spend charts require the `/v1/report`
reporting API (Pro/Enterprise only, billed per query), so they are a later, purely additive upgrade
that drops into the same slice — the base HUD shows the two credit numbers only.

Folder:

```
app/components/
  Dashboard.tsx            # the HUD: metric cards + store/social columns + feed + orb tags
app/api/
  metrics/route.ts         # GET last snapshot / POST refresh-now (reads vendors server-side)
lib/
  domain/metrics.ts        # the snapshot contract (per-vendor slices, DayPoint series)
  metrics/                 # vendor read adapters (installed as skills; not part of the base)
    gateway.ts             # built-in: AI Gateway credits via /v1/credits (uses AI_GATEWAY_API_KEY)
```

> **Suggested prompt**
>
> ```
> Build the live-metrics HUD. (1) lib/domain/metrics.ts: a MetricsSnapshot with a syncedAt timestamp
> and one slice per vendor (store: sales/orders/AOV/sessions/conversion + salesByDay/ordersByDay/
> sessionsByDay/conversionByDay + topProducts; social: accounts with per-platform followers +
> followersTotal, recent posts with impressions/likes/comments/clicks, ads with spend/impressions/
> clicks/spendByDay; gateway: balance + totalUsed in USD). Every slice has configured:boolean and
> error?:string; day series are {date,value}[]. (2) app/api/metrics/route.ts: GET returns the
> persisted snapshot (.data/metrics.json); POST re-reads all configured vendor adapters in
> lib/metrics/ — including the built-in gateway adapter (one GET https://ai-gateway.vercel.sh/v1/
> credits with Authorization: Bearer AI_GATEWAY_API_KEY → {balance,total_used}) — persists, returns
> the fresh snapshot. Never call vendors on GET or on a timer — refresh is user-triggered only. (3)
> app/components/Dashboard.tsx: arrange panels AROUND a central orb stage — a top row of ~6 metric
> cards (value, sparkline, ▲/▼/▸ trend of 2nd half vs 1st half of the window), a left store column
> (sales/day area + orders/day bars, sessions/day line + conversion %/day, top products, activity
> feed), a right social column (followers per platform, post impressions, reach meters, ads spend),
> a gateway cost tile (credits remaining headline + lifetime spend sub-line — infra cost, not ad
> spend), and 3-4 small data tags positioned around the orb. Charts with d3-shape/d3-scale: thin lines, dim
> grids, tabular numerals, cyan-on-dark monospace panels with backdrop blur. A REFRESH button in the
> top bar POSTs /api/metrics and shows "SYNCED <ago>". Slices with configured:false render a "not
> connected" note — never fake numbers. Do NOT add any brain "mode" or mock badge.
> ```

### Live agent activity (optional SSE feed)

A tiny in-process pub/sub bus: the ask route's brain-call path publishes events during a turn
(status/log/metric), and a BFF route streams them to the browser as SSE for the activity feed and
status line. This is **optional** — without it the feed shows vendor sync notes only, and the orb
still pulses from the voice signal. It slots into the existing ports/adapters shape:

```
app/api/
  events/route.ts            # GET: SSE stream of telemetry frames
lib/
  domain/telemetry.ts        # the HudEvent union + HudState + a pure reducer (the wire contract)
  ports/telemetry-source.ts  # interface: an async source of HudEvents
  server/bus.ts              # in-process pub/sub singleton the brain path publishes to
  adapters/bus-telemetry.ts  # adapts the bus to the telemetry-source port
```

The **wire contract** is the one piece worth fixing — server and client must agree on it:

```ts
export type AgentState = 'idle' | 'thinking' | 'tool' | 'speaking' | 'error';
export type HudEvent =
  | { type: 'status'; state: AgentState; model: string; uptime_s: number }
  | { type: 'tool'; name: string; phase: 'start' | 'end'; ms?: number; ok?: boolean }
  | { type: 'log'; ts: number; level: 'info' | 'warn' | 'error'; tag: string; text: string }
  | { type: 'metric'; tokens_per_s?: number; latency_ms?: number };
```

Where events come from: in `http-brain`, publish `status: 'thinking'` when a turn starts, `metric`
updates as tokens stream, and `status: 'speaking' → 'idle'` as it finishes. No "mode" concept —
just activity on the one real brain.

> **Suggested prompt**
>
> ```
> Add an optional live-activity feed. (1) lib/domain/telemetry.ts: define HudEvent (status | tool |
> log | metric), HudState, initialHudState, and a pure applyEvent(state, event) reducer — the shared
> wire contract. (2) lib/server/bus.ts: a globalThis-backed in-process pub/sub singleton (publish /
> subscribe HudEvents; survives HMR). (3) lib/ports/telemetry-source.ts + lib/adapters/bus-telemetry.ts:
> an async-iterable source backed by the bus. (4) app/api/events/route.ts: a GET handler that streams
> the source as SSE (text/event-stream, force-dynamic, with an idle heartbeat). (5) In
> lib/adapters/http-brain.ts, publish to the bus during a turn: status 'thinking' on start, metric
> updates while streaming, status 'speaking' then 'idle' at the end. Fold the events into the
> Dashboard's activity feed and status line. Do NOT add brain modes or a mock brain.
> ```

### Settings

A gear-button panel for editing config at runtime. It reads a non-secret snapshot from
`GET /api/config` and writes a subset back via `POST /api/config`, which persists to `.env.local`.
Fields: brain URL, brain secret, model, and the `boot_on_load` toggle. **No brain "mode" selector**
(see the no-modes note at the top of this file).

Folder:

```
app/components/
  Settings.tsx            # gear panel; GET/POST /api/config
app/api/
  config/route.ts         # GET snapshot (no secrets) + POST persist to .env.local
lib/
  server/env-file.ts      # read/write .env.local helper
```

> **Suggested prompt**
>
> ```
> Build a runtime settings panel. app/components/Settings.tsx: a gear (⚙) button toggling a small
> panel that GETs /api/config on open and POSTs edits on save. Fields: brain URL, brain secret
> (password input, "unchanged" placeholder), model, and a "boot sequence on load" checkbox.
> app/api/config/route.ts: GET returns a snapshot that NEVER echoes secrets (only *_set booleans +
> active enums); POST persists a subset to .env.local (via a lib/server/env-file.ts helper) and
> mirrors into process.env. Do NOT include a brain "mode" selector or a mock option.
> ```

## The page

`app/layout.tsx`:

```tsx
import type { Metadata } from 'next';
import './globals.css';

export const metadata: Metadata = { title: 'Jarvis', description: 'Your voice agent.' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

`app/page.tsx` — the orb is the centerpiece; the HUD arranges around it:

```tsx
import Orb from './components/Orb';
import Dashboard from './components/Dashboard';
import RealtimeVoice from './components/RealtimeVoice';
import Settings from './components/Settings';
import BootSequence from './components/BootSequence';

export default function Page() {
  return (
    <main>
      {/* The orb is the centerpiece; the HUD panels arrange around it. */}
      <div className="orb-stage">
        <Orb />
      </div>
      <Dashboard />
      <RealtimeVoice />
      <Settings />     {/* gear config panel; optional */}
      <BootSequence /> {/* boot overlay on top; gated by BOOT_ON_LOAD */}
    </main>
  );
}
```

`app/globals.css` — base styling for the orb stage and the voice controls; the HUD and boot overlay
bring their own classes (let your building agent generate those from the suggested prompts). Adapt
freely; this is the one place taste is welcome.

```css
:root { color-scheme: dark; }
* { box-sizing: border-box; }
html, body { margin: 0; height: 100%; background: #02030a; color: #cfe6ff;
  font-family: ui-sans-serif, system-ui, -apple-system, sans-serif; overflow: hidden; }

/* The orb's container — the canvas sizes itself to this box. */
.orb-stage { position: fixed; inset: 0; }
.scene { position: absolute; inset: 0; width: 100%; height: 100%; display: block; }
.cursor { position: fixed; top: 0; left: 0; width: 10px; height: 10px; margin: -5px 0 0 -5px;
  border-radius: 50%; background: #bfe6ff; opacity: 0; pointer-events: none;
  transition: opacity .2s; mix-blend-mode: screen; }
.cursor.active { opacity: .7; }

/* Voice control: the status pill + mic button, floating bottom-center. */
.chatbox { position: fixed; left: 50%; bottom: 5vh; transform: translateX(-50%);
  display: flex; flex-direction: column; align-items: center; gap: 14px; z-index: 10; }
.rt-status { display: inline-flex; align-items: center; gap: 8px; padding: 6px 14px;
  border-radius: 999px; font-family: ui-monospace, Menlo, monospace; font-size: 11px;
  letter-spacing: .14em; cursor: pointer; color: #9fc2e8;
  background: rgba(8, 16, 40, .55); backdrop-filter: blur(8px);
  border: 1px solid rgba(120, 170, 255, .25); transition: .18s ease; }
.rt-status .rt-dot { width: 7px; height: 7px; border-radius: 50%; background: currentColor; }
.rt-hot { color: #ff6b6b; border-color: rgba(255, 90, 90, .5); }
.rt-hot .rt-dot { box-shadow: 0 0 8px #ff5a5a; }
.rt-error { color: #ffb454; border-color: rgba(255, 180, 84, .5); }
.chat-mic { width: 56px; height: 56px; border-radius: 50%; cursor: pointer;
  font-size: 20px; color: #cfe6ff; background: rgba(20, 82, 255, .18);
  border: 1px solid rgba(120, 170, 255, .35); transition: transform .1s, background .2s; }
.chat-mic:hover { transform: scale(1.06); }
.chat-mic.hot { background: rgba(255, 60, 60, .35); border-color: rgba(255, 120, 120, .6); }
```

## Environment

Create `orb/.env.local` (never committed). The orb runs locally and points at the brain over
`BRAIN_URL`.

```bash
# The brain's address. Prod: the deployed brain's Vercel URL.
# For a brain running locally, set BRAIN_MODE=sidecar and BRAIN_URL=http://127.0.0.1:8787
BRAIN_URL=https://<brain>.vercel.app
BRAIN_MODE=remote
# Must equal BRAIN_SECRET in the brain (you set this in 02-backend.md).
BRAIN_SECRET=

# Vercel AI Gateway credential. Mints the realtime voice token server-side. Required.
AI_GATEWAY_API_KEY=

# Optional: override the realtime speech model (default openai/gpt-realtime-2).
# REALTIME_MODEL=

# UI: play the boot overlay on load (true/false). Editable in the settings panel.
BOOT_ON_LOAD=true
```

The person pastes `AI_GATEWAY_API_KEY` (you cannot get it for them). It comes from the Vercel
dashboard → AI Gateway. See `references.md`.

## Run it

```bash
cd orb
bun install
bun run dev
```

Open the page. You should see the orb assemble and spin, with the status pill reading
`○ TAP TO TALK`. Tap it, allow the microphone, and watch it go `CONNECTING…` → `● MIC LIVE`. Say
anything — the model hears you live and answers in voice, the orb stirring while it talks. Talk
over it mid-sentence and it stops and listens. Ask it something only the brain knows and it makes
the tool round-trip. Voice needs `AI_GATEWAY_API_KEY`; the tool round-trip needs `BRAIN_URL` and
`BRAIN_SECRET` pointing at a running brain (build it in `02-backend.md`).

> If you only want to confirm the orb renders (no mic/voice), open the page; the orb does not need a
> backend to draw.

Checklist before moving on:

- [ ] Orb renders and animates.
- [ ] Tap the pill → mic permission → `● MIC LIVE`; speaking gets a spoken reply and the orb stirs.
- [ ] Talking over the model interrupts it (barge-in).
- [ ] A question needing real data round-trips through `/api/realtime/ask` to the brain.
- [ ] View source / network: **no key and no secret appear in the client bundle** — the browser
      holds only the short-lived `vcst_` client token.

Next: `02-backend.md`. Build the brain, then point the orb at it.
