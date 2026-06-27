# AGENTS.md — start here

You are a coding agent. A person handed you this repository and asked you to build them a
**Jarvis-style voice agent**: a glowing particle orb they can talk to out loud, backed by a real
agent brain, deployed to the web so it is always on. This file is your entry point. Read it fully
before you touch anything.

This repo is a **guide, not a codebase**. It does not contain the app. It contains the architecture,
the build order, and the hard-to-get-right code pinned verbatim so you can recreate the app cleanly
in the person's own repo, on their own infrastructure, with their own keys. You write the code; you
spend the tokens; they own the result.

## The one job

Get the person to a **working foundation**: a particle-orb frontend wired to an agent backend,
deployed to Vercel, where they can say "hello" and hear a spoken reply. That is the deliverable. It
is the hard part done right, the clean base every powerful Jarvis-style system is built on.

Once "hello" works end to end, the foundation is complete. From there the same person can tell you
"now add X" and you build their backend out cleanly using `guide/07-extensibility.md`. The foundation
is deliberately unopinionated about what they build on top of it.

## The mental model (read `guide/00-overview.md` first)

Four named layers, one pattern that ties them together:

- **Frontend** — the orb the person sees and talks to. Renders, captures voice, plays speech.
- **Backend** — the agent brain. Takes a message, thinks, replies. This is where intelligence lives.
- **Context** — what the agent knows *during* a conversation: its instructions and the live transcript.
- **Memory** — what the agent knows *across* conversations: durable facts that outlive a session.

The pattern is **Backend-for-Frontend (BFF)**: the frontend never talks to a model or a third-party
service directly. It talks to **one backend that the person owns**, and that backend talks to
everything else. Secrets stay server-side, the frontend stays dumb, and every integration has exactly
one home. Learn this once and you can build anything on it.

Frontend and Backend are required for "hello." Context and Memory are taught here and are easy to add,
but they do **not** gate the hello-world foundation. Build the foundation, then extend.

## Build order

Follow these in sequence. Each guide step is self-contained and includes the code to write verbatim.

1. `guide/00-overview.md` — the mental model and the request flow. Understand it before building.
2. `guide/01-frontend.md` — build the orb: Next.js app, the BFF routes, the voice loop, the orb render.
3. `guide/02-backend.md` — build the EVE brain: the agent, its instructions, the OpenAI-compatible shim.
4. `guide/05-deploy.md` — deploy both to Vercel (env vars, deploy-on-merge, the subdirectory gotcha).
5. `guide/06-verify.md` — the acceptance test: say "hello," hear a reply. **Stop here. Foundation done.**

Then, only when the person asks for more:

6. `guide/03-context.md` — shape what the agent knows during a turn.
7. `guide/04-memory.md` — give the agent durable memory (Cognee).
8. `guide/07-extensibility.md` — add tools, skills, MCP, swap the backend or the voice. Build it out.

`docs/architecture.md` has the diagrams. `vendors.md` is the vendor + environment-variable checklist
(every key, brain vs orb, required vs optional). `docs/devops.md` is the repo + deploy workflow
(private repo, branch → merge-to-deploy, worktree cleanup). `references.md` has the platform docs and
recommendations. `PRD.md` is the product definition if you need the why.

## Rules of engagement

- **Build until "hello" works, then stop and report.** Do not pre-build integrations the person has
  not asked for. The foundation is the product; everything else is on request.
- **Pin the pinned code.** Where a guide step says "write this verbatim," reproduce it exactly. These
  snippets (the OpenAI-compatible shim, the voice loop, the orb shader, the BFF routes) are the parts
  that are easy to get subtly wrong. Assemble around them; do not rewrite them from memory.
- **Keep secrets in the backend.** The frontend only ever calls the person's own BFF routes. No model
  key, no provider key, no shim secret ever reaches the browser. If you are about to put a key in
  client code, you have made a mistake.
- **The person does the sign-ins.** You can write every file and run every local command. You cannot
  do browser logins (`vercel login`) or paste the person's API keys for them. When you hit one of
  those, ask for it in one line and continue.
- **Agnostic by design.** This guide assumes nothing about which agent you are. The instructions are
  plain prose and plain code. If your environment differs (package manager, shell), adapt the command,
  not the architecture.
- **Work in a private repo, ship by merging to `main`.** Create the person's repo **private**. Branch
  (or use a worktree) per change, open a PR, merge to `main` (that is the deploy), then **clean up the
  branch and worktree**. Full workflow in `docs/devops.md`. Keys never get committed; only
  `.env.example` files do.

## What "done" looks like

A deployed orb at a URL. The person opens it, taps the mic, says "hello," and hears a spoken reply
generated by their own agent brain running on their own Vercel project with their own keys. Nothing
of theirs runs on anyone else's infrastructure. That is the foundation. Report it, then ask what they
want to build on top.
