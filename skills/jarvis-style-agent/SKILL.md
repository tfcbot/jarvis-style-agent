---
name: jarvis-style-agent
description: Bootstrap a Jarvis-style voice agent from zero. Use when someone wants to build their own "Jarvis" — a talking particle-orb agent — and needs an agent to set up prerequisites (GitHub + Vercel), a clean workdir, clone the build guide, provision their private repo and Vercel project, and write the .env files. Run this first; it hands off to the guide's AGENTS.md to do the actual build.
---

# Jarvis-Style Agent — bootstrap

You are setting a person up to build a **Jarvis-style voice agent**: a glowing particle orb they talk
to out loud, backed by a real agent brain, deployed so it is always on. This skill is the **on-ramp**.
Your job here is **not** to build the app — it is to get every prerequisite in place so the build can
start cleanly. Walk through the five steps below **in order**. Do not skip ahead; each step gates the
next. When all five are green, hand off to the guide's `AGENTS.md`.

The person owns everything: their accounts, their keys, their repo, their Vercel project. You can run
every local command and write every file. You **cannot** do browser sign-ins or paste their secrets —
when you hit one, ask for it in one line and wait.

Work through it conversationally. Confirm each step out loud before moving on. Never echo a secret back
to the terminal.

---

## Step 1 — Prerequisites: GitHub + Vercel ready

The foundation needs exactly two vendors: **GitHub** (private repo + deploy trigger) and **Vercel**
(hosts the brain; its AI Gateway runs both the model and the voice). Confirm each tool is installed and
authenticated before continuing. Run the checks; only fix what fails.

**Local tooling**

- **Node 24+** — `node --version`. If older or missing, tell them to install Node 24+ (nvm: `nvm install 24 && nvm use 24`).
- **git** — `git --version`.
- **GitHub CLI** — `gh --version`, then `gh auth status`. If not authenticated, the person runs
  `gh auth login` themselves (it is a browser/device login — you cannot do it for them). Ask them to,
  then re-check `gh auth status`.
- **Vercel CLI** — `vercel --version` (install with `npm i -g vercel` if missing), then
  `vercel whoami`. If not logged in, the person runs `vercel login` themselves. Ask, then re-check.

**The one key you need now: the Vercel AI Gateway credential**

The AI Gateway runs **both** the model (brain) and the voice — speech in, speech out — keyless beyond
this one credential. There are **no separate OpenAI/Anthropic keys** for the foundation.

- Ask the person to create one: Vercel dashboard → **AI Gateway** → **API key**.
- They paste it to you as `AI_GATEWAY_API_KEY`. Hold it for Step 5 — **do not** echo it back or write
  it anywhere yet.

Do not proceed until: `node`≥24, `gh auth status` clean, `vercel whoami` resolves, and you have the
`AI_GATEWAY_API_KEY` value in hand.

---

## Step 2 — A clean, dedicated workdir

This project gets its own home. **Not the Desktop, not Downloads, not a folder that already has
files in it.** A stray directory is how repos rot and how the wrong files get committed.

- Ask the person where they want it, or propose a sensible default like `~/code/jarvis` or
  `~/projects/jarvis`.
- The directory must be **empty** (or not yet exist). Check it:
  ```bash
  DIR="$HOME/code/jarvis"   # use the person's chosen path
  [ -e "$DIR" ] && [ -n "$(ls -A "$DIR" 2>/dev/null)" ] && echo "NOT EMPTY — pick another" || echo "ok"
  ```
- If it is not empty, ask for a different path. Do not delete anything to make room.
- Create it and make it the working directory:
  ```bash
  mkdir -p "$DIR"
  cd "$DIR"
  ```

Confirm the chosen path back to the person before continuing.

---

## Step 3 — Clone the build guide

The guide is **reference material**, not the app. Your agent reads it and recreates the app from it.
Clone it into the workdir.

```bash
git clone https://github.com/tfcbot/jarvis-style-agent.git guide
```

This gives you `guide/AGENTS.md` (your build entry point), `guide/PRD.md` (the spec), `guide/vendors.md`
(the env-var checklist), and `guide/guide/00`–`07` (the ordered build steps). Read **`guide/AGENTS.md`**
now so you know what the build expects — but do not start building yet. Finish bootstrap first.

> The guide is a guide, not a fork. The person's actual app lives in their **own** repo (next step),
> not inside this clone. Keep the clone alongside as the thing you read from.

