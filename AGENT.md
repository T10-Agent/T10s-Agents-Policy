# AGENT.md — T10's Policy (version 1)

## Purpose

This repository uses an AI coding agent (e.g., Claude Sonnet 4) to assist with development. To protect safety, data, and systems, the agent MUST follow the rules below. These rules are language-agnostic and apply regardless of framework, package manager, or runtime.

## Scope

* **Workspace:** The workspace root and its subdirectories.
* **Project/Repo:** The repository and its subdirectories.
* **Execution:** Any action that runs user code, spawns services, or produces side effects outside the Project/Repo.
* **Agent:** Applies to Claude Sonnet 4 (and any other agent/tooling) when acting on project(s).

---

## Configuration (edit me)

> The agent MUST read and honor this configuration before proposing commands or emitting a **Run Plan**. If values are missing, the agent must infer sensible defaults using the **Detection & Fallbacks** below, but prefer the explicit settings here.

```yaml
agent:
  os: windows            # one of: windows | linux | macos
  shell: cmd             # one of: cmd | powershell | bash | zsh | fish
  terminal: vscode       # optional: vscode | system | wsl | git-bash | cygwin | msys2
  path_style: windows    # windows (\\) | posix (/)
  use_wsl: false         # if true, treat shell as bash in WSL regardless of OS
```

### Command Abstractions (agent MUST use)

When emitting shell commands (including the **Run Plan**), the agent MUST map these abstract operations to the concrete command for the configured `os`+`shell` using the **Adapter Matrix** below. If a path may contain spaces, always quote it.

**Abstract ops**

* `MKDIR_RECURSIVE <dir>` – create directory, creating parents if needed
* `RM_RECURSIVE_FORCE <path>` – delete file/dir tree without prompts
* `COPY_RECURSIVE <src> <dst>` – copy files/dirs recursively, overwrite
* `MOVE_FORCE <src> <dst>` – move/rename, overwrite
* `LIST_ALL [dir]` – list files including hidden
* `SET_ENV <NAME> <VALUE>` – set env var for current process

### Adapter Matrix (concrete commands)

| Abstract                     | bash/zsh/fish (Linux/macOS) | PowerShell                                               | cmd.exe                                                     |
| ---------------------------- | --------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| `MKDIR_RECURSIVE <dir>`      | `mkdir -p "<dir>"`          | `New-Item -ItemType Directory -Force -Path "<dir>"`      | `mkdir "<dir>"`                                             |
| `RM_RECURSIVE_FORCE <path>`  | `rm -rf "<path>"`           | `Remove-Item "<path>" -Recurse -Force`                   | `rmdir /s /q "<path>"` (dirs), `del /f /q "<path>"` (files) |
| `COPY_RECURSIVE <src> <dst>` | `cp -r "<src>" "<dst>"`     | `Copy-Item "<src>" -Destination "<dst>" -Recurse -Force` | `xcopy "<src>" "<dst>" /E /I /Y`                            |
| `MOVE_FORCE <src> <dst>`     | `mv -f "<src>" "<dst>"`     | `Move-Item "<src>" "<dst>" -Force`                       | `move /Y "<src>" "<dst>"`                                   |
| `LIST_ALL [dir]`             | `ls -la ["<dir>"]`          | `Get-ChildItem -Force ["<dir>"]`                         | `dir ["<dir>"] /a`                                          |
| `SET_ENV <NAME> <VALUE>`     | `export <NAME>="<VALUE>"`   | `$env:<NAME> = "<VALUE>"`                                | `set <NAME>=<VALUE>`                                        |

> **Windows note (cmd.exe):** Do **not** use POSIX-only flags like `-p`. `mkdir` on `cmd.exe` creates intermediate directories automatically. The agent MUST choose `cmd`-appropriate commands when `shell: cmd` is configured.

### Detection & Fallbacks

If `agent.os` or `agent.shell` are omitted:

1. If `use_wsl: true` → treat as `linux + bash`.
2. Else if `os` detected as `windows`:

   * Prefer `powershell` if commands include PowerShell-specific syntax in the repo scripts; otherwise default to `cmd`.
3. Else default to `bash` on `linux`/`macos`.

When unsure, the agent MUST output commands using the safest cross-platform choice (e.g., `npm run <script>` rather than invoking a binary directly), and avoid POSIX flags on Windows shells.

---

### Shell-Aware Command Rules (mandatory)

* The agent MUST consult **Configuration** and the **Adapter Matrix** before emitting any command.
* The agent MUST NOT assume a POSIX shell. If `shell: cmd`, avoid POSIX flags (e.g., `-p`) and prefer Windows-native builtins.
* In `Run Plan` blocks, commands MUST already be translated to the configured shell (no abstract placeholders).
* For npm/pnpm/yarn scripts, prefer script invocations over direct binaries to leverage per-OS script shims.

---

## No-Execute Policy (language-agnostic)

You may build / compile / lint / format / generate docs **without approval** in order to surface errors. **Do not execute programs without explicit approval.**

### Allowed without approval

**Non-executing operations:** build/compile, lint/static analysis, format, doc generation, dependency resolution/lockfile updates.

**Unit test execution for this repository** (e.g., `cargo test`, `go test`, `npm test`, `pytest`). Tests must run locally with **no external network/services** and **no destructive side effects** outside the Project/Repo. If tests require network/services or privileged resources, ask for approval.

**Test compilation-only modes** (e.g., `cargo test --no-run`).

### Requires approval (execution of user code)

Any command that runs user code or starts a service, including for example:

* Run/serve/start/exec/bench commands (e.g., `cargo run`, `cargo bench`; `npm run start`; `pnpm dev`; `yarn dev`; `dotnet run`; `go run`; `java -jar …`; `deno run`).
* Launching interpreters with a script (e.g., `python script.py`, `node app.js`).
* Executing built binaries directly (e.g., from `target/`, `bin/`, `dist/`).
* Starting servers, long-running watchers that execute user code, containers/VMs/DBs.

### How to get approval (mandatory, agent-generated code)

When execution is requested, **do not run**. Instead:

* **Output a Run Plan** (command, working directory, expected side effects, stop method, logging path).

* **Generate a random 6-digit numeric code** and include it as the final bullet in the Run Plan in this exact format:

  `Approval Code: 483920`

* **Do not include any additional instruction text** (no “Please reply with …”).

* **Wait for a user message** that contains a valid approval (see rules below). Then and only then, execute.

### Rules for approval codes

* **One-time use**, valid for **10 minutes** or **until the next user message**—whichever comes first.
* **Acceptance:** either a line containing **only** the 6 digits (whitespace allowed), or the **exact phrase** `APPROVE RUN: <code>`.
* Digits must match the **last issued code**. Do not accept codes embedded in sentences or older codes.
* Up to **3 mismatches** allowed; on mismatch/expiration: **abort** and **re-issue a new code** with the Run Plan.
* **Record** the issued code and the user’s confirmation in `./_agentlog/`.

**Scope note:** Approval codes apply only to the No-Execute Policy above.

## Run Plan (template to output verbatim when execution is requested)

* **Command:** `<exact command to run>`
* **Working Directory:** `<absolute or repo/project-relative path>`
* **Expected Side Effects:** `<files created/modified, ports opened, env vars, resource usage>`
* **Stop Method:** `<ctrl-c, kill PID, docker stop, test timeout, etc.>`
* **Logging Path:** `./_runlogs/<YYYYMMDD-HHMMSS>.log`
* **Approval Code:** `123456`

> The agent must emit the bullets above exactly, with the 6-digit **Approval Code** as the final bullet. No extra prose.
