---
name: better-ps-cmd-skill
description: "Use when: WSL 运行 Windows 命令、WSL Windows 交叉命令、Git Bash bash.exe 路径问题、powershell.exe 引号转义、cmd.exe 换行符、WSL /mnt/drive-letter 路径转换、.ps1/.bat/.cmd Windows 脚本编写；or when: Codex runs Windows-side commands from WSL; or when: bare bash.exe path/quoting/nesting errors; or when: writing portable Windows scripts (.cmd, .bat, .ps1). Works across agent runtimes: Codex, Claude Code, Cursor, OpenClaw, or any bash-capable agent. Prefers Git for Windows bash.exe over powershell.exe or bare bash.exe for WSL-to-Windows bridges.
---

# better-ps-cmd-skill

## Core Rule

> **适用环境**：适用于任意支持 bash 调用的 agent runtime（Codex / Claude Code / Cursor / OpenClaw 等）。只要能跑 bash，就能用 Git for Windows bash.exe 做 WSL ↔ Windows 桥接。

For Windows-side work launched from WSL, run the `bash.exe` that belongs to Git for Windows. Do not use bare `bash.exe`, because it may resolve to the Windows WSL launcher and run Linux bash instead of Git Bash.

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/path/to/windows-project && git status'
```

Use native WSL bash for Linux-only tasks. For generated Windows scripts, prefer native cmd/PowerShell and do not introduce Git Bash unless the user explicitly requires a developer-only script.

## Quick Decision Tree

```
用户在 WSL 里要跑 Windows 命令？
├── 是 → 用 Git Bash bash.exe
│   ├── 路径 /mnt/c/... → 转换为 /c/...
│   ├── 路径含空格 → 内层命令加单引号
│   └── 目标 Windows 工具 → 放 -lc '' 内
└── 否（纯 Linux 工具）→ 用原生 WSL bash
```

## Decision Flow

1. If the command targets a repo or file under `/mnt/drive-letter/...` and should use Windows-installed tools, run it through Git for Windows `bash.exe -lc`.
2. If the command is pure inspection from WSL (`rg`, `sed`, `find`, reading files), keep it in WSL bash.
3. If creating or editing `.cmd`, `.bat`, or `.ps1`, write portable native Windows script code first. Do not require Git, Git Bash, or WSL for scripts intended for ordinary users.
4. If a temporary Codex command previously failed because of `powershell.exe -Command`, `cmd.exe /c`, bare `bash.exe`, nested quotes, Windows backslashes, or path translation, retry through Git for Windows `bash.exe` before changing the underlying task.
5. If Git for Windows `bash.exe` is unavailable or cannot be verified for temporary Codex command execution, stop and tell the user to expose/fix Git for Windows Bash. Do not auto-search installation paths, prepend PATH, introduce a wrapper, use `git-bash.exe`, or fall back to running `git`/`gh` in PowerShell/cmd.

## Git Bash Command Pattern

> **参数说明**：`-lc` = login shell (`-l`) + execute command (`-c`)；`/c/` = Git Bash 驱动路径格式（不是 `/mnt/c/`，不是 `C:\`）

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

## Edge Cases & Boundary Conditions

| 场景 | 判断条件 | 行为 |
|------|---------|------|
| WSL 路径不存在 | `test -d /mnt/x` 返回非0 | 停止，报「路径不可达」，提示检查 WSL 挂载 |
| 目标路径含空格 | 路径含空白字符 | Git Bash 路径加引号：`'/c/Program Files/...'` |
| Windows 路径含中文 | 路径含非 ASCII 字符 | PowerShell 优先，Git Bash 可能乱码 |
| 多层嵌套引号 | `-lc` 内层已有引号 | 用双引号包 `-lc`，单引号包内层命令 |
| Git Bash 不可用 | 探针返回非 MINGW/MSYS | 按 Failure Handling 规则 fail-fast |
| WSL 无 `/mnt/` 挂载 | `df /mnt/c` 失败 | 告知用户检查 WSL 安装状态 |
| CRLF 文件在 Git Bash 执行 | Windows .bat 含 `\r\n` | Git Bash 报语法错误；执行前用 `sed -i 's/\r$//' file.bat` 或 `unix2dos -a` 转换 |

## Failure Handling

Before doing git or gh work through this skill, probe Git for Windows Bash explicitly:

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'uname -s'
```

Expected output starts with `MINGW` or `MSYS`. If the command is not found, returns `Linux`, produces no usable stdout, or cannot execute, fail fast and tell the user to expose/fix Git for Windows Bash. Do not silently solve the environment problem inside the task.

If the command fails with `UtilBindVsockAnyPort`, treat it as a permissions or sandbox-scope restriction rather than a Git Bash command design problem. Ask the user to allow elevated execution for the same Git for Windows `bash.exe` command, then retry without adding wrappers, PATH fixes, PowerShell/cmd launchers, `git-bash.exe`, or bare `bash.exe`.
