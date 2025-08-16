# T10 Agents Policy (Version 1.0.0)

**A language‑agnostic, cross‑platform `AGENT.md` policy for safer AI coding agents**

[![Policy](https://img.shields.io/badge/policy-no--execute-blue)](#no-execute-policy-language-agnostic)
[![Language Agnostic](https://img.shields.io/badge/language-agnostic-success)](#)
[![Security Warning](https://img.shields.io/badge/SECURITY-Review%20AGENT.md%20before%20use-critical)](#security-warning)

> Drop this into any **repo or local workspace** to standardize how AI agents (Claude, GPT, etc.) behave when proposing commands or automations. Works across Windows/macOS/Linux, any runtime or package manager. No Git host required.

> 🚨 **General Warning:** Treat any `AGENT.md` as executable policy. Verify source, read it fully, pin to a commit, and test in a sandbox. Beware of auto-run, networked tests, data exfiltration, Git hook changes, and destructive deletes.

---

## ✨ What is this?

`T10 Agents Policy` is a small, public specification (the `AGENT.md` file) you can add to your repositories **or local workspaces/codebases**. It defines:

* A consistent agent configuration (OS, shell, path style, terminal)
* A mandatory **Adapter Matrix** that maps abstract operations → concrete commands per shell
* Cross‑repo **No‑Execute** safety rules
* A strict **Approval** workflow with one‑time codes before any execution of user code
* A copy‑paste **Run Plan** template agents must output before running anything

The result: agents stay deterministic, shell‑aware, and safe—without you having to rewrite guidance per project.

---

## 🚀 Quick start

1. **Copy** [`AGENT.md`](./AGENT.md) to the root of your repo **or** local workspace.
2. **Edit** the `Configuration (edit me)` block at the top to match your environment:

   ```yaml
   agent:
     os: windows            # windows | linux | macos
     shell: cmd             # cmd | powershell | bash | zsh | fish
     terminal: vscode       # optional: vscode | system | wsl | git-bash | cygwin | msys2
     path_style: windows    # windows (\\) | posix (/)
     use_wsl: false         # if true, treat shell as bash in WSL regardless of OS
   ```
3. **If you use version control, commit it.** Otherwise just keep `AGENT.md` in your workspace root. Agents should read and honor `AGENT.md` automatically when acting on your repo or local workspace.
4. (Optional) Add **guardrails** (local hook or any CI) from the section below.

---

## 🧭 How agents should use this

Agents MUST:

* Read `AGENT.md` **before** proposing commands or emitting a **Run Plan**
* Translate abstract ops to concrete commands using the **Adapter Matrix** for the configured `os`+`shell`
* Prefer script shims (e.g., `npm run`) over direct binaries to stay cross‑platform
* Follow the **No‑Execute Policy** and **Approval Code** workflow below

Recommended snippets to paste into your agent setup or task description:

> "This workspace (repo or local) includes `AGENT.md`. You MUST read and follow it."

> "This workspace (repo or local) includes `AGENT.md`. You MUST read and follow it. When proposing commands, use the configured `os`/`shell`, map abstract ops via the Adapter Matrix, and do not execute user code without an approved Run Plan and 6‑digit code."

---

## 🧩 Adapter Matrix (snapshot)

**Abstract ops**

* `MKDIR_RECURSIVE <dir>` – create directory, creating parents if needed
* `RM_RECURSIVE_FORCE <path>` – delete file/dir tree without prompts
* `COPY_RECURSIVE <src> <dst>` – copy files/dirs recursively, overwrite
* `MOVE_FORCE <src> <dst>` – move/rename, overwrite
* `LIST_ALL [dir]` – list files including hidden
* `SET_ENV <NAME> <VALUE>` – set env var for current process

**Concrete commands** (see `AGENT.md` for the full table and notes)

| Abstract                     | bash/zsh/fish (Linux/macOS) | PowerShell                                               | cmd.exe                                                     |
| ---------------------------- | --------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| `MKDIR_RECURSIVE <dir>`      | `mkdir -p "<dir>"`          | `New-Item -ItemType Directory -Force -Path "<dir>"`      | `mkdir "<dir>"`                                             |
| `RM_RECURSIVE_FORCE <path>`  | `rm -rf "<path>"`           | `Remove-Item "<path>" -Recurse -Force`                   | `rmdir /s /q "<path>"` (dirs), `del /f /q "<path>"` (files) |
| `COPY_RECURSIVE <src> <dst>` | `cp -r "<src>" "<dst>"`     | `Copy-Item "<src>" -Destination "<dst>" -Recurse -Force` | `xcopy "<src>" "<dst>" /E /I /Y`                            |
| `MOVE_FORCE <src> <dst>`     | `mv -f "<src>" "<dst>"`     | `Move-Item "<src>" "<dst>" -Force`                       | `move /Y "<src>" "<dst>"`                                   |
| `LIST_ALL [dir]`             | `ls -la ["<dir>"]`          | `Get-ChildItem -Force ["<dir>"]`                         | `dir ["<dir>"] /a`                                          |
| `SET_ENV <NAME> <VALUE>`     | `export <NAME>="<VALUE>"`   | `$env:<NAME> = "<VALUE>"`                                | `set <NAME>=<VALUE>`                                        |

> **Windows note:** For `cmd.exe` do **not** use POSIX flags like `-p`. `mkdir` already creates intermediate directories.

---

## 🛡️ No‑Execute Policy (language‑agnostic)

You may build/compile/lint/format/generate docs **without approval** to surface errors.

**Allowed without approval**

* Build/compile, lint/static analysis, format, docs
* Dependency resolution/lockfile updates
* **Unit tests** that run locally with **no external services/network** and no destructive side effects (or compilation‑only modes like `cargo test --no-run`)

**Requires approval (execution of user code)**

* Any `run/serve/start/exec/bench` command (e.g., `npm run start`, `cargo run`, `go run`, `dotnet run`)
* Launching interpreters with project scripts (`python script.py`, `node app.js`)
* Executing built binaries from `target/`, `bin/`, `dist/`
* Starting servers, watchers, containers/VMs/DBs

When execution is requested, **do not run**. Instead, emit a **Run Plan** and wait for approval.

---

## ✅ Approval workflow

Agents must output the **Run Plan** template **verbatim** (with a fresh random code):

```
* **Command:** `<exact command to run>`
* **Working Directory:** `<absolute or repo/project-relative path>`
* **Expected Side Effects:** `<files created/modified, ports opened, env vars, resource usage>`
* **Stop Method:** `<ctrl-c, kill PID, docker stop, test timeout, etc.>`
* **Logging Path:** `./_runlogs/<YYYYMMDD-HHMMSS>.log`
* **Approval Code:** `123456`
```

**Rules for approval codes**

* One‑time use, valid for **10 minutes** or **until the next user message** (whichever comes first)
* **Acceptance formats:** a line containing **only** the 6 digits (whitespace allowed), or the exact phrase `APPROVE RUN: <code>`
* Digits must match the **last issued code**
* Up to **3 mismatches** allowed; on mismatch/expiration: abort and re‑issue a new code with the Run Plan
* **Record** the issued code and the user’s confirmation in `./_agentlog/`

---

## 🛠️ Example usage

**Scenario:** Build and test a Node.js project on Windows with `cmd.exe`.

* Allowed without approval: `npm ci`, `npm run build`, `npm test` (if local, no network/services)
* Requires approval: `npm run start`

**Run Plan (example):**

```
* **Command:** npm run start
* **Working Directory:** .\
* **Expected Side Effects:** starts local dev server on port 5173; writes logs to .\_runlogs; reads .env.local
* **Stop Method:** ctrl-c in the terminal
* **Logging Path:** .\_runlogs\20250115-141230.log
* **Approval Code:** 829461
```

---

## 🏠 Local machine usage (no repo required)

You can use `AGENT.md` in a plain folder on an end‑user PC without any Git hosting or CI:

1. Place `AGENT.md` in the **workspace root**.
2. Ensure your agent is pointed at that folder. Most agents read repo/workspace files automatically.
3. The policy’s paths (e.g., `./_agentlog/`, `./_runlogs/`) are **relative to the workspace root** and will be created on demand.
4. Use the **Adapter Matrix** to keep commands shell‑correct on that machine (Windows `cmd`, PowerShell, bash, etc.).

---

## 🔧 Guardrails (local or any CI) to require `AGENT.md`

Use one of these lightweight checks to ensure `AGENT.md` exists and is versioned.

**Bash (macOS/Linux)**

```bash
# scripts/check-agent-policy.sh
set -euo pipefail
test -f AGENT.md
grep -E "^# AGENT.md — T10 Agents Policy \(version [0-9]+\)" AGENT.md
```

**PowerShell (Windows)**

```powershell
# scripts/check-agent-policy.ps1
$ErrorActionPreference = "Stop"
if (-not (Test-Path -Path "AGENT.md")) { throw "AGENT.md missing" }
Select-String -Path "AGENT.md" -Pattern "^# AGENT.md — T10 Agents Policy \(version [0-9]+\)" | Out-Null
```

**Pre-commit hook (any VCS, optional)**

```bash
# .git/hooks/pre-commit
#!/usr/bin/env bash
set -e
test -f AGENT.md
grep -Eq "^# AGENT.md — T10 Agents Policy \(version [0-9]+\)" AGENT.md || { echo "AGENT.md missing or malformed"; exit 1; }
```

**Bring-your-own CI**
Run one of the scripts above in your CI of choice (GitLab CI, Jenkins, Azure Pipelines, etc.).

---

## 📁 Suggested workspace layout

```
/
├─ AGENT.md               # The policy (required)
├─ README.md              # This file
├─ LICENSE                # CC BY-NC 4.0 notice
├─ _agentlog/             # Agent approval ledger (created on demand)
├─ _runlogs/              # Runtime logs from approved runs (created on demand)
└─ scripts/               # Optional local guard scripts
```

---

## 🔄 Versioning

* The policy is versioned inside `AGENT.md` header, e.g., `version 1.0.0`.
* Use SemVer for repository tags (`v1.0.0`).
* If you change semantics, bump the policy version and summarize changes in **Changelog**.

---

## 🧪 Compatibility

The policy is agent‑agnostic. It has been designed to work with tools that can read repo files and follow instructions, including (non‑exhaustive): Claude, GPT‑based agents, local LLM assistants, and CI “bots.”

---

## ❓ FAQ

**Why a separate `AGENT.md` instead of more prompts?**
It lives with the code. If you use a repo, it’s reviewable via PRs; if you’re local‑only, you can still enforce it with a simple script or pre‑commit hook. Prompts drift; files persist.

**Will this break on Windows?**
No—the Adapter Matrix drives concrete commands for `cmd.exe` and PowerShell. The Windows note forbids POSIX‑only flags on `cmd.exe`.

**Do I have to use the matrix?**
Yes—agents MUST translate abstract ops to the configured shell before emitting commands.

**Can agents ever execute without approval?**
Only non‑executing actions and strictly local unit tests per the No‑Execute Policy.

---

## 🤝 Contributing

Issues and PRs welcome! Please:

* Keep examples shell‑accurate per the Adapter Matrix
* Add tests or CI checks when changing policy semantics
* Update the **Changelog** and bump the policy version when necessary

---

## 📜 License

**Creative Commons Attribution-NonCommercial 4.0 International License (CC BY-NC 4.0)**

Copyright (c) 2025 T10 Agents Policy Authors

This work is licensed under the Creative Commons Attribution-NonCommercial 4.0 International License. To view a copy of this license, visit [http://creativecommons.org/licenses/by-nc/4.0/](http://creativecommons.org/licenses/by-nc/4.0/) or send a letter to Creative Commons, PO Box 1866, Mountain View, CA 94042, USA.

**You are free to:**

* **Share:** copy and redistribute the material in any medium or format
* **Adapt:** remix, transform, and build upon the material

**Under the following terms:**

* **Attribution:** You must give appropriate credit to the T10 Agents Policy Authors, provide a link to the license, and indicate if changes were made. You may do so in any reasonable manner, but not in any way that suggests the licensor endorses you or your use.
* **NonCommercial:** You may not use the material for commercial purposes.
* **No additional restrictions:** You may not apply legal terms or technological measures that legally restrict others from doing anything the license permits.

The full license text is available at: [http://creativecommons.org/licenses/by-nc/4.0/legalcode](http://creativecommons.org/licenses/by-nc/4.0/legalcode)

---

## 🗒️ Changelog

See [CHANGELOG.md](./CHANGELOG.md) (create it on first release).

---

## 🏷️ Badge (optional)

Add this to your README or docs to signal adoption:

```md
![Agent Policy: T10 Agents Policy](https://img.shields.io/badge/agent--policy-T10--Agents--Policy-2BB869)
```

---

**Thanks for using T10 Agents Policy!** If you adopt it, consider opening an issue with feedback or examples so others can learn from your setup.
