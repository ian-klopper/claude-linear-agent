# linear-routines-bridge

## Two versions of this bridge

There are two implementations of this idea. Pick the one that matches how you pay for Claude:

- **This repo** drives Claude via [Claude Code Routines](https://claude.ai/code/routines). Use it if you want to lean on a Claude.ai subscription instead of paying for API usage. Honestly, it's a hacky workaround — it piggybacks on a consumer surface to avoid API billing.
- **[NorthIsUp/claude-linear-agent](https://github.com/NorthIsUp/claude-linear-agent)** drives Claude via the Managed Agents Sessions API. Use it if you have an Anthropic API key. No Routine to configure, sessions persist across restarts, and it ships a Helm chart and multi-arch Docker images for production deploys.

Both work. The fork is the cleaner integration; this repo exists for people who'd rather not put Anthropic usage on a credit card.

## What it does

Bridges the gap between [Linear](https://linear.app) and [Claude Code Routines](https://claude.ai/code/routines).

Assign a Linear issue to Claude, and this bridge spins up a Claude cloud session with the full issue context. Claude does the work in its own sandbox and (if you set it up) comments back on the issue with progress and results.

I built this because I wanted to hand off Linear tickets to Claude without leaving Linear or copy-pasting context. Each Claude cloud session works directly off `origin/main` — it clones the repo, does the work, and you can watch the session by clicking a link in the Linear sidebar.

## How a hand-off looks

1. You assign a Linear issue to the agent user.
2. The bridge spins up a Claude Code Routine with the issue title, description, and metadata.
3. The bridge posts a link to the Claude session in the Linear issue sidebar — you can click it to watch Claude work.
4. Claude does the work off `origin/main` in its own cloud sandbox.
5. If you've set up the Linear MCP (see below), Claude posts status comments and the final summary back on the Linear issue.
6. If you reply in the Linear agent thread, the bridge starts a **fresh** Claude session. (More on that under "Gotchas.")

## Recommended: set up Linear MCP access for Claude

Without MCP, Claude does the work but has no way to talk back to Linear — you'd only see output in the Claude session itself.

With the Linear MCP connector enabled in your Routine, Claude can:

- Read the issue and its prior comments for context.
- Post progress updates as Linear comments.
- Post a final summary when it finishes.

You set this up once in the Routine config at [claude.ai/code/routines](https://claude.ai/code/routines). In the Routine prompt, tell Claude to sign every comment with something like `— Claude Code Agent` so readers can tell it's from the bot, not from you.

## Setup

You'll do this three times: once in Linear, once in Claude, once in your `.env`.

### 1. Create a Linear OAuth app

Go to [linear.app/settings/api/applications](https://linear.app/settings/api/applications) → **Create new**.

- **Actor:** `App user` (this creates a dedicated agent user — don't pick "User")
- **Scopes:** `read`, `write`, `app:assignable`
- **Redirect URI:** `<BASE_URL>/oauth/callback`
- **Webhook URL:** `<BASE_URL>/webhook`
- **Webhook events:** check `Agent session events`

Copy the `Client ID`, `Client secret`, and `Webhook signing secret`.

### 2. Create a Claude Routine

Go to [claude.ai/code/routines](https://claude.ai/code/routines) → **Create a Routine**.

- Connect the GitHub repo you want Claude to work in.
- Connect the Linear MCP (recommended — lets Claude comment back on issues).
- In the Routine prompt, tell Claude how to behave. Something like:

  ```
  You were invoked from a Linear issue. The human is NOT watching your
  Claude session — they only see what you post to Linear.

  1. Keep your own session output terse. It's scratch space.
  2. Post all user-facing updates as Linear comments via the Linear MCP.
  3. Sign every comment "— Claude Code Agent".
  4. On follow-up invocations, read the Linear issue's prior comments
     first — you do not remember previous sessions.
  ```

Copy the Routine's trigger ID (starts with `trig_`) and an Anthropic API key with Routines access.

### 3. Configure `.env`

```sh
cp .env.example .env
```

Fill in these six values:

| Variable | What it is |
|----------|------------|
| `LINEAR_CLIENT_ID` | From your Linear OAuth app |
| `LINEAR_CLIENT_SECRET` | From your Linear OAuth app |
| `LINEAR_WEBHOOK_SECRET` | The webhook signing secret from Linear |
| `CLAUDE_ROUTINE_ID` | Your Routine's trigger ID (`trig_…`) |
| `CLAUDE_ROUTINE_TOKEN` | Anthropic API key with Routines access |
| `BASE_URL` | Public HTTPS URL where this bridge is reachable |

### 4. Run it

```sh
npm install
npm run dev
```

Then visit `<BASE_URL>/oauth/authorize` once and approve the app. You should see `Installed for workspace: <name>` in the server logs.

### 5. Try it

Assign any issue to the agent user in Linear. Within a few seconds you should see activity appear in the Linear issue's agent sidebar, and a link to the Claude cloud session.

## Running it on a public URL

Linear needs to reach the bridge over HTTPS. Pick one:

**Local + cloudflared** (easiest for testing, no signup):

```sh
brew install cloudflared
cloudflared tunnel --url http://localhost:3001
```

Use the printed `https://...trycloudflare.com` URL as `BASE_URL`.

**Local + ngrok** (also fine):

```sh
brew install ngrok
ngrok config add-authtoken <your-token>
ngrok http 3001
```

**A hosting platform** (Fly, Railway, Render, etc.): set the env vars as secrets, and use the platform's HTTPS URL as `BASE_URL`. There's no database or queue — just the one Node process.

> Heads up: free-tier tunnel URLs change when you restart them. If the URL changes, update both `BASE_URL` in `.env` **and** the Redirect URI + Webhook URL in your Linear app settings.

## Gotchas (worth knowing before you use it heavily)

**1. Replies start a brand new Claude session.**
Every time you reply to the agent in Linear, the bridge fires a fresh Claude Routine. Claude is told to read the prior Linear comments via MCP to catch up — but that means every reply re-reads context and re-clones the repo. It can get expensive with long back-and-forths. Keep replies meaningful. *(The fork linked above doesn't have this limitation — its sessions are persistent.)*

**2. No "done" signal.**
The bridge doesn't know when Claude is finished. It posts the session link and moves on. You find out Claude is done either by watching the Claude session or by seeing a summary comment on the Linear issue (which is why MCP is recommended).

**3. Comments appear as whoever connected the MCP.**
This is a Linear/MCP limitation — Claude can't post as the agent user. That's why the sign-off in the Routine prompt matters.

**4. Restarts wipe the OAuth token.**
The token lives in memory. If the server restarts, you have to visit `/oauth/authorize` again. For local dev, set `DEV_PERSIST_TOKEN=1` to cache it in a gitignored file. Don't use that in production.

**5. Protect the webhook secret.**
Anyone with `LINEAR_WEBHOOK_SECRET` can fake Linear events and burn your Anthropic credits. Treat it like the Anthropic API key.

## Contributing

If you spot something broken or see a better way to do it, please open an issue or PR — feedback welcome.

### Dev setup

```sh
git clone <this-repo>
cd claude-linear-agent
npm install
cp .env.example .env
# fill in the six values from the Setup section above
npm run dev
```

`npm run dev` uses `tsx watch` so it reloads on file changes. In another terminal, start a tunnel so Linear can reach your local server:

```sh
cloudflared tunnel --url http://localhost:3001
```

Paste the `https://...trycloudflare.com` URL into `BASE_URL` in `.env`, and into the Redirect URI and Webhook URL fields in your Linear app settings. Then visit `<BASE_URL>/oauth/authorize` once to install.

From there, assigning a Linear issue to the agent user should trigger the whole flow end-to-end.

### Useful dev env vars

- `DEV_PERSIST_TOKEN=1` — caches the OAuth token to `.token-dev.json` so `tsx watch` reloads don't wipe it. Local only.
- `DEBUG_PAYLOAD=1` — logs the first 6 KB of each Linear webhook payload. Handy for figuring out what Linear is actually sending.

## License

MIT. See [LICENSE](LICENSE).
