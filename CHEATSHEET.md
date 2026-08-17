# Git submodules — cheatsheet

Companion to **[LESSON.md](LESSON.md)**. No timed blocks, no quiz — pure lookup.
Verified on Windows 11 · **Windows PowerShell 5.1** · **git 2.52.0.windows.1** · **.NET SDK 10.0.101**, 2026-08-17.

## 1 · The four places

| # | Where | Shared? | Written by | Is |
|---|---|---|---|---|
| 1 | the **gitlink**: `160000 commit <sha> libs/shared-lib` in the tree | **shared** (tracked) | `git add <path>` + commit | **the pin** — one line, not the files |
| 2 | `.gitmodules` | **shared** (tracked) | `submodule add` / `set-url` / `set-branch` / `git rm` | path → URL (+ optional `branch`, `ignore`) |
| 3 | `.git\config` → `submodule.<name>.url` / `.active` — the submodule's **name**, which equals the path only until someone runs `git mv` | local | `submodule add` / `submodule init` / `deinit` | your opt-in |
| 4 | `.git\modules\<path>` (`<path>\.git` is a one-line pointer file) | local | `submodule add` / `submodule update` | the submodule's real `.git` dir |

A plain `git clone` of the parent brings **1 and 2 only** → empty folder, clean `git status`, failing build.

## 2 · Two directions

```
A. I move the pin                              B. The pin moves me
   git -C <path> checkout <tag|sha>               git pull            (moves the gitlink)
   (or git submodule update --remote <path>)      git submodule update   (moves your checkout)
   git add <path>                                 # NEVER `git add <path>` here — that commits
   git commit                                     # your OLD checkout = a silent downgrade
   git push --recurse-submodules=on-demand
   # submodule first, then parent. Always.
```

## 3 · `git submodule status` prefixes

| Prefix | Meaning |
|---|---|
| (space) | checkout == pin — fine |
| `-` | not **populated** — no checkout. Still shown after `git submodule init`, which writes only the local opt-in (place 3) — run `update --init` |
| `+` | checkout != pin — **in either direction**; read `git diff --submodule=log` to see which |
| `U` | merge conflict on the gitlink (all-zero SHA) |

## 4 · `git status` suffixes

| Long | `--short` | Meaning |
|---|---|---|
| `(new commits)` | ` M` | checkout is at a different commit than the pin |
| `(modified content)` | ` m` | uncommitted edits inside |
| `(untracked content)` | ` ?` | untracked files inside (usually `bin`/`obj` — the submodule needs its own `.gitignore`) |

## 5 · The commands that matter

```powershell
# --- get it ------------------------------------------------------------------
git clone --recurse-submodules <url>            # clone + init + update, one go
git submodule update --init --recursive         # after a plain clone, or after pulling a NEW submodule
                                                # (`git submodule update` alone is a SILENT no-op)

# --- add one -----------------------------------------------------------------
git submodule add <url> libs/shared-lib         # leaves it ON its default branch (not detached)
git -C libs/shared-lib checkout v1.0.0          # pin to a release tag
git add libs/shared-lib; git commit -m "Add shared-lib submodule pinned to v1.0.0"

# --- bump it -----------------------------------------------------------------
git submodule update --remote libs/shared-lib   # -> tip of submodule.<name>.branch (default: remote HEAD)
git -C libs/shared-lib fetch --tags; git -C libs/shared-lib checkout v1.1.0   # or pin a tag by hand
git add libs/shared-lib; git commit -m "Bump shared-lib to v1.1.0"
git submodule set-branch --branch main libs/shared-lib   # writes `branch = main` into .gitmodules

# --- after someone else's bump ------------------------------------------------
git pull; git submodule update                  # or: git pull --recurse-submodules

# --- change code inside -------------------------------------------------------
git -C libs/shared-lib switch main               # ALWAYS branch first: a detached-HEAD commit is lost
                                                 # by the next `submodule update` (reflog only)
# ...edit, then:
git -C libs/shared-lib commit -am "..."
git add libs/shared-lib; git commit -m "Bump shared-lib (...)"
git push --recurse-submodules=on-demand          # pushes submodule, then parent
git push --recurse-submodules=check              # refuses if a submodule commit is unpushed (exit 128)

# --- remove it ----------------------------------------------------------------
git submodule deinit libs/shared-lib             # -f if dirty        (place 3 + checkout)
git rm libs/shared-lib                           # edits .gitmodules  (places 1 + 2)
git commit -m "Remove shared-lib submodule"
Remove-Item -Recurse -Force .git\modules\libs\shared-lib   # place 4 — skip it and re-adding FAILS

# --- inspect ------------------------------------------------------------------
git submodule status                             # prefix + sha + (git describe)
git ls-files -s libs/shared-lib                  # 160000 <sha> 0  libs/shared-lib   <- the gitlink
git ls-tree HEAD libs/shared-lib                 # the same, from the last commit instead of the index
git diff --submodule=log                         # commits, not SHAs;  > forward   < + (rewind) backward
git submodule summary                            # like --submodule=log, but for what is staged
git show HEAD -- libs/shared-lib                 # +Subproject commit <sha>
git config --get-regexp '^submodule\.'           # place 3
Get-Content libs\shared-lib\.git                 # gitdir: ../../.git/modules/libs/shared-lib
git submodule foreach 'git push'                 # SINGLE quotes in PS 5.1 ($name/$sm_path/$sha1 are the shell's)
git submodule sync                               # copy a changed URL from .gitmodules into .git\config
```

