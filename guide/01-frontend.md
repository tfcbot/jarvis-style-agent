# 01 — Frontend: build the orb

Build the **orb**: a Next.js app that draws a glowing particle sphere, captures the person's voice,
plays the agent's voice back, and pulses in time with it. It holds **no secrets** and contains **no
intelligence**. It talks only to its own server routes (the BFF), which do the talking to everything
else.

> Pinned code below is marked **PIN — write verbatim**. Reproduce it exactly. These are the pieces
> that are easy to get subtly wrong (the streaming SSE parse, the sentence cutter, the Web-Audio
> amplitude loop, the orb shader). Everything else you may adapt to the person's setup.

## Stack

- **Next.js 16**, **React 19** (App Router).
- **three.js** for the orb.
- The **Vercel AI SDK** (`ai`) with **`@ai-sdk/gateway`** for keyless voice (TTS + STT), and
  **`@ai-sdk/elevenlabs`** for the optional premium voice.

Build the orb against a **mock brain** first (included below). It streams a canned reply with no
backend, so you can get the whole frontend working before `02-backend.md` exists. Then flip one env
var to point at the real brain.

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
      chat/route.ts               # BFF: proxy a turn to the brain, stream SSE back   (PIN)
      speak/route.ts              # BFF: text -> speech (server-side)                  (PIN)
      transcribe/route.ts         # BFF: speech -> text (server-side)                  (PIN)
    components/
      Orb.tsx                     # the particle orb                                   (PIN)
      ChatBox.tsx                 # mic + stop controls, runs a turn                   (PIN)
      voice.ts                    # speaker (TTS queue) + mic recorder                 (PIN)
      voice-signal.ts             # shared 0..1 "is speaking" amplitude                (PIN)
  lib/
    domain/chat.ts                # turn shape + the BFF->browser stream contract      (PIN)
    cut-sentences.ts              # split streamed text into sentences for TTS         (PIN)
    openai-sse.ts                 # parse OpenAI SSE into content deltas               (PIN)
    ports/brain.ts                # the Brain interface                                (PIN)
    adapters/http-brain.ts        # real brain over the OpenAI-compatible shim         (PIN)
    adapters/mock-brain.ts        # offline canned brain (build/dev without a backend) (PIN)
    adapters/brain-select.ts      # pick the brain from env                            (PIN)
    voice/tts.ts                  # pick TTS model + voice by tier                     (PIN)
```

## Project config

`package.json`:

```json
{
  "name": "orb",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@ai-sdk/elevenlabs": "^3.0.0-canary.49",
    "@ai-sdk/gateway": "^4.0.0-canary.107",
    "ai": "^7.0.0-canary.176",
    "next": "16.2.9",
    "react": "19.2.4",
    "react-dom": "19.2.4",
    "three": "^0.184.0"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "@types/three": "^0.184.1",
    "typescript": "^5"
  }
}
```

> Voice (`experimental_generateSpeech` / `experimental_transcribe`) needs the AI SDK **canary**
> versions above. Keep them.

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

This is the shape the browser and the routes agree on. Keep it tiny.

`lib/domain/chat.ts` — **PIN — write verbatim:**

```ts
// Domain: chat turn shape and the BFF->browser stream contract for /api/chat.

export type ChatRole = 'user' | 'assistant' | 'system';
export type ChatMessage = { role: ChatRole; content: string };

// What /api/chat streams to the browser (one JSON object per SSE `data:` frame).
export type ChatDelta =
  | { type: 'delta'; text: string }
  | { type: 'done' }
  | { type: 'error'; error: string };
```

## The BFF: the brain port + adapters

The orb does not call the brain directly from a route; it calls a small **`Brain` port** and an env
switch picks the adapter (the real HTTP brain, or the offline mock). This is the BFF pattern in
miniature: the route depends on an interface, not a provider.

`lib/ports/brain.ts` — **PIN:**

```ts
// Port: the brain. Runs a chat turn and streams reply deltas.

import type { ChatMessage, ChatDelta } from '@/lib/domain/chat';

