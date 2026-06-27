# Vendors & environment keys

Everything external this build touches, and every environment variable you will set, in one place.
Use it as the credentials checklist. The person creates the accounts and pastes the keys; you (the
agent) wire them in. Secrets live in the **brain** or the **orb** env, **never** in client code.

## Vendors

| Vendor | What it does in this build | Required? | Where the key comes from |
| ------ | -------------------------- | --------- | ------------------------ |
| **Vercel** | Hosts the brain (deploy-on-merge); the orb runs locally | **Required** | vercel.com → account |
| **Vercel AI Gateway** | Runs the **model** (brain) and the **voice** — TTS + STT (orb), keyless | **Required** | Vercel dashboard → AI Gateway → API key |
| **GitHub** | Private repo hosting + the deploy trigger (merge to `main`) | **Required** | github.com → account (+ `gh` CLI) |
| **Anthropic (model)** | The brain's LLM (e.g. `anthropic/claude-haiku-4.5`) | **Required**, but **no separate key** | Routed *through* the AI Gateway — you do not hold an Anthropic key |
| **Cognee** | Durable memory (knowledge graph) when you add `04-memory.md` | Optional | Cognee Cloud → API key |
| **ElevenLabs** | Premium voice (swap from the default gateway voice) | Optional | elevenlabs.io → API key |

Key point: with the AI Gateway, **one credential (`AI_GATEWAY_API_KEY`) covers both the model and the
voice.** You do not need separate OpenAI/Anthropic keys for the foundation. Cognee and ElevenLabs are
the only extra vendor keys, and only if the person opts into memory or premium voice.

## Environment variables — the brain (`brain/.env`)

| Variable | Required? | Purpose |
| -------- | --------- | ------- |
| `AI_GATEWAY_API_KEY` | **Required** | Gateway credential that runs the agent's model |
| `BRAIN_SECRET` | **Required** | Shared bearer guarding `/v1/chat/completions`. **Same value as the orb's `BRAIN_SECRET`.** Generate with `openssl rand -hex 32` |
| `AGENT_MODEL` | **Required** | The model id — a **dotted** gateway id, e.g. `anthropic/claude-haiku-4.5`. Dashes fail the build |
| `COGNEE_API_KEY` | Optional | Cognee Cloud key, only if you added memory (`04-memory.md`) |
| `COGNEE_API_URL` | Optional | Override Cognee's base URL (defaults to the Cognee Cloud API) |
| _(per-tool keys)_ | Optional | Any tool you add in `07-extensibility.md` brings its own key (e.g. `WEATHER_API_KEY`). Always on the brain, never the orb |

## Environment variables — the orb (`orb/.env.local`)

| Variable | Required? | Purpose |
| -------- | --------- | ------- |
| `AI_GATEWAY_API_KEY` | **Required** | Gateway credential that runs the voice (TTS + STT). Same key as the brain |
| `BRAIN_URL` | **Required** | The brain's address. The deployed brain's Vercel URL (or `http://127.0.0.1:8787` for a local brain) |
| `BRAIN_SECRET` | **Required** | The shared bearer. **Same value as the brain's `BRAIN_SECRET`** |
| `BRAIN_MODE` | **Required** | `remote` (the deployed brain, default) or `sidecar` (a brain you run locally) |
| `VOICE_TIER` | Optional | `gateway` (default, keyless) or `elevenlabs` (premium) |
| `VOICE` | Optional | Gateway voice name (default `onyx`) |
| `ELEVENLABS_KEY` | Optional | ElevenLabs API key, only when `VOICE_TIER=elevenlabs` |
| `ELEVENLABS_VOICE` | Optional | ElevenLabs voice id |
| `ELEVENLABS_MODEL` | Optional | ElevenLabs model (default `eleven_flash_v2_5`) |
| `TTS_MODEL` | Optional | Override the gateway TTS model (default `openai/tts-1-hd`) |
| `STT_MODEL` | Optional | Override the gateway STT model (default `openai/gpt-4o-transcribe`) |

## The values that must match

| The brain's... | must equal the orb's... | why |
| -------------- | ----------------------- | --- |
| `BRAIN_SECRET` | `BRAIN_SECRET` | One shared bearer. If they differ, the orb gets `401` / `brain_auth_failed` |
| deployed URL | `BRAIN_URL` | The orb has to know where the brain lives |

And `AGENT_MODEL` must be a **dotted** gateway id everywhere it appears (brain env + any local
override). Dashes are not gateway ids and fail the build.

## Minimum to reach "hello"

Only four values, all from two vendors (Vercel + GitHub):

- Brain: `AI_GATEWAY_API_KEY`, `BRAIN_SECRET`, `AGENT_MODEL`
- Orb: `AI_GATEWAY_API_KEY`, `BRAIN_URL`, `BRAIN_SECRET`, `BRAIN_MODE=remote` (`AI_GATEWAY_API_KEY`
  and `BRAIN_SECRET` are the same values as the brain's)

Everything else (Cognee, ElevenLabs, tool keys) is opt-in, added when the person asks for that
capability. See `references.md` for vendor docs and `docs/devops.md` for how to run the repo/deploy.
