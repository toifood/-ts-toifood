ISSUE LOG
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ISSUE ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES.

REQUIRED FORMAT FOR EACH ISSUE ENTRY:

## ISSUE:{NAME OF ENVIRONMENT} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD ALL NEW ISSUE ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES-->
## ISSUE:toifood 2026-06-08 → toifood-dev org created — prod source repos to migrate from jayreck996

`toifood-dev` GitHub org created 2026-06-08. Houses production source code repos (`ts-toifood-back`, `ts-toifood-front`, `ts-toifood-web`), separate from `toifood` (pipeline/analysis).

**Pending after transfer:**
- Update Mac Mini git remote: `~/ts-toifood-back` points to `jayreck996/ts-toifood-back` — must update after transfer (PM2 service runs from this path)
- Update `would-update.md` skill: source URL `jayreck996/ts-toifood-{suffix}` → `toifood-dev/ts-toifood-{suffix}`
- Verify `TOIFOOD_CROSS_REPO_TOKEN` has `repo` access to `toifood-dev` repos

## ISSUE:toifood 2026-06-08 → summary index deferred — useful once ts-front and ts-web are live

A `-toifood/README.md` summary index (linking to each pipeline repo's `could/` analysis, last run timestamp, category list) was considered. Decision: defer until ts-front and ts-web are live and the org is shared with other team members. No value adding it for a single pipeline repo. When added: `would-update` skill writes the last run timestamp + GitHub links to `could/` per category as a final step.
## ISSUE:toifood 2026-06-07 16:00 → skill was reading wrong branch — zipball/latest unresolved

`zipball/latest` in the skill was an ambiguous ref. GitHub treats it as a literal branch/tag name "latest" — since no such ref exists on `ts-toifood-back`, behavior was undefined (likely falling back to default branch `main`). `main` has no `-MUST/` directory, so the 5 standard category prompts were not being read.

**Fix:** skill now uses `compare/main...{branch}` per branch to find creation date, picks newest created. First successful detection: `1-1-1` (created 2026-06-07), which has `-MUST/`.

## ISSUE:toifood 2026-06-07 16:00 → two clones of ts-toifood-back found locally

Two local clones of `jayreck996/ts-toifood-back` exist:
- `~/ts-toifood-back` — branch `1-1-1`, active service (PM2 runs from here), has `-MUST/`
- `~/Documents/GitHub/ts-toifood-back` — branch `main`, stale reference clone, no `-MUST/`

Not a conflict — different branches, different purposes. `~/ts-toifood-back` is the live copy.
## ISSUE:toifood 2026-06-07 13:58 → runner group blocked public repos — fixed before first successful run

**Symptom:** Self-hosted runner online and listening but not picking up queued jobs from `ts-back`.

**Root cause:** Default runner group had `allows_public_repositories: false`. `ts-back` is a public repo — jobs were silently dropped.

**Fix:** `gh api --method PATCH orgs/toifood/actions/runner-groups/1 --field allows_public_repositories=true`

**Hard rule:** When adding a self-hosted runner to a free GitHub org, always check runner group public repo access if any repos are public.

## ISSUE:toifood 2026-06-07 13:09 → self-hosted runner runs under jayreck account, not jayagent

Decision: use `jayreck` account on Mac Mini for the GitHub Actions self-hosted runner, not `jayagent`.

**Why `jayreck`:** Claude Code is already installed and Claude Pro OAuth is already authenticated under `jayreck` (`~/.claude/` exists and is valid). PM2 is already running under this account managing both Cloudflare tunnels (`cloudflare-tunnel`, `toigroup-tunnel`). Adding the runner to PM2 follows the same pattern — no new account setup, no new auth.

**Why not `jayagent`:** `jayagent` would require a separate Claude Code install + fresh OAuth browser login. No benefit over `jayreck` which is already fully configured.

**Pending:** Runner registration only — `toifood` org → Settings → Actions → Runners → New → macOS ARM64 → `./config.sh` → `pm2 start run.sh --name toifood-runner` → `pm2 save`.

## ISSUE:toifood 2026-06-07 → GitHub Actions hosted runner cannot use Claude Pro — OAuth auth is interactive-only

Claude Code CLI authenticates via Claude Pro OAuth (browser login → token saved to `~/.claude/`). GitHub Actions hosted runners (`ubuntu-latest`) are ephemeral VMs — blank slate on every run, no `~/.claude/`, no browser available to complete the OAuth flow. `claude` fails immediately with "not authenticated".

**Why `ANTHROPIC_API_KEY` solves it but breaks the business goal:** API key skips OAuth entirely and works headlessly, but it is separate billing from Claude Pro — defeats the purpose of aligning with the existing subscription.

**Why self-hosted runner (Mac Mini) solves it:** `jayagent` on the Mac Mini already completed the OAuth flow once manually. `~/.claude/` persists between runs. GitHub Actions dispatches to the Mac Mini runner, `claude` reads the existing token, and the job executes under Claude Pro auth. Token refreshes automatically while the subscription is active.
## ISSUE:toifood 2026-06-07 → pipeline LLM decision — Claude skill via Mac Mini self-hosted runner

**Decision:** Use Claude Code CLI (Claude Pro) running on the Mac Mini server as a GitHub Actions self-hosted runner. GitHub Actions triggers on schedule → dispatches to Mac Mini → `claude --print "/would-update ts-back"` → writes to category docs → commits and pushes.

**Why not GitHub-hosted runner:** GitHub Actions hosted runners (`ubuntu-latest`) cannot use Claude Pro — they require `ANTHROPIC_API_KEY` (separate API billing). Claude Pro OAuth auth is machine-local.

**Why Mac Mini is valid:** Mac Mini is always-on infrastructure (PM2-managed, auto-restarts after Friday 3am reboot, `jayagent` auto-login). Not a "local machine" in the transient sense — it is a server.

**Why not Ollama (toiflow pattern):** Business goal is to align with Claude Pro subscription. Ollama remains available as fallback.

**Pending setup:**
1. GitHub Actions self-hosted runner installed on Mac Mini (`jayagent`) as LaunchAgent
2. Claude Code installed + Claude Pro OAuth auth on Mac Mini
3. `would-update.md` skill updated from Windows/PowerShell → macOS/bash
4. `would-update.yml` workflow updated to `runs-on: [self-hosted, mac-mini]`
## ISSUE:toifood 2026-06-07 → Claude skill tool execution requires local machine — GitHub Actions hosted runners incompatible

Claude skills split into two parts: LLM inference runs on Anthropic servers (covered by Claude Pro), tool execution (Bash, Read, Write, file I/O) runs on the local machine. GitHub Actions hosted runners (`ubuntu-latest`) have no connection to the local machine, so Claude's tool calls have nowhere to land.

**Options excluding local machine:**
- `anthropics/claude-code-action` — runs Claude Code in GitHub Actions but requires `ANTHROPIC_API_KEY` (API billing, not Claude Pro)
- Direct `api.anthropic.com` call — same billing
- Self-hosted runner — uses Claude Pro but requires local machine to be on

**Conclusion:** Excluding local machine = Anthropic API billing (~$1–3/month at Haiku rates). Decision pending.
## ISSUE:toifood 2026-06-07 → Claude skills are CLI-only — cannot be called from GitHub Actions

Confirmed: Claude skills run in the interactive Claude Code CLI session covered by Claude Pro. GitHub Actions cannot invoke them. Resolved by running the `/would-update` skill locally via wmux cron instead — no GitHub Actions LLM dependency, no `ANTHROPIC_API_KEY` needed.
## ISSUE:toifood 2026-06-07 → Claude skills (Claude Code) are CLI-only — not callable from GitHub Actions

Confirmed: skills run in the interactive CLI session, covered by Claude Pro subscription. Cannot be invoked from GitHub Actions workflows. Rules out using Claude Code/skills as the pipeline LLM backend — `api.anthropic.com` is used instead (`ANTHROPIC_API_KEY` org secret).
