---
name: ripREA
description: Reverse engineer shipped software end to end - goal routing plus the fastest local CLI technique per target, no MCP. Covers Electron/JS bundles, Mach-O/ELF/PE, Go/Rust/.NET/Flutter/WASM, APKs, SQLite, pcaps, firmware and websites. Use when the user asks how an app or binary works internally, why it crashes or fails, wants to clone or rebuild a feature, audit an app's security, compare two versions, or extract versions/dependencies/metadata.
---

# ripREA: goal → tool → depth

One file: what to investigate, which command answers it, and when to stop. Fix the goal first, then run that strategy's cheapest step. The commands for every target sit in the router below.

## Rules

- Work on a copy (`cp -c` on APFS is instant), one directory per target under `$TMPDIR/re/<name>`, reused.
- Route on `file -b T` and the magic bytes, then take the cheapest inventory, then read code.
- Bundlers hide code in `.webpack/` or `.vite/`, and `rg` skips hidden and ignored files: search extracted trees with `rg -uu`.
- Never `cat` a big or minified file. Slice it: `rg -o '.{0,200}NEEDLE.{0,200}' F | head`. Dump a disassembly to a file once, then grep the file.
- Batch independent probes into one call, separated by `;` and tagged with `echo '--- name'` (quote the marker: zsh expands a bare `==x`). Cap every output with `head` or `-m`.
- Evidence beats vibes: mark each claim **seen** (file:offset, symbol, string or command output) or **inferred** (with the reasoning). Something you did not find is **unknown**, not "absent". Reading a binary statically is not observing it run.
- Turns cost more than bytes: aim for 2-4 tool calls. A missing tool is not a blocker; use the row's fallback, or install it if the user agrees.

## Which question is it?

| Goal | Question | Deliverable | Stop when |
|------|----------|-------------|-----------|
| **FEATURE** | How does X work? | Flow trace UI/API → effect, one hop per line, cited | Every hop is evidenced |
| **DEBUG** | Why does X fail? | Trigger → root-cause chain, minimal fix, repro steps | Cause explains the symptom |
| **REBUILD** | Clone X | Behavior contract (inputs, edge cases, outputs) + parity checks | Functional parity |
| **SECURITY** | What are the risks? | Findings ranked by severity, each with a trigger path → impact | Threat model done |
| **COMPARE** | v1 → v2? | Delta table: added / removed / changed | Delta documented |
| **EXTRACT** | Versions, deps? | Metadata table + the command used per field | All fields found |

End every investigation with its goal's deliverable and the claims marked as above. A request holding several goals gets split, cheapest first (EXTRACT and COMPARE before FEATURE and DEBUG), each sub-investigation with its own stop condition. An unclear goal is treated as FEATURE, with one clarifying question.

## Depth

`L1 SURFACE` strings, exports, imports, entry points (~1 call, seconds) → `L2 BOUNDARIES` component and data-flow map (~1-2 calls) → `L3 LOGIC` implementation read, pseudocode (minutes) → `L4 DEEP` disassembly or decompilation (~10× the cost of L1-L3: confirm with the user unless they explicitly asked for that depth).

Climb only while the deliverable is incomplete. Three calls with no progress mean the wrong layer or the wrong tool: re-route instead of pushing on. Most answers stop at L1 or L2. L4 exists for native or obfuscated code that L3 cannot answer. The examples below are illustrations, not recordings.

## Strategies

### FEATURE: follow the action to its effect

1. **L1**: `rg -uu 'name'`, or the router row, to find the handler, IPC channel or API endpoint.
2. **L2**: trace handler → model → storage, one hop per line.
3. **L3**: read the implementation only for the hops you still cannot explain.
4. **L4**: only when a native boundary blocks the trace.

Stop when the chain from trigger to effect is cited end to end. "Calls closed-source lib X" is a valid hop.
*Example: Notion search. L1 finds the `search:query` IPC channel, L2 traces it to `SearchIndex.node`, L3 sees the `fulltext_search` export. The answer is complete without touching the disassembler.*

### DEBUG: bisect from symptom to cause

1. **Reproduce** once; record actual against expected.
2. **Trace back**: `rg 'error string'` to the function that emits the symptom.
3. **Inspect state**: read it, find the violated condition.
4. **Trace upstream** through the data or property to the root cause.

Pick the branch that matches the symptom. A crash with a stack starts at the crash frame, checking null and bounds. Wrong output with no error starts at the output function and diffs. A silent failure skips static work and goes dynamic. A target you cannot execute (iOS, firmware, malware, no build) skips reproduction: infer the trigger from logs, paths and strings, validate statically, and say "not reproduced".
*Example: settings crash. The crash log points at a null dereference in `loadPreferences`; `prefsCache` is only set in `viewDidLoad` when the user is logged in, which explains the failure for logged-out users.*

### REBUILD: behaviour, not internals

