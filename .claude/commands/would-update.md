Analyse the codebase and write 14 issue/asset entries for the target repo.

## Arguments
`$ARGUMENTS` is the target repo name, e.g. `ts-back`.

## Steps

### 1. Read codebase context

Read `/tmp/would-read-content.txt` — prepared by `would-read-md.js`. Contains:
- `## CODEBASE` section: README, package.json, schema.prisma, src/ file tree
- `## INSTRUCTIONS` section: per-category -MUST/ prompts for migrate, price, recovery, usage, instruction

Hold this entire context in mind for all 14 analyses.

### 2. Prepare output

Run in bash:
```bash
mkdir -p /tmp/would-results
TS=$(TZ=Pacific/Auckland date '+%Y-%m-%d %H:%M')
echo "Timestamp: $TS"
```

### 3. Generate 14 analyses

For each category in `migrate`, `price`, `recovery`, `usage`, `instruction`, `bug`, `analysis` — generate both ISSUE and ASSET entries.

**For `migrate`, `price`, `recovery`, `usage`, `instruction`:** use the matching `### {category} {type}` instruction from the `## INSTRUCTIONS` section of `/tmp/would-source.txt`.

**Embedded prompts for `bug` and `analysis`:**
- **bug ISSUE**: Analyze for undiscovered bugs — edge cases, race conditions, null dereferences, async pitfalls, production risks.
- **bug ASSET**: Identify existing bug-prevention assets — error handling, validation, test coverage, logging, gaps.
- **analysis ISSUE**: Overall code quality — technical debt, architectural concerns, missing patterns, scalability risks.
- **analysis ASSET**: Codebase health summary — what is well-built, the tech stack, production-ready vs in progress.

For each category and type:
1. Generate a concise analysis grounded in the codebase context
2. Format as:
   ```
   ## ISSUE:{category} {TS} → {one-line summary}
   
   {analysis content}
   ```
   (use `ASSET:` prefix for asset type)
3. Write to `/tmp/would-results/{category}-issue.txt` or `/tmp/would-results/{category}-asset.txt`
