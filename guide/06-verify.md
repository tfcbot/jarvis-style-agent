# 06 — Verify: say "hello," hear a reply

This is the acceptance test. When it passes, the **foundation is done.** Stop building, tell the
user, and ask what they want to build on top. Do not pre-build anything they have not asked for.

## The one test that matters

Run the orb locally (`bun run dev`) and open `http://localhost:3000`. Tap the status pill
(`○ TAP TO TALK`), allow the microphone, and watch it go `CONNECTING…` → `● MIC LIVE`. Say
**"hello."** You should:

1. Hear a **spoken reply** from the agent, live — no send button, no pipeline pause.
2. See the **orb stir** in time with the voice.
3. Be able to **talk over it mid-sentence and have it stop** and listen (barge-in).

Then ask something only the brain can answer (anything needing its data or tools). The reply takes
a beat longer — that is the tool round-trip: the voice model calls `ask_jarvis_brain`, the orb's
`/api/realtime/ask` runs the **deployed** brain over `BRAIN_URL`, and the voice model speaks the
answer. Voice in, agent brain on Vercel, voice out, all on the user's own infrastructure.

## Full checklist

Frontend (local):

- [ ] `bun run dev` serves the orb at `http://localhost:3000`; the orb renders and animates.
- [ ] The status pill reads `○ TAP TO TALK`; tapping it prompts for microphone permission.
- [ ] The pill walks `CONNECTING…` → `● MIC LIVE`; speaking gets a live spoken reply and the pill
      shows `JARVIS SPEAKING` while it talks.
- [ ] The orb stirs while the agent speaks; **talking over it interrupts it**.
- [ ] A question needing real data round-trips to the brain and comes back spoken.

Backend (deployed):

- [ ] The brain's Vercel URL is live; `POST /v1/chat/completions` with the right bearer streams a
      reply and ends with `data: [DONE]`.
- [ ] A wrong or missing bearer returns `401`.
- [ ] `AGENT_MODEL` is a dotted gateway id; the deploy built without a model-metadata error.

Wiring:

- [ ] The orb's `BRAIN_URL` is the brain's prod URL, and `BRAIN_SECRET` is the same value on the orb
      and the brain.
- [ ] `BRAIN_MODE=remote` on the orb.

Security (the BFF promise):

- [ ] View page source or the Network tab: **no `AI_GATEWAY_API_KEY`, no `BRAIN_SECRET`, no model
      key** appears in the client bundle or any request the browser makes directly. The browser
      calls `/api/realtime/setup` and `/api/realtime/ask`, and opens the gateway WebSocket with only
      the short-lived `vcst_` client token the setup route minted.

## Curl the brain directly

```bash
curl -N https://<brain>.vercel.app/v1/chat/completions \
  -H "authorization: Bearer $BRAIN_SECRET" \
  -H "content-type: application/json" \
  -d '{"stream":true,"messages":[{"role":"user","content":"hello"}]}'
```

Streamed `data:` frames ending in `data: [DONE]` mean the brain is live. A `401` means the bearer is
wrong. This call is also exactly how an agent or any OpenAI-compatible client reaches the brain
(`base_url + api_key`, where the api key is `BRAIN_SECRET`).

You can exercise the tool path without a microphone too:

```bash
curl -s -X POST http://localhost:3000/api/realtime/ask \
  -H 'content-type: application/json' -d '{"message":"hello"}'
```

`{"text":"…"}` means the ask route reaches the brain; the voice model uses this exact path.

## If something is off

The status pill is the diagnostic — a tap always produces a visible state, and failures name
themselves:

- **`⚠ MIC BLOCKED`:** the browser denied the microphone. Fix the site's mic permission and tap
  again.
- **Stuck on `CONNECTING…` / `⚠ CONNECT FAILED — <reason>`:** the session could not open. Check
  `/api/realtime/setup` in the Network tab — a 502 there means `AI_GATEWAY_API_KEY` is missing or
  invalid in `orb/.env.local`, or the `REALTIME_MODEL` id is wrong (default
  `openai/gpt-realtime-2`).
- **Connected, but data questions fail** (the model apologizes or says it cannot reach its brain):
  the tool round-trip is failing. Curl `/api/realtime/ask` as above — `brain_auth_failed` means
  `BRAIN_SECRET` differs between the orb and the brain; `brain_unreachable` means `BRAIN_URL` is
  wrong or the brain is not deployed.
- **It answers itself / overlapping voices on speakers:** the mic is re-capturing the model's own
  output (the WebSocket transport is outside the browser's echo canceller). Use headphones, or flip
  the `HALF_DUPLEX` flag in `RealtimeVoice.tsx` — it mutes the mic while the model speaks, at the
  cost of voice interruption.
- **Brain build failed:** almost always a dashed model id, or `brain` got excluded by a root
  `.vercelignore` (see the subdirectory gotcha in `05-deploy.md`). Confirm Root Directory = `brain`
  and Node 24.x.

## Done. Now hand it back.

Report to the user, plainly: the brain is live at `<url>`, the orb runs locally and talks to it, and
it is all theirs. Then ask the real question:

> "The foundation is up and working. What do you want it to be able to *do*? I can give it tools,
> memory, a sharper persona, live dashboard data, or expose it to your other apps, whichever you
> want first."

From here, build on request using `07-extensibility.md`. The foundation is the clean base;
everything the user dreams up gets built on top of it, one capability at a time.