1. **API surface**: probe UI, network and IPC for input and output shapes.
2. **Behaviour matrix**: probe edge cases by hand into a truth table.
3. **Selective peek**, only for a real unknown: identify the algorithm or endpoint name, then research the algorithm elsewhere.
4. **Implement and verify**: write your own and test parity on the same inputs.

How deep to go: simple CRUD or REST stops at step 2. A complex algorithm justifies step 3 to name it. A proprietary protocol or crypto justifies packet inspection, maybe disassembly.
*Example: Slack's emoji picker. The picker calls `GET /emoji.list`, typing `:smi` prefix-filters rather than fuzzy-matches, and custom emoji differ by a category field. Fetch, filter, grid: behaviour reproduced.*

### SECURITY: perimeter, boundaries, impact

1. **Surface map**: `rg` for network, IPC, file, subprocess, eval and shell entry points.
2. **Trust boundaries**: trace each entry to its sink and check the validation.
3. **Capability**: entitlements, permissions and dangerous APIs give the blast radius.
4. **Pattern scan**: read the risky paths for SQLi, XSS, overflow, path traversal.

Priorities: a sandboxed app starts with entitlements; anything parsing untrusted data starts with the parsers; a web-to-native bridge (Electron preload, WebView bridge) comes first. Never call a finding confirmed without a concrete trigger path; otherwise it is inferred.
*Example: an Electron app with 12 `ipcMain.handle` calls and a custom protocol. The preload shows `contextIsolation: false`, and the window loads a remote URL: renderer RCE from any XSS.*

### COMPARE: inventories before code

1. **Metadata**: `go version -m` on both, or the bundle's `package.json`.
2. **Symbols**: `nm` diff for added and removed functions.
3. **Strings**: `strings | sort` diff for new UI, errors and URLs.
4. **Code**: disassemble only the functions that changed.

By gap: a patch release means steps 2 and 4, ignoring the unchanged 99%. A major version means all four, and expect architecture changes. Hunting a fix means step 3 for new error strings, then step 4 for the added validation.
*Example: a new `exportPDF` and `handlePrint`, new "Export to PDF" strings, and `exportPDF` shelling out to wkhtmltopdf: one feature added, 500 other functions untouched.*

### EXTRACT: metadata, no code

One shot each: Go `go version -m T`; Rust `cargo audit bin T` or the `/rustc/` hash; .NET `ilspycmd --version`; JS `jq .dependencies`; native `otool -L` plus `strings | rg version`; APK `aapt dump badging a.apk`. Then stop. No disassembly, ever.

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

- Strings and imports first: on a small binary, `nm -u`, `strings` and the export list answer "what does it do" without disassembly.
- To find who uses a string, take its address from the "literal pool for" comment in `otool -tV` output (or `rg -n 'literal pool for: "X'`), then look for the enclosing function label above that line.
- A C++ name in a `bl` target already tells you the protocol (`getpeereid`, `write`, `std::hash`). Read the call sequence, not every instruction.
- In JS, work inward from the native boundary: native call, caller class, IPC registration, then renderer and preload.
- Comparing versions: diff inventories (`nm`, module list, `go version -m`) before diffing any code.
- Rebuilding: minimal reimplementation, then test it against the original on the same inputs.

## Escalation, first that applies

- Minified JS → `sourceMappingURL` → `jq .sourcesContent`; otherwise `npx -y prettier --parser babel`.
- Stripped native → `nm -u`, `otool -L`, `go version -m` and `strings` still work; then Ghidra auto-analysis before reading by hand.
- Packed or encrypted binary → identify the packer (die, BinInfo), then dump memory after it unpacks via lldb or frida.
- Obfuscated strings → find the decrypt routine, break there, dump the decrypted values.
- Static exhausted → dynamic (`lldb`, `frida`, `dtrace`, `--remote-debugging-port`).
- Disassembly unreadable → decompiler (Ghidra, IDA, r2 `pdc`).

## Safety and scope

- SECURITY work on software the user does not own: confirm authorization once. Unclear scope means passive static analysis only, no exploitation attempts.
- Dynamic analysis on copies only (VM, snapshot, throwaway container). Never execute an untrusted binary on the host, never attach to a production process.
- REBUILD stays black-box: reimplement behaviour; never copy code, assets, strings or product names. A selective peek names an algorithm to research, not code to transplant.
- L4 needs the user's go-ahead unless they asked for that depth.

## When the plan stops fitting

If a goal fits no strategy, or the plan does not survive contact, say so and think out loud: what is the actual question, what is the minimum evidence, which tool is fastest, how will I know I am done? Then run that. This is a guide, not a constraint.

## Install (on request)

`brew install jadx apktool radare2 binwalk wabt wireshark ghidra` · `npm i -g @electron/asar` · `dotnet tool install -g ilspycmd` · `cargo install cargo-audit`.