---

## Step 4 — The person's private repo + Vercel project

The app the person builds lives in **their own private GitHub repo** and deploys to **their own Vercel
project**. Scaffold the project root next to the guide, then create both artifacts.

```bash
cd "$DIR"
mkdir -p app && cd app          # the person's project root (orb/ at root, brain/ as a subfolder later)
git init
printf "node_modules/\n.env\n.env.local\n.vercel/\n.next/\n" > .gitignore
git add -A
git commit -m "chore: initialize" --allow-empty
```

**GitHub — private, always.** Never public, never a fork of the guide. This repo will hold the
person's config and product.

```bash
gh repo create <name> --private --source=. --remote=origin --push
```

Pick `<name>` with the person (e.g. `my-jarvis`). Confirm it created **private** with
`gh repo view --json visibility`.

**Vercel — link a project for the brain.** The brain deploys here on merge-to-`main`; the orb runs
locally. Link the project now so env vars have a home in Step 5. (The brain lives in a `brain/`
subfolder the build creates later — that is fine; link the project root now.)

```bash
vercel link        # the person picks/creates the Vercel project, interactively
```

If `vercel link` needs choices only the person can make (team, project name), let them drive that one
command and continue once it reports linked.

---

## Step 5 — Write the .env files

Now place the environment files with the **right keys**. The foundation needs only **four values**,
all from the two vendors already set up. Secrets live in gitignored `.env` files locally **and** in
Vercel project env vars — never committed, never echoed.

Generate the shared bearer once; the **same value** goes in both files:

```bash
BRAIN_SECRET=$(openssl rand -hex 32)
```

**The brain — `brain/.env`** (the build creates `brain/`; you can write the file ahead of it):

```bash
mkdir -p brain
cat > brain/.env <<EOF
AI_GATEWAY_API_KEY=<the key the person gave you in Step 1>
BRAIN_SECRET=$BRAIN_SECRET
AGENT_MODEL=anthropic/claude-haiku-4.5
EOF
```

`AGENT_MODEL` must be a **dotted** gateway id (`anthropic/claude-haiku-4.5`). Dashes are not gateway
ids and fail the build.

**The orb — `orb/.env.local`:**

```bash
mkdir -p orb
cat > orb/.env.local <<EOF
AI_GATEWAY_API_KEY=<same key as the brain>
BRAIN_URL=http://127.0.0.1:8787
BRAIN_SECRET=$BRAIN_SECRET
BRAIN_MODE=remote
EOF
```

`BRAIN_URL` is a placeholder until the brain deploys — the build updates it to the deployed Vercel URL
in `guide/guide/05-deploy.md`. `BRAIN_SECRET` **must match** between the two files (it already does,
since both came from the one `$BRAIN_SECRET`). If they differ, the orb gets `401 / brain_auth_failed`.

**Mirror the secrets into Vercel** (so the deployed brain has them):

```bash
cd app   # the linked project root
printf '%s' "<the AI_GATEWAY_API_KEY>" | vercel env add AI_GATEWAY_API_KEY production
printf '%s' "$BRAIN_SECRET"            | vercel env add BRAIN_SECRET production
printf '%s' "anthropic/claude-haiku-4.5" | vercel env add AGENT_MODEL production
```

Confirm `.env` and `.env.local` are gitignored (`git status` shows neither). Commit **only**
`.env.example` stubs if you make them — never the real files.

> **Optional, later (do not set now):** `COGNEE_API_KEY` (memory, `guide/guide/04-memory.md`) and
> `ELEVENLABS_KEY` (premium voice). Both are opt-in add-ons the person brings in when they ask for that
> capability.

---

## Done — hand off to the build

When all five are green:

- ✅ Node 24+, `gh` and `vercel` authenticated, `AI_GATEWAY_API_KEY` in hand.
- ✅ A clean, dedicated workdir.
- ✅ The guide cloned into `guide/`.
- ✅ The person's **private** GitHub repo created and a Vercel project linked.
- ✅ `brain/.env` and `orb/.env.local` written with the four required values; secrets in Vercel.

Tell the person bootstrap is complete, then **read `guide/AGENTS.md` and follow its build order** to
build the orb and the brain, deploy the brain, and stop at "say hello, hear a reply." That clean
foundation is the deliverable. From there, the person tells you what to build on top and you use
`guide/guide/07-extensibility.md`.
