# Git submodules — a 45-minute hands-on lesson

This repository is the **parent** (superproject) of a self-contained, timed lesson on git submodules.
You clone it twice — once as the author, once as the teammate — and add
[`git-submodules-lesson-lib`](https://github.com/tzaczek/git-submodules-lesson-lib) as a submodule at
`libs/shared-lib`.

## Start here

**[LESSON.md](LESSON.md)** — 45 minutes, nine timed blocks, every **block** with its real expected output.
**[CHEATSHEET.md](CHEATSHEET.md)** — the one-page lookup for afterwards.

Everything was verified end to end on 2026-08-17 against these two repositories: Windows 11,
Windows PowerShell 5.1, git 2.52.0.windows.1, .NET SDK 10.0.101.

## What is in here

```
src\App\App.csproj      console app; ProjectReference -> ..\..\libs\shared-lib\src\SharedLib\SharedLib.csproj
src\App\Program.cs      Console.WriteLine(Greeter.Greet(Environment.UserName));
```

The `ProjectReference` deliberately points at a path that **does not exist yet** — Block 2 starts by running
`dotnet run --project src/App` and watching it fail, then makes it work by adding the submodule. There is no
`libs/` folder and no `.gitmodules` in this repository at the start; you create both.

## The cast

| Role | Clone | Does |
|---|---|---|
| **ALICE** — author | `app-alice` | adds the submodule, pins it to `v1.0.0`, bumps it to `v1.1.0`, and pulls Bob's library change back |
| **BOB** — teammate | `app-bob` | clones, hits the empty folder, pulls a bump, contributes back, removes it |

Lab root: `C:\Users\tomas\Repo\Trivago\git-submodules-lab`.

## Resetting

Both repositories carry a lightweight tag **`lesson-start`** marking the starting state, so you can run the
lesson again. From inside a fresh clone of each:

```powershell
git push --force origin refs/tags/lesson-start:refs/heads/main
```

Full procedure — including deleting the lab folder — is **Appendix B** of LESSON.md.
