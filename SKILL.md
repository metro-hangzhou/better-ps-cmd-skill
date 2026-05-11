---
name: better-ps-cmd-skill
description: Use when Codex runs Windows-side commands from WSL or Codex CLI and PowerShell/cmd quoting, path conversion, newline, encoding, or command translation errors are likely. Prefer this skill for Windows-hosted projects under /mnt/drive-letter paths, npm/node/cargo/python commands that must run with Windows tooling, GitHub/CI probes launched from WSL into Windows, and any previous failure involving powershell.exe -Command, cmd.exe /c, backslash paths, nested quotes, or WSL-to-Windows argument drift. It provides a better default than direct PowerShell/cmd by invoking Git for Windows Bash directly whenever practical.
---

# better-ps-cmd-skill

## Core Rule

For Windows-side work launched from WSL, invoke Git for Windows Bash through the `bash.exe` command after ensuring Git for Windows `bin` takes precedence in `PATH`. Do not execute the Git Bash binary by absolute path, and do not add an extra wrapper script unless a task specifically needs a reusable project script.

```bash
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/path/to/windows-project && git status'
```

Use native WSL bash for Linux-only tasks. Use PowerShell or cmd only when the task specifically requires PowerShell cmdlets, `.ps1`, `.bat`, registry APIs, COM, or Windows shell behavior that Git Bash cannot provide.

## Decision Flow

1. If the command targets a repo or file under `/mnt/drive-letter/...` and should use Windows-installed tools, run it through Git for Windows Bash.
2. If the command is pure inspection from WSL (`rg`, `sed`, `find`, `git diff`, reading files), keep it in WSL bash.
3. If a command previously failed because of `powershell.exe -Command`, `cmd.exe /c`, nested quotes, Windows backslashes, or path translation, retry through Git Bash before changing the underlying command.
4. If PowerShell/cmd is unavoidable, keep the shell hop as shallow as possible and prefer invoking a checked-in script file over embedding a long command string.

## Git Bash Command Pattern

Use Git Bash drive paths inside the `-lc` command:

```bash
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && npm run build'
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && npx vue-tsc --noEmit'
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && gh auth status'
```

When starting from a WSL path, convert only the working directory boundary:

```bash
# /mnt/c/projects/my-app becomes /c/projects/my-app for Git Bash.
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && git status --short'
```

Use shell operators inside the Git Bash command string only after the `cd`:

```bash
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && git status --short && npm run test'
```

## Path And Quoting Rules

- Keep the WSL-to-Windows boundary to one shell hop: WSL bash -> Git for Windows Bash -> target tool.
- Put `/mnt/c/Program Files/Git/bin` before Windows System32 in `PATH`; otherwise `bash.exe` may resolve to the Windows WSL launcher instead of Git Bash.
- Use Git Bash paths such as `/c/projects/app`, not Windows backslash paths, inside `bash.exe -lc`.
- Prefer single quotes around the whole `-lc` command when launching from WSL bash.
- Avoid mixing `/mnt/c/...`, `/c/...`, and `C:\...` in the same command string.
- Avoid WSL -> PowerShell -> cmd -> tool chains. They multiply quoting, newline, and encoding failure modes.
- If Git Bash is installed outside the default location, prepend that installation's `bin` directory to `PATH` before running `bash.exe`.

## Failure Handling

If Git Bash is missing, report that clearly and ask whether to install/configure Git for Windows or proceed with a task-specific PowerShell fallback. If a Git Bash command fails, first verify the Git Bash executable path, then check that the `cd` target exists in Git Bash path form and that the intended Windows tool is available inside Git Bash.
