# Architecture

This is the whole system on one page. Four layers, one pattern (Backend-for-Frontend), and the path a
single spoken "hello" travels. Read this before building; the guide steps assume it.

## The four layers

```mermaid
flowchart TB
  subgraph FE["FRONTEND  (orb/ — Next.js, the only thing the browser runs)"]
    UI["Particle orb + mic + HUD/boot overlay<br/>captures voice, plays speech, pulses while talking"]
    BFF["BFF routes (server-side)<br/>/api/chat · /api/speak · /api/transcribe<br/>/api/events (SSE) · /api/config — optional"]
    UI <--> BFF
  end

  subgraph BE["BACKEND  (brain/ — the EVE agent, the user owns it)"]
    SHIM["OpenAI-compatible shim<br/>POST /v1/chat/completions (bearer-guarded)"]
    LOOP["Agent loop: model + tools"]
    SHIM --> LOOP
  end

  subgraph CTX["CONTEXT  (per conversation)"]
    INSTR["instructions.md (persona + rules)"]
    TRANS["live transcript (this turn)"]
  end

  subgraph MEM["MEMORY  (across conversations — optional add-on)"]
    DUR["durable store (Cognee, BYO key)"]
  end

  GW["Vercel AI Gateway<br/>model · TTS · STT (keyless)"]

  BFF -- "Bearer shim secret" --> SHIM
  BFF -- "TTS / STT" --> GW
  LOOP -- "runs the model" --> GW
  LOOP -. reads .-> INSTR
  LOOP -. reads .-> TRANS
  LOOP -. "remember / recall" .-> DUR

  classDef opt stroke-dasharray: 4 3;
  class MEM,DUR opt;
```

The dashed Memory layer is optional: the foundation ("hello" works) needs only Frontend, Backend, and
the per-turn Context. Memory is added when the user wants it.

## The pattern: Backend-for-Frontend (BFF)

The single rule that makes the rest fall into place:

> The frontend never talks to a model or a third-party service directly. It talks to **one backend the
> user owns**, and that backend talks to everything else.

Why it matters here:

- **Secrets stay server-side.** The browser holds no keys. The model key, the gateway key, and the
  shim secret all live in the backend / BFF routes. If a key is in client code, the pattern is broken.
- **The frontend stays dumb.** It renders, captures a mic, plays audio. It does not know what a model
  is. Swap the brain and the orb does not change.
- **One home per integration.** Every external service is reached from exactly one place. Adding a
  tool, a data source, or a new surface is a backend change, never a frontend scavenger hunt.

There are two BFF hops in this build, and that is intentional:

1. The orb's own server routes (`/api/*`) are the BFF for the **browser** — they hold the gateway voice
   credential and the brain's bearer.
2. The brain's OpenAI-compatible shim is the BFF for **the model and tools** — it holds the model key
   and runs the agent loop.

## A single "hello," end to end

```mermaid
sequenceDiagram
  participant P as Person
  participant Orb as Orb UI (browser)
  participant BFF as Orb BFF (/api/*)
  participant GW as Vercel AI Gateway
  participant Brain as EVE brain (/v1/chat/completions)

  P->>Orb: taps mic, says "hello"
  Orb->>BFF: POST /api/transcribe (audio bytes)
  BFF->>GW: STT
  GW-->>BFF: "hello"
  BFF-->>Orb: { text: "hello" }

  Orb->>BFF: POST /api/chat (messages, stream)
  BFF->>Brain: POST /v1/chat/completions (Bearer shim secret)
  Brain->>GW: run the model
  GW-->>Brain: reply tokens
  Brain-->>BFF: OpenAI SSE (streamed deltas)
  BFF-->>Orb: SSE deltas

  loop per finished sentence
    Orb->>BFF: POST /api/speak (sentence)
    BFF->>GW: TTS
    GW-->>BFF: mp3 bytes
    BFF-->>Orb: audio
    Orb->>P: plays speech, orb pulses
  end
```

Two details that make the voice feel live, both pinned in `guide/01-frontend.md`:

- The reply is **cut into sentences as it streams**, and each finished sentence is sent to TTS
  immediately. Speech starts before the agent has finished thinking.
- The played audio's amplitude is sampled (Web Audio `AnalyserNode`) into a shared signal the orb
  reads each frame, so the orb **pulses in time with the voice**.

## Deploy topology

```mermaid
flowchart LR
  Repo["One GitHub repo<br/>orb/ + brain/"]
  Local["Developer machine<br/>orb on localhost (bun run dev)"]

  subgraph Vercel
    BrainProj["Vercel project: BRAIN<br/>Eve framework, root = brain/"]
  end

  GW["Vercel AI Gateway"]

  Repo -- "merge to main" --> BrainProj
  Repo -- "clone + run" --> Local
  Local -- "BRAIN_URL + BRAIN_SECRET" --> BrainProj
  Local -- "voice (keyless)" --> GW
  BrainProj -- "model (keyless)" --> GW
```

- **One Vercel project off the repo.** The brain deploys with its **Root Directory set to `brain/`**
  and Vercel's **Eve** framework preset. It redeploys on merge to `main`.
- **The orb runs on localhost** (`bun run dev`) and reaches the brain by URL (`BRAIN_URL`) with a
  shared bearer (`BRAIN_SECRET`, set to the same value on both). The orb proxies server-side, so there
  is **no CORS** to deal with.
- The brain is bearer-guarded and OpenAI-compatible, so it is safe to expose, and any agent reaches it
  with `base_url + api_key`. The orb holds the gateway key and brain bearer in its server routes, so it
  stays local rather than becoming a public endpoint.
- One subdirectory gotcha (the brain lives in a subfolder of the orb's repo) is covered in
  `guide/05-deploy.md`. Get it wrong and the brain build fails; get it right and it is invisible.
