# 05 — Deploy: put it on Vercel, always on

Take the orb and the brain from "works on localhost" to "deployed and always on." Both go to
**Vercel**, as **two projects off the one repo**, each redeploying when the person merges to `main`.
No laptop required after this.

> Use the **Vercel CLI + git integration** (you run the commands; the person does one `vercel login`,
> or sets `VERCEL_TOKEN` for fully headless). Do **not** use the Vercel SDK — that is for platforms
> deploying on someone else's behalf. Here the person deploys to their own Vercel.

## What changes vs local

- **Same code, same wiring.** The request/response shapes are stateless, so chat and voice behave
  identically deployed. No code changes to ship.
- **Re-set env on each Vercel project.** A missing env var is the number-one "worked locally, broke in
  prod" cause.
- **Rewire the URL, keep the secret.** The orb's `BRAIN_URL` points at the deployed brain;
  `BRAIN_SECRET` is the same value on both projects. The orb proxies server-side, so **there is no CORS**.

## One repo, two projects

The repo holds both `orb/` (at the root) and `brain/` (a subfolder). Each becomes its own Vercel
project:

| Project | Framework (Vercel preset) | Root Directory | Redeploys on |
| ------- | ------------------------- | -------------- | ------------ |
| orb     | Next.js (auto-detected)   | repo root      | merge to `main` |
| brain   | Eve (auto-detected)       | `brain`        | merge to `main` |

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
vercel env add BRAIN_SECRET      # the shared bearer (long random string)
vercel env add AGENT_MODEL             # a DOTTED gateway id, e.g. anthropic/claude-haiku-4.5
# If you added memory (04-memory.md):
# vercel env add COGNEE_API_KEY
```

After this, a merge to `main` builds and deploys the brain. Note its production URL; that is the orb's
`BRAIN_URL`.

> ### The subdirectory gotcha (read this)
> The brain lives in a subfolder of the repo that also holds the orb. With the brain project's **Root
> Directory = `brain`**, do **NOT** add `brain` to a repo-root `.vercelignore` — that would strip the
> brain's own source and break `eve build` (`eve: command not found`, exit 127). Keep the orb's
> exclusion of the brain in the orb's **`tsconfig.json`** instead (`"exclude": ["brain"]`, already set
> in `01-frontend.md`). In short: `tsconfig` excludes the brain from the *orb build*; nothing in
> `.vercelignore` touches `brain`.

## Orb → Vercel

Standard Next.js; the CLI detects it.

```bash
cd <repo root>                  # the orb is the root app
vercel link --project jarvis-orb
vercel git connect <repo-url>   # deploy-on-merge (or `vercel deploy --prod` to ship now)
vercel env add AI_GATEWAY_API_KEY   # runs the voice (TTS + STT)
vercel env add BRAIN_URL            # = the deployed brain's URL
vercel env add BRAIN_SECRET         # the SAME value you set on the brain
vercel env add BRAIN_MODE           # set to: sidecar
# Optional voice overrides: VOICE_TIER, VOICE, ELEVENLABS_KEY, TTS_MODEL, STT_MODEL.
```

> Voice and inference bill to the person's Vercel team (the gateway runs both). Optionally put the orb
> page behind a password while testing.

## The env that must match

One value is shared and must be identical on both projects, or the orb cannot reach the brain. The
brain validates the bearer; the orb sends it. Same name, same value:

```
BRAIN_SECRET            same value on the brain AND the orb (the shared bearer)
brain's prod URL   ->   orb's BRAIN_URL
```

And the model id, in both the brain env and any local override, must be a **dotted** gateway id.
Dashes fail the build.

## Ship it

1. Commit the repo (orb + brain), push, open a PR, merge to `main`.
2. Both Vercel projects build and deploy on the merge.
3. Set `BRAIN_URL` on the orb to the brain's now-known prod URL (re-deploy the orb if you set it after
   the first build).
4. Open the orb's URL. Go to `06-verify.md`.

> The person does the `vercel login` (or provides `VERCEL_TOKEN`) and pastes the keys. You can run
> every other command. The brain's token/model calls and the orb's voice calls all bill to their
> Vercel account, by design — it is their system.

Next: `06-verify.md`.
