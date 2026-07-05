# Vendors & environment keys

Everything external this build touches, and every environment variable you will set, in one place.
Use it as the credentials checklist. The person creates the accounts and pastes the keys; you (the
agent) wire them in. Secrets live in the **brain** or the **orb** env, **never** in client code.

## Vendors

| Vendor | What it does in this build | Required? | Where the key comes from |
| ------ | -------------------------- | --------- | ------------------------ |
| **Vercel** | Hosts the brain (deploy-on-merge); the orb runs locally | **Required** | vercel.com → account |
| **Vercel AI Gateway** | Runs the **model** (brain) and the **realtime voice** (orb mints an ephemeral session token) | **Required** | Vercel dashboard → AI Gateway → API key |
| **GitHub** | Private repo hosting + the deploy trigger (merge to `main`) | **Required** | github.com → account (+ `gh` CLI) |
| **Anthropic (model)** | The brain's LLM (e.g. `anthropic/claude-haiku-4.5`) | **Required**, but **no separate key** | Routed *through* the AI Gateway — you do not hold an Anthropic key |
| **OpenAI (realtime voice)** | The speech model (`openai/gpt-realtime-2`) | **Required**, but **no separate key** | Routed *through* the AI Gateway — you do not hold an OpenAI key |
| **Cognee** | Durable memory (knowledge graph) when you add `04-memory.md` | Optional | Cognee Cloud → API key |

Key point: with the AI Gateway, **one credential (`AI_GATEWAY_API_KEY`) covers both the model and
the realtime voice.** You do not hold separate OpenAI/Anthropic keys at all. Cognee is the only
extra vendor key, and only if the person opts into memory; any metrics/tool vendor added later
brings its own key (see `07-extensibility.md`).

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
| `AI_GATEWAY_API_KEY` | **Required** | Gateway credential that mints the realtime voice token (server-side). Same key as the brain |
| `BRAIN_URL` | **Required** | The brain's address. The deployed brain's Vercel URL (or `http://127.0.0.1:8787` for a local brain) |
| `BRAIN_SECRET` | **Required** | The shared bearer. **Same value as the brain's `BRAIN_SECRET`** |
| `BRAIN_MODE` | **Required** | `remote` (the deployed brain, default) or `sidecar` (a brain you run locally) |
| `REALTIME_MODEL` | Optional | Override the realtime speech model (default `openai/gpt-realtime-2`) |
| `BOOT_ON_LOAD` | Optional | Play the boot overlay on load (default `true`; editable in the settings panel) |

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

Everything else (Cognee, metrics/tool vendor keys) is opt-in, added when the person asks for that
capability. See `references.md` for vendor docs and `docs/devops.md` for how to run the repo/deploy.
