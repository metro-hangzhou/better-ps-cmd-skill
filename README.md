# better-ps-cmd-skill

## English

`better-ps-cmd-skill` is a Codex skill for running Windows-side commands from WSL with fewer PowerShell and cmd quoting failures. It standardizes WSL-to-Windows command execution by invoking Git for Windows Bash directly, so agents can run Windows-installed tools such as `git`, `gh`, `npm`, `node`, `cargo`, and project build commands with less path, newline, and nested-quote drift.

Use it when a Codex task needs to operate on a Windows-hosted project from WSL, especially when direct `powershell.exe -Command` or `cmd.exe /c` calls are likely to break because of quoting or path conversion.

This skill intentionally does not require a wrapper script. The normal pattern is a Git Bash command call:

```bash
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && git status'
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && gh auth status'
```

The `PATH` prefix makes `bash.exe` resolve to Git for Windows Bash instead of the Windows WSL launcher. Use Git Bash drive paths, such as `/c/projects/my-app`, inside the `-lc` command.

## 中文

`better-ps-cmd-skill` 是一个 Codex skill，用来显著降低从 WSL 侧调用 Windows 侧命令时常见的 PowerShell/cmd 转义、路径转换、换行和嵌套引号问题。它不额外引入包装脚本，而是直接调用 Git for Windows Bash，让 agent 更稳定地调用 Windows 侧安装的 `git`、`gh`、`npm`、`node`、`cargo` 以及项目构建命令。

当 Codex 需要从 WSL 操作 Windows 侧项目，尤其是直接使用 `powershell.exe -Command` 或 `cmd.exe /c` 容易因为引号和路径转译失败时，优先使用这个 skill。

常用模式是通过 `bash.exe` 命令进入 Git Bash：

```bash
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && git status'
PATH="/mnt/c/Program Files/Git/bin:$PATH" bash.exe -lc 'cd /c/projects/my-app && gh auth status'
```

前面的 `PATH` 前缀用于确保 `bash.exe` 解析到 Git for Windows Bash，而不是 Windows 自带的 WSL launcher。在 `-lc` 命令内部使用 Git Bash 路径，例如 `/c/projects/my-app`。

## License

MIT
