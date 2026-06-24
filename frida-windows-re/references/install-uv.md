# Installing frida-tools with uv (Windows)

Two PyPI packages, two roles. `frida-tools` is the CLI suite (`frida`, `frida-ps`, `frida-trace`, …). `frida` is the native core + Python binding you `import frida`. Installing `frida-tools` pulls `frida` automatically. The importable module is `frida_tools`; the binding module is `frida`.

| Package | Provides | Pulled by |
|---|---|---|
| `frida-tools` | CLI commands (`frida_tools`) | you install it |
| `frida` | native core wheel + `import frida` | dependency of `frida-tools` (`frida >= 17.10.0, < 18.0.0`) |

## Why uv

- One isolated venv per tool — frida-tools and its deps never collide with project packages or system Python.
- uv can supply the Python interpreter itself (`--python 3.12`) — no system Python needed.
- Fast wheel resolution; `uv pip` is a drop-in replacement for `pip` inside a venv.
- `uvx` runs the tools ephemerally with zero persistent install.

## Quick install (recommended)

```powershell
uv tool install frida-tools
```

Creates a dedicated isolated venv, installs `frida-tools` + deps (`frida`, `colorama`, `prompt-toolkit`, `pygments`, `websockets`), and drops `.exe` shims for every `frida-*` command into uv's tool bin directory (`%USERPROFILE%\.local\bin`, i.e. `C:\Users\<you>\.local\bin`).

If that bin dir isn't on PATH, uv prints a warning. Fix it:

```powershell
uv tool update-shell      # adds the tool bin dir to PATH (persistent)
# then OPEN A NEW TERMINAL -- PATH changes don't apply to the current session
```

Verify or override the bin location:

```powershell
uv tool dir --bin                 # prints the exact bin dir uv uses
$env:UV_TOOL_BIN_DIR              # env var that overrides it
```

One-shot PATH fix for the current session only (no new terminal):

```powershell
$env:PATH = "$env:USERPROFILE\.local\bin;$env:PATH"
```

Pick the interpreter for the tool's venv:

```powershell
uv tool install --python 3.12 frida-tools
```

## One-off runs (no install) — `uvx`

`uvx` is exactly `uv tool run`: it builds (or reuses a cached) ephemeral env and runs a command without a persistent install.

**Gotcha: the command name differs from the package name.** A bare `uvx frida-ps` tries to install a package literally named `frida-ps` (does not exist). Always use `--from frida-tools` and then name the `frida-*` binary:

```powershell
uvx --from frida-tools frida-ps
uvx --from frida-tools frida --version
uvx --from frida-tools frida-trace -n notepad.exe
```

Long form:

```powershell
uv tool run --from frida-tools frida-ps
```

Pin a version ephemerally, or add a matching core:

```powershell
uvx --from "frida-tools==14.10.2" frida-ps
uvx --from frida-tools --with "frida==17.15.3" frida-ps
```

