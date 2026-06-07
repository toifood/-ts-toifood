ASSET LOG
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ASSET ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES.

REQUIRED FORMAT FOR EACH ASSET ENTRY:

## ASSET:{NAME OF ENVIRONMENT} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD ALL NEW ASSET ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES-->
## ASSET:toifood 2026-06-07 14:16 → ts-back file/folder structure aligned to toiflow/ts-anz pattern

Four JS files + `would/` output folder now mirrors ts-anz exactly.

| ts-anz | ts-back equivalent |
|---|---|
| `would-read-md.js` (RSS fetch) | `would-read-md.js` (-MUST/ + codebase read) |
| `would-update-md.js` (Ollama call) | `would-update-md.js` (claude skill runner) |
| `would-update-content.js` (write docs) | `would-update-content.js` (write would/ via API) |
| `would-update-csv.js` (CSV log) | `would-update-csv.js` (codebase headline log) |
| `would/-content-issue-v1.md` | `would/migrate-issue-v1.md` × 5 categories |
| `would/-content-asset-v1.md` | `would/migrate-asset-v1.md` × 5 categories |
| `would/-log-asset-v1.csv` | `would/-log-codebase-v1.csv` |

**Skill path updated:** `$GITHUB_WORKSPACE/would/{category}-issue-v1.md` / `would/{category}-asset-v1.md`

## ASSET:toifood 2026-06-07 13:58 → pipeline fully operational — end-to-end summary

**Status:** Live. First run succeeded 2026-06-07 13:55. Next run: Monday 2026-06-09 18:00 NZST.

**Full infrastructure state:**

| Component | Detail |
|---|---|
| Org | `toifood` (GitHub free plan, all repos public) |
| Runner | `mac-mini` — `jayreck` account, `~/actions-runner/` v2.334.0 |
| PM2 process | `toifood-runner` (id 7) — alongside `cloudflare-tunnel`, `toigroup-tunnel`, `toifood-back`, `postgres`, `redis`, `slack-bot` |
| Runner group | Default — `visibility: all`, `allows_public_repositories: true` |
| Claude auth | Claude Pro OAuth, `~/.claude/` persisted under `jayreck` |
| Org secret | `TOIFOOD_CROSS_REPO_TOKEN` — `repo` + `workflow` scopes, used for checkout + git push |
| Schedule | `0 6 * * 1` — weekly Monday 06:00 UTC = 18:00 NZST |

**Repos:**
| Repo | Role |
|---|---|
| `toifood/-toifood` | Reusable workflows + skill (`would-update.md`) + org docs |
| `toifood/ts-back` | Target — category docs updated weekly by skill |

**Skill flow (`would-update ts-back`):**
1. `gh api zipball jayreck996/ts-toifood-back@latest` → `/tmp/`
2. Read `README.md`, `package.json`, `prisma/schema.prisma`, `src/` tree
3. Read 10 `-MUST/` instruction prompts from source repo
4. Generate 10 analyses under Claude Pro
5. Prepend entries to 10 category docs in `$GITHUB_WORKSPACE`
6. `git commit + push` from workspace
7. `rm -rf /tmp/toifood-source*`

**Troubleshooting log (first run):**
- Runner auto-updated v2.325→v2.334 on first pickup → session conflict → removed + re-registered
- Runner group `allows_public_repositories` was false → jobs silently queued → patched via API
- Both resolved before first successful run

## ASSET:toifood 2026-06-07 13:58 → schedule set to weekly Monday 6pm NZST

`would-update.yml` cron updated from daily `0 18 * * *` → weekly `0 6 * * 1` (Monday 06:00 UTC = Monday 18:00 NZST).

**Why weekly:** codebase analysis doesn't need daily frequency; weekly cadence reduces runner load and keeps doc entries meaningful.

## ASSET:toifood 2026-06-07 13:09 → pipeline architecture finalised — Mac Mini self-hosted runner under jayreck