## 6 · Config worth setting

```powershell
# in a repo (safest — these are per-clone):
git config --local submodule.recurse true          # pull/checkout/switch/reset/restore also move submodule
                                                   # CHECKOUTS; fetch/grep recurse too (fetch only downloads
                                                   # objects, it never moves the checkout); and plain
                                                   # `git push` behaves like --recurse-submodules=on-demand.
                                                   # Does NOT cover `git clone` or `git ls-files` — git
                                                   # excludes those two by design. `git stash` ignores submodules.
git config --local push.recurseSubmodules check    # refuse to push a pin whose commit is not on any remote
git config --local diff.submodule log              # never read a raw SHA pair again
git config --local status.submoduleSummary true    # `git status` shows the commit range

# machine-wide (changes your global git config — your call):
git config --global submodule.recurse true
git config --global push.recurseSubmodules check
git config --global diff.submodule log
```

> **Keep that order.** `submodule.recurse` and `push.recurseSubmodules` both set the same internal
> push-recursion mode and **the last one git reads wins** — git reads the config file top to bottom. Set
> `submodule.recurse` first, `push.recurseSubmodules` second; reversed, `submodule.recurse=true` silently
> downgrades the `check` guard to `on-demand`. `git config` folds a key into a matching section if one
> already exists and only otherwise appends a new section at the end, so confirm with
> `Get-Content .git\config` that `[submodule] recurse` really sits **above** `[push] recurseSubmodules`.

CI: `actions/checkout@v4` with `with: { submodules: recursive }` — plus a token/deploy key that can read the
submodule repo if it is private.

## 7 · `Look` — print all four places at once

Paste once into your PowerShell window, then run `Look` after any submodule command:

```powershell
function Look {
  param([string]$Sub = 'libs/shared-lib')
  $s = $Sub.Replace('/', '\')
  Write-Host "`n-- parent status" -ForegroundColor Cyan;        git status --short
  Write-Host "`n-- gitlink in INDEX" -ForegroundColor Cyan;     git ls-files -s -- $Sub
  Write-Host "`n-- gitlink in HEAD" -ForegroundColor Cyan;      git ls-tree HEAD -- $Sub
  Write-Host "`n-- .gitmodules (shared)" -ForegroundColor Cyan; if (Test-Path .gitmodules) { Get-Content .gitmodules } else { '(none)' }
  Write-Host "`n-- submodule status [' ' ok  '-' not init  '+' checkout != pin  'U' conflict]" -ForegroundColor Cyan
  git submodule status
  Write-Host "`n-- .git\config submodule.* (local opt-in)" -ForegroundColor Cyan; git config --get-regexp '^submodule\.'
  Write-Host "`n-- .git\modules (local repos)" -ForegroundColor Cyan
  if (Test-Path .git\modules) { (Get-ChildItem .git\modules -Recurse -Directory -Filter 'shared-lib').FullName } else { '(none)' }
  Write-Host "`n-- submodule HEAD" -ForegroundColor Cyan
  if (Test-Path "$s\.git") { @(git -C $Sub status -sb)[0] } else { '(no checkout)' }
}
```

## 8 · PowerShell 5.1 traps

| Trap | Do this instead |
|---|---|
| `cmd-a && cmd-b` | `cmd-a; cmd-b` — there is no `&&`/`\|\|`/`?:`/`??` |
| `git … 2>&1` | never — PS 5.1 wraps stderr in `System.Management.Automation.RemoteException` |
| `... > file.txt` | `Set-Content` / `Add-Content` (`>` writes UTF-16) |
| `git show 'HEAD@{1}'`, `'refs^{commit}'` | single-quote anything with `{ } ^ $` — unquoted, PS 5.1 eats the `{…}`: `HEAD@{1}` becomes `HEAD@` (`fatal: ambiguous argument 'HEAD@'`) and `^{commit}` becomes `-encodedCommand …` (``error: unknown switch `e'``) |
| `git submodule foreach "echo $name"` | single quotes: `'echo $name'` |
| `rm -rf .git/modules/x` | `Remove-Item -Recurse -Force .git\modules\x` |
| `cd libs\shared-lib` then `git add libs/shared-lib` | `fatal: pathspec … did not match` — the gitlink is staged from the **parent** (`cd ..\..`), or use `git -C` |
| `LF will be replaced by CRLF` | harmless (`core.autocrlf=true`), not an error |

## 9 · Reset this lesson

From inside a clone of **each** repo (the `lesson-start` tag must exist locally):

```powershell
git push --force origin refs/tags/lesson-start:refs/heads/main
```

Full procedure: LESSON.md, Appendix B.