export interface Brain {
  readonly mode: 'sidecar' | 'remote' | 'mock';
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

`lib/adapters/mock-brain.ts` — **PIN** (offline: lets the whole frontend run with no backend):

```ts
// Adapter: a keyless mock brain. Streams a canned reply word-by-word so the
// chat loop and orb can be exercised with no EVE backend (build/dev/offline).

import type { Brain } from '@/lib/ports/brain';
import type { ChatMessage, ChatDelta } from '@/lib/domain/chat';

const REPLY =
  'Hello. I am online and listening. Everything looks nominal. ' +
  'Ask me anything, or tell me what you would like to build.';

function delay(ms: number, signal: AbortSignal): Promise<void> {
  return new Promise((resolve) => {
    const id = setTimeout(resolve, ms);
    signal.addEventListener('abort', () => { clearTimeout(id); resolve(); }, { once: true });
  });
}

export function mockBrain(): Brain {
  return {
    mode: 'mock',
    async *chat(_messages: ChatMessage[], signal: AbortSignal): AsyncIterable<ChatDelta> {
      await delay(280, signal);
      const words = REPLY.split(' ');
      for (let i = 0; i < words.length; i++) {
        if (signal.aborted) break;
        yield { type: 'delta', text: i === 0 ? words[i] : ` ${words[i]}` };
        await delay(45, signal);
      }
      yield { type: 'done' };
    },
  };
}
```

`lib/adapters/brain-select.ts` — **PIN** (env switch; default is the real sidecar/remote brain, fall
back to the mock when no URL is configured or `BRAIN_MODE=mock`):

```ts
// Select the brain adapter from env. Default `sidecar` (HTTP brain against a
// local EVE shim). Falls back to the mock when no URL, or BRAIN_MODE=mock.

import type { Brain } from '@/lib/ports/brain';
import { httpBrain } from '@/lib/adapters/http-brain';
import { mockBrain } from '@/lib/adapters/mock-brain';

export function selectBrain(): Brain {
  const mode = (process.env.BRAIN_MODE ?? 'sidecar').toLowerCase();
  const url = process.env.BRAIN_URL ?? 'http://localhost:8787';
  const secret = process.env.BRAIN_SECRET ?? '';
  const model = process.env.AGENT_MODEL;

  if (mode === 'sidecar' || mode === 'remote') {
    if (url) return httpBrain({ mode, url, secret, model });
  }
  return mockBrain();
}
```

## The BFF routes

Three server routes. The browser calls only these; they hold the secrets.

`app/api/chat/route.ts` — **PIN** (run a turn against the selected brain, stream deltas as SSE):

```ts
import type { ChatMessage } from '@/lib/domain/chat';
import { selectBrain } from '@/lib/adapters/brain-select';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

export async function POST(req: Request): Promise<Response> {
  let messages: ChatMessage[] = [];
  try {
    const body = await req.json();
    messages = Array.isArray(body?.messages) ? body.messages : [];
  } catch {
    return Response.json({ error: 'bad_request' }, { status: 400 });
  }
  if (messages.length === 0) return Response.json({ error: 'no_messages' }, { status: 400 });

  const brain = selectBrain();
  const ac = new AbortController();
  req.signal.addEventListener('abort', () => ac.abort());
  const encoder = new TextEncoder();

  const stream = new ReadableStream<Uint8Array>({
    async start(controller) {
      try {
        for await (const delta of brain.chat(messages, ac.signal)) {
          controller.enqueue(encoder.encode(`data: ${JSON.stringify(delta)}\n\n`));
        }
      } catch {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ type: 'error', error: 'brain_failed' })}\n\n`));
      } finally {
        try { controller.close(); } catch { /* already closed */ }
      }
    },
    cancel() { ac.abort(); },
  });

  return new Response(stream, {
    headers: {
      'content-type': 'text/event-stream; charset=utf-8',
      'cache-control': 'no-cache, no-transform',
      connection: 'keep-alive',
    },
  });
}
```

`lib/voice/tts.ts` — **PIN** (pick the TTS model + voice; default keyless gateway, optional
ElevenLabs). The speak route uses this:

```ts
// Select the TTS model + voice by tier. Default: keyless gateway
// (openai/tts-1-hd, onyx). Premium: ElevenLabs (Daniel) when VOICE_TIER=
// elevenlabs and ELEVENLABS_KEY is set. Both run server-side; no key reaches the page.

import { experimental_generateSpeech as generateSpeech } from 'ai';
import { gateway } from '@ai-sdk/gateway';
import { createElevenLabs } from '@ai-sdk/elevenlabs';

type SpeechModelArg = Parameters<typeof generateSpeech>[0]['model'];

