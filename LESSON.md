# Git submodules — a 45-minute hands-on lesson

> **Total timed content: 45 minutes** (hard ceiling). The appendices are reference material — not counted.
>
> | Block | Topic | Who | Minutes |
> |---|---|---|---|
> | 0 | Pre-flight: versions, identity, one window | — | 3 |
> | 1 | Mental model: four places, two directions | — | 4 |
> | 2 | Add a submodule and pin it to a release | ALICE | 7 |
> | 3 | Clone a repo that has one — the empty folder | BOB | 5 |
> | 4 | Bump the pin to a newer version | ALICE | 5 |
> | 5 | The teammate's stale checkout — and the downgrade you must not commit | BOB | 6 |
> | 6 | Change code *inside* the submodule and push it in the right order | BOB → ALICE | 8 |
> | 7 | Remove a submodule properly | BOB | 3 |
> | 8 | Recap and self-quiz | — | 4 |
> | | **Total** | | **45** |
>
> **Read everything above Block 0 before you start the clock — about 9 minutes** (1,300 words at the
> 150 wpm used below).
>
> **How to use the blocks, and where the 45 minutes went.** Each ```` ```powershell ```` block is meant to be
> **pasted whole**; the ```` ```text ```` block under it is the *combined* output of all its lines, in order —
> minus `git clone` progress and any failure the prose repeats from an earlier block (Block 3's `dotnet run`
> is the one case, marked with `(…)` there) — so you check once per block rather than once per line. Five
> blocks print nothing worth showing and carry no output block at all. There are **34 such blocks and 97
> commands** across Blocks 0–8. Budgeted at ~8 s to paste and settle per block, ~7 s to read each command's
> output, and prose at 150 words a minute, the nine blocks come to **~40 minutes** — the remaining five are a
> worst-case allowance for the **five clones** (two of the app repo, three of the library — in 2.2, 3.1 and
> Block 7) and the first `dotnet build` in each clone, which are slower than everything else here; on a warm
> machine with a warm NuGet cache they cost well under a minute in total. Run the commands one at a time
> instead and you will need closer to an hour; that is a fine way to do it, just not a 45-minute one.
>
> **Every block builds on the previous one — you cannot skip a whole block, only trim it.** If you fall
> behind, Blocks 1, 5 and 6 are the ones to read slowly — they carry the entire "why does this break for my
> team" payload. Skim the prose in 2 and 3 but still run every command; cut the inspection in **2.3**; and
> drop **Block 7** entirely — it is the one thing you can look up when you need it.
>
> **Verified on this machine on 2026-08-17**, end to end against the real GitHub repos:
> Windows 11 Pro, **Windows PowerShell 5.1**, **git 2.52.0.windows.1**, **.NET SDK 10.0.101**,
> `gh` authenticated as `tzaczek`. Every "you see" block below is real captured output, not a guess.

---

## What you will be able to do at the end

- Say exactly what a superproject stores about a submodule — and what it does **not** store.
- Add a submodule, pin it to a release tag, and explain why the pin is one line of text.
- Explain why a fresh `git clone` leaves an empty folder and a **clean** `git status`, and fix it.
- Bump a submodule to a new version, and read `git diff --submodule=log` instead of a raw SHA pair.
- Recognise the two situations that produce `modified: <path> (new commits)` and tell them apart —
  including the one where committing would silently **downgrade** the dependency for everyone.
- Commit inside a submodule without losing the commit, and push in the order that does not break your team.
- Remove a submodule so that re-adding it later actually works.
- Argue in an interview when a submodule is the right tool and when NuGet, subtree or a monorepo wins.

---

## The cast, and what this leaves on disk

You play **two people** in two clones of the *same* app repository. That is the whole point: almost every
submodule surprise happens to the *second* person, not the author.

| Role | Folder | Does |
|---|---|---|
| **ALICE** — the author | `app-alice` | Blocks 2, 4, and the end of 6 — adds the submodule, pins it, bumps it |
| **BOB** — the teammate | `app-bob` | Blocks 3, 5, 6, 7 — clones, pulls, contributes back, removes |

Both push as `tzaczek` to the same two repositories. That is expected — the point is the *sequence*,
not the identity.

```
C:\Users\tomas\Repo\Trivago\git-submodules-lab\
├── app-alice\                 clone of git-submodules-lesson  (ALICE)
│   ├── libs\shared-lib\       the submodule  -> git-submodules-lesson-lib
│   └── src\App\
└── app-bob\                   clone of git-submodules-lesson  (BOB)
    ├── libs\shared-lib\
    └── src\App\
```

## Frozen names — never rename these

| Thing | Value |
|---|---|
| Lab root (`$lab`) | `C:\Users\tomas\Repo\Trivago\git-submodules-lab` |
| Parent repo (**APP**) | `https://github.com/tzaczek/git-submodules-lesson.git` |
| Library repo (**LIB**) | `https://github.com/tzaczek/git-submodules-lesson-lib.git` |
| Submodule path | `libs/shared-lib` (forward slashes in git commands, always) |
| App project | `src/App` — run it with `dotnet run --project src/App` from the repo root |
| Library releases | `v1.0.0` and `v1.1.0` — **annotated** tags |
| Reset marker | `lesson-start` — **lightweight** tag in both repos (Appendix B) |
| Commit messages you will type | `Add shared-lib submodule pinned to v1.0.0` · `Bump shared-lib to v1.1.0` · `Unrelated README tweak` · `Friendlier farewell` · `Bump shared-lib (friendlier farewell)` · `Remove shared-lib submodule` |

## SHAs in the expected outputs

The two library commits are frozen and will be *identical* on your machine — you can compare them character
by character:

| Symbol | Real value | What it is |
|---|---|---|
| **L1** | `de9cf89608b4b699a6e4944c288524b26ad00251` (`de9cf89`) | LIB commit `Greeter.Greet (1.0.0)` = tag `v1.0.0` |
| **L2** | `63d40c0a83e0e07d5684e85eae7a3769ed6745c1` (`63d40c0`) | LIB commit `Add Version + Farewell (1.1.0)` = tag `v1.1.0` = LIB `main` |
| `<C>` | *yours will differ* | the library commit **you** make in Block 6 |
| `<C7>` | *yours will differ* | the first 7 characters of `<C>`. Only `<C>` has a separate short form, because Block 6 shows both side by side; the `<start>`/`<P…>` symbols stand for the commit at whatever length git happens to print it |
| `<start>`, `<P1>`…`<P5>` | *yours will differ* | commits in the **app** repo (`<start>` = the one that is already there; `<P3>` is the deliberate mistake you make and immediately reset away in Block 5.1 — you see it printed once, in 5.1, and it never lands in your final history, so Block 6 jumps straight to `<P4>`) |

Anything else that differs — timestamps, GitHub's `remote: Enumerating objects…` / `Receiving objects…`
progress lines, and the `Cloning into …` / `Resolving deltas …` / `done.` lines your own `git clone` prints
where no output block is shown — is noise. Everything else should match.

## PowerShell 5.1 rules that apply to every command in this file

