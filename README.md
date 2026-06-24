# frida-windows-skills

An AI Agent **skill** — [`frida-windows-re`](frida-windows-re/SKILL.md) — for
dynamic instrumentation and security analysis of Windows binaries with
[Frida](https://frida.re) and the `frida-tools` CLI. It's a cheatsheet + bundled
reference docs for runtime/dynamic analysis on a Windows process, DLL, or `.exe`:
API hooking and tracing, dumping decrypted buffers / crypto keys, monitoring
file/registry/network activity, anti-debug / anti-Frida bypass, and
product-security / app-pentest work (TLS interception, cert-pinning bypass,
secret/key auditing, license/auth checks, fuzzing).

```
frida-windows-skills/
├── README.md                     ← you are here (setup / install)
└── frida-windows-re/
    ├── SKILL.md                  ← the cheatsheet entry point
    └── references/               ← bundled reference docs (CLI, JS API, recipes, anti-debug, product-sec)
```

The skill itself **assumes `frida-tools` is already installed and on `PATH`**
(`frida --version` works). Install it once with the guide below, then use the skill.

---

## Install frida-tools (uv)

frida-tools ships **prebuilt wheels** for Windows — no compiler needed. The cleanest
install is as an isolated [`uv`](https://docs.astral.sh/uv/) tool.

Two PyPI packages, two roles: `frida-tools` is the CLI suite (`frida`, `frida-ps`,
`frida-trace`, …); `frida` is the native core + Python binding you `import frida`.
Installing `frida-tools` pulls `frida` automatically.

| Package | Provides | Pulled by |
|---|---|---|
| `frida-tools` | CLI commands | you install it |
| `frida` | native core wheel + `import frida` | dependency of `frida-tools` (`frida >= 17.10.0, < 18.0.0`) |

### Quick install (recommended)

```powershell
uv tool install frida-tools
uv tool update-shell      # add the tool bin dir to PATH (persistent) — then OPEN A NEW TERMINAL
frida --version           # verify (prints the frida CORE version, e.g. 17.15.3)
frida-ps                  # lists local processes -> proves the native core loaded
```

`uv tool install` creates a dedicated isolated venv, installs `frida-tools` + deps
(`frida`, `colorama`, `prompt-toolkit`, `pygments`, `websockets`), and drops `.exe`
shims for every `frida-*` command into uv's tool bin dir (`%USERPROFILE%\.local\bin`).

One-shot PATH fix for the **current** session (no new terminal):

```powershell
$env:PATH = "$env:USERPROFILE\.local\bin;$env:PATH"
uv tool dir --bin                 # prints the exact bin dir uv uses
```

Pick the interpreter for the tool's venv (uv can supply it — no system Python needed):

```powershell
uv tool install --python 3.12 frida-tools
```

## Versioning (read once)

`frida-tools` (the CLI — currently **14.x**) and the `frida` **core** (the native
engine — **17.x**, what `frida --version` prints) are **separate version lines**;
`uv tool install frida-tools` pulls a matching core automatically (e.g. frida-tools
14.10.2 + frida 17.15.3 — not a typo). Pin with `uv tool install "frida-tools==<v>"`.

The version match that **does** matter is **frida core ⇔ `frida-server` / `frida-gadget`**
for remote/USB targets — they must be the **same** core version (the wire protocol
isn't stable across versions). This is **irrelevant to local Windows analysis**: the
agent ships inside the `frida` core wheel, so attaching/spawning local processes needs
no server.

| Component | Must version-match `frida` core? |
|---|---|
| `frida-tools` (CLI) | Yes — enforced by the `frida < 18.0.0` pin |
| `frida-server` (remote/USB) | Yes — exact version |
| `frida-gadget` (embedded) | Yes — exact version |
| Local Windows attach/spawn | N/A — agent is in the core wheel |

---

## Windows prerequisites

- **Python**: latest 3.x recommended. Frida core wheels are `cp37-abi3` → **Python
  3.7+** works (one abi3 wheel covers all 3.x). uv can supply the interpreter via
  `--python 3.x` even with no system Python installed.
- **No compiler needed**: `frida` ships prebuilt wheels — `win_amd64` (x64), `win32`
  (x86), `win_arm64` (ARM64). If install tries to *build from sdist*, no matching wheel
  was found for your Python/arch — fix the interpreter/arch, don't install a compiler.
- **VC++ runtime**: the native payload relies on the Microsoft Visual C++
  Redistributable. Win10/11 usually already has it; if `frida`/`frida-ps` fails with a
  native DLL-load error, install the latest **VC++ Redistributable (x64)**.
- **Architecture**: prefer a **64-bit Python** on x64 Windows — its 64-bit core can
  inject into both 32- and 64-bit local targets. For a 32-bit-only target, install
  frida-tools under a 32-bit Python.
- **Elevation**: run the terminal **as Administrator** to attach to elevated/SYSTEM
  processes (PPL/protected processes may still refuse injection).

---

## Verify the install

```powershell
frida --version            # frida core version, e.g. 17.15.3
uv tool list               # confirms frida-tools + its 17 exposed commands
Start-Process notepad.exe
frida-ps | Select-String notepad
frida-trace -n notepad.exe -i "CreateFileW"     # smoke test
```

If `frida --version` errors with a DLL-load failure, that's the VC++ runtime /
wrong-arch-wheel symptom — install the latest VC++ Redistributable (x64) and confirm a
supported interpreter/arch.

---

## Using the skill

With frida-tools installed, open **[`frida-windows-re/SKILL.md`](frida-windows-re/SKILL.md)**
— the cheatsheet entry point. It links to bundled reference docs under
`frida-windows-re/references/` (every `frida-*` tool + REPL magics, a terse JS API
reference, Win32/Native hooking recipes, anti-debug / anti-Frida bypass + the Check
Point Anti-Debug encyclopedia, and product-security / app-pentest recipes).