const ELEVEN_DANIEL = 'onwK4e9ZLuTAKqWW03F9';

export type TtsChoice = { model: SpeechModelArg; voice: string };

export function selectTts(requestedVoice?: string): TtsChoice {
  const tier = (process.env.VOICE_TIER ?? 'gateway').toLowerCase();

  const elKey = process.env.ELEVENLABS_KEY ?? process.env.ELEVENLABS_API_KEY;
  if (tier === 'elevenlabs' && elKey) {
    const el = createElevenLabs({ apiKey: elKey });
    return {
      model: el.speech(process.env.ELEVENLABS_MODEL ?? 'eleven_flash_v2_5'),
      voice: requestedVoice ?? process.env.ELEVENLABS_VOICE ?? ELEVEN_DANIEL,
    };
  }

  return {
    model: gateway.speechModel(process.env.TTS_MODEL ?? 'openai/tts-1-hd'),
    voice: requestedVoice ?? process.env.VOICE ?? 'onyx',
  };
}
```

`app/api/speak/route.ts` — **PIN** (text → mp3, server-side):

```ts
import { experimental_generateSpeech as generateSpeech } from 'ai';
import { selectTts } from '@/lib/voice/tts';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

export async function POST(req: Request): Promise<Response> {
  let text = '';
  let voice: string | undefined;
  try {
    const body = await req.json();
    text = typeof body?.text === 'string' ? body.text : '';
    voice = typeof body?.voice === 'string' ? body.voice : undefined;
  } catch {
    return Response.json({ error: 'bad_request' }, { status: 400 });
  }
  if (!text.trim()) return Response.json({ error: 'no_text' }, { status: 400 });

  try {
    const tts = selectTts(voice);
    const { audio } = await generateSpeech({ model: tts.model, text, voice: tts.voice, outputFormat: 'mp3' });
    return new Response(new Uint8Array(audio.uint8Array), {
      headers: { 'content-type': 'audio/mpeg', 'cache-control': 'no-store' },
    });
  } catch {
    return Response.json({ error: 'tts_failed' }, { status: 502 });
  }
}
```

`app/api/transcribe/route.ts` — **PIN** (speech → text, server-side):

```ts
import { experimental_transcribe as transcribe } from 'ai';
import { gateway } from '@ai-sdk/gateway';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

export async function POST(req: Request): Promise<Response> {
  const bytes = new Uint8Array(await req.arrayBuffer());
  if (bytes.length === 0) return Response.json({ error: 'no_audio' }, { status: 400 });

  try {
    const { text } = await transcribe({
      model: gateway.transcriptionModel(process.env.STT_MODEL ?? 'openai/gpt-4o-transcribe'),
      audio: bytes,
    });
    return Response.json({ text });
  } catch {
    return Response.json({ error: 'transcription_failed' }, { status: 502 });
  }
}
```

## The voice loop

This is what makes it feel live. Two ideas: **cut the reply into sentences as it streams** and speak
each one immediately; and **sample the playing audio's amplitude** into a shared signal the orb reads.

`lib/cut-sentences.ts` — **PIN — write verbatim** (guards the decimal-point case so "12.5" is not a
sentence end):

```ts
// Split streamed text into complete sentences for per-sentence TTS, holding
// back the trailing partial.

export function cutSentences(s: string): { done: string[]; rest: string } {
  const done: string[] = [];
  let start = 0;
  for (let i = 0; i < s.length; i++) {
    const c = s[i];
    if (c !== '.' && c !== '!' && c !== '?' && c !== '…') continue;
    if (c === '.' && /[0-9]/.test(s[i - 1] ?? '') && /[0-9]/.test(s[i + 1] ?? '')) continue;
    let j = i + 1;
    while (j < s.length && /\s/.test(s[j]!)) j++;
    done.push(s.slice(start, i + 1).trim());
    start = j;
    i = j - 1;
  }
  return { done, rest: s.slice(start) };
}
```

`app/components/voice-signal.ts` — **PIN — write verbatim** (a module singleton the speaker writes
and the orb reads, no prop drilling):

```ts
'use client';

// Shared 0..1 "agent is speaking" amplitude. The speaker writes it from the live
// TTS audio envelope; the Orb reads it each frame to pulse.

let amplitude = 0;