> - **Use ONE PowerShell window for the whole lesson.** `$lab` and the identity variables in Block 0 die
>   with the window, and a second window will not have them.
> - There is **no `&&` and no `||`**. Chain with `;` or use separate lines.
> - **Never append `2>&1`.** PS 5.1 wraps a native program's stderr in `System.Management.Automation.RemoteException`
>   noise. Git's messages already appear in the console.
> - **Do not write files with `>` or `>>`** — PS 5.1 writes UTF-16 there. Use `Set-Content` / `Add-Content`.
> - Quote anything containing `{`, `}`, `^` or `$` in **single** quotes: `'HEAD@{1}'`,
>   `'v1.1.0^{commit}:refs/heads/main'`. Unquoted, PS 5.1 eats the `{…}` and git reports a nonsense error —
>   `fatal: ambiguous argument 'HEAD@'`, or ``error: unknown switch `e'`` for the `^{commit}` form.
> - Delete git directories with `Remove-Item -Recurse -Force`.
> - `git -C <path> <cmd>` runs a command in another repo **without** `cd`. Used deliberately below —
>   the one place you really do `cd` into the submodule is Block 6, where being inside it is the point.
> - The warning `LF will be replaced by CRLF the next time Git touches it` is **harmless** here
>   (`core.autocrlf=true` on this machine). It is not an error.

---

## Block 0 — Pre-flight · 3 min

> **Goal:** the right versions, an identity that works *inside* a submodule, and one window to keep them in.

### 0.1 Versions and auth

```powershell
git --version
dotnet --list-sdks
gh auth status
git ls-remote https://github.com/tzaczek/git-submodules-lesson-lib.git refs/heads/main
```

| Check | Expected on this machine |
|---|---|
| `git --version` | `git version 2.52.0.windows.1` |
| `dotnet --list-sdks` | a line starting `10.0.101` |
| `gh auth status` | `✓ Logged in to github.com account tzaczek` |
| `git ls-remote …` | one line: a SHA, a tab, `refs/heads/main` |

`gh` is never used again — it is checked because it is what seeded Windows Credential Manager, and the
`git ls-remote` probe exercises the same credential path the pushes in Blocks 2, 4 and 6 use, in two
seconds. Nothing here needs to match exactly except that git is **2.25 or newer**: the timed blocks use
`git switch` (git 2.23) and `submodule.recurse` (2.14), the cheatsheet uses `git submodule set-branch`
(2.22) and `git submodule set-url` (2.25), and the `protocol.file.allow=always` workaround in Appendix A
needs 2.38.1.

### 0.2 The lab folder

```powershell
$lab = 'C:\Users\tomas\Repo\Trivago\git-submodules-lab'
New-Item -ItemType Directory -Force $lab | Out-Null   # no output — Out-Null swallows New-Item's DirectoryInfo object
Set-Location $lab
```

### 0.3 Identity — the one that also works inside the submodule

This machine has **no global `user.name` / `user.email`** (deliberately). Setting it per repo with
`git config user.email …` in `app-bob` is **not enough**: a submodule has its *own* git directory
(`.git\modules\libs\shared-lib\config`), so the parent's per-repo config never reaches it, and your
commit in Block 6 dies with `Author identity unknown`. Set it for the whole session instead:

```powershell
$env:GIT_AUTHOR_NAME  = $env:GIT_COMMITTER_NAME  = 'tzaczek'
$env:GIT_AUTHOR_EMAIL = $env:GIT_COMMITTER_EMAIL = 'tzaczek@users.noreply.github.com'
git var GIT_COMMITTER_IDENT
```

```text
tzaczek <tzaczek@users.noreply.github.com> 1786993411 +0200
```

> All **four** variables are needed: with only `GIT_AUTHOR_*` set, git fails with
> `Committer identity unknown` instead. (The other option is `git config --global user.email …`, which
> changes your machine-wide git config — this lesson deliberately does not do that.)

---

## Block 1 — The mental model: four places, two directions · 4 min

> **Goal:** be able to predict every output in Blocks 2–7 instead of memorising commands.

A submodule is **a pinned commit of another repository, checked out in a folder of yours**. The outer
repo is the *superproject*. Git keeps exactly four pieces of state about it — and the split between
them explains every surprise in this lesson:

```
  SHARED  (tracked; travels in your commits, everyone gets them)
  ┌──────────────────────────────────────────────────────────────────────┐
  │ 1. THE GITLINK  — a tree entry:  160000 commit <sha>  libs/shared-lib │
  │       one line of text. THE PIN. Not the files.                      │
  │ 2. .gitmodules  — a tracked file: path -> URL map                    │
  └──────────────────────────────────────────────────────────────────────┘
  LOCAL   (never pushed; exist only in your clone)
  ┌──────────────────────────────────────────────────────────────────────┐
  │ 3. .git\config  submodule.libs/shared-lib.url / .active               │
  │       = "I opted in".  Written by `submodule add` / `submodule init`  │
  │ 4. .git\modules\libs\shared-lib  — the submodule's REAL .git dir      │
  │       libs\shared-lib\.git is a one-line file pointing at it          │
  └──────────────────────────────────────────────────────────────────────┘
```

A `git clone` of the parent brings you places **1 and 2 only**. That is the whole reason the folder is
empty and `git status` is clean — nothing is missing from git's point of view (Block 3).

Every command in this lesson moves in one of **two directions**:

| Direction | Situation | The motion |
|---|---|---|
| **A — I move the pin** | you want a different version of the library | change the submodule's checkout → `git add libs/shared-lib` → commit **the parent** → push **submodule first, then parent** |
| **B — the pin moves me** | someone else moved it and you pulled | `git pull` updates the gitlink; your checkout still lags → `git submodule update` |

**Detached HEAD is normal here, but not always.** A pin is a *commit*, not a branch, so anything that
applies a pin (`git submodule update`, `git clone --recurse-submodules`, `git checkout <tag>` inside,
`git submodule update --remote`) leaves the submodule with a **detached HEAD**. The one exception is
`git submodule add`, which is a fresh clone and leaves you **on `main`**. This asymmetry is why the
author of a submodule rarely hits the lost-commit trap and every teammate eventually does (Block 6).

**The .NET analogy.** `.gitmodules` is roughly *where* the dependency comes from (like a `ProjectReference`
path or a feed), and the gitlink is *exactly which build* you use — a one-line lock file. Which is also
the honest argument for using NuGet instead: see **Appendix C**.

---

## Block 2 — ALICE adds the submodule and pins it to v1.0.0 · 7 min

> **You are: ALICE (the author).**
> **Goal:** watch all four places appear, and learn that the pin is one line of text.

```powershell
Set-Location $lab
git clone https://github.com/tzaczek/git-submodules-lesson.git app-alice
Set-Location "$lab\app-alice"
```

### 2.1 First, the failure that motivates all of this

`src\App\App.csproj` already has a `ProjectReference` to a project that is not there yet.

```powershell
dotnet run --project src/App
```

```text
C:\Program Files\dotnet\sdk\10.0.101\Microsoft.Common.CurrentVersion.targets(2189,5): warning MSB9008: The referenced project ..\..\libs\shared-lib\src\SharedLib\SharedLib.csproj does not exist. [C:\Users\tomas\Repo\Trivago\git-submodules-lab\app-alice\src\App\App.csproj]
C:\Users\tomas\Repo\Trivago\git-submodules-lab\app-alice\src\App\Program.cs(1,7): error CS0246: The type or namespace name 'SharedLib' could not be found (are you missing a using directive or an assembly reference?) [C:\Users\tomas\Repo\Trivago\git-submodules-lab\app-alice\src\App\App.csproj]

