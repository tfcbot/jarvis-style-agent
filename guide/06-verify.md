# 06 — Verify: say "hello," hear a reply

This is the acceptance test. When it passes, the **foundation is done.** Stop building, tell the
user, and ask what they want to build on top. Do not pre-build anything they have not asked for.

## The one test that matters

Run the orb locally (`bun run dev`) and open `http://localhost:3000`. Tap the mic, allow the
microphone, say **"hello,"** tap again to send. You should:

1. Hear a **spoken reply** from the agent.
2. See the **orb pulse** in time with the voice.

The orb is talking to the **deployed** brain over `BRAIN_URL`. Voice in, agent brain on Vercel, voice
out, all on the user's own infrastructure.

## Full checklist

Frontend (local):

- [ ] `bun run dev` serves the orb at `http://localhost:3000`; the orb renders and animates.
- [ ] Tapping the mic prompts for microphone permission and records.
- [ ] After sending, a reply streams into view and is spoken sentence by sentence.
- [ ] The orb pulses while the agent speaks; the stop button barges in.

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
      key** appears in the client bundle or any request the browser makes directly. The browser only
      ever calls `/api/chat`, `/api/speak`, `/api/transcribe`.

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

## If something is off

- **Orb renders but no voice in:** `/api/transcribe` needs `AI_GATEWAY_API_KEY` in `orb/.env.local`.
  Check the Network tab for a 502 from `/api/transcribe`.
- **Transcribes but no reply:** the orb cannot reach the brain. Check `BRAIN_URL` (the brain's prod
  URL) and that `BRAIN_SECRET` is identical on the orb and the brain. `brain_auth_failed` in the reply
  box means the values differ; `brain_unreachable` means the URL is wrong or the brain is not deployed.
- **Reply text but no audio out:** `/api/speak` needs `AI_GATEWAY_API_KEY` too; or the tab has not had
  a user gesture yet (the mic tap counts, so this is rare).
- **Brain build failed:** almost always a dashed model id, or `brain` got excluded by a root
  `.vercelignore` (see the subdirectory gotcha in `05-deploy.md`). Confirm Root Directory = `brain`
  and Node 24.x.

## Done. Now hand it back.

Report to the user, plainly: the brain is live at `<url>`, the orb runs locally and talks to it, and
it is all theirs. Then ask the real question:

> "The foundation is up and working. What do you want it to be able to *do*? I can give it tools,
> memory, a sharper persona, or expose it to your other apps, whichever you want first."

From here, build on request using `07-extensibility.md`. The foundation is the clean base; everything
the user dreams up gets built on top of it, one capability at a time.