export const speechAmplitude = {
  set(v: number) { amplitude = v < 0 ? 0 : v > 1 ? 1 : v; },
  get() { return amplitude; },
};
```

`app/components/voice.ts` — **PIN — write verbatim** (sentence-queued speaker through Web Audio +
one-shot mic recorder; the amplitude RMS loop drives the orb):

```ts
'use client';

import { cutSentences } from '@/lib/cut-sentences';
import { speechAmplitude } from './voice-signal';

// Browser-side voice: a sentence-queued speaker (TTS via /api/speak, played in
// order through Web Audio so we can read its amplitude, interruptible) and a
// one-shot mic recorder (-> /api/transcribe). The amplitude drives the orb pulse.

export type Speaker = {
  push: (delta: string) => void;
  finish: () => void;
  stop: () => void;
};

let _ctx: AudioContext | null = null;
let _analyser: AnalyserNode | null = null;
function getAudio(): { ctx: AudioContext; analyser: AnalyserNode } {
  if (!_ctx) {
    const Ctx = window.AudioContext ?? (window as unknown as { webkitAudioContext: typeof AudioContext }).webkitAudioContext;
    _ctx = new Ctx();
    _analyser = _ctx.createAnalyser();
    _analyser.fftSize = 256;
    _analyser.connect(_ctx.destination);
  }
  if (_ctx.state === 'suspended') void _ctx.resume().catch(() => {});
  return { ctx: _ctx, analyser: _analyser! };
}

export function createSpeaker(opts?: { onIdle?: () => void }): Speaker {
  let buffer = '';
  const queue: string[] = [];
  let pumping = false;
  let stopped = false;
  let current: AudioBufferSourceNode | null = null;
  let ampRaf = 0;

  const sampleAmplitude = () => {
    const { analyser } = getAudio();
    const data = new Uint8Array(analyser.fftSize);
    const loop = () => {
      analyser.getByteTimeDomainData(data);
      let sum = 0;
      for (let i = 0; i < data.length; i++) {
        const v = (data[i] - 128) / 128;
        sum += v * v;
      }
      speechAmplitude.set(Math.min(1, Math.sqrt(sum / data.length) * 2.4));
      ampRaf = requestAnimationFrame(loop);
    };
    cancelAnimationFrame(ampRaf);
    ampRaf = requestAnimationFrame(loop);
  };
  const stopAmplitude = () => {
    cancelAnimationFrame(ampRaf);
    ampRaf = 0;
    speechAmplitude.set(0);
  };

  async function pump() {
    if (pumping) return;
    pumping = true;
    sampleAmplitude();
    while (!stopped && queue.length > 0) {
      const sentence = queue.shift()!;
      try {
        const res = await fetch('/api/speak', {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ text: sentence }),
        });
        if (stopped || !res.ok) continue;
        const { ctx, analyser } = getAudio();
        const audioBuf = await ctx.decodeAudioData(await res.arrayBuffer());
        if (stopped) break;
        await new Promise<void>((resolve) => {
          const src = ctx.createBufferSource();
          src.buffer = audioBuf;
          src.connect(analyser);
          src.onended = () => resolve();
          current = src;
          src.start();
        });
        current = null;
      } catch {
        // skip this sentence on failure
      }
    }
    pumping = false;
    stopAmplitude();
    if (!stopped && queue.length === 0) opts?.onIdle?.();
  }

  const enqueue = (s: string) => {
    const t = s.trim();
    if (t) queue.push(t);
  };

  return {
    push(delta) {
      if (stopped) return;
      buffer += delta;
      const { done, rest } = cutSentences(buffer);
      buffer = rest;
      done.forEach(enqueue);
      void pump();
    },
    finish() {
      if (stopped) return;
      enqueue(buffer);
      buffer = '';
      void pump();
    },
    stop() {
      stopped = true;
      queue.length = 0;
      if (current) { try { current.stop(); } catch { /* already stopped */ } current = null; }
      stopAmplitude();
      opts?.onIdle?.();
    },
  };
}

export type Recorder = { stop: () => Promise<Blob> };

export async function startRecording(): Promise<Recorder> {
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  const mr = new MediaRecorder(stream);
  const chunks: BlobPart[] = [];
  mr.ondataavailable = (e) => { if (e.data.size) chunks.push(e.data); };
  mr.start();
  return {
    stop: () =>
      new Promise<Blob>((resolve) => {
        mr.onstop = () => {
          stream.getTracks().forEach((t) => t.stop());
          resolve(new Blob(chunks, { type: mr.mimeType || 'audio/webm' }));
        };
        mr.stop();
      }),
  };
}

