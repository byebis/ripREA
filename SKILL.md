---
name: ripREA
description: WHEN to use which reverse-engineering technique, based on your goal. Use when the user asks how an app or binary works internally, why something crashes or fails, wants to clone or rebuild a feature, audit an app's security, compare two versions, or extract versions/dependencies/metadata. Goal routing, progressive depth ladders, stop conditions. Decision layer for leanrea - load WITH leanrea for full tool coverage.
---

# ripREA: Strategic layer for reverse engineering

**Extends leanrea with:** Goal routing → Depth strategy → Stop conditions

## Goal-first routing (identify BEFORE tools)

| Goal | Need | Max depth | Stop when |
|------|------|-----------|-----------|
| **FEATURE** "How does X work?" | Data flow UI→storage | Function boundaries | Trace complete |
| **DEBUG** "Why fails?" | Error path + state | Root cause | Reproducible explanation |
| **REBUILD** "Clone X" | API surface + behavior | Black-box equiv | Functional parity |
| **SECURITY** "Risks?" | Attack surface + boundaries | Vuln patterns | Threat model done |
| **COMPARE** "v1→v2 diff?" | Capability delta | Changed symbols only | Delta documented |
| **EXTRACT** "Versions?" | Metadata only | No code | All versions found |

**Route:** Match goal → apply strategy below. Multiple goals in one question → split into sub-investigations, cheapest first (EXTRACT/COMPARE before FEATURE/DEBUG), each with its own stop condition. Unknown goal → use FEATURE + ask clarification.

**Report contract:** end every investigation with its goal's deliverable below, every claim marked **seen** (cite file/line or command output) / **inferred** (state the reasoning) / **unknown** (admit it):

| Goal | Deliverable |
|------|-------------|
| FEATURE | Flow trace API/UI → effect, one hop per line, cited |
| DEBUG | Trigger → root-cause chain, minimal fix, repro steps |
| REBUILD | Behavior contract (inputs, edge cases, outputs) + parity checks |
| SECURITY | Findings ranked by severity, each with a concrete trigger path → impact |
| COMPARE | Delta table: added / removed / changed (symbol or file) |
| EXTRACT | Metadata table, command used per field |

Never label a SECURITY finding **confirmed** without a concrete trigger path - mark it **inferred**.

---

## Strategies by goal

### FEATURE: Trace user action → effect

**Depth ladder** (stop when answered):
1. **SURFACE** (1 call, 10s): `rg -uu 'featureName'` → entry points (handlers/IPC/API)
2. **BOUNDARIES** (1-2 calls, 30s): Trace handler→model→storage → component map
3. **LOGIC** (2-3 calls, 2-5m): Read implementations → pseudocode
4. **DEEP** (only if stuck): Disassemble/decompile native

**When to stop:**
- L1: Just checking if feature exists or what it's called
- L2: Mapping components/communication ("what talks to what?")
- L3: Need to rebuild logic or understand complex behavior
- L4: ONLY if native+obfuscated AND L3 insufficient

**Example:** Search in Notion: L1 finds `'search:query'` IPC → L2 traces handler to SearchIndex.node → L3 sees `fulltext_search` export → STOP. We know UI→IPC→native FTS without disassembling.

### DEBUG: Find divergence point

**Bisect symptom→cause:**
1. **REPRODUCE**: Confirm trigger, observe actual vs expected
2. **TRACE BACK** (1-2 calls): `rg 'error string'` → function that outputs symptom
3. **INSPECT STATE** (1 call): Read function → find violated condition
4. **TRACE UPSTREAM** (1-2 calls): Follow data flow back → root cause

**When to branch:**
- Crash with stack trace? → Start Step 2 at crash point, check null/bounds
- Wrong output, no error? → Step 2 at output function, diff actual vs expected
- Silent failure? → Skip static, go dynamic (lldb/frida for call chain)
- Can't execute the target (iOS, firmware, malware, no build)? → Skip REPRODUCE: infer the trigger from crash logs/strings/paths, validate statically, report "not reproduced"

