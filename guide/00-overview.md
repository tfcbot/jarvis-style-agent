# 00 — Overview: the mental model

Read this once. It is the spine of everything else. If you understand the four layers and the one
pattern, every later step is just filling them in.

## Four layers

A talking agent is always the same four pieces. Naming them is the whole trick.

### Frontend — what the person talks to

The orb. It runs in the browser. Its entire job is sensory: draw something alive, capture the
person's voice from the mic, play the agent's voice back, and look like it is listening and speaking.
It contains **no intelligence and no secrets**. It does not know what a model is. You could replace
the brain behind it and the orb would not notice.

### Backend — the brain

The agent. It takes a message and returns a reply. This is where the intelligence lives: the model,
the persona, the tools, the keys. In this build the backend is an [EVE](https://eve.dev) agent that
exposes one door the frontend can knock on: an **OpenAI-compatible endpoint** (`POST
/v1/chat/completions`). Anything that can speak that protocol can drive the brain, which is why the
orb (and later, other clients) can talk to it without bespoke glue.

### Context — what it knows during a conversation

Two things, assembled fresh for every turn: the agent's **instructions** (who it is, how it behaves)
and the **transcript** of the conversation so far. Context is short-lived. It is what makes a single
conversation coherent. It is not memory; close the tab and it is gone.

### Memory — what it knows across conversations

Durable facts that outlive a session: "my store is X," "I prefer short answers," "last week we
shipped Y." Memory is a store the agent writes to and reads from on purpose. It is **optional** for
the foundation and added later (Cognee, bring-your-own-key). Build the foundation without it first.

## One pattern: Backend-for-Frontend (BFF)

Here is the rule that ties the layers together and keeps the whole thing safe and simple:

> The frontend talks to **one backend you own**, and nothing else. That backend talks to everything
> else on the frontend's behalf.

The orb never calls a model API. It never calls a voice API. It never holds a key. It calls its own
server routes (`/api/chat`, `/api/speak`, `/api/transcribe`), and *those* call out. Those routes are
the **Backend-for-Frontend**: a thin server layer that exists to serve this one frontend, holding the
secrets and doing the talking.

Three payoffs, every time:

1. **Secrets stay server-side.** The browser bundle is public. Anything in it is leaked. The BFF keeps
   keys where the public cannot reach them.
2. **The frontend stays replaceable.** It speaks only to your routes, in your shapes. Swap the model,
   the voice, or the whole brain behind those routes and the orb is unchanged.
3. **One home per integration.** Want the agent to read your calendar? That is one new path in the
   backend. The frontend never grows a tangle of third-party SDKs and tokens.

In this build BFF shows up twice, by design:

- The **orb's `/api/*` routes** are the BFF for the browser (they hold the voice credential and the
  brain's bearer).
- The **brain's OpenAI-compatible shim** is the BFF for the model and tools (it holds the model key
  and runs the agent loop).

Same idea at two scales. Learn it here and you will see it everywhere.

## What the foundation requires

| Layer    | Needed for "hello"? | Built in            |
| -------- | ------------------- | ------------------- |
| Frontend | yes                 | `01-frontend.md`    |
| Backend  | yes                 | `02-backend.md`     |
| Context  | the minimum (instructions + transcript per turn, which the backend already does) | `03-context.md` to go deeper |
| Memory   | no                  | `04-memory.md` later |

So: build the **Frontend** and the **Backend**, **deploy** them (`05-deploy.md`), and **verify**
"hello" works (`06-verify.md`). That is the foundation. Context and Memory deepen it afterward.

## The build, start to finish

1. **Frontend** (`01-frontend.md`) — scaffold the Next.js orb, write the three BFF routes, the voice
   loop, and the orb render. Run it against a mock brain so it works before the real one exists.
2. **Backend** (`02-backend.md`) — scaffold the EVE brain, write its instructions and the
   OpenAI-compatible shim, run it locally, and point the orb at it. Now "hello" works on localhost.
3. **Deploy** (`05-deploy.md`) — two Vercel projects off one repo, env vars set, deploy-on-merge.
4. **Verify** (`06-verify.md`) — the acceptance checklist. When it passes, **stop**. The foundation is
   done. Tell the person, and ask what they want to build on top.

Then, on request: `03-context.md` (persona/context), `04-memory.md` (Cognee), `07-extensibility.md`
(tools, MCP, backend/voice swaps).

Next: `01-frontend.md`.
