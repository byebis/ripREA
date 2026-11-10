---
name: ripREA
description: Reverse engineer shipped software end to end - goal routing plus the fastest local CLI technique per target, no MCP. Covers Electron/JS bundles, Mach-O/ELF/PE, Go/Rust/.NET/Flutter/WASM, APKs, SQLite, pcaps, firmware and websites. Use when the user asks how an app or binary works internally, why it crashes or fails, wants to clone or rebuild a feature, audit an app's security, compare two versions, or extract versions/dependencies/metadata.
---

# ripREA: goal → tool → depth

One file: what to investigate, which command answers it, and when to stop. Identify the goal first, then run that strategy's cheapest step; commands for every target are in the router.

## Rules

- Work on a copy (`cp -c` on APFS is instant). One dir per target under `$TMPDIR/re/<name>`, reused.
- Route on `file -b T` and the magic bytes, then the cheapest inventory, then code.
- Bundlers hide code (`.webpack/`, `.vite/`) and `rg` skips hidden/ignored files: search extracted trees with `rg -uu`.
- Never `cat` a big or minified file - slice it: `rg -o '.{0,200}NEEDLE.{0,200}' F | head`. Dump a disassembly once to a file, then grep the file.
- Batch independent probes into ONE call separated by `;` with `echo '--- name'` markers (quoted: zsh expands a bare `==x`); cap every output with `head` or `-m`.
- Evidence over vibes: mark each claim **seen** (file:offset, symbol, string or command output) or **inferred** (with the reasoning). What you did not find is **unknown**, not "absent". Static reading is not observed execution.
- Turns cost more than bytes: 2-4 tool calls is the target. A missing tool is not a blocker - use the row's fallback, or install it if the user allows.

## Goal routing (decide before touching a tool)

| Goal | Question | Deliverable | Stop when |
|------|----------|-------------|-----------|
| **FEATURE** | How does X work? | Flow trace UI/API → effect, one hop per line, cited | Every hop is evidenced |
| **DEBUG** | Why does X fail? | Trigger → root-cause chain, minimal fix, repro steps | Cause explains the symptom |
| **REBUILD** | Clone X | Behavior contract (inputs, edge cases, outputs) + parity checks | Functional parity |
| **SECURITY** | What are the risks? | Findings ranked by severity, each with a trigger path → impact | Threat model done |
| **COMPARE** | v1 → v2? | Delta table: added / removed / changed | Delta documented |
| **EXTRACT** | Versions, deps? | Metadata table + the command used per field | All fields found |

End each investigation with its goal's deliverable, every claim marked per the rules. Multiple goals in one request → split into sub-investigations, cheapest first (EXTRACT/COMPARE before FEATURE/DEBUG), each with its own stop condition. Unknown goal → treat as FEATURE and ask one clarifying question.

## Depth ladder (shared)

`L1 SURFACE` strings, exports, imports, entry points (~1 call, seconds) → `L2 BOUNDARIES` component and data-flow map (~1-2 calls) → `L3 LOGIC` implementation read, pseudocode (minutes) → `L4 DEEP` disassembly/decompile (~10× the cost of L1-L3: confirm with the user unless they explicitly asked for that depth).

Climb only while the deliverable is incomplete. Three calls with no progress means the wrong layer or the wrong tool: re-route, do not push. Most answers stop at L1/L2; L4 exists for native or obfuscated code where L3 cannot answer. Examples below are illustrative.

## Strategies

### FEATURE - trace user action to effect

1. **L1**: `rg -uu 'name'` (or the router row) → handler, IPC channel, API endpoint
2. **L2**: trace handler → model → storage; one hop per line
3. **L3**: read the implementation only for hops you still cannot explain
4. **L4**: only when a native boundary blocks the trace

STOP: the chain from trigger to effect is cited end to end. "Calls closed-source lib X" is a valid hop.
*Ex: Notion search - L1 finds the `search:query` IPC channel, L2 traces it to `SearchIndex.node`, L3 sees the `fulltext_search` export. Answer complete without disassembling.*

### DEBUG - bisect symptom to cause

1. **REPRODUCE**: confirm the trigger, record actual vs expected
2. **TRACE BACK**: `rg 'error string'` → the function that emits the symptom
3. **INSPECT STATE**: read it, find the violated condition
4. **TRACE UPSTREAM**: follow the data or property back to the root cause

Branches: crash with a stack → start at the crash frame, check null/bounds; wrong output, no error → start at the output function and diff; silent failure → skip static, go dynamic; cannot execute the target (iOS, firmware, malware, no build) → skip REPRODUCE, infer the trigger from logs/paths/strings, validate statically, report "not reproduced".
*Ex: settings crash - crash log → null deref in `loadPreferences` → `prefsCache` is set only in `viewDidLoad` when logged in → root cause for logged-out users.*