**Flow:**
```
GitHub Actions schedule (daily 06:00 NZST = 18:00 UTC)
  → would-update.yml (ts-back) — runs-on: [self-hosted, mac-mini]
    → checkout ts-back (TOIFOOD_CROSS_REPO_TOKEN)
    → checkout toifood/-toifood → copy would-update.md → ~/.claude/commands/
    → claude --dangerously-skip-permissions --print "/would-update ts-back"
        → gh api zipball ts-toifood-back@latest → /tmp/
        → read -MUST/ prompts + codebase context (README, package.json, prisma, src/)
        → generate 10 analyses (migrate/price/recovery/usage/instruction × ISSUE/ASSET)
        → prepend entries to category docs in $GITHUB_WORKSPACE
        → git commit + push
        → rm -rf /tmp/toifood-source*
```

**What already exists under jayreck:**
| Component | Status |
|---|---|
| Claude Code installed | ✅ (`npm install -g @anthropic-ai/claude-code`) |
| Claude Pro OAuth (`~/.claude/`) | ✅ authenticated |
| PM2 running | ✅ manages `cloudflare-tunnel` + `toigroup-tunnel` |
| PM2 startup on boot | ✅ (tunnels survive Friday 3am reboot) |
| Node.js, `gh` CLI, `git` | ✅ |
| `TOIFOOD_CROSS_REPO_TOKEN` org secret | ✅ set |

**What still needs to be done:**
| Step | Action |
|---|---|
| 1 | `toifood` org → Settings → Actions → Runners → New runner → macOS ARM64 → run `./config.sh --name mac-mini --labels mac-mini` |
| 2 | `pm2 start ~/actions-runner/run.sh --name toifood-runner && pm2 save` |

**No new secrets needed.** Skill auto-copies from `-toifood` on every run — stays in sync with repo.

## ASSET:toifood 2026-06-07 → Claude Pro auth model confirmed — why self-hosted runner is the only headless path

| Scenario | Auth path | Works headlessly |
|---|---|---|
| `claude` on local machine (first run) | Browser OAuth → token saved to `~/.claude/` | N/A — interactive |
| `claude` on local machine (subsequent) | Reads `~/.claude/` token | ✅ Yes |
| `claude` on GitHub hosted runner | No `~/.claude/`, no browser — OAuth impossible | ❌ No |
| `claude` with `ANTHROPIC_API_KEY` | Skips OAuth, uses key directly | ✅ Yes — but separate API billing |
| `claude` on Mac Mini self-hosted runner | Reads persisted `~/.claude/` from prior manual login | ✅ Yes — Claude Pro |

**Key principle:** Claude Pro OAuth requires a human + browser exactly once per machine. After that, the token in `~/.claude/` is reused automatically. Self-hosted runner on Mac Mini is the only GitHub Actions path that has this persistent auth state.
## ASSET:toifood 2026-06-07 → pipeline architecture — Claude skill on Mac Mini self-hosted runner

**Flow:**
```
GitHub Actions schedule (daily 06:00)
  → would-update.yml (ts-back) — runs-on: [self-hosted, mac-mini]
    → checkout ts-back + -toifood
    → cp .toifood/.claude/commands/would-update.md ~/.claude/commands/
    → claude --dangerously-skip-permissions --print "/would-update ts-back"
        → gh api zipball ts-toifood-back@latest → /tmp/
        → read -MUST/ prompts + codebase context
        → generate 10 analyses (migrate/price/recovery/usage/instruction × ISSUE/ASSET)
        → prepend entries to category docs in ts-back
        → git commit + push
        → rm -rf /tmp/toifood-source*
```

**Mac Mini — what already exists:**
| Component | Status |
|---|---|
| `jayagent` account + PM2 | ✅ Running |
| cloudflared Cloudflare tunnel | ✅ Running |
| Friday 3am reboot + auto-recovery | ✅ Configured |
| Node.js (for npm/Claude Code) | ✅ Available |

**What needs to be built:**
| Step | Action |
|---|---|
| 1 | Register self-hosted runner: `toifood` org → Settings → Actions → Runners → New → macOS ARM64; install via `./config.sh` + `./svc.sh install` on Mac Mini (`jayagent`) |
| 2 | Install Claude Code: `npm install -g @anthropic-ai/claude-code`; auth: `claude` → OAuth → Claude Pro login |
| 3 | Update `would-update.md` skill: replace PowerShell with bash (`curl`, `unzip`, `/tmp/`, `$GITHUB_WORKSPACE`) |
| 4 | Update `would-update.yml`: replace `runs-on: ubuntu-latest` → `runs-on: [self-hosted, mac-mini]`; add skill copy step |