The build failed. Fix the build errors and run again.
```

> Remember `MSB9008` + `CS0246` together. It is the signature of a **missing submodule checkout**, and you
> will see it again in Block 3 in a repo where `git status` says everything is fine.
>
> Why the library builds differently *inside* the superproject than in its own clone — MSBuild walks up
> past the git boundary — is **Appendix E**.

### 2.2 Do it — add it

```powershell
git submodule add https://github.com/tzaczek/git-submodules-lesson-lib.git libs/shared-lib
```

```text
Cloning into 'C:/Users/tomas/Repo/Trivago/git-submodules-lab/app-alice/libs/shared-lib'...
(progress lines omitted)
warning: in the working copy of '.gitmodules', LF will be replaced by CRLF the next time Git touches it
```

### 2.3 Three of the four places, one command each (inspection — cut this if you are behind)

```powershell
Get-Content .gitmodules
git ls-files -s libs/shared-lib
Get-Content libs\shared-lib\.git
```

```text
[submodule "libs/shared-lib"]
	path = libs/shared-lib
	url = https://github.com/tzaczek/git-submodules-lesson-lib.git

160000 63d40c0a83e0e07d5684e85eae7a3769ed6745c1 0	libs/shared-lib

gitdir: ../../.git/modules/libs/shared-lib
```

Read those three against the diagram in Block 1: the map (**place 2**), the **gitlink** — mode `160000`, the
mode git uses for "a commit, not a blob and not a tree" — sitting *staged in the index*, which is what
becomes **place 1** the moment you commit it in 2.5, and the pointer file to the real git dir (**place 4**).
`add` wrote **place 3** here too, but you will read it in Block 3.1, where `git submodule init` writes it a
second time and it is the only thing that happens.

`add` cloned the library at its default branch, so right now the pin is **L2** (`v1.1.0`). You want a
deliberate release, not "whatever `main` happened to be":

### 2.4 Do it — pin it to v1.0.0

```powershell
git -C libs/shared-lib checkout v1.0.0
```

```text
Note: switching to 'v1.0.0'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.
(…advice…)

HEAD is now at de9cf89 Greeter.Greet (1.0.0)
```

```powershell
git status
git submodule status
git diff --submodule=log
```

```text
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   .gitmodules
	new file:   libs/shared-lib

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   libs/shared-lib (new commits)

+de9cf89608b4b699a6e4944c288524b26ad00251 libs/shared-lib (v1.0.0)

Submodule libs/shared-lib 63d40c0..de9cf89 (rewind):
  < Add Version + Farewell (1.1.0)
```

Three things to notice, all of them will matter later:

- A *directory* is listed as `new file:` — because to git it is one line, not a directory.
- The same path is both **staged** (`new file:`, the pin git recorded during `add`) and **unstaged**
  (`modified: … (new commits)`, your `checkout` moved the checkout away from it). The `+` in
  `git submodule status` says the same thing: **checkout ≠ recorded pin**. A leading *space* there — which is
  what you would have seen a minute ago — means checkout == pin.
- `--submodule=log` translates the SHA pair into commits, and says `(rewind)` because you moved
  **backwards**. Configure `diff.submodule=log` once and you never read a raw SHA pair again.

### 2.5 Record the pin and prove it is one line

```powershell
git add libs/shared-lib
git commit -m "Add shared-lib submodule pinned to v1.0.0"
git show HEAD -- libs/shared-lib
```

```text
[main <P1>] Add shared-lib submodule pinned to v1.0.0
 2 files changed, 4 insertions(+)
 create mode 100644 .gitmodules
 create mode 160000 libs/shared-lib

(…commit header…)
diff --git a/libs/shared-lib b/libs/shared-lib
new file mode 160000
index 0000000..de9cf89
--- /dev/null
+++ b/libs/shared-lib
@@ -0,0 +1 @@
+Subproject commit de9cf89608b4b699a6e4944c288524b26ad00251
```

> **`+Subproject commit …` is the entire content of the submodule as far as your repo is concerned.**
> Four insertions in that commit: three lines of `.gitmodules` and this one.

```powershell
dotnet run --project src/App
git push
```

```text
Hello, tomas! (shared-lib 1.0.0)

To https://github.com/tzaczek/git-submodules-lesson.git
   <start>..<P1>  main -> main
```

> **If you typed `git add libs/shared-lib` and got `fatal: pathspec 'libs/shared-lib' did not match any files`**
> you are still *inside* the submodule. `cd ..\..` first — the gitlink lives in the parent, and only the
> parent can stage it.
>
> **Two notes for real projects.** Release tags should be **annotated** (`git tag -a v1.0.0 -m …`) as these
> are: `git submodule status` runs `git describe`, which only considers **annotated** tags and falls back to
> `git describe --tags` (which does see lightweight ones) only when that finds nothing. So a lightweight tag
> usually still shows — but the moment any annotated tag is reachable it wins, and you read
> `v0.9-2-gabc1234` instead of your release name. And teams usually write the URL **relative**
> (`url = ../git-submodules-lesson-lib.git`), which is resolved against the parent's `origin`, so HTTPS
> users and SSH users both get a working URL from the same `.gitmodules` — see Appendix A.

---

## Block 3 — BOB clones it: the empty folder · 5 min

> **You are: BOB (the teammate).**
> **Goal:** see that a clean `git status` proves nothing, and learn what `init` and `update` each do.

```powershell
Set-Location $lab
git clone https://github.com/tzaczek/git-submodules-lesson.git app-bob
Set-Location "$lab\app-bob"
git status
Get-ChildItem libs\shared-lib
git submodule status
dotnet run --project src/App   # fails — same MSB9008 + CS0246 as 2.1
```

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean

-de9cf89608b4b699a6e4944c288524b26ad00251 libs/shared-lib

(…same MSB9008 + CS0246 as 2.1…)
```

`Get-ChildItem` prints nothing between those two — the folder exists and is empty. And `dotnet run` fails
with the same `MSB9008` + `CS0246` as in Block 2.1 — while `git status` says **clean**. That combination is
the tell. The other tell is the **`-`** prefix: *not populated*. Note there is no `(v1.0.0)` suffix either —
git cannot `describe` a repository it has not cloned yet.

> **`git -C libs/shared-lib status` does not help you here.** The folder is empty, so git walks *up* out of
> it and reports the **parent's** status — `On branch main` / `nothing to commit, working tree clean`, exit
> code 0. It reads like a healthy submodule sitting on a branch, which is the exact opposite of the truth.
> Ask `git submodule status` (the one with the `-` prefix) instead.

### 3.1 Do it — the trap: `update` alone does nothing

```powershell
git submodule update
```

No output. Exit code 0. **Nothing happened** — `update` only acts on submodules that are *initialised*.
This silent no-op is why the command you should type by reflex is `git submodule update --init --recursive`
— **but do not run that yet.** It would do both halves at once and you would see neither. Do it in two
steps this once, to see who writes what:

```powershell
git submodule init
git config --get-regexp '^submodule\.'
```

```text
Submodule 'libs/shared-lib' (https://github.com/tzaczek/git-submodules-lesson-lib.git) registered for path 'libs/shared-lib'

submodule.libs/shared-lib.active true
submodule.libs/shared-lib.url https://github.com/tzaczek/git-submodules-lesson-lib.git
```

