# Architecture

This is the whole system on one page. Four layers, one pattern (Backend-for-Frontend), and the path a
single spoken "hello" travels. Read this before building; the guide steps assume it.

## The four layers

```mermaid
flowchart TB
  subgraph FE["FRONTEND  (orb/ — Next.js, the only thing the browser runs)"]
    UI["Particle orb + realtime voice session + HUD/boot overlay<br/>hears and speaks live, pulses while talking"]
    BFF["BFF routes (server-side)<br/>/api/realtime/setup · /api/realtime/ask<br/>/api/metrics · /api/events (SSE) · /api/config — optional"]
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

  GW["Vercel AI Gateway<br/>model · realtime voice (one credential)"]

  BFF -- "Bearer shim secret" --> SHIM
  BFF -- "mints the ephemeral token" --> GW
  UI == "WebSocket: audio both ways<br/>(vcst_ token only)" ==> GW
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

- **Secrets stay server-side.** The browser holds no keys — for the realtime session it carries only
  a short-lived `vcst_` client token the BFF minted. The model key, the gateway key, and the shim
  secret all live in the backend / BFF routes. If a key is in client code, the pattern is broken.
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
  participant GW as Vercel AI Gateway (gpt-realtime-2)
  participant Brain as EVE brain (/v1/chat/completions)

  P->>Orb: taps the status pill
  Orb->>BFF: POST /api/realtime/setup
  BFF->>GW: getToken (AI_GATEWAY_API_KEY)
  GW-->>BFF: ephemeral vcst_ token + tool defs
  BFF-->>Orb: token
  Orb->>GW: open WebSocket (vcst_ token)
  Note over Orb,GW: ● MIC LIVE — audio streams both ways

  P->>Orb: "hello"
  GW-->>Orb: spoken reply (audio), orb pulses
  Note over P,GW: talking over it fires server-VAD speech-started → playback stops (barge-in)

  P->>Orb: "how did the store do this week?"
  GW->>Orb: tool call ask_jarvis_brain({message})
  Orb->>BFF: POST /api/realtime/ask {message}
  BFF->>Brain: POST /v1/chat/completions (Bearer shim secret)
  Brain-->>BFF: OpenAI SSE (streamed deltas)
  BFF-->>Orb: { text } (accumulated)
  Orb->>GW: tool output { text }
  GW-->>Orb: speaks the answer in its own voice
```

Two details that make the voice feel live, both specified in `guide/01-frontend.md`:

- **Turn-taking is server-side voice activity detection.** The session hears the person live; when
  they speak over a reply, the server emits `speech-started`, playback stops, and the interrupted
  reply is truncated — interruption works like talking to a person.
- The session's **playing state drives a shared amplitude signal** the orb reads each frame, so the
  orb **stirs and breathes in time with the voice**.

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
  Local -- "realtime voice (token minted locally)" --> GW
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
