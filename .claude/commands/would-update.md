Analyse the source codebase and update all 10 category docs in the target repo.

## Arguments
`$ARGUMENTS` is the target repo name, e.g. `ts-back`, `ts-front`, `ts-web`.

## Derived values
- Source repo: `jayreck996/ts-toifood-{suffix}` where suffix = strip `ts-` from `$ARGUMENTS` (e.g. `ts-back` → `back` → `ts-toifood-back`)
- Target path: `$GITHUB_WORKSPACE` (set by GitHub Actions on the self-hosted runner)
- Categories: `migrate`, `price`, `recovery`, `usage`, `instruction`

## Steps

### 1. Download and extract source repo

Run in bash:
```bash
suffix="${ARGUMENTS#ts-}"
zipPath="/tmp/toifood-source.zip"
extractPath="/tmp/toifood-source"
rm -rf "$extractPath"
gh api "repos/jayreck996/ts-toifood-${suffix}/zipball/latest" > "$zipPath"
unzip -q "$zipPath" -d "$extractPath"
root=$(find "$extractPath" -mindepth 1 -maxdepth 1 -type d | head -1)
echo "$root"
```

Note the `$root` path printed — all source file reads use this as the base.

### 2. Read codebase context

Read these files from `$root`:
- `README.md`
- `package.json`
- `prisma/schema.prisma` (skip if not present)
- List contents of `src/` directory tree (glob `src/**/*` from `$root`)

Hold this codebase context in mind for all 10 analyses.

### 3. For each of the 5 categories × issue/asset (10 total)

For each category in `migrate`, `price`, `recovery`, `usage`, `instruction`:

#### 3a. ISSUE analysis
1. Read `$root/-MUST/{category}-ISSUE.md` — this is your analysis instruction/prompt
2. Using that instruction and the codebase context from step 2, generate a concise analysis
3. Format the entry:
   ```
   ## ISSUE:{category} {YYYY-MM-DD HH:MM} → {one-line summary}
   
   {analysis content}
   ```
4. Prepend the entry into `$GITHUB_WORKSPACE/would/{category}-issue-v1.md` directly below the `####### <!-- ANCHOR MARKER` line — never edit existing entries below it

#### 3b. ASSET analysis
1. Read `$root/-MUST/{category}-ASSET.md` — this is your analysis instruction/prompt
2. Using that instruction and the codebase context from step 2, generate a concise analysis
3. Format the entry:
   ```
   ## ASSET:{category} {YYYY-MM-DD HH:MM} → {one-line summary}
   
   {analysis content}
   ```
4. Prepend the entry into `$GITHUB_WORKSPACE/would/{category}-asset-v1.md` directly below the `####### <!-- ANCHOR MARKER` line — never edit existing entries below it

### 4. Commit and push

Run in bash from `$GITHUB_WORKSPACE`:
```bash
cd "$GITHUB_WORKSPACE"
git config user.name "would-update"
git config user.email "admin@toigroup.co.nz"
git add migrate-ISSUE.md migrate-ASSET.md price-ISSUE.md price-ASSET.md recovery-ISSUE.md recovery-ASSET.md usage-ISSUE.md usage-ASSET.md instruction-ISSUE.md instruction-ASSET.md
git commit -m "would-update: $(date '+%Y-%m-%d %H:%M') codebase analysis"
git push
```

### 5. Clean up
```bash
rm -rf /tmp/toifood-source.zip /tmp/toifood-source
```