That is **place 3** — copied out of `.gitmodules` into your local `.git\config`. Nothing was downloaded.

```powershell
git submodule update
git submodule status
git -C libs/shared-lib status
dotnet run --project src/App
```

```text
Cloning into 'C:/Users/tomas/Repo/Trivago/git-submodules-lab/app-bob/libs/shared-lib'...
Submodule path 'libs/shared-lib': checked out 'de9cf89608b4b699a6e4944c288524b26ad00251'

 de9cf89608b4b699a6e4944c288524b26ad00251 libs/shared-lib (v1.0.0)

HEAD detached at de9cf89
nothing to commit, working tree clean

Hello, tomas! (shared-lib 1.0.0)
```

`update` created **place 4** and the checkout. Note `HEAD detached at de9cf89` — it says a *SHA*, not a
tag: git checks out the pin itself.

| `git submodule status` prefix | Meaning |
|---|---|
| (space) | checkout matches the recorded pin — all good |
| `-` | not **populated** — there is no checkout. Still shown right after `git submodule init`, which writes only the local opt-in (place 3). Run `update --init` |
| `+` | checkout ≠ recorded pin — **in either direction** |
| `U` | merge conflict on the gitlink |

> **The one-liner for real life:** `git clone --recurse-submodules <url>` does clone + init + update in
> one go. And **nested** submodules are the reason `--recursive` exists: their git dirs nest as
> `.git\modules\<a>\modules\<b>`, and every `clone`/`update`/`status` needs `--recursive` to reach them.
>
> After pulling a commit that *adds* a new submodule you also need `git submodule update --init` again —
> the new one is uninitialised for exactly the same reason.

---

## Block 4 — ALICE bumps the pin to v1.1.0 · 5 min

> **You are: ALICE.**
> **Goal:** move the pin the way you will do it at work, and review it like a reviewer.

```powershell
Set-Location "$lab\app-alice"
git submodule update --remote libs/shared-lib
git status
git submodule status
```

```text
Submodule path 'libs/shared-lib': checked out '63d40c0a83e0e07d5684e85eae7a3769ed6745c1'

On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   libs/shared-lib (new commits)

no changes added to commit (use "git add" and/or "git commit -a")

+63d40c0a83e0e07d5684e85eae7a3769ed6745c1 libs/shared-lib (v1.1.0)
```

> **`update` vs `update --remote` — the one-sentence difference.** `git submodule update` moves your
> checkout **to the pin recorded in the parent** (direction B). `git submodule update --remote` ignores the
> pin, fetches the submodule's remote and moves your checkout **to the tip of its branch** (direction A,
> the first half). Which branch? `submodule.<name>.branch` from `.gitmodules` if set — otherwise the
> remote's default branch. Set it explicitly on a real project with
> `git submodule set-branch --branch main libs/shared-lib` (it adds `branch = main` to `.gitmodules`).
> Block 2.4 did the same thing by hand with `git -C libs/shared-lib checkout v1.0.0`, which is what you
> want when you pin to **tags** rather than follow a branch.

### 4.1 Read the change like a human

```powershell
git diff
git diff --submodule=log
```

```text
diff --git a/libs/shared-lib b/libs/shared-lib
index de9cf89..63d40c0 160000
--- a/libs/shared-lib
+++ b/libs/shared-lib
@@ -1 +1 @@
-Subproject commit de9cf89608b4b699a6e4944c288524b26ad00251
+Subproject commit 63d40c0a83e0e07d5684e85eae7a3769ed6745c1

Submodule libs/shared-lib de9cf89..63d40c0:
  > Add Version + Farewell (1.1.0)
```

The raw diff is what a reviewer sees by default on a pull request: two SHAs and no information.
`--submodule=log` lists the commits you are actually pulling in, and `>` means *forward*.

