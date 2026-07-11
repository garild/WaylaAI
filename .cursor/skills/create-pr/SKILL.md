---
name: create-pr
description: >-
  Commit scoped changes under repo-local git identity (set once from GitHub
  user), push to remote, and open a pull request. Use when the user asks to
  create a PR, push changes, or ship a branch for review.
---

# Create PR

End-to-end workflow to commit, push, and open a pull request.

**Do not run local build or unit tests** — CI validates on push/PR. Fix failures from CI if needed.

---

## Fast path (apply before every phase)

Evaluate these **once** at the start. Skip phases/steps when the condition is true.

| Check | If true | Skip |
|-------|---------|------|
| `git config user.name` and `git config user.email` both non-empty | identity already in repo | Phase 0 `gh api user` — read config only |
| `git status --porcelain` empty | nothing to commit | Phase 2 entirely |
| `git rev-list --count HEAD..origin/develop` = 0 (after fetch) | branch up to date with base | Phase 3 merge |
| No new commits this session and no merge | nothing changed author-wise | Phase 0 verify (until just before push) |
| `gh pr list --head "$(git branch --show-current)" --json number -q '.[0].number'` returns a number | PR already open | `gh pr create` → use `gh pr edit` or report existing URL |
| User says "one commit" or diff is one logical scope | small change | split-by-scope — single commit instead |

**Single bootstrap batch** — run Phase 1 + sync check in **one parallel call** (one `git fetch` for the whole workflow):

```bash
git fetch origin
git status
git diff
git diff --staged
git log --oneline origin/develop..HEAD
git rev-list --count HEAD..origin/develop   # BEHIND — reuse in Phase 3, do not fetch again
```

Default base branch: `develop`

---

## Phase 0 — Git identity (resolve once, set once, reuse)

**Every commit must be authored and committed by the repo owner — never the agent, bot, or Cursor.**

Run Phase 0 **once** at the start of the workflow. After repo-local `git config` is set, use plain `git commit` for all commits — no per-commit `--author` flags or env exports.

**Fast path:** if repo-local `git config user.name` and `git config user.email` are both set, use them as `$USER_NAME` / `$USER_EMAIL` and skip `gh api` entirely. Only call GitHub when config is missing.

### 1. Resolve identity (single lookup)

Priority order — use the first source that yields both name and email (skip sources already satisfied by fast path):

| Priority | Source | How |
|----------|--------|-----|
| 1 | Authenticated GitHub user | `gh api user` → `.name` (fallback `.login`), `.email` |
| 2 | GitHub primary email | `gh api user/emails` → entry with `"primary": true` (if user scope missing, skip) |
| 3 | Repo-local git config | `git config user.name`, `git config user.email` |
| 4 | Global git config | `git config --global user.name`, `git config --global user.email` |

**Bash (run once):**

```bash
USER_JSON=$(gh api user)
USER_NAME=$(echo "$USER_JSON" | jq -r '.name // .login')
USER_EMAIL=$(echo "$USER_JSON" | jq -r '.email // empty')
[ -z "$USER_EMAIL" ] && USER_EMAIL=$(gh api user/emails -q '.[] | select(.primary==true) | .email' 2>/dev/null | head -1)
[ -z "$USER_NAME" ] && USER_NAME=$(git config user.name || git config --global user.name)
[ -z "$USER_EMAIL" ] && USER_EMAIL=$(git config user.email || git config --global user.email)
```

**PowerShell (run once):**

```powershell
$user = gh api user | ConvertFrom-Json
$USER_NAME = if ($user.name) { $user.name } else { $user.login }
$USER_EMAIL = $user.email
if (-not $USER_EMAIL) { $USER_EMAIL = git config user.email }
if (-not $USER_NAME) { $USER_NAME = git config user.name }
```

If either `$USER_NAME` or `$USER_EMAIL` is still empty, **stop and ask the user** — do not commit.

### 2. Set repo-local git config (once)

Write identity into **this repo only** (not `--global`):

```bash
git config user.name "$USER_NAME"
git config user.email "$USER_EMAIL"
```

Confirm and **keep these values for the entire workflow** — do not re-read or re-export identity on later phases.

```bash
git config user.name   # must match $USER_NAME
git config user.email  # must match $USER_EMAIL
```

### 3. Commit normally

All commits and merges use git config automatically:

```bash
git commit -m "..."
git merge origin/develop --no-edit   # merge commit picks up config too
```

Do **not** pass `--author`, `GIT_AUTHOR_*`, or `GIT_COMMITTER_*` on individual commits after Phase 0.

### Forbidden