**Example:** Settings crash on tap
```
Step 2: grep crash log → null deref in loadPreferences
Step 3: read loadPreferences → accesses self->prefsCache without check
Step 4: when init? → grep constructor → only set in viewDidLoad IF logged in
ROOT CAUSE: prefsCache uninitialized for logged-out users
```

### REBUILD: Black-box contract, not internals

**Strategy:**
1. **API SURFACE** (1 call): UI/network/IPC inspection → input/output shapes
2. **BEHAVIOR MATRIX** (manual): Test edge cases → truth table
3. **SELECTIVE PEEK** (1-2 calls, only if opaque): Grep algo name or check imports
4. **IMPLEMENT + VERIFY**: Write yours, test parity

**When to go deeper:**
- Simple CRUD/REST? → Steps 1-2 sufficient, write your own SQL/queries
- Complex algorithm? → Step 3 identifies algo name, then research algo separately (not their code)
- Proprietary protocol/encryption? → Step 3 must reverse deeper (packet inspection, maybe disassembly)

**Example:** Clone Slack emoji picker
```
Step 1: Click picker → network shows GET /emoji.list → returns {emojis: [{name, url}]}
Step 2: Type ":smi" → filters to matches. Test: prefix or fuzzy? → Prefix only confirmed
Step 3: Check custom emoji → same endpoint, category field differentiates
Step 4: Implement: fetch JSON, prefix-filter, render grid → test against Slack → matches
STOP: We replicated behavior. Don't need their React implementation.
```

### SECURITY: Perimeter → boundaries → impact

**Strategy:**
1. **SURFACE MAP** (1-2 calls): Grep network/IPC/file/subprocess/eval → entry points
2. **TRUST BOUNDARIES** (1 call): Trace entries to usage → check validation
3. **CAPABILITY** (1 call): Check entitlements/permissions/dangerous APIs → impact scope
4. **PATTERN SCAN** (1-2 calls, if risks found): Read vulnerable paths for SQLi/XSS/overflow/etc

**Priority targets:**
- Sandboxed app? → Check entitlements FIRST (no network/fs = limited blast radius)
- Parses untrusted data? → Focus parsers (JSON/XML/image/etc are vuln-prone)
- Web→native bridge? → HIGH PRIORITY (Electron/WebView contextBridge/preload)

**Example:** Electron app security audit
```
Step 1: rg 'ipcMain.handle' → 12 handlers, rg 'protocol.register' → 1 custom protocol
Step 2: Check preload for contextIsolation → FALSE (renderer has full Node.js)
Step 3: Check webPreferences → allowRunningInsecureContent + loads remote URL
Step 4: Analysis: renderer + remote content + no isolation = RCE from any XSS
STOP: Critical finding documented. Full handler audit now lower priority.
```

### COMPARE: Diff inventories, not code

**Strategy:**
1. **METADATA** (1 call): strings diff or `go version -m` diff → dep changes
2. **SYMBOLS** (1 call): nm diff → added/removed functions
3. **STRINGS** (1 call): strings diff → new UI/errors/URLs
4. **CODE** (1-2 calls, only for interesting symbols): Disasm changed funcs only

**Depth by version gap:**
- Patch (1.0.1→1.0.2)? → Steps 2+4 only (symbols + changed funcs). Ignore unchanged 99%.
- Major (1.x→2.x)? → All steps. Expect architectural changes.
- Looking for bug fixes? → Step 3 for new error strings, Step 4 for added validation.

**Example:** What's new in v2.5 vs v2.4?
```
Step 1: go version -m both → deps unchanged
Step 2: nm both, diff -u → shows +exportPDF, +handlePrint, +renderPDFPage
Step 3: strings both, diff -u → "Export to PDF", "Print Preview", new menu items
Step 4: Disassemble exportPDF only → calls wkhtmltopdf subprocess
CONCLUSION: Added PDF export via wkhtmltopdf. Other 500+ functions untouched.
```

