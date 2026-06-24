---
name: frida-windows-re
description: >-
  Cheatsheet for dynamic instrumentation and security analysis of Windows
  binaries with Frida and the frida-tools CLI (frida REPL, frida-trace,
  frida-ps). Use it for runtime/dynamic analysis of a Windows process, DLL, or
  .exe: hooking and tracing Win32/Native (kernel32, ntdll, advapi32, ws2_32,
  bcrypt) API calls, intercepting/modifying arguments and return values, dumping
  decrypted buffers or crypto keys at runtime, monitoring file/registry/network
  activity, tracing process injection (VirtualAllocEx/WriteProcessMemory/
  CreateRemoteThread), unpacking, and bypassing anti-debug / anti-Frida checks in
  malware or protected software. Reach for it on phrasings like "hook this
  Windows function", "intercept CreateFileW", "dump the decrypted strings",
  "frida script for Windows", or any Frida / frida-trace mention. Also covers
  installing frida-tools with uv. Complements rizin-windows-re (static RE).
---

# Frida on Windows — Security Analysis Cheatsheet

Frida is a dynamic instrumentation toolkit: it injects a JavaScript engine
(GumJS) into a running native process so you can hook functions, read/write
memory, trace execution, and rewrite behavior live — no recompilation, no source.
On Windows this means instrumenting any `.exe`/`.dll`, watching what malware
*actually* does, dumping data after it's decrypted, and neutralizing
anti-analysis tricks.

This skill is a **cheatsheet and an entry point to bundled reference docs**, not
a workflow. Skim the inline fast paths below, then open the matching reference
file for depth. Everything here targets **local Windows native analysis**
(x86/x64), not Android/iOS.

> Use only on software you are authorized to analyze (your own systems, malware
> in a lab/VM, CTF targets, sanctioned engagements). Analyze untrusted samples in
> an isolated, disposable VM with no network you care about.

---

## 1. Install (uv)

frida-tools ships **prebuilt wheels** for Windows — no compiler needed. The
cleanest install is as an isolated `uv` tool:

```powershell
# one isolated environment, all frida-* commands on PATH
uv tool install frida-tools
uv tool update-shell        # ensure the tool bin dir is on PATH (new shell after)

frida --version             # verify (prints frida core version, e.g. 17.x)
frida-ps                    # list running processes
```

One-off run without installing, or inside a project venv for the `import frida`
Python API:

```powershell
uvx --from frida-tools frida-ps        # ephemeral run
uv venv; uv pip install frida-tools    # project venv: gives the `frida` python module too
```

**Version rule:** `frida-tools` (the CLI — currently **14.x**) and the `frida`
**core** (the native engine — **17.x**, what `frida --version` prints) are
*separate* version lines; `uv tool install frida-tools` pulls a matching core
automatically (e.g. frida-tools 14.10.2 + frida 17.15.3 — not a typo). The
version match that *does* matter is **core ⇔ `frida-server`/`frida-gadget`** for
remote/USB targets (irrelevant to local Windows analysis). Pin with
`uv tool install "frida-tools==<v>"`. Full command list, upgrading, and
troubleshooting → **[references/install-uv.md](references/install-uv.md)**.

---

## 2. Mental model (read this once)

- **Local injection needs no server.** On Windows, `frida`/`frida-trace` inject
  directly into local processes. `frida-server` is only for *remote* targets
  (Android/iOS/another box over `-H host:port`). `frida-gadget` is for embedding.
- **Spawn vs attach.**
  - `frida -f program.exe -l hooks.js` — **spawn**: Frida loads your script while
    the process is held suspended, so hooks land *before* the first instruction
    (packers, early anti-debug), then it **auto-resumes**. Add `--pause` to hold at
    the REPL prompt instead and release manually with `%resume`.
  - `frida <name|PID>` — **attach** to an already-running process.
- **Match the architecture.** A 64-bit Frida injects 64-bit targets; a 32-bit
  target needs the 32-bit Frida. `uv tool install` matches your Python's arch —
  use a matching Python for 32-bit targets, or just check `Process.arch` once
  attached.
