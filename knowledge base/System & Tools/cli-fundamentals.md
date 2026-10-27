# CLI Fundamentals

## What Is CLI?
CLI (Command Line Interface) is a **universal, platform-agnostic** approach — not tied to any single framework or platform.
It's a general way of interacting with systems via text commands, applicable across:
- Operating systems (Linux, macOS, Windows)
- Languages and frameworks (Python, Node.js, Go, Java, etc.)
- Cloud platforms (AWS, Azure, GCP)
- Containerization (Docker, Kubernetes)
- Version control (Git), package managers (npm, pip, apt, etc.)

The fundamentals — pipes, streams, exit codes, environment variables — are **core computing concepts**.
Different platforms have their own CLI, but they all follow similar patterns.

## OS Heritage & CLI Landscape
Unix was created in the 1970s. Modern OS-es relate to it differently:

| OS | Relation to Unix | Default Shell |
|----|-----------------|---------------|
| **Linux** | Unix-inspired (free rewrite, no original code) | bash (extends `sh`) |
| **macOS** | Based on Darwin/BSD (evolved from Unix) | zsh (bash-compatible) |
| **Windows** | Built independently from DOS — no Unix relation | cmd.exe / PowerShell |

Because Linux and macOS share Unix heritage, they use the same CLI philosophy and most tools.
Windows went its own way — hence different command syntax and paradigms.

### Windows CLI options
Windows has several CLI environments — they are **not** the same:

| CLI | Color | Notes |
|-----|-------|-------|
| `cmd.exe` | black | Old Command Prompt, limited syntax, Windows-specific |
| **PowerShell** | blue | Modern, object-based pipeline, Windows-native but powerful |
| **WSL** | black/custom | Windows Subsystem for Linux — runs actual Linux bash |
| **Git Bash** | black | Bash emulation via MinGW, installed with Git |

- `cmd.exe` is its own thing — **not** bash, **not** PowerShell
- Linux equivalent of `cmd.exe` is `sh` (original basic shell); bash extends `sh`
- Bash is **not** built into Windows — you need WSL or Git Bash to get it

### Shell progression summary
- **Windows**: `cmd.exe` (basic) → PowerShell (advanced)
- **Linux/Unix**: `sh` (basic) → bash (advanced)

## Why CLI Still Matters
- Scriptable & reproducible (CI/CD, infra-as-code)
- Faster than GUI for power users
- Available everywhere (SSH, containers, servers without desktop)
- Composable via pipes — small tools doing one thing well (Unix philosophy)

## Anatomy of a Command
```
git commit -m "fix bug" --amend
└─┬─┘ └──┬─┘ └──┬─┘ └──────┬──────┘ └──┬──┘
 cmd  subcmd   short opt    arg     long opt
```

- **Short options**: `-m`, often combinable (`-la` = `-l -a`)
- **Long options**: `--message`, more readable
- **Arguments**: positional, order matters
- **`--`**: end-of-options marker (`rm -- -filename`)

## Streams & Redirection (POSIX & PowerShell)
- `stdin` (0), `stdout` (1), `stderr` (2)
- `>`  redirect stdout (overwrite), `>>` append
- `2>` redirect stderr; `2>&1` merge stderr into stdout
- `|` pipe stdout of one command into stdin of next
- `<` read stdin from file

```bash
build.sh > log.txt 2>&1        # capture all output
grep ERROR log.txt | wc -l     # count error lines
```

## Exit Codes
- `0` = success, non-zero = failure (specifics vary)
- `$?` (bash) / `$LASTEXITCODE` (PowerShell) holds last exit code
- Used by scripts and CI to detect failure
- Chain with `&&` (run if previous succeeded) or `||` (run if previous failed)
  - Note: in PowerShell pre-7, `&&`/`||` are not supported; use `if ($?)`

## Environment Variables
- Configure tools without flags (`PATH`, `EDITOR`, `LANG`, `HTTP_PROXY`)
- Set per-session (`export FOO=bar` / `$env:FOO = "bar"`)
- Persist via shell profile (`~/.bashrc`, `$PROFILE`) or system settings
- Inject secrets/config in containers and CI

## Globs & Quoting
- `*` matches any chars, `?` one char, `[abc]` set
- **Single quotes**: literal, no expansion (`'$HOME'`)
- **Double quotes**: variables expand (`"$HOME"`)
- Quote paths with spaces

## Essential Unix Tools
| Tool | Purpose |
|------|---------|
| `ls` / `Get-ChildItem` | list files |
| `cd` / `Set-Location` | change dir |
| `cat` / `Get-Content` | print file |
| `grep` / `Select-String` | search text |
| `find` / `Get-ChildItem -Recurse` | find files |
| `awk`, `sed` | text processing |
| `cut`, `sort`, `uniq`, `wc` | column/line ops |
| `xargs` | turn stdin into args |
| `curl`, `wget` | HTTP |
| `jq` | parse/query JSON |
| `tar`, `zip` | archive |
| `ssh`, `scp`, `rsync` | remote ops |
| `top`/`htop`, `ps` | processes |
| `df`, `du`, `free` | disk/memory |

## PowerShell Specifics (Windows)
- Cmdlets are `Verb-Noun` (`Get-Process`, `Set-Item`)
- Pipeline passes **objects**, not text — `Get-Process | Where-Object CPU -gt 100`
- Aliases: `ls`, `cat`, `cd` map to cmdlets
- `Get-Help <cmd> -Examples`, `Get-Command`, tab completion

## Writing Reliable Scripts
- Use **strict mode**: `set -euo pipefail` (bash) / `Set-StrictMode -Version Latest` (PS)
- Quote variables to handle spaces / empty values
- Check exit codes; fail fast
- Use `--` to separate flags from filenames that may start with `-`
- Idempotent where possible (rerunnable)
- Use `mktemp` for temp files; clean up with `trap`

## Discoverability
- `<cmd> --help` or `-h`
- `man <cmd>` (Unix), `Get-Help <cmd>` (PowerShell)
- `which <cmd>` / `Get-Command <cmd>` to find executable path
- `<cmd> --version` to confirm what's installed