export async function transcribe(blob: Blob): Promise<string> {
  const res = await fetch('/api/transcribe', {
    method: 'POST',
    headers: { 'content-type': blob.type || 'audio/webm' },
    body: blob,
  });
  if (!res.ok) throw new Error('transcription_failed');
  const { text } = (await res.json()) as { text: string };
  return text;
}
```

## The orb

`app/components/Orb.tsx` — **PIN — write verbatim**. A three.js particle sphere (Fibonacci shell +
bright rings) with a bloom pass. The key wiring for the voice: a `uPulse` shader uniform that each
frame eases toward `speechAmplitude.get()`, so the orb expands and brightens while the agent talks.

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

function buildOrbPoints(): { pos: Float32Array; bright: Float32Array; count: number } {
  const pos: number[] = [];
  const bright: number[] = [];
  const golden = Math.PI * (3 - Math.sqrt(5));

  const shell = Math.floor(COUNT * 0.86);
  for (let i = 0; i < shell; i++) {
    const y = 1 - (i / (shell - 1)) * 2;
    const rad = Math.sqrt(Math.max(0, 1 - y * y));
    const th = golden * i;
    const jr = 1 + (Math.random() - 0.5) * 0.05;
    pos.push(Math.cos(th) * rad * jr, y * jr, Math.sin(th) * rad * jr);
    bright.push(Math.random() < 0.14 ? 0.85 + Math.random() * 0.15 : 0.32 + Math.random() * 0.3);
  }

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

export default function Orb() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const cursorRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current!;
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
          p += 0.015 * vec3(sin(t), cos(t * 1.2), sin(t * 0.8));
          p *= 1.0 + uPulse * 0.05;
          vec4 mv = modelViewMatrix * vec4(p, 1.0);
          gl_PointSize = aSize * uPixelRatio * (7.0 / -mv.z) * (1.0 + uPulse * 0.5);
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
          gl_FragColor = vec4(col, core * (0.25 + 0.8 * vBright) * (1.0 + uPulse * 0.7));
        }
      `,
    });

    const points = new THREE.Points(geo, mat);
    points.frustumCulled = false;
    scene.add(points);

    const composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));
    const bloom = new UnrealBloomPass(new THREE.Vector2(1, 1), 0.5, 0.65, 0.0);
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

    const pointer = new THREE.Vector2(0, 0);
    const pointerWorld = new THREE.Vector3(0, 0, 999);
    const rawPointer = { active: false };
    const raycaster = new THREE.Raycaster();
    const facePlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);
    let burst = 0;
    let cursorTimer: ReturnType<typeof setTimeout>;

    const updatePointer = (cx: number, cy: number) => {
      rawPointer.active = true;
      pointer.x = (cx / window.innerWidth) * 2 - 1;
      pointer.y = -(cy / window.innerHeight) * 2 + 1;
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
    const onDown = () => { burst = 1.0; };
    const onTouch = (e: TouchEvent) => { if (e.touches[0]) updatePointer(e.touches[0].clientX, e.touches[0].clientY); };
    window.addEventListener('pointermove', onMove);
    window.addEventListener('pointerleave', onLeave);
    window.addEventListener('pointerdown', onDown);
    window.addEventListener('touchmove', onTouch, { passive: true });

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
      bloom.strength = 0.5 + pulse * 0.9;

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
        if (burst > 0.001) {
          const dx = arr[ix], dy = arr[iy], dz = arr[iz], d = Math.sqrt(dx * dx + dy * dy + dz * dz) + 1e-3;
          vx += (dx / d) * burst * 0.04; vy += (dy / d) * burst * 0.04; vz += (dz / d) * burst * 0.04;
        }
        vx *= DAMP; vy *= DAMP; vz *= DAMP;
        velocities[ix] = vx; velocities[iy] = vy; velocities[iz] = vz;
        arr[ix] += vx; arr[iy] += vy; arr[iz] += vz;
      }
      posAttr.needsUpdate = true;
      burst *= 0.9;

      composer.render();
      frame = requestAnimationFrame(tick);
    };

    const resize = () => {
      const w = window.innerWidth, h = window.innerHeight;
      renderer.setSize(w, h, false); composer.setSize(w, h); bloom.setSize(w, h);
      camera.aspect = w / h;
      camera.position.z = w / h < 0.85 ? 11 : 8.4;
      camera.updateProjectionMatrix();
    };
    window.addEventListener('resize', resize);
    resize();
    tick();

    return () => {
      cancelAnimationFrame(frame);
      clearTimeout(cursorTimer);
      window.removeEventListener('pointermove', onMove);
      window.removeEventListener('pointerleave', onLeave);
      window.removeEventListener('pointerdown', onDown);
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

## The controls

`app/components/ChatBox.tsx` — **PIN — write verbatim**. Voice-only: a mic button (tap to talk, tap
again to send) and a stop button to barge in. It runs a turn: transcribe → `/api/chat` (SSE) →
feed deltas to the speaker, which speaks and drives the orb.

```tsx
'use client';

import { useRef, useState } from 'react';
import type { ChatDelta, ChatMessage } from '@/lib/domain/chat';
import { createSpeaker, startRecording, transcribe, type Recorder, type Speaker } from './voice';

export default function ChatBox() {
  const [reply, setReply] = useState('');
  const [busy, setBusy] = useState(false);
  const [recording, setRecording] = useState(false);
  const [speaking, setSpeaking] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const history = useRef<ChatMessage[]>([]);
  const recorder = useRef<Recorder | null>(null);
  const speaker = useRef<Speaker | null>(null);

  async function runTurn(text: string) {
    if (!text.trim() || busy) return;
    setReply('');
    setError(null);
    setBusy(true);
    setSpeaking(true);
    history.current = [...history.current, { role: 'user', content: text }];
    speaker.current = createSpeaker({ onIdle: () => setSpeaking(false) });

    try {
      const res = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ messages: history.current, stream: true }),
      });
      if (!res.ok || !res.body) throw new Error('request_failed');

      const reader = res.body.getReader();
      const decoder = new TextDecoder();
      let buf = '';
      let acc = '';
      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        buf += decoder.decode(value, { stream: true });
        let idx: number;
        while ((idx = buf.indexOf('\n\n')) !== -1) {
          const frame = buf.slice(0, idx);
          buf = buf.slice(idx + 2);
          const m = /^data:\s?(.*)$/m.exec(frame);
          if (!m) continue;
          const ev = JSON.parse(m[1]) as ChatDelta;
          if (ev.type === 'delta') { acc += ev.text; setReply(acc); speaker.current?.push(ev.text); }
          else if (ev.type === 'error') setError(ev.error);
        }
      }
      speaker.current?.finish();
      if (acc) history.current = [...history.current, { role: 'assistant', content: acc }];
    } catch {
      setError('brain_unreachable');
      setSpeaking(false);
    } finally {
      setBusy(false);
    }
  }

  async function toggleMic() {
    speaker.current?.stop();
    if (recording) {
      const rec = recorder.current;
      recorder.current = null;
      setRecording(false);
      if (!rec) return;
      try {
        const blob = await rec.stop();
        setBusy(true);
        const text = await transcribe(blob);
        setBusy(false);
        if (text.trim()) await runTurn(text);
      } catch {
        setBusy(false);
        setError('transcription_failed');
      }
      return;
    }
    try {
      recorder.current = await startRecording();
      setRecording(true);
      setError(null);
    } catch {
      setError('mic_unavailable');
    }
  }

  function stopSpeaking() {
    speaker.current?.stop();
    setSpeaking(false);
  }

  return (
    <div className="chatbox">
      {(reply || error) && (
        <div className={`chat-reply${error ? ' err' : ''}`}>{error ? `⚠ ${error}` : reply}</div>
      )}
      <div className="chat-controls">
        <button
          type="button"
          className={`chat-mic${recording ? ' rec' : ''}`}
          onClick={toggleMic}
          aria-label={recording ? 'Stop and send' : 'Speak'}
          title={recording ? 'Stop and send' : 'Speak'}
        >
          {recording ? '■' : '🎙'}
        </button>
        <button
          type="button"
          className="chat-stop"
          onClick={stopSpeaking}
          disabled={!speaking}
          aria-label="Stop talking"
          title="Stop talking"
        >
          🔇
        </button>
      </div>
    </div>
  );
}
```

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

`app/page.tsx`:

```tsx
import Orb from './components/Orb';
import ChatBox from './components/ChatBox';

export default function Page() {
  return (
    <main>
      <Orb />
      <ChatBox />
    </main>
  );
}
```

`app/globals.css` — minimal styling for the classes the components use (orb canvas fills the screen;
controls float bottom-center). Adapt freely; this is the one place taste is welcome.

```css
:root { color-scheme: dark; }
* { box-sizing: border-box; }
html, body { margin: 0; height: 100%; background: #02030a; color: #cfe6ff;
  font-family: ui-sans-serif, system-ui, -apple-system, sans-serif; overflow: hidden; }

.scene { position: fixed; inset: 0; width: 100vw; height: 100vh; display: block; }
.cursor { position: fixed; top: 0; left: 0; width: 10px; height: 10px; margin: -5px 0 0 -5px;
  border-radius: 50%; background: #bfe6ff; opacity: 0; pointer-events: none;
  transition: opacity .2s; mix-blend-mode: screen; }
.cursor.active { opacity: .7; }

.chatbox { position: fixed; left: 50%; bottom: 5vh; transform: translateX(-50%);
  display: flex; flex-direction: column; align-items: center; gap: 14px; z-index: 10; }
.chat-reply { max-width: 70ch; padding: 10px 16px; border-radius: 14px; text-align: center;
  background: rgba(8, 16, 40, .55); backdrop-filter: blur(8px);
  border: 1px solid rgba(120, 170, 255, .18); font-size: 15px; line-height: 1.4; }
.chat-reply.err { color: #ff9b9b; border-color: rgba(255, 120, 120, .35); }
.chat-controls { display: flex; gap: 12px; }
.chat-mic, .chat-stop { width: 56px; height: 56px; border-radius: 50%; cursor: pointer;
  font-size: 20px; color: #cfe6ff; background: rgba(20, 82, 255, .18);
  border: 1px solid rgba(120, 170, 255, .35); transition: transform .1s, background .2s; }
.chat-mic:hover, .chat-stop:hover:not(:disabled) { transform: scale(1.06); }
.chat-mic.rec { background: rgba(255, 60, 60, .35); border-color: rgba(255, 120, 120, .6); }
.chat-stop:disabled { opacity: .35; cursor: default; }
```

## Environment

Create `orb/.env.local` (never committed). For the foundation, the only required value is the gateway
credential for voice; everything else has a working default.

```bash
# Brain selection. Default 'sidecar' talks to a local brain at BRAIN_URL.
# Set BRAIN_MODE=mock to run the whole UI with NO backend (canned replies).
BRAIN_MODE=sidecar
BRAIN_URL=http://127.0.0.1:8787
# Must equal OPENAI_SHIM_SECRET in the brain (you set this in 02-backend.md).
BRAIN_SECRET=

# Vercel AI Gateway credential — runs the voice (TTS + STT). Required.
# (Or run the UI under `vercel dev` to use an OIDC token instead.)
AI_GATEWAY_API_KEY=

# Voice: 'gateway' (keyless, default) or 'elevenlabs' (premium, needs ELEVENLABS_KEY).
VOICE_TIER=gateway
VOICE=onyx
# ELEVENLABS_KEY=
```

The person pastes `AI_GATEWAY_API_KEY` (you cannot get it for them). It comes from the Vercel
dashboard → AI Gateway. See `references.md`.

## Run it (against the mock brain)

```bash
cd orb
npm install        # or bun / pnpm
# Run with the mock brain so the UI works before the backend exists:
BRAIN_MODE=mock npm run dev
```

Open the page. You should see the orb assemble and spin. Tap the mic, allow the microphone, say
anything, tap again to send. With `BRAIN_MODE=mock` the canned reply streams in, gets spoken sentence
by sentence, and the orb pulses with the voice. If that works, the **entire frontend is correct** and
the only thing left is to point it at a real brain.

> If you only want to confirm the orb renders (no mic/voice), open the page; the orb does not need a
> backend to draw.

Checklist before moving on:

- [ ] Orb renders and animates.
- [ ] Mic captures, `/api/transcribe` returns text (needs `AI_GATEWAY_API_KEY`).
- [ ] `BRAIN_MODE=mock`: a turn streams a reply, it is spoken, the orb pulses.
- [ ] View source / network: **no key and no secret appear in the client bundle.**

Next: `02-backend.md` — build the real brain, then flip `BRAIN_MODE` back to `sidecar`.