- **Run elevated** for system/protected processes (launch the terminal "as
  Administrator"). PPL/protected processes may still refuse injection.
- **Wide strings everywhere.** Win32 `...W` APIs take UTF-16 (`LPCWSTR`):
  read them with `arg.readUtf16String()`, *not* `readUtf8String()`. `...A` APIs
  use `readAnsiString()`.

---

## 3. CLI fast paths

```powershell
# Discover targets
frida-ps                       # all local processes (name + PID)
frida-ps -a                    # only applications (with a main window)
frida-ls-devices               # local + connected/remote devices

# REPL: attach and explore interactively
frida notepad.exe              # attach by name
frida -p 1234                  # attach by PID
frida -f malware.exe --pause   # spawn, hold at prompt (type %resume); without --pause it auto-runs
frida -f app.exe -l hooks.js   # spawn + load script (auto-reloads on save)
frida -f openssl.exe -l hooks.js -- enc -aes-256-cbc -in a -out b  # pass argv to the target: dashed args AFTER --
frida -p 1234 -l hooks.js -o session.log   # tee all output to a file

# Trace API calls with zero scripting (auto-generates editable handlers)
frida-trace -p 1234 -i "CreateFileW" -i "ReadFile" -i "WriteFile"
frida-trace -f app.exe -i "ws2_32.dll!*" -i "*!*Crypt*"   # by module / glob
frida-trace -p 1234 -I "ntdll.dll"      # include every export of a module (-X excludes)

# Other tools
frida-discover -f app.exe      # find hot functions worth tracing
frida-kill 1234                # kill via Frida's device layer
```

Inside the REPL, **`%` magic commands** drive the session — the full set is just
`%resume`, `%load <path>`, `%reload`, `%unload`, `%exec <path>`, `%time <expr>`,
`%autoperform`, `%autoreload`, `%help`. (There is **no** `%hexdump`/`%dump`/`%dis`/
`%backtrace` — hexdump is automatic for binary values, and symbolication/disasm is
done inline in JS: `hexdump(ptr)`, `DebugSymbol.fromAddress(ptr)`, `Instruction.parse(ptr)`.)
Full tool/flag/magic tables → **[references/cli-tools.md](references/cli-tools.md)**.
frida-trace's include/exclude syntax and the `__handlers__` editing workflow →
**[references/frida-docs/frida-trace.md](references/frida-docs/frida-trace.md)**.

---

## 4. The one hook you'll write most

The universal pattern: resolve an export → `Interceptor.attach` → read args in
`onEnter` (stash on `this`) → act in `onLeave`.

```js
// CreateFileW: log every file the target opens
const k32 = Process.getModuleByName('kernel32.dll');     // -> Module object
Interceptor.attach(k32.getExportByName('CreateFileW'), { // -> NativePointer
  onEnter(args) {
    this.path = args[0].readUtf16String();   // LPCWSTR (wide!)
  },
  onLeave(retval) {
    console.log(`CreateFileW("${this.path}") -> ${retval}`);
  }
});

// Search ALL modules for an export (when you don't know which DLL):
Interceptor.attach(Module.getGlobalExportByName('connect'), { /* ... */ });
```

Current Frida (17.x) API: `Process.getModuleByName(name)` returns a `Module`,
and you call `.getExportByName(fn)` **on it**, or
`Module.getGlobalExportByName(fn)` to search globally. (The old
`Module.getExportByName('kernel32.dll','CreateFileW')` two-arg form seen in most
blog posts was **removed in Frida 17** and now throws — see
[references/windows-hooking-recipes.md](references/windows-hooking-recipes.md) §0
for a version-safe shim.)

Other essentials, copy-paste ready:

```js
retval.replace(0);                      // force a return value (e.g. fake "not found")
args[0] = Memory.allocUtf16String('x'); this.keep = args[0];  // replace a wide-string arg (keep it alive!)
Interceptor.replace(addr, new NativeCallback(() => 1, 'int', []));  // stub a whole function
const fn = new NativeFunction(addr, 'int', ['pointer','uint32'], 'stdcall'); // call native (x86 WINAPI = 'stdcall'; x64 = default/'win64')
console.log(hexdump(ptr, { length: 64, ansi: true }));         // dump a buffer
console.log(Thread.backtrace(this.context, Backtracer.ACCURATE)
  .map(DebugSymbol.fromAddress).join('\n'));                   // who called me
```

Hooking recipes for files, registry, networking, crypto (dump keys/plaintext),
and process-injection APIs → **[references/windows-hooking-recipes.md](references/windows-hooking-recipes.md)**.
Apps that bundle their own crypto (OpenSSL `libcrypto`, BoringSSL, NSS) don't call
Windows CAPI/CNG — hook the library's own exports instead (e.g. OpenSSL's
`EVP_CipherInit_ex`/`EVP_CipherUpdate`); see that file's §4.9.
Condensed API reference (Process/Module/Memory/NativePointer/Interceptor/
NativeFunction with exact signatures) →
**[references/js-api-cheatsheet.md](references/js-api-cheatsheet.md)**.