(`uvx frida` happens to work only because a command named `frida` matches a package named `frida` — don't rely on that for the tools; use `--from frida-tools`.)

## Project / venv install — for `import frida`

When you also write Python scripts that `import frida`, install into a project venv:

```powershell
uv venv                                  # creates .venv with a uv-managed Python
.\.venv\Scripts\Activate.ps1             # PowerShell activation
uv pip install frida-tools
```

Without activating, target the venv's interpreter explicitly:

```powershell
uv venv
uv pip install --python .\.venv\Scripts\python.exe frida-tools
```

- Choose the Python version: `uv venv --python 3.12`
- The `frida-*` shims land in `.venv\Scripts\*.exe` and are on PATH only while the venv is active.
- Verify inside the venv:
  ```powershell
  frida --version
  python -c "import frida; print(frida.__version__)"
  ```

## Pinning versions and the same-major-version rule

Pin the tools package (and optionally the core, a separate package with its own version line):

```powershell
uv tool install "frida-tools==14.10.2"
uv tool install "frida-tools>=14,<15"
uv tool install "frida-tools==14.10.2" --with "frida==17.15.3"
uvx --from "frida-tools==14.10.2" frida-ps
```

**The matching rule:** `frida-tools` (CLI) depends on `frida` (core binding + embedded agent); the `frida >= 17.10.0, < 18.0.0` constraint in `frida-tools` enforces their pairing, so installing frida-tools pulls a compatible `frida` core automatically.

For **remote / USB targets** the `frida-server` running on the target — and `frida-gadget` when embedded in an app — **must be the same Frida version as your host-side `frida` core**. Frida's wire protocol is not guaranteed stable across versions; a mismatch typically fails to connect. Check the host version and match the server/gadget download from Frida's GitHub releases:

```powershell
frida --version          # must equal the frida-server / frida-gadget you deploy
```

This server-match constraint does **not** apply to local Windows instrumentation (attaching to local `notepad.exe`-style processes) — the agent ships inside the `frida` core wheel; no separate server is needed.

| Component | Must version-match `frida` core? |
|---|---|
| `frida-tools` (CLI) | Yes — enforced by the `< 18.0.0` pin |
| `frida-server` (remote/USB) | Yes — exact version |
| `frida-gadget` (embedded) | Yes — exact version |
| Local Windows attach/spawn | N/A — agent is in the core wheel |

## Windows prerequisites

- **Python**: latest 3.x recommended. Frida core wheels are `cp37-abi3`, so **Python 3.7+** works (one abi3 wheel covers all 3.x). uv can supply the interpreter via `--python 3.x` even with no system Python installed.
- **No compiler needed**: `frida` core ships prebuilt Windows wheels — `win_amd64` (x64), `win32` (x86), `win_arm64` (ARM64), all `cp37-abi3`. uv downloads the matching wheel; you do **not** need Visual Studio / MSVC build tools. If installation tries to *build from sdist*, that means no matching wheel was found for your Python/arch — fix the interpreter/arch rather than installing a compiler.
- **VC++ runtime**: the native payload generally relies on the Microsoft Visual C++ Redistributable. Windows 10/11 usually already has it; if `frida`/`frida-ps` fails with a native DLL-load error, install the latest **Microsoft Visual C++ Redistributable (x64)**.
- **Architecture**: prefer a **64-bit Python** on x64 Windows — its 64-bit core can inject into both 32- and 64-bit local targets.

## Every console command the package installs

`uv tool install frida-tools` exposes all of these as `.exe` shims (17 commands):

| Command | Purpose |
|---|---|
| `frida` | Interactive REPL — inject a JS agent, live-load scripts, eval code, magic commands. |
| `frida-ps` | List processes (or apps) on a device. |
| `frida-trace` | Dynamically trace function calls; auto-generates editable JS handlers + live Web UI. |
| `frida-discover` | Use Stalker to discover hot/internal functions being called (call-count profiling). |
| `frida-kill` | Kill a process on a device by name or PID. |
| `frida-ls-devices` | List all connected Frida devices (Id, Type, Name, OS). |
| `frida-ls` | List files on a device's filesystem. |
| `frida-itrace` | Instruction-level tracer; follows a thread/range via Stalker into a binary `.itrace` file. |
| `frida-strace` | System-call tracer. |
| `frida-join` | Make a target process join a Frida Portal (control plane). |
| `frida-create` | Scaffold a new Frida project (TypeScript agent or C CModule). |
| `frida-compile` | Compile/bundle a Frida agent (TypeScript/JS) into a single script. |
| `frida-rm` | Remove files/directories on a device (`-r`, `-f`). |
| `frida-pull` | Pull remote file(s) from a device to the local machine. |
| `frida-push` | Push local file(s) to a device. |
| `frida-apk` | Make an Android APK debuggable and optionally inject a gadget. |
| `frida-pm` | Frida package manager — search/install npm-style Frida packages (bridges, etc.). |

## Verify the install

```powershell
frida --version            # prints the Frida core version, e.g. 17.15.3
frida-ps                   # lists local processes -> proves the native core loaded OK
python -c "import frida; print(frida.__version__)"
uv tool list               # confirm frida-tools + its exposed commands (uv install path)
```

Quick local smoke test:

```powershell
Start-Process notepad.exe
frida-ps | Select-String notepad
frida-trace -n notepad.exe -i "CreateFileW"
```

If `frida --version` errors with a DLL-load failure, that's the VC++ runtime / wrong-arch-wheel symptom — install the latest VC++ Redistributable (x64) and confirm you're on a supported interpreter/arch.

## Upgrading and uninstalling

```powershell
uv tool upgrade frida-tools      # upgrade within existing constraints
uv tool upgrade --all            # upgrade every uv-installed tool
uv tool list                     # show installed tools + exposed commands
uv tool uninstall frida-tools    # remove the tool and its shims
```

For a venv install instead, upgrade in place:

```powershell
uv pip install --upgrade frida-tools
```

### Gotchas recap

1. `uvx frida-ps` alone fails — use `uvx --from frida-tools frida-ps` (command name ≠ package name).
2. `frida-server` / `frida-gadget` must equal your `frida` core version for remote/USB; local Windows attach needs no server.
3. After `uv tool update-shell`, **open a new terminal**; bin dir is `%USERPROFILE%\.local\bin` (verify via `uv tool dir --bin`).
4. No compiler on Windows — abi3 wheels for amd64/win32/arm64; an sdist build attempt means an unsupported interpreter/arch.