# better-ps-cmd-skill

## English

`better-ps-cmd-skill` is a Codex skill for running Windows-side commands from WSL with fewer PowerShell/cmd quoting failures. It standardizes WSL-to-Windows command execution by explicitly using the `bash.exe` that ships with Git for Windows, so agents can run Windows-installed tools such as `git`, `gh`, `npm`, `node`, `cargo`, and project build commands with less path, newline, and nested-quote drift.

Use it when a Codex task needs to operate on a Windows-hosted project from WSL, especially when direct `powershell.exe -Command`, `cmd.exe /c`, or bare `bash.exe` calls are likely to break because of quoting, path conversion, or resolving to the wrong executable.

The intended command shape is:

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && git status'
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && gh auth status'
```

Use Git Bash drive paths, such as `/c/projects/my-app`, inside the `-lc` command. Do not use bare `bash.exe`; on many systems it resolves to the WSL launcher and returns `Linux`, not Git Bash. If the explicit Git for Windows `bash.exe` cannot be verified with `uname -s` returning `MINGW` or `MSYS`, the skill should fail fast and ask the user to fix Git for Windows Bash.

When writing Windows scripts for ordinary users, do not require Git Bash. Use native `.cmd`, `.bat`, or `.ps1` code:

```bat
@echo off
setlocal
cd /d "%~dp0"
if not exist "dist" mkdir "dist"
exit /b %ERRORLEVEL%
```

```powershell
$ErrorActionPreference = 'Stop'
Set-Location -LiteralPath $PSScriptRoot
New-Item -ItemType Directory -Force -Path 'dist' | Out-Null
```

If the failure mentions `UtilBindVsockAnyPort`, ask the user to allow elevated execution for the same Git for Windows `bash.exe` command and retry.

## 中文

`better-ps-cmd-skill` 是一个 Codex skill，用来显著降低从 WSL 侧临时调用 Windows 侧命令时常见的 PowerShell/cmd 转义、路径转换、换行和嵌套引号问题。Codex 自己临时执行命令时，它明确使用 Git for Windows 自带的 `bash.exe`，让 agent 更稳定地调用 Windows 侧安装的 `git`、`gh`、`npm`、`node`、`cargo` 以及项目构建命令。

当 Codex 需要从 WSL 操作 Windows 侧项目，尤其是直接使用 `powershell.exe -Command`、`cmd.exe /c` 或裸 `bash.exe` 容易因为引号、路径转译或解析到错误可执行文件而失败时，优先使用这个 skill。

目标命令形态：

```bash
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && git status'
"/mnt/c/Program Files/Git/bin/bash.exe" -lc 'cd /c/projects/my-app && gh auth status'
```

在 `-lc` 命令内部使用 Git Bash 路径，例如 `/c/projects/my-app`。不要使用裸 `bash.exe`；很多系统上它会解析到 WSL launcher，输出 `Linux`，而不是进入 Git Bash。如果明确的 Git for Windows `bash.exe` 无法通过 `uname -s` 返回 `MINGW` 或 `MSYS` 来验证，skill 应该直接失败并提示用户修复 Git for Windows Bash。

当编写给普通用户运行的 Windows 脚本时，不要依赖 Git Bash。应写原生 `.cmd`、`.bat` 或 `.ps1`：

```bat
@echo off
setlocal
cd /d "%~dp0"
if not exist "dist" mkdir "dist"
exit /b %ERRORLEVEL%
```

```powershell
$ErrorActionPreference = 'Stop'
Set-Location -LiteralPath $PSScriptRoot
New-Item -ItemType Directory -Force -Path 'dist' | Out-Null
```

如果失败信息包含 `UtilBindVsockAnyPort`，应提示用户允许同一条 Git for Windows `bash.exe` 命令提权执行后重试。

## License

MIT