### REBUILD - black-box contract, not internals

1. **API SURFACE**: UI, network and IPC probes → input/output shapes
2. **BEHAVIOR MATRIX**: probe edge cases by hand → truth table
3. **SELECTIVE PEEK** (only for a real unknown): identify the algorithm or endpoint name, then research the algorithm elsewhere
4. **IMPLEMENT + VERIFY**: write your own, test parity on the same inputs

Go deeper only when: simple CRUD/REST → steps 1-2 suffice; complex algorithm → step 3 names it; proprietary protocol or crypto → deeper packet inspection, maybe disassembly.
*Ex: Slack emoji picker - the picker hits `GET /emoji.list`, typing `:smi` prefix-filters rather than fuzzy-matches, custom emoji differ by a category field → fetch + filter + grid reproduces the behavior.*

### SECURITY - perimeter, boundaries, impact

1. **SURFACE MAP**: `rg` network, IPC, file, subprocess, eval, shell → entry points
2. **TRUST BOUNDARIES**: trace each entry to its sink, check validation
3. **CAPABILITY**: entitlements, permissions, dangerous APIs → blast radius
4. **PATTERN SCAN**: read the risky paths (SQLi, XSS, overflow, path traversal)

Priorities: sandboxed app → entitlements first; parses untrusted data → parsers first; web→native bridge (Electron preload, WebView bridge) → highest. Never label a finding "confirmed" without a concrete trigger path; otherwise **inferred**.
*Ex: Electron - 12 `ipcMain.handle` plus a custom protocol; preload shows `contextIsolation: false`; the window loads a remote URL → renderer RCE from any XSS.*

### COMPARE - diff inventories before code

1. **METADATA**: `go version -m` both, or the bundle's `package.json`
2. **SYMBOLS**: `nm` diff → added/removed functions
3. **STRINGS**: `strings | sort` diff → new UI, errors, URLs
4. **CODE**: disassemble only the changed functions

By gap: patch → steps 2+4, ignore the unchanged 99%; major → all four, expect architecture changes; hunting a fix → step 3 for new error strings, then step 4 for added validation.
*Ex: +`exportPDF`, +`handlePrint`, new "Export to PDF" strings, and `exportPDF` shells out to wkhtmltopdf → one feature added, 500 other functions untouched.*

### EXTRACT - metadata, zero code

One shot: Go `go version -m T`; Rust `cargo audit bin T` or the `/rustc/` hash; .NET `ilspycmd --version`; JS `jq .dependencies`; native `otool -L` plus `strings | rg version`; APK `aapt dump badging a.apk`.
**STOP.** No disassembly, ever.

## Router: fastest path per target

