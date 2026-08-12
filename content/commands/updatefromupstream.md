---
description: Sync this fork with upstream comol/ai_rules_1c — fetch, analyze, merge (asking on real conflicts), push, then offer /updaterules
---

# /updatefromupstream — sync the fork with upstream

This command is for the **fork owner** — it operates on your local clone of `denisvmiheev/ai_rules_1c` (the rules repository itself), not on a 1C project. Run it from a project session; it locates or clones your fork's working copy separately from the current project's own files.

Companion: `/updaterules` propagates the fork's *current* state into one project. This command updates the fork itself from upstream first — run this, then `/updaterules` in each project that should pick up the result.

## Steps

### 1. Resolve the local clone path

- Check `memory.md` / `1c-templates-fastcode` (`recall` with `fork clone path`, `ai_rules_1c`) for a previously saved path.
- If found, verify it still exists and is a git repo with `origin` pointing at `github.com/denisvmiheev/ai_rules_1c` (`git -C <path> remote -v`). If it doesn't match, treat as not found.
- If not found, ask the user once: "Где лежит локальный клон твоего форка `denisvmiheev/ai_rules_1c`? (или скажи 'клонируй', и я склонирую в `C:\Work\git\ai_rules_1c`)". Save the resolved path via `remember` (or a `memory.md` entry if the server is unavailable) so future runs skip this question.

### 2. Ensure the `upstream` remote exists

```powershell
git -C <clone> remote get-url upstream 2>$null
if (-not $?) { git -C <clone> remote add upstream https://github.com/comol/ai_rules_1c.git }
```

### 3. Fetch and check working-tree cleanliness

```powershell
git -C <clone> fetch upstream
git -C <clone> status --porcelain
```

If `status --porcelain` is non-empty (uncommitted local changes) — **stop** and report it to the user; do not stash or discard anything automatically. The user must commit or stash first.

### 4. Analyze what changed

```powershell
git -C <clone> log HEAD..upstream/main --oneline
git -C <clone> diff --stat HEAD upstream/main
```

- **No commits ahead** → report "форк уже актуален" and stop (noop, nothing to merge, nothing to push).
- **Commits found** → summarize for the user in 3-6 lines: how many commits, which top-level areas changed (`content/rules/`, `content/agents/`, `content/commands/`, `adapters/`, `install.ps1`, etc.), and flag whether any changed path overlaps the fork's known hand-edited set (`content/mcp-servers.json`, `content/skills/mcp-1c-tools/**`, `content/rules/{mcp-first-search,verification-gates,tooling-playbooks,systematic-debugging,dev-standards-architecture,subagents,forms-add,getconfigfiles}.md`, `content/agents/*.md`, `content/commands/{checkmcp,updaterules}.md`, `AGENTS.md`, `AGENT-INSTALL.md`, `README.md`) — those are exactly where a merge conflict is likely.

### 5. Merge

```powershell
git -C <clone> merge upstream/main --no-commit --no-ff
```

Then `git -C <clone> status --porcelain` again to classify:

- **`UU` / `AA` / `DD` entries (real conflicts)** — go to Step 6.
- **No conflict markers, merge staged cleanly** — go to Step 7.

### 6. Resolve conflicts — ask, don't guess

For each conflicted file, open it and look at the conflict hunks.

- **Mechanical conflicts** (the conflict is purely upstream's `comol/ai_rules_1c` URL/branding vs. this fork's `denisvmiheev/ai_rules_1c`, or the classic-catalog tool name vs. the `edt-mcp`/`v8std-local` name this fork already substituted in that exact line) — resolve by keeping this fork's side. This is safe because it repeats a substitution already made and reviewed earlier in this fork's history.
- **Anything else** — a new rule, a rewritten section, a restructured file where upstream's intent and this fork's intent both touch real content — this is a **material fork** per `AGENTS.md → CONFUSION`. Do **not** silently pick a side. Present it to the user:

  ```
  CONFUSION: merge conflict in <file>
  Options:
    A) Keep this fork's version — <one-line summary of what this fork's side does>
    B) Take upstream's version — <one-line summary of what upstream's side does>
    C) Merge both manually — <what combining would look like, if feasible>
  → Which one to pick?
  ```

  Batch conflicts from the same file / same logical change into one question when reasonable; don't ask once per hunk if the file has ten trivial ones — batch those into the mechanical bucket instead.
- After resolving a file: `git -C <clone> add <file>`.
- If the user wants to abandon the merge at any point: `git -C <clone> merge --abort` and stop — report that upstream was fetched but not merged, and the fork is unchanged.

### 7. Finalize the merge locally

```powershell
git -C <clone> diff --cached --stat
```

Show this to the user as the final summary of what the merge actually changed, then:

```powershell
git -C <clone> commit --no-edit
```

**Safety check — no leftover conflict markers.** Before committing (or immediately after, before push), scan for anything a manual resolution might have missed:

```powershell
git -C <clone> grep -n "^<<<<<<<\|^=======\|^>>>>>>>" -- . ':!*.md.orig'
```

Any hit means a conflict marker survived into a committed file — fix it (re-open the file, resolve properly, `git add`, `git commit --amend --no-edit`) before proceeding. Do not push with markers present.

### 8. Push — confirm first

Push is visible to anyone with access to the fork and not trivially reversible (force-push would be, but this is a plain fast-forward-able merge commit, so a revert is cheap — still, confirm before acting on a shared remote per the project's standing rule on visible/hard-to-reverse actions).

Ask: "Смёрджено локально, N коммитов из апстрима. Запушить в `denisvmiheev/ai_rules_1c` (`origin/main`)?" On yes:

```powershell
git -C <clone> push origin main
```

### 9. Offer to propagate into the current project

The merge only updated the fork on GitHub — it does **not** touch the project you're sitting in right now. Ask: "Форк обновлён. Запустить `/updaterules` в этом проекте сейчас, чтобы подтянуть изменения?" If yes, proceed exactly as `/updaterules` describes. If no, remind the user that other projects on this fork need their own `/updaterules` run later, whenever they next open a session there.

## Final report

- Commits merged (count + one-line list of the notable ones).
- Files touched by the merge (from the `--stat` in Step 7).
- Conflicts encountered and how each was resolved (mechanical vs. user-decided), or "no conflicts".
- Push result (pushed / declined / skipped because nothing to merge).
- Whether `/updaterules` was run in the current project as a follow-up.

## Limits

- Never force-push (`git push --force`) — if `origin/main` has diverged (someone else pushed to the fork, or a previous run left an unpushed local commit that changed), stop and report the divergence instead of overwriting it.
- Never resolve a non-mechanical conflict without asking — see Step 6.
- Never discard uncommitted local changes in the clone (Step 3) — stop and let the user handle them.
- This command does not run for the *upstream* repository itself — it only ever merges upstream → fork, one direction.