### EXTRACT: Metadata only, zero code

**One-shot:** Go: `go version -m`. Rust: `strings | rg '/rustc/'`. .NET: `ilspycmd --version`. JS: `jq .dependencies`. Native: `otool -L` + `strings | rg version`.  
**STOP immediately.** No disassembly ever.

---

## Tactical rules

**Stop when:**
- Question answered (don't dig for completeness)
- Boundary hit (e.g., "calls closed-source lib X" is valid answer)
- 3+ calls with no progress (wrong layer, re-route)

**Go deeper when:**
- Answer incomplete (found handler, not implementation)
- Obfuscation blocks (minified/stripped/encrypted)
- User explicitly asks

**Switch tools:**
- Strings/symbols exhausted → disassembly
- Disassembly unreadable → decompiler (Ghidra/IDA)
- Static stuck → dynamic (lldb/frida/dtrace)

**Obfuscation escalation** (first that applies):
- Minified JS → sourceMappingURL → `jq .sourcesContent`; else `npx prettier --parser babel` (never cat minified)
- Stripped native → what's left is still readable: `nm -u`, `otool -L`, `go version -m`, `strings`; then Ghidra auto-analysis before manual reading
- Packed/encrypted binary → identify packer (die/BinInfo) → dump memory after unpacking via lldb/frida
- Obfuscated strings → find the decrypt routine, break there, dump decrypted values

**Safety & scope:**
- SECURITY on software you don't own → confirm authorization first (ask once); unclear scope → passive static analysis only, no exploitation attempts
- Dynamic analysis on copies only (VM/snapshot/throwaway container): never execute an untrusted binary on the host, never attach a debugger to a production process
- REBUILD stays black-box: reimplement behavior; never copy code, assets, strings or product names. The selective peek names an algorithm to research - it is not code to transplant
- L4 (disassembly/decompile) costs ~10× L1-L3 → confirm with the user before going there unless they explicitly asked for depth

**Efficiency (from leanrea):**
- Work on copy (`cp -c` instant on APFS)
- Route by `file` + magic first
- Never `cat` minified (slice with `rg -o`)
- Batch probes (one shell call with `;` markers)
- `rg -uu` for extracted trees (bundlers hide in dot-folders)
- Mark claims: **seen** (cite) / **inferred** (explain) / **unknown** (honest)
- Dump disassembly once, then grep

**Tools & speed patterns:** See leanrea skill (load together with ripREA).

**Quick tool reference if leanrea unavailable:**
- **Electron**: `npx -y @electron/asar extract app.asar X` → `rg -uu 'ipcMain|ipcRenderer|contextBridge' X`
- **Native (Mach-O)**: `nm -gU T` (exports), `nm -u T` (imports), `otool -tV T > dis.txt`, `strings -a T`
- **Go**: `go version -m T` (all metadata in one shot)
- **Rust**: `cargo audit bin T` or `strings | rg '/rustc/'`
- **Android**: `jadx -d X a.apk` → `rg`
- **.NET**: `ilspycmd -p -o X T.dll`
- **Web/JS**: Find sourceMappingURL, `jq .sourcesContent` or `npx prettier --parser babel`
- **Disasm**: `otool -tV` (macOS), `objdump -d -M intel` (Linux), then `rg` the output
- **Dynamic**: `lldb -p PID`, `frida`, `dtrace`, Chrome `--remote-debugging-port`

Full tool router + speed patterns in leanrea. Use together for best results.

---

## Meta-rule

If goal doesn't match taxonomy or strategy doesn't fit, **reason out loud:**
1. What's the actual question?
2. Minimum info needed?
3. Fastest tool?
4. When am I done?

Then execute. This is a guide, not a constraint.