| Target (detect) | Fastest path |
|---|---|
| Electron (`Contents/Resources/app.asar`) | `npx -y @electron/asar extract app.asar out` (needs Node ≥22.12), then `jq '{name,version,main}' out/package.json` for the entry point; native addons stay in `app.asar.unpacked/`. Channels built at runtime hide from plain grep: `rg -uu -o '"[a-z]+:[a-z0-9:-]+"' out/<main> \| sort -u`. Fuses and ASAR integrity: `npx -y @electron/fuses read --app A.app`; the header hash the `EnableEmbeddedAsarIntegrityValidation` fuse checks lives in `plutil -extract ElectronAsarIntegrity xml1 -o - A.app/Contents/Info.plist`. |
| Web bundle / minified JS | Start at `sourceMappingURL`: a map carrying `sourcesContent` gives the tree back (`jq -r '.sources[]'`). No map: find the module id, slice ±250 chars around the hit. Prettier output: `npx -y prettier --parser babel F > F.pretty.js`. |
| Hermes (`index.android.bundle`, magic `c6 1f bc 03`) | Read the bytecode version out of the header first. `hermes-dec` (`pipx install hermes-dec`) covers current bundles: `hbc-disassembler B out.hasm`, `hbc-decompiler B out.js`. `hbctool disasm B D` parses only HBC 59-85; newer versions need `r2hermes` (`r2pm -ci r2hermes`, then `r2 B` and `pd:ha`), which handles 51-99. `.jsc` from bytenode has no decompiler: strings plus runtime hooks. |
| Mach-O | `lipo -info`, thin the slice first (`lipo -thin arm64 -o T1 T`); `otool -L` for links, `xcrun dyld_info -exports -imports` for the symbol table, `nm -gU` exports and `nm -u` imports (the import list is a capability map), `strings -a -n 6`, `otool -l \| rg -A3 'LC_(RPATH\|BUILD_VERSION)'`, `codesign -d --entitlements :- T`, `otool -ov` for ObjC classes, `xcrun swift-demangle --compact`. |
| Native code logic | Dump once, grep forever: `otool -tV T > $D/dis.txt` (or `llvm-objdump -d --no-show-raw-insn -C`). Locate a function with `rg -n '^_?.*Name.*:$'`, then read only its trail: `awk '/^FUNC:/{p=1} p&&/^[_a-zA-Z].*:$/&&!/FUNC/{exit} p' dis.txt \| rg 'bl\|call\|literal\|cmp'`. Pseudocode: Ghidra `analyzeHeadless $D p -import T -postScript DecompileAll.java -deleteProject`, or `r2 -qc 'aaa; pdc @ sym.X' T`. |
| ELF / PE | `readelf -hlSW` (layout), `readelf -dW` (NEEDED = runtime deps), `nm -D --defined-only` (exports), `objdump -d -M intel`. PE: `objdump -x`, resources with `wrestool -l` or `pefile`, hardening with `pwn checksec`. |
| Go (`__go_buildinfo`, `.go.buildinfo`) | `go version -m T` prints toolchain, module, deps and build flags in one shot. No Go installed: `strings -a T \| rg -m1 -A200 'Go buildinf'`. Stripping does not remove Go symbols: `go tool nm T \| rg main\.`. The same command reads a Windows `.exe`. |
| Rust | Built with cargo-auditable → `cargo audit bin T`, or `rust-audit-info T` to get the dependency tree as JSON. Compiler: `strings -a T \| rg -o '/rustc/[0-9a-f]{40}' \| sort -u`. Crate versions leak through path strings: `strings -a T \| rg -o '[a-z0-9_-]+-[0-9]+\.[0-9]+\.[0-9]+/src' \| sort -u`. |
| .NET (Mono / CoreCLR) | `ilspycmd -p -o out T.dll`, then `rg` the tree; P/Invoke calls: `rg DllImport out`. NativeAOT and IL2CPP output is native code - use the rows above. |
| Android APK | `jadx -d out --show-bad-code a.apk`, then `rg`. Manifest and resources only: `apktool d --no-src -o out a.apk`. Fast inventory: `unzip -l a.apk`. `lib/*/libflutter.so` means Flutter (`blutter`), `libhermes.so` means React Native (Hermes row). |
| Flutter snapshot hash | `libapp.so` starts `f5 f5 dc dc`; the 32 hex chars at +20 are the snapshot hash - that is what maps the binary to a Dart version. Dump the snapshot with `blutter`. |
| WASM | `wasm-objdump -x` for sections and imports, `wasm2wat` to read, `wasm-decompile` to get C-like output (all WABT). |
| SQLite | Copy the database together with its `-wal`, then `sqlite3 'file:C?mode=ro' '.schema'` and `SELECT * FROM t LIMIT 20`. |
| plist / nib / car | `plutil -p F` for any plist; Info.plist carries bundle id, URL schemes and privacy strings; `assetutil --info Assets.car` lists image assets. |
| pcap / HAR | `tshark -r C -q -z conv,tcp` for the flow table, `tshark -r C -T fields -e dns.qry.name -e tls.handshake.extensions_server_name -e http.request.full_uri \| sort -u` for the trace; HAR: `jq -r '.log.entries[].request \| "\(.method) \(.url)"'`. |
| Python packaged (PyInstaller `_MEIPASS`, Nuitka) | `pyinstxtractor-ng app` unpacks the archive into `.pyc` files without needing the interpreter that built it (`-i` only lists), then `pycdc` or `decompyle3` turns those into source. Nuitka compiles to C, so its output goes through the ELF/PE rows instead. |
| Node SEA single-file (`NODE_SEA_BLOB`) | The script travels as a blob appended to the runtime: `rg -ab 'NODE_SEA_BLOB' app` finds it, and the bytes around it hold the snapshot strings. Restoring the body is not supported upstream - read strings, then hook the entry point at runtime. |
| Unity IL2CPP (`global-metadata.dat`, `libil2cpp.so`, `GameAssembly.dll`) | `Il2CppDumper <exe> global-metadata.dat out` writes `dump.cs`, the dummy DLLs and Ghidra/IDA scripts; `Cpp2IL` handles the newer metadata versions better. Method bodies stay in the native binary - load the scripts there. |
| Java JAR / `.class` | `cfr app.jar --outputdir out` (or `procyon`) recovers readable Java, `jdeps -summary app.jar` gives the dependency graph. Obfuscated names stay obfuscated; map them by strings and call sites. |
| iOS IPA | `unzip -o app.ipa -d out`, then `out/Payload/*.app`: main binary through the Mach-O rows, `plutil -p Info.plist` for bundle id and URL schemes, `assetutil --info Assets.car` for images, `Frameworks/` as separate binaries. |
| macOS pkg / Windows installer | `pkgutil --expand-full app.pkg out` for a flat package; `innoextract -d out setup.exe` for Inno Setup, `msiextract` for MSI, `7z x` for the generic case; then re-route each payload. |
| Firmware | `binwalk -e F`, or `unblob -e out F` for modern containers, then re-route every extracted binary through this table. |
| Website | `curl -sL URL` for the HTML and its script sources, fetch each bundle and slice it. Runtime view: a headless browser with a request log (Playwright `page.on('request')`) or Chrome `--remote-debugging-port`. |
| Running app / process | macOS: `lldb -p PID`, `fs_usage`, `log stream --predicate 'process=="X"'`. Electron main process: `--inspect` if the RunAsNode and inspect fuses permit it. Android: `adb logcat`, `frida -U -f pkg -l hook.js`. |

