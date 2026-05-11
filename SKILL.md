---
name: better-ps-cmd-skill
description: Use when Codex runs Windows-side commands from WSL or Codex CLI and PowerShell/cmd quoting, path conversion, newline, encoding, or command translation errors are likely, and when creating or editing Windows scripts (.cmd, .bat, .ps1). Prefer this skill for Windows-hosted projects under /mnt/drive-letter paths, npm/node/cargo/python commands that must run with Windows tooling, GitHub/CI probes launched from WSL into Windows, Windows script authoring for ordinary users, and any previous failure involving powershell.exe -Command, cmd.exe /c, bare bash.exe, backslash paths, nested quotes, or WSL-to-Windows argument drift. It uses Git for Windows bash.exe only for Codex's temporary command execution, while Windows script outputs should be native cmd/PowerShell and avoid Git Bash dependencies.
---

# better-ps-cmd-skill

## Core Rule

For Windows-side work launched from WSL, run the `bash.exe` that belongs to Git for Windows. Do not use bare `bash.exe`, because it may resolve to the Windows WSL launcher and run Linux bash instead of Git Bash.

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/path/to/windows-project && git status'
```

Use native WSL bash for Linux-only tasks. For generated Windows scripts, prefer native cmd/PowerShell and do not introduce Git Bash unless the user explicitly requires a developer-only script.

## Decision Flow

1. If the command targets a repo or file under `/mnt/drive-letter/...` and should use Windows-installed tools, run it through Git for Windows `bash.exe -lc`.
2. If the command is pure inspection from WSL (`rg`, `sed`, `find`, reading files), keep it in WSL bash.
3. If creating or editing `.cmd`, `.bat`, or `.ps1`, write portable native Windows script code first. Do not require Git, Git Bash, or WSL for scripts intended for ordinary users.
4. If a temporary Codex command previously failed because of `powershell.exe -Command`, `cmd.exe /c`, bare `bash.exe`, nested quotes, Windows backslashes, or path translation, retry through Git for Windows `bash.exe` before changing the underlying task.
5. If Git for Windows `bash.exe` is unavailable or cannot be verified for temporary Codex command execution, stop and tell the user to expose/fix Git for Windows Bash. Do not auto-search installation paths, prepend PATH, introduce a wrapper, use `git-bash.exe`, or fall back to running `git`/`gh` in PowerShell/cmd.

## Git Bash Command Pattern

Use Git Bash drive paths inside the `-lc` command:

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && npm run build'
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && npx vue-tsc --noEmit'
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && gh auth status'
```

When starting from a WSL path, convert only the working directory boundary:

```bash
# /mnt/c/projects/my-app becomes /c/projects/my-app for Git Bash.
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && git status --short'
```

Use shell operators inside the Git Bash command string only after the `cd`:

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && git status --short && npm run test'
```

## Path And Quoting Rules

- Keep the WSL-to-Windows boundary shallow: WSL bash -> Git for Windows `bash.exe` -> target tool.
- Do not use bare `bash.exe`; it is ambiguous and may be the WSL launcher.
- Do not use `git-bash.exe` for non-interactive command execution by default; use Git for Windows `bin/bash.exe -lc`.
- Do not use PowerShell/cmd as a Git Bash launcher from WSL by default.
- Do not use Windows backslash paths as the project path inside Git Bash; use `/c/projects/app`.
- Avoid mixing `/mnt/c/...`, `/c/...`, and `C:\...` for the same working directory.
- Put all real work (`git`, `gh`, `npm`, builds, tests) inside the Git Bash `-lc` command.

## Windows Script Authoring

When creating or editing `.cmd`, `.bat`, or `.ps1` files, assume the script may run on a normal user's Windows machine with no Git for Windows, no Git Bash, and no WSL. Do not use Git Bash, Unix commands, Git Bash paths, or `bash.exe` inside these scripts unless the user explicitly asks for a developer-only script.

For `.cmd` or `.bat`:

```bat
@echo off
setlocal
cd /d "%~dp0"
rem Use native Windows commands here. Avoid bash, sh, sed, grep, awk, rm, cp, and /c/... paths.
if not exist "dist" mkdir "dist"
exit /b %ERRORLEVEL%
```

For `.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
Set-Location -LiteralPath $PSScriptRoot
# Use native PowerShell here. Avoid bash, sh, sed, grep, awk, rm, cp, and /c/... paths.
New-Item -ItemType Directory -Force -Path 'dist' | Out-Null
exit $LASTEXITCODE
```

If a script needs external tooling, use tools that the script checks for explicitly and reports clearly. Do not make Git Bash an implicit dependency for ordinary Windows users.

## Failure Handling

Before doing git or gh work through this skill, probe Git for Windows Bash explicitly:

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'uname -s'
```

Expected output starts with `MINGW` or `MSYS`. If the command is not found, returns `Linux`, produces no usable stdout, or cannot execute, fail fast and tell the user to expose/fix Git for Windows Bash. Do not silently solve the environment problem inside the task.

If the command fails with `UtilBindVsockAnyPort`, treat it as a permissions or sandbox-scope restriction rather than a Git Bash command design problem. Ask the user to allow elevated execution for the same Git for Windows `bash.exe` command, then retry without adding wrappers, PATH fixes, PowerShell/cmd launchers, `git-bash.exe`, or bare `bash.exe`.
