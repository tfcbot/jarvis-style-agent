# 05 — Deploy: the brain on Vercel, the orb on localhost

The brain is the only deployed surface. It runs on Vercel as one project, always on, reachable by its
URL and guarded by a bearer. The orb runs on the user's machine with `bun run dev` and points at the
deployed brain. The orb's server routes hold the secrets, so the orb stays local and never becomes a
public endpoint.

> Use the **Vercel CLI + git integration** (you run the commands; the user does one `vercel login`,
> or sets `VERCEL_TOKEN` for fully headless). Do **not** use the Vercel SDK. That is for platforms
> deploying on someone else's behalf. Here the user deploys to their own Vercel.

## One repo, one deployed project

The repo holds both `orb/` (at the root) and `brain/` (a subfolder). Only the brain becomes a Vercel
project.

| Surface | Runs on | Framework (Vercel preset) | Root Directory |
| ------- | ------- | ------------------------- | -------------- |
| brain   | Vercel, deploy-on-merge   | Eve (auto-detected)       | `brain` |
| orb     | localhost (`bun run dev`) | Next.js                   | repo root |

## Brain → Vercel (deploy-on-merge)

The brain is a Nitro app (`eve build`) that Vercel detects as the **Eve** framework. Set it up
git-connected so a merge to `main` deploys it.

```bash
cd brain
vercel link --project jarvis-brain     # once: create the project
vercel git connect <repo-url>          # connect the GitHub repo for deploy-on-merge
# Then on the project (Dashboard, or the v9 projects API):
#   Root Directory   = brain
#   Production Branch = main
#   Node.js Version   = 24.x
vercel env add AI_GATEWAY_API_KEY      # the gateway credential that runs the model
vercel env add BRAIN_SECRET            # the shared bearer (long random string)
vercel env add AGENT_MODEL             # a DOTTED gateway id, e.g. anthropic/claude-haiku-4.5
# If you added memory (04-memory.md):
# vercel env add COGNEE_API_KEY
```

A merge to `main` builds and deploys the brain. Note its production URL; that is the orb's
`BRAIN_URL`.

> Set the env vars before the first deploy. `eve build` reads `AI_GATEWAY_API_KEY` to resolve the
> model's gateway metadata, so a build with the key missing fails.

> ### The subdirectory gotcha (read this)
> The brain lives in a subfolder of the repo that also holds the orb. Set the brain project's **Root
> Directory = `brain`**. Do **not** add `brain` to a repo-root `.vercelignore`; that strips the
> brain's own source and breaks `eve build` (`eve: command not found`, exit 127). The orb's exclusion
> of the brain lives in the orb's **`tsconfig.json`** (`"exclude": ["brain"]`, set in
> `01-frontend.md`). `tsconfig` keeps the brain out of a local orb build; nothing in `.vercelignore`
> touches `brain`.

## Orb → localhost

The orb runs on the user's machine and talks to the deployed brain. Point it at the brain's
production URL in `orb/.env.local`:

```bash
BRAIN_URL=https://<brain>.vercel.app   # the deployed brain
BRAIN_MODE=remote
BRAIN_SECRET=                          # the SAME value set on the brain
AI_GATEWAY_API_KEY=                    # runs the voice (TTS + STT) from the local orb
```

Run it:

```bash
cd orb
bun install
bun run dev                            # http://localhost:3000
```

The browser calls only the orb's own `/api/*` routes; those hold the gateway key and the brain bearer
server-side. No key reaches the page, and the brain answers only a request carrying the bearer.

> Voice and inference bill to the user's Vercel team (the gateway runs both).

## The values that must match

```
BRAIN_SECRET            same value on the brain AND the orb (the shared bearer)
brain's prod URL   ->   orb's BRAIN_URL
```

`AGENT_MODEL` on the brain is a **dotted** gateway id (e.g. `anthropic/claude-haiku-4.5`). A dashed id
fails the build.

## Why the brain is the deployed surface and the orb is not

The brain's only door is `POST /v1/chat/completions`, guarded by `BRAIN_SECRET`. The bearer is the
lock and never leaves the server. The door speaks the OpenAI protocol, so any OpenAI-compatible client
or agent drives it with `base_url + api_key`, where the api key is the bearer.

The orb's `/api/*` routes hold the gateway key and the brain bearer and run with no auth in front of
them. A publicly deployed orb is an open endpoint that spends the user's gateway credits for anyone
with the link. On the Vercel Pro plan, Deployment Protection does not cover a project's production
domain without the paid Advanced Deployment Protection add-on, so a deployed orb is not gated for
free. Running the orb on localhost keeps the secrets local and removes the public surface. Deploy the
brain; run the orb.

## Ship it

1. Commit the repo (orb + brain), push, open a PR, merge to `main`.
2. The brain project builds and deploys on the merge.
3. Set `BRAIN_URL` in `orb/.env.local` to the brain's prod URL.
4. Run the orb with `bun run dev` and go to `06-verify.md`.

> The user does the `vercel login` (or provides `VERCEL_TOKEN`) and pastes the keys. You run every
> other command. Production deploys happen on merge to `main`; an org policy may block a manual
> `vercel deploy --prod`, so rely on the git connection.

Next: `06-verify.md`.