## First move: one batched call

Native (Mach-O):
```sh
T=./bin; D=$TMPDIR/re/$(basename $T); mkdir -p $D; file $T; lipo -info $T; otool -L $T; echo '--- EXPORTS'; nm -gU $T | head -150; echo '--- IMPORTS'; nm -u $T | rg -v '^__Z|napi_' | tr '\n' ' '; echo; echo '--- STRINGS'; strings -a -n 6 $T | rg -v '^[_$.]' | head -400; otool -tV $T > $D/dis.txt; echo '--- FUNCS'; rg '^[_a-zA-Z].*:$' $D/dis.txt | rg -v '^_?_?Z?N?St3' | head -150
```
Then read one function: `awk '/^FUNC_LABEL:$/{p=1;next} p&&/^[_a-zA-Z].*:$/{exit} p' $D/dis.txt | rg 'bl|b\.|cmp|literal|mov\s+w2'`.

Electron:
```sh
A=./app.asar; X=$TMPDIR/re/asar; rm -rf $X; npx -y @electron/asar extract $A $X && jq '{name,version,main}' $X/package.json; find $X -path '*/node_modules' -prune -o -type f \( -name '*.js' -o -name '*.node' \) -print | head -60; ls ${A}.unpacked/node_modules 2>/dev/null; rg -uu -l --glob '!node_modules' 'exposeInMainWorld' $X
```
Surface scan: `rg -n -o '.{0,120}(ipcMain\.(handle|handleOnce|on|once)|ipcRenderer\.(invoke|send|sendSync|on)|contextBridge\.exposeIn(Main|Isolated)World|webPreferences|preload:|nodeIntegration|contextIsolation|sandbox|\.node\b).{0,160}' -uu $X --glob '!node_modules'`. Then slice the needle across the bundle: `rg -uu -o --glob '!node_modules' '.{0,250}NEEDLE.{0,250}' $X | head -20`.

Go: `go version -m T` alone usually answers the question.

## Speed patterns

- Strings and imports first: on a small binary, `nm -u` + `strings` + the export list answer "what does it do" without disassembly.
- Who uses a string: get its address from the `otool -tV` "literal pool for" comment, or `rg -n 'literal pool for: "X'`, then find the enclosing function label above that line.
- A C++ name in a `bl` target already tells you the protocol (`getpeereid`, `write`, `std::hash`). Read the call sequence, not every instruction.
- JS: work from the native/JS boundary inward - native call, caller class, IPC registration, then renderer/preload.
- Comparing versions: diff inventories (`nm`, module list, `go version -m`) before diffing any code.
- Rebuilding: minimal reimplementation, then test it against the original with the same inputs.

## Escalation (first that applies)

- Minified JS → `sourceMappingURL` → `jq .sourcesContent`; else `npx -y prettier --parser babel`.
- Stripped native → `nm -u`, `otool -L`, `go version -m`, `strings` still work; then Ghidra auto-analysis before reading by hand.
- Packed/encrypted binary → identify the packer (die/BinInfo), dump memory after unpacking via lldb/frida.
- Obfuscated strings → find the decrypt routine, break there, dump the decrypted values.
- Static exhausted → dynamic (`lldb`, `frida`, `dtrace`, `--remote-debugging-port`).
- Disassembly unreadable → decompiler (Ghidra/IDA/r2 `pdc`).

## Safety and scope

- SECURITY work on software the user does not own → confirm authorization once; unclear scope → passive static analysis only, no exploitation attempts.
- Dynamic analysis on copies only (VM, snapshot, throwaway container). Never execute an untrusted binary on the host; never attach to a production process.
- REBUILD stays black-box: reimplement behavior; never copy code, assets, strings or product names. A selective peek names an algorithm to research - it is not code to transplant.
- L4 needs the user's go-ahead unless they asked for that depth.

## Meta-rule

If the goal fits no strategy or the plan does not survive contact, say so and reason out loud: what is the actual question, what is the minimum evidence, which tool is fastest, how will I know I am done? Then run it. This is a guide, not a constraint.

## Install (on request)

`brew install jadx apktool radare2 binwalk wabt wireshark ghidra` · `npm i -g @electron/asar` · `dotnet tool install -g ilspycmd` · `cargo install cargo-audit`.