- No `Co-authored-by:` trailers
- No bot/agent/cursor identities in author or committer fields
- No re-resolving identity mid-workflow unless config was wrong

### Verify before push

Run **only** when new commits were created this session, a merge just completed, or this is the last step before `git push`:

```bash
git log origin/develop..HEAD --format='%h %an <%ae> | committer: %cn <%ce>'
```

**All** author and committer lines must match the Phase 0 `$USER_NAME` and `$USER_EMAIL`. If any commit fails, fix unpushed history:

```bash
git rebase origin/develop --exec 'git commit --amend --reset-author --no-edit'
```

Re-verify. Do not push until clean.

---

## Phase 1 — Inspect changes

Use the **single bootstrap batch** from Fast path (includes `BEHIND` count). Do not run a second `git fetch` later unless push fails due to stale refs.

---

## Phase 2 — Commit by scope

**Fast path:** if `git status --porcelain` is empty, skip this phase.

### Rules

- Stage **only** files that belong together in one scope
- Never `git add .` when splitting commits
- Never commit secrets (`.env`, keys, credentials)
- Match recent `git log` message style; 1–2 sentences focused on **why**
- Identity comes from Phase 0 repo-local `git config` — plain `git commit` only
- **Single-commit mode:** one logical scope or user asked for one commit → one `git add` + one `git commit`, no splitting

### Split by scope

| Scope | Example subject |
|-------|-----------------|
| Feature | `feat(api): add JWT validation middleware` |
| Bug fix | `fix(worker): retry on transient errors` |
| CI / infra | `ci(github): add build-test-deploy workflow` |
| Chore | `chore(dotnet): pin SDK version in global.json` |
| Docs | `docs(readme): update deployment steps` |

Order: foundations first (config, types), then implementation, then tests/docs.

```bash
git add <scoped-files>
git commit -m "$(cat <<'EOF'
ci(github): add build-test-deploy workflow

Add GitHub Actions workflow for build, test, and deploy on push to develop.
EOF
)"
```

Repeat until the working tree is clean. Run **Phase 0 verify** once after all commits (if any were created).

---

## Phase 3 — Sync with base branch

Use `BEHIND` from the bootstrap batch — **do not fetch again**.

**Fast path:** if `BEHIND` = 0, skip this phase.

If `BEHIND` > 0:

```bash
git merge origin/develop --no-edit
```

If merge needs a manual commit after conflict resolution, use plain `git commit -m "merge: sync with origin/develop"` — identity from Phase 0 git config.

Resolve conflicts, run **Phase 0 verify**, then continue.

---

## Phase 4 — Push

```bash
git push -u origin HEAD
```

No force-push unless the user explicitly requests it.

---

## Phase 5 — Open pull request

Ensure `git` is on `PATH` when calling `gh` on Windows (e.g. prepend `C:\Program Files\Git\bin`).

**Fast path:** check for an existing PR first:

```bash
gh pr list --head "$(git branch --show-current)" --json number,url -q '.[0]'
```

If a PR exists → `gh pr edit <number>` (title/body/base as needed) and report its URL. Otherwise:

```bash
gh pr create --base develop --title "<Short description>" --body "$(cat <<'EOF'
## Summary

- <what changed and why>

## Test plan

- [ ] CI build and tests pass on PR
- [ ] <manual check if applicable>
EOF
)"
```

If the base is wrong on an existing PR, run `gh pr edit <number> --base develop`.

---

## Phase 6 — Report back

Return to the user:

1. **PR URL**
2. **Branch name** and commit list (with author verification result)
3. **Git identity** — name and email set in Phase 0
4. **Sync status** with base branch

---

## Hard rules

| Rule | Detail |
|------|--------|
| User author only | Phase 0 sets repo-local `user.name`/`user.email` from GitHub user; every commit uses that identity |
| No local build/test | Do not run `dotnet build`, `dotnet test`, or equivalent — CI handles validation |
| Fast path first | Apply Fast path table before each phase; one fetch per workflow |
| Scoped commits | One concern per commit; message reflects scope |
| No secrets | Never commit credentials |
| PR base branch | Always target `develop` (`--base develop`) |
| No force-push to main/develop | Warn user if requested |
| `gh` for GitHub | Use `gh` for PRs and checks |

---

## Quick reference

```
Fast path checks → Phase 0 (skip gh if git config set)
  → Bootstrap batch (one fetch + BEHIND) → Commit if dirty (else skip)
  → Merge if BEHIND > 0 (else skip) → Verify authors if commits/merge happened
  → Push → gh pr list --head (edit if exists, else create) → Report URL
```