> **Reviewing a submodule bump at work:** `git diff --submodule=log origin/main...origin/feature` shows the
> commits being adopted, and then check the new SHA is actually reachable from a branch or tag in the
> library (a PR can legitimately point at a commit that only exists on someone's laptop — Block 6).

### 4.2 Do it — record and publish

```powershell
dotnet run --project src/App
git add libs/shared-lib
git commit -m "Bump shared-lib to v1.1.0"
git push
```

```text
Hello, tomas! (shared-lib 1.1.0)
Goodbye, tomas!

[main <P2>] Bump shared-lib to v1.1.0
 1 file changed, 1 insertion(+), 1 deletion(-)

To https://github.com/tzaczek/git-submodules-lesson.git
   <P1>..<P2>  main -> main
```

**One file changed, one line.** A dependency upgrade that reviews as a single line, and reproduces exactly
— that is the actual selling point of submodules.

---

## Block 5 — BOB pulls: the stale checkout and the downgrade trap · 6 min

> **You are: BOB.**
> **Goal:** the single most common submodule incident on a team — and how not to cause it.

```powershell
Set-Location "$lab\app-bob"
git pull
git status
git submodule status
dotnet run --project src/App
```

```text
From https://github.com/tzaczek/git-submodules-lesson
   <P1>..<P2>  main       -> origin/main
Updating <P1>..<P2>
Fast-forward
 libs/shared-lib | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

(…)
	modified:   libs/shared-lib (new commits)

+de9cf89608b4b699a6e4944c288524b26ad00251 libs/shared-lib (v1.0.0)

Hello, tomas! (shared-lib 1.0.0)
```

You pulled the bump. Your build still prints **1.0.0**. Nothing is broken: `git pull` moved the *pin*
(place 1, a tracked file), and never touches your *checkout* (place 4, local). Your checkout is now
**behind** the pin.

> ### ⛔ Read this before you type anything
> `modified: libs/shared-lib (new commits)` means only **"checkout ≠ recorded pin"** — it does **not** say
> which is newer. In Block 4 you were *ahead* (you had moved the checkout deliberately). Here you are
> **behind**. **Do not** run `git add libs/shared-lib`, `git commit -a` or `git add -A` now: you would
> commit *your old checkout* as the new pin and silently **downgrade the library for the whole team**.
> `git diff --submodule=log` tells you the direction — `>` newer, `<` plus `(rewind)` older.

```powershell
git diff --submodule=log
```

```text
Submodule libs/shared-lib 63d40c0..de9cf89 (rewind):
  < Add Version + Farewell (1.1.0)
```

### 5.1 Do it wrong once, on purpose

```powershell
Add-Content README.md 'unrelated tweak'
git commit -am "Unrelated README tweak"
git show --stat HEAD
```

```text
[main <P3>] Unrelated README tweak
 2 files changed, 2 insertions(+), 1 deletion(-)

(…commit header…)
 README.md       | 1 +
 libs/shared-lib | 2 +-
 2 files changed, 2 insertions(+), 1 deletion(-)
```

There it is: a commit about a README that also **reverted the dependency**, because `-a` stages every
tracked modification and the gitlink is a tracked file. On a real team this is a mystery bug the next
morning. `git show --submodule=log HEAD` spells it out — its last lines read
`Submodule libs/shared-lib 63d40c0..de9cf89 (rewind):` / `  < Add Version + Farewell (1.1.0)`.

```powershell
git reset --hard HEAD~1
git submodule update
git submodule status
dotnet run --project src/App
```

```text
HEAD is now at <P2> Bump shared-lib to v1.1.0

Submodule path 'libs/shared-lib': checked out '63d40c0a83e0e07d5684e85eae7a3769ed6745c1'

 63d40c0a83e0e07d5684e85eae7a3769ed6745c1 libs/shared-lib (v1.1.0)

Hello, tomas! (shared-lib 1.1.0)
Goodbye, tomas!
```

`git submodule update` is direction B, all of it: make the checkout match the pin. The prefix is a space
again, and the app finally prints what Alice shipped.

> **The setting that prevents this class of bug:** `git config submodule.recurse true` makes
> `pull`/`checkout`/`switch`/`reset`/`restore` update your submodule **checkouts** automatically, makes
> `fetch`/`grep` recurse into submodules (`fetch` only downloads the submodule's objects — it never moves the
> checkout), and turns a plain `git push` into `--recurse-submodules=on-demand`. It does **not** cover
> `git clone` or `git ls-files`: git excludes those two by design, even when the setting is `--global`, so on
> a clone you still type `--recurse-submodules` yourself. `git stash` ignores submodules entirely.
> **Do not set it yet** — it would have made the `git pull` at the top of this block move your checkout for
> you, and the stale checkout you just spent five minutes on would never have appeared. Set it at the end of
> Block 6.

---

## Block 6 — BOB changes the library and pushes it in the right order · 8 min

> **You are: BOB**, then **ALICE** at the end.
> **Goal:** the other half of direction A — and the mistake that strands your whole team.

```powershell
Set-Location "$lab\app-bob\libs\shared-lib"
git status
```

```text
HEAD detached at 63d40c0
nothing to commit, working tree clean
```

You are inside **another repository** now — its own history, its own remote, its own branches. And its
HEAD is detached, because it is sitting on a pin.

> **Commit here and the commit belongs to no branch.** The next `git submodule update` moves HEAD away
> and the commit is reachable only through the reflog. **Rule: switch to a branch before you edit inside a
> submodule.** (Recovery, if you ever forget, is in Appendix A.)

```powershell
git switch main
(Get-Content src\SharedLib\Greeter.cs) -replace 'Goodbye', 'Farewell' | Set-Content src\SharedLib\Greeter.cs
git diff --stat
```

```text
Your branch is up to date with 'origin/main'.
Switched to branch 'main'

 src/SharedLib/Greeter.cs | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

> PS 5.1 interleaves git's stderr and stdout, so those first two lines can appear in either order. Nothing
> is wrong if `Switched to branch 'main'` comes first.
>
> If `git switch main` says your branch is *behind* `origin/main`, run `git pull` here before editing —
> you would otherwise build a commit on a stale base.

### 6.1 What the parent sees, before and after the submodule commit

```powershell
Set-Location "$lab\app-bob"
git status
git diff
```

```text
(…)
  (commit or discard the untracked or modified content in submodules)
	modified:   libs/shared-lib (modified content)

diff --git a/libs/shared-lib b/libs/shared-lib
--- a/libs/shared-lib
+++ b/libs/shared-lib
@@ -1 +1 @@
-Subproject commit 63d40c0a83e0e07d5684e85eae7a3769ed6745c1
+Subproject commit 63d40c0a83e0e07d5684e85eae7a3769ed6745c1-dirty
```

`(modified content)` and the `-dirty` suffix — **not** `(new commits)`. The parent can only ever point at
a *commit*, so uncommitted work in the submodule is not something it can record. Commit it:

```powershell
git -C libs/shared-lib commit -am "Friendlier farewell"
git status
git submodule status
dotnet run --project src/App
```

```text
[main <C7>] Friendlier farewell
 1 file changed, 1 insertion(+), 1 deletion(-)

(…)
	modified:   libs/shared-lib (new commits)

+<C> libs/shared-lib (v1.1.0-1-g<C7>)

Hello, tomas! (shared-lib 1.1.0)
Farewell, tomas!
```

`(v1.1.0-1-g<C7>)` is `git describe`: *one commit after v1.1.0*. Now record the new pin in the parent:

```powershell
git add libs/shared-lib
git commit -m "Bump shared-lib (friendlier farewell)"
```

```text
[main <P4>] Bump shared-lib (friendlier farewell)
 1 file changed, 1 insertion(+), 1 deletion(-)
```

This is `<P4>` — `<P3>` was the README commit you threw away in Block 5.1, so that number never lands in
your history.

### 6.2 Do it — the push that must fail

Your parent commit now points at a library commit that **exists only on your disk**.

```powershell
git push --recurse-submodules=check
```

```text
The following submodule paths contain changes that can
not be found on any remote:
  libs/shared-lib

Please try

	git push --recurse-submodules=on-demand

or cd to the path and use

	git push

to push them to a remote.

fatal: Aborting.
```

Exit code 128, and **nothing** was pushed — verify with
`git ls-remote https://github.com/tzaczek/git-submodules-lesson.git refs/heads/main` if you like. This is
the guard you want as a team default; without it a plain `git push` succeeds happily and your teammates'
next `git pull` dies (the exact error is in Appendix A — it is *the* classic "submodules are broken" ticket).

```powershell
git push --recurse-submodules=on-demand
```

```text
Pushing submodule 'libs/shared-lib'
To https://github.com/tzaczek/git-submodules-lesson-lib.git
   63d40c0..<C7>  main -> main
To https://github.com/tzaczek/git-submodules-lesson.git
   <P2>..<P4>  main -> main
```

**Submodule first, then parent** — the only safe order, in one command.

> **This is exactly why you ran `git switch main` first.** `on-demand` pushes the submodule the way a plain
> `git push` inside it would. On a detached HEAD (or on a branch whose name does not match what the
> submodule's remote expects) it prints `Pushing submodule 'libs/shared-lib'` followed by
> `Everything up-to-date` — it pushed your stale local `main`, not the detached commit — and then the check
> that `on-demand` runs afterwards aborts the whole push with `fatal: Aborting.` and exit code 128, so
> **neither** remote moves. The misleading part is the reassuring `Pushing submodule` line, not the exit
> status.

### 6.3 ALICE picks it up — and the three settings you keep

```powershell
Set-Location "$lab\app-alice"
git pull --recurse-submodules
dotnet run --project src/App
```

```text
From https://github.com/tzaczek/git-submodules-lesson
   <P2>..<P4>  main       -> origin/main
Fetching submodule libs/shared-lib
From https://github.com/tzaczek/git-submodules-lesson-lib
   63d40c0..<C7>  main       -> origin/main
Updating <P2>..<P4>
Fast-forward
 libs/shared-lib | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
Submodule path 'libs/shared-lib': checked out '<C>'

Hello, tomas! (shared-lib 1.1.0)
Farewell, tomas!
```

One command, both repos, working build — compare with Block 5, where the same pull left a stale checkout.
Now set the three defaults, in **this clone** — `app-alice`, where the `Set-Location` above left you. Leave
`app-bob` alone until the lesson is over: with `submodule.recurse=true` set there, Block 7's closing
`git reset --hard origin/main` does not just behave differently — it **fails**, because by then you have
deleted `.git\modules\libs\shared-lib` by hand and the recursion goes looking for it
(`fatal: not a git repository: ../../.git/modules/libs/shared-lib` / `fatal: could not reset submodule index`).
Drop `--local` and add `--global` only if you want them machine-wide.

```powershell
git config --local submodule.recurse true
git config --local push.recurseSubmodules check
git config --local diff.submodule log
```

> **Order matters for the first two.** `submodule.recurse` and `push.recurseSubmodules` both set the same
> internal push-recursion mode, and **the last one git reads wins** — git reads `.git\config` top to bottom.
> So write `submodule.recurse` **first** and `push.recurseSubmodules` **second**: the other way round,
> `submodule.recurse=true` lands after `[push]` and silently downgrades your `check` guard to `on-demand`.
> Each `git config` call folds the key into a **matching section if one already exists** and only otherwise
> appends a new section at the end — so in a clone that already has a bare `[submodule]` section (any
> `git clone --recurse-submodules`, and `--global` configs vary too) the order you type them in may not be
> the order in the file. Check with `Get-Content .git\config` that `[submodule] recurse` ends up *before*
> `[push] recurseSubmodules`, and reorder by hand if it does not. The third line closes the loop opened in
> Block 2.4: no more raw SHA pairs in `git diff`.

---

## Block 7 — BOB removes the submodule · 3 min

> **You are: BOB.** This removal stays local — **do not push it**; the repos must keep working for the reset.
> **Goal:** know why deleting the folder is not removal, and why re-adding later fails.

```powershell
Set-Location "$lab\app-bob"
git submodule deinit libs/shared-lib
git submodule status
Test-Path .git\modules\libs\shared-lib
```

```text
Cleared directory 'libs/shared-lib'
Submodule 'libs/shared-lib' (https://github.com/tzaczek/git-submodules-lesson-lib.git) unregistered for path 'libs/shared-lib'

-<C> libs/shared-lib

True
```

`deinit` undid place 3 and the checkout (add `-f` if you have local modifications). Place 4 survives.

```powershell
git rm libs/shared-lib
git status --short
git commit -m "Remove shared-lib submodule"
Remove-Item -Recurse -Force .git\modules\libs\shared-lib
```

```text
rm 'libs/shared-lib'

M  .gitmodules
D  libs/shared-lib

[main <P5>] Remove shared-lib submodule
 2 files changed, 4 deletions(-)
 delete mode 160000 libs/shared-lib
```

`git rm` handled places 1 and 2 — including editing `.gitmodules`, which is now a **tracked, empty (0-byte)
file** you can `git rm` as well if this was the last submodule. Only the manual `Remove-Item` finishes the
job; skip it and a future `git submodule add … libs/shared-lib` fails with
`fatal: A git directory for 'libs/shared-lib' is found locally with remote(s)`.

Order matters: `deinit` **after** `git rm` fails with `pathspec 'libs/shared-lib' did not match`.

```powershell
git reset --hard origin/main
git submodule update --init
dotnet run --project src/App
```

```text
HEAD is now at <P4> Bump shared-lib (friendlier farewell)

Submodule 'libs/shared-lib' (https://github.com/tzaczek/git-submodules-lesson-lib.git) registered for path 'libs/shared-lib'
Cloning into 'C:/Users/tomas/Repo/Trivago/git-submodules-lab/app-bob/libs/shared-lib'...
Submodule path 'libs/shared-lib': checked out '<C>'

Hello, tomas! (shared-lib 1.1.0)
Farewell, tomas!
```

Back to a working tree — and because you deleted `.git\modules\…`, this one really re-clones (`init` has to
register place 3 again too, because `deinit` removed it). The pin is now `<C>`, **your own** library commit,
and that is what the app prints: the whole lesson in two lines.

---

## Block 8 — Recap and self-quiz · 4 min

| Situation | Command | What actually moves |
|---|---|---|
| Add a dependency | `git submodule add <url> <path>` | places 2+3+4, and **stages** the gitlink — place 1 exists only once you commit |
| Pin it to a release | `git -C <path> checkout v1.0.0` then `git add <path>` + commit | checkout, then the gitlink |
| Fresh clone | `git clone --recurse-submodules` (or `update --init --recursive`) | 3+4 and the checkout; 1+2 came with the clone |
| Bump to the library's latest | `git submodule update --remote <path>` then `git add <path>` + commit | checkout, then the gitlink |
| After a teammate's bump lands | `git submodule update` | checkout only — **never** `git add <path>` here |
| Changed code inside | `git -C <path> switch main` → edit → commit **inside** → `git add <path>` + commit → `git push --recurse-submodules=on-demand` | the submodule's history, then the gitlink, then both remotes |
| Remove it | `deinit` → `git rm <path>` → commit → `Remove-Item .git\modules\<path>` | 3+checkout, then 1+2, then 4 |

Print **Appendix D** ([CHEATSHEET.md](CHEATSHEET.md)) and keep it beside the terminal — it is this table
plus the glyphs, the traps and the config, on one page.

<details><summary><b>Q1.</b> A colleague says "the submodule's files are in our repo". Correct them in one sentence.</summary>

The parent stores **one line** — a gitlink, `160000 commit <sha> libs/shared-lib` — plus the path→URL map
in `.gitmodules`. The files live in the other repository; your commit only records *which commit* of it.
</details>

<details><summary><b>Q2.</b> You pulled, git says <code>modified: libs/shared-lib (new commits)</code>, and the app still prints the old version. What happened, and what is the one command?</summary>

`git pull` updated the recorded pin; your checkout (a local thing, place 4) still sits at the old commit,
i.e. you are *behind*. `git submodule update`. Committing instead would push your old checkout as the new
pin and downgrade everyone.
</details>

<details><summary><b>Q3.</b> What is the difference between <code>.gitmodules</code> and the <code>submodule.*</code> entries in <code>.git\config</code>, and what does <code>git submodule init</code> do?</summary>

`.gitmodules` is **tracked and shared** — it travels in commits. `.git\config` is **local** and never
pushed; it is your opt-in and can legitimately hold a different URL from everyone else's. `git submodule init`
copies the mapping from the first into the second, and downloads nothing.
</details>

<details><summary><b>Q4.</b> You pushed the parent but not the submodule commit. What does your teammate see, and what do you run?</summary>

You never saw this one happen — Block 6.2's `check` stopped you before you could cause it; the captured
failure is in **Appendix A**. Their `git pull` fails at the fetch stage with
`fatal: remote error: upload-pack: not our ref <sha>` / `Errors during submodule fetch:` (their branch does
not even advance). You fix it with `git -C libs/shared-lib push` — **not** with
`git push --recurse-submodules=on-demand`, which now says `Everything up-to-date` and pushes nothing,
because the parent is already up to date. Prevent it with `push.recurseSubmodules=check`.
</details>

<details><summary><b>Q5.</b> Why is the submodule's HEAD detached after <code>git submodule update</code>, and what must you do before committing in there?</summary>

Because a pin is a commit, not a branch — `update` checks out that exact SHA. Before editing, `git switch <branch>`
(and pull it), otherwise your commit belongs to no branch: the next `update` moves HEAD away and only the
reflog remembers it. It also breaks `push --recurse-submodules=on-demand`.
</details>

---

**Everything below is untimed reference material — it is not counted against the 45 minutes.**

## How to say this in an interview

**"What are git submodules and when would you use one?"**
> A submodule is a pinned commit of another repository, recorded in the parent as a single tree entry —
> mode `160000` — plus a `.gitmodules` file mapping that path to a URL. So the parent stores *which commit*
> of the dependency it builds against, not its files. I reach for it when I need source-level coupling with
> an exact, reviewable, reproducible pin: a shared library I also develop, a vendored fork I patch, or config
> that must version with the app. If the dependency is a stable, versioned artifact, a NuGet package is the
> better answer — the dependency graph, transitive resolution and restore caching come for free.

**"What goes wrong with them on a team?"**
> Three things, all of them the pin/checkout split. First: a plain clone leaves an empty folder and a clean
> `git status`, so CI and new joiners get a build error that looks unrelated — `--recurse-submodules` on clone
> and `submodules: recursive` in the CI checkout step. Second: after a pull, the checkout lags the pin and
> `git commit -a` will silently commit the *downgrade* — `submodule.recurse=true` plus reading
> `git diff --submodule=log` in review. Third: pushing the parent while the submodule commit is still local
> strands everyone with `upload-pack: not our ref` (the captured failure is in Appendix A — the lesson's
> `check` guard is what stopped you from ever producing it) — `push.recurseSubmodules=check` as a team-wide
> default.
> And pin to annotated release tags, not to a moving branch.

**"How do you roll out a library update across services?"**
> The library releases a tag; each consumer bumps its pin in its own commit and PR. The diff is one line,
> the review reads `git diff --submodule=log`, CI builds the exact combination, and the parent's history is a
> reproducible record of which library commit shipped when — you can check out any old commit of the service
> and rebuild it exactly. The cost is that the rollout is N pull requests, which is precisely the trade-off a
> monorepo removes and a package feed automates.

## Appendix A — troubleshooting

**`Author identity unknown` when committing inside the submodule.** Its config is
`.git\modules\libs\shared-lib\config`, so `git config user.email` set in the parent does not apply. Use the
four `GIT_AUTHOR_*` / `GIT_COMMITTER_*` variables from Block 0 (all four — with only the author pair set you
get `Committer identity unknown`), or `git config --global`.

**The classic: parent pushed, submodule commit not pushed.** The teammate's plain `git pull` aborts *at the
fetch stage* — their branch does not even advance. (`fetch.recurseSubmodules` has defaulted to `on-demand`
for many releases, git 2.52 included, and it follows `submodule.recurse` when that is set — so after Block
6.3 it recurses unconditionally.)

```text
From https://github.com/tzaczek/git-submodules-lesson
   <x>..<y>  main       -> origin/main
Fetching submodule libs/shared-lib
fatal: remote error: upload-pack: not our ref <sha>
Errors during submodule fetch:
	libs/shared-lib
```

`git pull --no-recurse-submodules` fast-forwards the parent, and then `git submodule update` fails with:

```text
fatal: remote error: upload-pack: not our ref <sha>
fatal: Fetched in submodule path 'libs/shared-lib', but it did not contain <sha>. Direct fetching of that commit failed.
```

**The fix is on the author's side:** `git -C libs/shared-lib push` (or `git submodule foreach git push`).
Running `git push --recurse-submodules=on-demand` *after* the parent is already pushed prints
`Everything up-to-date` and pushes **nothing** — on-demand only pushes submodules for commits it is
pushing right now. Note that nothing in `git status` warns you that a submodule commit is unpushed; only
`push.recurseSubmodules=check` catches it.

**I committed inside the submodule on a detached HEAD and `git submodule update` ate it.** No warning is
printed. The commit is in the submodule's reflog:

```powershell
git -C libs/shared-lib reflog
git -C libs/shared-lib branch rescue <sha>
```

```text
7e9c5b1 HEAD@{0}: checkout: moving from c219369… to 7e9c5b1…
c219369 HEAD@{1}: commit: WIP on detached HEAD
```

(Quote `'HEAD@{1}'` in PowerShell if you reference it directly.)

**`fatal: A git directory for 'libs/shared-lib' is found locally with remote(s):`** — you removed the
submodule but left `.git\modules\libs\shared-lib`. Either `Remove-Item -Recurse -Force .git\modules\libs\shared-lib`
first, or `git submodule add --force <url> libs/shared-lib`, which answers
`Reactivating local git directory for submodule 'libs/shared-lib'`.

**`fatal: 'libs/shared-lib' already exists in the index`** — you ran `add` while the path is still staged.
`git rm --cached libs/shared-lib` (or finish the removal) first.

**`fatal: pathspec 'libs/shared-lib' did not match any files`** — you are inside the submodule. `cd ..\..`.

**I deleted the submodule folder by accident.** `git status` shows ` D libs/shared-lib`;
`git submodule update --init` restores it instantly and **without cloning** — the repository is still in
`.git\modules`.

**`no submodule mapping found in .gitmodules for path 'x'` / `No url found for submodule path 'x'`.**
Someone committed a nested repository instead of adding a submodule (git warned them with
`warning: adding embedded git repository:`). Fix at the source: `git rm --cached x`, then a real
`git submodule add`.

**Teammate pulls your removal:** `warning: unable to rmdir 'libs/shared-lib': Directory not empty`, then the
path shows up under `Untracked files:` — build output left behind. Delete the folder by hand.

**URL changed upstream.** `.gitmodules` is tracked, `.git\config` is not — after pulling a URL change, run
`git submodule sync` (`Synchronizing submodule url for 'libs/shared-lib'`) to copy it into your local config.
`git submodule set-url <path> <url>` edits `.gitmodules` for everyone. **Relative URLs**
(`url = ../git-submodules-lesson-lib.git`) resolve against the parent's `origin`, so HTTPS and SSH users
share one mapping — the trap is that a **fork** then resolves to the fork's namespace; override locally with
`git config submodule.libs/shared-lib.url <url>` **before** `git submodule update --init`.

**`fatal: Unable to find refs/remotes/origin/<b> revision in submodule path 'libs/shared-lib'`** — a
`branch =` in `.gitmodules` that does not exist upstream; fix it or `git submodule set-branch --default`.

**`Failed to clone 'libs/shared-lib'. Retry scheduled` / `… a second time, aborting`** — usually auth or a
wrong URL. `git submodule update --init --recursive` is idempotent, so fix the cause and re-run it.

**`fatal: transport 'file' not allowed`** — since **2.38.1** git refuses `file://`/path submodules by default
(CVE-2022-39253). For local experiments only: `git -c protocol.file.allow=always submodule update --init`.

**Merge conflict on a gitlink.** `CONFLICT (submodule): Merge conflict in libs/shared-lib`, `git status` shows
`both modified:`, and `git submodule status` shows a `U` with an all-zero SHA. Resolve it *in* the submodule
(check out the commit you want — usually the newer one, or a merge of both), then `git add libs/shared-lib`
in the parent.

**Status suffixes, and their short forms.**

| Long form | `--short` | Meaning |
|---|---|---|
| `(new commits)` | ` M` | checkout is at a different commit than the pin |
| `(modified content)` | ` m` | uncommitted edits to tracked files inside |
| `(untracked content)` | ` ?` | untracked files inside (build output, usually) |

They combine: `(new commits, modified content, untracked content)`. Your parent's `.gitignore` does **not**
apply inside a submodule — the submodule needs its own (this one has it). You can silence the noise per
submodule in `.gitmodules` with `ignore = untracked` (or `dirty`); **never** `ignore = all`, which also hides
pin moves and turns `git add -A` into a loaded gun.

**Other things that surprise people** — `git checkout <old-commit>` and `git reset --hard` leave the submodule
where it is unless you pass `--recurse-submodules` (or set `submodule.recurse=true`); `git stash` ignores
submodule content entirely; `git submodule foreach` needs **single** quotes in PS 5.1
(`git submodule foreach 'echo $sm_path'`, since `$sm_path`/`$name`/`$sha1` are the *shell's* variables — with
double quotes PowerShell expands them to empty first); `git worktree remove` on a tree with submodules needs
`--force`; `git mv` on a submodule path updates `.gitmodules` but leaves the name and `.git\modules` path
unchanged; `git clone --recurse-submodules --shallow-submodules` is the CI-friendly form.

*Not verified on this machine:* GitHub renders a bump as a `shared-lib @ <sha>` link in the PR file list;
`actions/checkout@v4` needs `with: { submodules: recursive }` plus a token or deploy key that can read the
submodule repo (the default `GITHUB_TOKEN` cannot read a *different* private repo); Visual Studio, Rider and
VS Code all show the submodule as a separate repository in their git UI and none of them will update it for
you on branch switch unless `submodule.recurse` is set.

## Appendix B — reset the lesson to its starting state

Both repositories carry a lightweight tag **`lesson-start`** marking the state this lesson expects. Force
`main` back to it. **The push must be run against a clone of each repo** (the tag has to exist locally —
running it in a non-repo folder just gives `fatal: not a git repository`).

> ### ⛔ Destructive — and only ever these three things
> It rewinds `main` on **github.com/tzaczek/git-submodules-lesson** and
> **github.com/tzaczek/git-submodules-lesson-lib**, and it deletes
> `C:\Users\tomas\Repo\Trivago\git-submodules-lab` — including Bob's work. It touches no other repository
> and no other folder. The pushes use `git -C`, so if a clone above fails they cannot fall through onto a
> repo you happen to be standing in.

```powershell
$tmp = "$env:TEMP\reset-submodules-lesson"
Remove-Item -Recurse -Force $tmp -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force $tmp | Out-Null
git clone https://github.com/tzaczek/git-submodules-lesson.git "$tmp\app"
git clone https://github.com/tzaczek/git-submodules-lesson-lib.git "$tmp\lib"
git -C "$tmp\app" push --force origin refs/tags/lesson-start:refs/heads/main
git -C "$tmp\lib" push --force origin refs/tags/lesson-start:refs/heads/main
Set-Location $env:USERPROFILE
Remove-Item -Recurse -Force $tmp
Remove-Item -Recurse -Force 'C:\Users\tomas\Repo\Trivago\git-submodules-lab' -ErrorAction SilentlyContinue
```

```text
To https://github.com/tzaczek/git-submodules-lesson.git
 + <P4>...<start>  lesson-start -> main (forced update)
To https://github.com/tzaczek/git-submodules-lesson-lib.git
 + <C7>...63d40c0  lesson-start -> main (forced update)
```

Two pushes, two pairs of lines. The **left** SHA is whatever your run left behind — `<P4>` and `<C7>` if you
followed the lesson to the end, since Block 7's removal is never pushed. The **right** one is the
`lesson-start` commit, which never changes.

Without `--force` you get `! [rejected] … (non-fast-forward)`. Do **not** use an annotated tag as the source
of such a push — every git server, GitHub included, answers
`! [remote rejected] … (invalid new value provided)`, because `refs/heads/*` must point at a commit and an
annotated tag is a tag object. That is exactly why `lesson-start` is lightweight while `v1.0.0`/`v1.1.0` are
annotated. And never write `lesson-start^{commit}:main` unquoted in PS 5.1 — PowerShell parses `{commit}` as
a script block and rewrites the argument into `-encodedCommand <base64> -inputFormat xml -outputFormat text`,
so git answers ``error: unknown switch `e'``. Single-quote the whole refspec if you ever need that form.

Old clones do not notice a rewound remote (`git pull` says `Already up to date.`) — delete them, or
`git fetch; git reset --hard origin/main`.

## Appendix C — when NOT to use submodules

| Situation | Better tool | Why |
|---|---|---|
| Stable, versioned, reusable library | **NuGet** (nuget.org / GitHub Packages / Azure Artifacts) | transitive dependency resolution, restore caching, no source coupling; a `PackageReference` is one line and every .NET tool understands it |
| Same code, many services, one team, atomic cross-cutting changes | **Monorepo** | one commit changes library + consumers; no N-PR rollout, no pin drift |
| Consumers should not know it is a separate repo | **`git subtree`** | history is merged into the parent; plain `git clone` just works, no extra commands for anyone; the cost is a messier history and awkward upstreaming |
| Third-party code you patch and never upstream | **Vendoring** (copy it in) | simplest possible thing; you already own the maintenance |
| You need a submodule but consumers keep forgetting the commands | still a submodule, plus `submodule.recurse=true`, `push.recurseSubmodules=check` and CI `--recurse-submodules` | the ergonomics problem is configuration, not architecture |

Rule of thumb: **submodules buy you an exact, reviewable source pin, and charge you in ceremony on every
clone, pull and push.** If you are not actively developing the dependency alongside the consumer, that is a
bad trade.

## Appendix D — the one-page cheatsheet

See **[CHEATSHEET.md](CHEATSHEET.md)** in this repository: the four places, the status prefixes and suffixes,
the two directions, ~30 commands grouped by task, the config worth setting, the PowerShell 5.1 traps, a
`Look` helper that prints all four places at once, and the reset one-liner.

## Appendix E — .NET and MSBuild across the submodule boundary

- **Keep the submodule a sibling of your projects** (`src\App\` and `libs\shared-lib\`), never *inside* a
  project folder: SDK-style projects glob `**\*.cs`, so the library's sources would be compiled into the app
  as well. First you get `warning CS0436` — the globbed copy silently shadows the type coming from the
  referenced assembly and the build still *succeeds* — and then, on the next build once the nested project's
  `obj\` exists and gets globbed in too, hard `error CS0579: Duplicate 'System.Reflection.AssemblyTitleAttribute'
  attribute` errors that make no sense until you see why.
- **MSBuild walks up past the git boundary.** `Directory.Build.props`, `Directory.Packages.props` and
  `global.json` are found by walking parent directories — which does not stop at `libs\shared-lib`. So the
  same library builds differently inside the superproject than in its own clone: the parent's
  `TreatWarningsAsErrors` can fail it, and Central Package Management makes its `PackageReference`s fail with
  `NU1008`. Fence it with a `Directory.Build.props` at the library's root (this library ships one) containing
  `<Project><PropertyGroup><ImportDirectoryPackagesProps>false</ImportDirectoryPackagesProps></PropertyGroup></Project>`.
  `global.json` cannot be fenced — the SDK is resolved before any project file is read.
- **Build output inside the submodule** shows up in the parent as `(untracked content)` unless the submodule
  has its own `.gitignore` (this one does). The parent's `.gitignore` does not reach inside it.
- **`dotnet run --project src/App` from the repo root**, not `cd src\App` — the relative
  `ProjectReference ..\..\libs\shared-lib\…` is resolved against the *project* file, but running from the root
  keeps every path in this lesson consistent.