**Secrets required:**
| Secret | Scope | Status |
|---|---|---|
| `TOIFOOD_CROSS_REPO_TOKEN` | toifood org | ✅ Set |

No new secrets needed — Claude Pro auth and `gh` auth are stored on-machine under `jayagent`.

**Risk table:**
| Risk | Mitigation |
|---|---|
| Claude Pro auth expires | Job fails loudly → re-run `claude` interactively on Mac Mini |
| Mac Mini offline | Jobs queue → run when runner comes back online |
| Friday reboot vs job timing | Reboot 03:00, job 06:00 — no overlap |

**Comparison with toiflow:**
| | toiflow | toifood |
|---|---|---|
| LLM | Ollama `qwen2.5:7b` | Claude Pro (via CLI) |
| Runner | GitHub hosted | Self-hosted (Mac Mini) |
| LLM auth | `OLLAMA_SECRET` WAF header | Claude Pro OAuth on Mac Mini |
| Cost | Free | Free (Claude Pro already paid) |
## ASSET:toifood 2026-06-07 → Claude skill execution model confirmed

| Layer | Where it runs | Billing |
|---|---|---|
| LLM inference | Anthropic servers | Claude Pro (interactive) or API key (automated) |
| Tool execution (Bash, Read, Write) | Local machine | Free |
| GitHub Actions hosted runner | No local machine | Must use API key |
| GitHub Actions self-hosted runner | Local machine | Claude Pro ✅ |

Claude Pro covers LLM inference only in interactive CLI sessions. Any automated/headless trigger (GitHub Actions hosted) requires `ANTHROPIC_API_KEY`.
## ASSET:toifood 2026-06-07 → /would-update skill created — org-level reusable codebase analyser

File: `.claude/commands/would-update.md`

Invoked as `/would-update {ts-back|ts-front|ts-web}`. Derives source repo (`ts-toifood-{suffix}`) automatically.

Flow: `gh api zipball/latest` → `Expand-Archive` → read -MUST/ prompts + codebase context → generate 10 analyses → prepend to category docs → `git commit + push` → cleanup. Runs under Claude Pro, no API key required.
## ASSET:toifood 2026-06-07 → LLM backend confirmed — Anthropic API (`api.anthropic.com`)

Claude Code skills are CLI-only (Claude Pro subscription, interactive session). GitHub Actions cannot invoke them. Decision finalised: toifood pipeline uses `api.anthropic.com/v1/messages` with `ANTHROPIC_API_KEY` org secret. No Ollama dependency.
## ASSET:toifood 2026-06-07 → architecture — mirrors toiflow, scaled to 5 categories

Follows the same pattern as `toiflow` org. Key deltas:

| | toiflow | toifood |
|---|---|---|
| Reusable workflow LLM | Ollama via Cloudflare tunnel | Claude API (`api.anthropic.com`) |
| Auth secret | `OLLAMA_SECRET` (WAF header) | `ANTHROPIC_API_KEY` |
| Content jobs per repo | `issue` + `asset` (2) | 5 categories × issue/asset (10) |
| Job parallelism | Serialised (Ollama single-threaded) | Parallel (Claude API handles concurrency) |
| Source data | RSS / Gmail / Calendar | Codebase from source repo via `TOIFOOD_CROSS_REPO_TOKEN` |
| Output | `would/` files + CSV | `would/` files per category |

**Job flow (ts-back):** `fetch` → 10 parallel content jobs → `update`

**Same across both orgs:** org secrets via `secrets: inherit`, `would-read-md.js` + `would-update-content.js` pattern, `would/` output directory, ISSUE/ASSET doc format.

## ASSET:toifood 2026-06-07 → must-update-content.yml created — Claude API reusable workflow

- Calls `api.anthropic.com/v1/messages` with `x-api-key: ANTHROPIC_API_KEY`
- Inputs: `prompt` (required), `model` (default: `claude-haiku-4-5-20251001`)
- Output: `response` — Claude's text response
- Guard: exits 1 if response is empty or null
- Same interface as `toiflow/-toiflow/must-update-content.yml` (Ollama equivalent)
- `ANTHROPIC_API_KEY` required as `toifood` org secret