---

## 5. Anti-debug / anti-Frida bypass

Malware and packers check for debuggers and for Frida itself. Defeat them from a
Frida script — usually by hooking the detection API and lying, or patching the
flag it reads.

```js
// IsDebuggerPresent / CheckRemoteDebuggerPresent -> always "no debugger"
const k32 = Process.getModuleByName('kernel32.dll');
Interceptor.replace(k32.getExportByName('IsDebuggerPresent'),
  new NativeCallback(() => 0, 'int', []));

// PEB!BeingDebugged byte (read directly to dodge API hooks)
// 64-bit PEB+0x02; clear it once at startup.
```

For `NtQueryInformationProcess` (ProcessDebugPort=7, ProcessDebugObjectHandle=0x1e,
ProcessDebugFlags=0x1f), thread-hiding (`NtSetInformationThread` /
ThreadHideFromDebugger=0x11), timing checks (RDTSC/GetTickCount/QPC), hardware
breakpoints, and **anti-Frida** signals (thread names like `gum-js-loop`/`gmain`,
TCP port `27042`, in-memory `"frida"`/`"FridaScriptEngine"` strings, inline-hook
integrity checks) plus stealth tactics →
**[references/anti-debug-bypass.md](references/anti-debug-bypass.md)**.

The bundled **Check Point Anti-Debug encyclopedia** documents each detection
trick in detail (detection + mitigation), organized by mechanism →
**[references/anti-debug-db/](references/anti-debug-db/)** (`debug-flags.md`,
`timing.md`, `exceptions.md`, `process-memory.md`, `object-handles.md`,
`assembly.md`, `interactive.md`, `misc.md`).

---

## 6. Reference map — what to open when

| Need | Open |
|------|------|
| Install/upgrade frida-tools with uv; full command list | [references/install-uv.md](references/install-uv.md) |
| Every `frida-*` tool, its flags, and all REPL `%` magics | [references/cli-tools.md](references/cli-tools.md) |
| Terse JS API reference (security-focused) | [references/js-api-cheatsheet.md](references/js-api-cheatsheet.md) |
| Copy-paste Win32/Native hooking recipes | [references/windows-hooking-recipes.md](references/windows-hooking-recipes.md) |
| Defeat anti-debug / anti-Frida checks | [references/anti-debug-bypass.md](references/anti-debug-bypass.md) |
| The detection tricks themselves, in depth | [references/anti-debug-db/](references/anti-debug-db/) |
| `frida-trace` deep dive + handler editing | [references/frida-docs/frida-trace.md](references/frida-docs/frida-trace.md) |
| **Canonical** full JS API (every class/method) | [references/frida-docs/javascript-api.md](references/frida-docs/javascript-api.md) |
| Stalker — instruction/branch tracing & coverage | [references/frida-docs/stalker.md](references/frida-docs/stalker.md) |
| `send()`/`recv()`/`rpc.exports` messaging (script ⇄ Python) | [references/frida-docs/messages.md](references/frida-docs/messages.md) |
| String/arg-replacement pitfalls (keep-alive, read-only) | [references/frida-docs/best-practices.md](references/frida-docs/best-practices.md) |
| NativeFunction/Callback type & ABI details | [references/frida-docs/functions.md](references/frida-docs/functions.md) |
| Gadget (embed Frida without injection) | [references/frida-docs/gadget.md](references/frida-docs/gadget.md) |
| Worked Windows example (hook by IDA address) | [references/frida-docs/examples-windows.md](references/frida-docs/examples-windows.md) |
| REPL intro / quickstart / modes / troubleshooting | [references/frida-docs/](references/frida-docs/) |

### Typical pairing with static RE
Use **rizin-windows-re** (or IDA/Ghidra) to locate an interesting function or
offset statically, then hook it here. For an internal (unexported) function at
file offset `0xXXXX`:

```js
const base = Process.getModuleByName('target.exe').base;
Interceptor.attach(base.add(0x14230), { onEnter(args) { /* ... */ } });
```

---

## Provenance

Bundled references are copied/adapted from local, openly-licensed sources:
`references/frida-docs/` from the **Frida website docs** (frida.re,
`oleavr` et al.); `references/anti-debug-db/` from the **Anti-Debug-DB / Check
Point Anti-Debug encyclopedia** (MIT). The synthesized cheatsheets
(`install-uv.md`, `cli-tools.md`, `js-api-cheatsheet.md`,
`windows-hooking-recipes.md`, `anti-debug-bypass.md`) distill those plus the
`frida-tools` source.
