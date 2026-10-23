# ripREA Examples

Practical examples showing how ripREA's goal-first routing changes investigation strategy.

---

## Example 1: Feature Investigation

**User:** "How does Discord's screen sharing work?"

### ripREA decision process
```
1. Match goal: FEATURE (how does X work?)
2. Apply strategy: Progressive depth L1→L2→L3
3. Execute with stop conditions
```

### Investigation
```
L1 SURFACE (10s):
$ npx @electron/asar extract Discord.app/Contents/Resources/app.asar /tmp/discord
$ rg -uu 'screen.*share|screenshare|SCREEN_SHARE' /tmp/discord
→ Found: 'SCREEN_SHARE_START' IPC channel, DesktopCapturer API

L2 BOUNDARIES (30s):
$ rg -A5 'SCREEN_SHARE_START' /tmp/discord/main.js
→ Handler in main process calls desktopCapturer.getSources()
→ Sends source list to renderer via IPC reply
→ Renderer captures via getUserMedia with chromeMediaSourceId

L3 LOGIC (check if needed):
Question: "How does it work?" → ANSWERED
- UI triggers IPC
- Main process enumerates screens via desktopCapturer
- Renderer captures via WebRTC getUserMedia
- Stream sent via WebRTC datachannel

STOP: Complete trace from UI to wire. No need for L4 (native disassembly).
```

**Time:** ~1 minute, 2-3 tool calls  
**Depth:** Stopped at L2 (boundaries clear, logic evident)

---

## Example 2: Debugging

**User:** "Why does my Electron app freeze when opening a large file?"

### ripREA decision process
```
1. Match goal: DEBUG (why fails?)
2. Apply strategy: Bisect symptom → cause
3. Branch: Performance issue → profile first
```

### Investigation
```
REPRODUCE:
- Open file >10MB → UI freezes for 5+ seconds
- Expected: Progressive load or worker thread
- Actual: Main thread blocks

TRACE BACK:
$ rg -uu 'openFile|readFile|fs\.read' app.asar-extracted/
→ Found: ipcMain.handle('file:open', readFileSync)
→ Ah! Synchronous read in main process

INSPECT STATE:
Reading the handler code:
```js
ipcMain.handle('file:open', async (event, path) => {
  const content = fs.readFileSync(path, 'utf-8'); // BLOCKING!
  return content;
});
```

ROOT CAUSE: fs.readFileSync blocks main process event loop

FIX: Use fs.promises.readFile or worker_threads

STOP: Root cause found, solution clear.
```

**Time:** ~30 seconds  
**Depth:** Stopped at TRACE BACK (symptom → root cause identified)

---

## Example 3: Security Audit

**User:** "Is this Electron app safe to run?"

### ripREA decision process
```
1. Match goal: SECURITY (risks?)
2. Apply strategy: Perimeter → boundaries → impact
3. Priority: Check contextIsolation, nodeIntegration, remote content
```

### Investigation
```
SURFACE MAP:
$ npx @electron/fuses read --app App.app
→ RunAsNode: ENABLED (risky)
→ EnableNodeCliInspectArguments: ENABLED (risky)

$ rg -uu 'webPreferences|BrowserWindow' app/main.js
→ Found 3 BrowserWindow creations

TRUST BOUNDARIES:
Window 1 (main):
  nodeIntegration: false ✓
  contextIsolation: true ✓
  sandbox: true ✓
  
Window 2 (settings):
  nodeIntegration: false ✓
  contextIsolation: false ✗ RISK
  preload: settings-preload.js
  
Window 3 (helper):
  nodeIntegration: true ✗ CRITICAL
  webSecurity: false ✗ CRITICAL
  allowRunningInsecureContent: true ✗ CRITICAL

$ rg 'loadURL.*http' app/main.js
→ Window 3 loads: `loadURL(userProvidedURL)` ← UNTRUSTED INPUT

CAPABILITY:
$ codesign -d --entitlements :- App.app
→ com.apple.security.network.client: true
→ com.apple.security.files.user-selected.read-write: true

VERDICT:
🚨 CRITICAL: Window 3 has nodeIntegration + loads remote content
   → RCE from any attacker-controlled URL
   
⚠️  HIGH: Fuses allow Node CLI args → possible escape
   
Impact: Full system access (network + filesystem entitlements)

STOP: Critical findings documented. Full code audit now lower priority.
```

**Time:** ~2 minutes  
**Depth:** Stopped at PATTERN SCAN (critical vulns found, no need for deeper audit)

---

## Example 4: Rebuilding a Feature

**User:** "Clone the command palette from VS Code"

### ripREA decision process
```
1. Match goal: REBUILD (clone X)
2. Apply strategy: Black-box behavior → selective peek
3. Stop at: Functional parity (don't need their code)
```

### Investigation
```
API SURFACE (test in VS Code):
- Trigger: Cmd+Shift+P
- Input: Text field with fuzzy search
- Output: Filtered command list
- Action: Enter executes, Esc cancels

BEHAVIOR MATRIX (manual testing):
Input "file op" → matches "File: Open", "File: Open Recent", etc.
Test: Is it prefix? → NO ("op" matches "Open")
Test: Is it fuzzy? → YES ("flop" matches "File: Open")
Test: Does it learn? → YES (recent commands appear first)

Character test:
- "FiOp" → matches "File: Open" (capital letters)
→ Algorithm: Fuzzy finder with capital letter preference + recency boost

SELECTIVE PEEK (only what's not obvious):
How to get command list in my app?
→ I don't need VS Code's list, I need MY app's commands
→ Skip this step

IMPLEMENT:
- Command registry: Map<id, {label, action}>
- Fuzzy matcher: score by character matches + capital preference
- Sort: recency boost × fuzzy score
- UI: Input + virtual scrolled list

VERIFY:
Test against my app → same UX as VS Code
✓ Fuzzy matching works
✓ Capital letters preferred
✓ Recent commands boosted

STOP: Functional parity achieved. Don't need VS Code's React internals.
```

**Time:** ~10 minutes (includes implementation)  
**Depth:** Stopped at BEHAVIOR MATRIX + IMPLEMENT (didn't reverse VS Code's code)

---

## Example 5: Version Comparison

**User:** "What changed in Chrome 118 vs 117?"

### ripREA decision process
```
1. Match goal: COMPARE (diff?)
2. Apply strategy: Inventory diff → selective code diff
3. Major version → full inventory pass
```

### Investigation
```
METADATA DIFF:
$ strings chrome-117 | rg 'version' > /tmp/v117-versions
$ strings chrome-118 | rg 'version' > /tmp/v118-versions
$ diff -u /tmp/v117-versions /tmp/v118-versions
→ V8 9.5 → 9.6
→ Blink version bump

SYMBOLS DIFF:
$ nm -gU chrome-117 > /tmp/v117-symbols
$ nm -gU chrome-118 > /tmp/v118-symbols  
$ diff -u /tmp/v117-symbols /tmp/v118-symbols | rg '^[+-]' | head -50
→ +_WebGPUCreateInstance (NEW WebGPU API)
→ +_CSSContainerQueryEvaluate (NEW CSS container queries)
→ -_LegacyLayoutNGMethod (removed)

STRINGS DIFF:
$ strings chrome-117 > /tmp/v117-strings
$ strings chrome-118 > /tmp/v118-strings
$ diff -u /tmp/v117-strings /tmp/v118-strings | rg 'feature|api|new' | head -30
→ "WebGPU API enabled by default"
→ "CSS Container Queries Level 1"
→ "Deprecated: document.domain setter"

CODE DIFF (only for interesting new symbols):
$ # Skip: Release notes already document these features
$ # WebGPU and Container Queries are public specs

CONCLUSION:
Major additions:
- WebGPU API (new GPU access)
- CSS Container Queries
- V8 9.6 performance improvements

Removals:
- Legacy layout engine code
- document.domain setter (deprecated)

STOP: High-level delta documented. 
Don't need to disassemble WebGPU implementation (it's spec'd).
```

**Time:** ~3 minutes  
**Depth:** Stopped at SYMBOLS + STRINGS (new features identified from inventory)

---

## Example 6: Metadata Extraction

**User:** "What Go version was used to build this binary?"

### ripREA decision process
```
1. Match goal: EXTRACT (versions?)
2. Apply strategy: One-shot metadata (no code reading)
3. Stop immediately after metadata dump
```

### Investigation
```
ONE-SHOT:
$ go version -m ./binary
./binary: go1.21.5
	path	github.com/cli/cli/v2/cmd/gh
	mod	github.com/cli/cli/v2	v2.40.1
	dep	github.com/AlecAivazis/survey/v2	v2.3.7
	dep	github.com/MakeNowJust/heredoc	v1.0.0
	...
	build	-compiler=gc
	build	CGO_ENABLED=1
	build	GOARCH=arm64
	build	GOOS=darwin

ANSWER:
- Go toolchain: 1.21.5
- Module: github.com/cli/cli/v2 v2.40.1
- CGO: enabled
- Target: macOS ARM64

STOP: Metadata extraction complete. No code reading needed.
```

**Time:** 5 seconds  
**Depth:** EXTRACT (metadata only, never read code)

---

## Key Patterns Across Examples

### Stop condition recognition
- **Example 1:** Stopped at L2 (boundaries clear)
- **Example 2:** Stopped at root cause (fix is obvious)
- **Example 3:** Stopped at critical vuln (blocking issue found)
- **Example 4:** Stopped at parity (behavior matches)
- **Example 5:** Stopped at inventory diff (features identified)
- **Example 6:** Stopped immediately (metadata goal)

### Avoided over-investigation
- **Example 1:** Didn't disassemble native capturer
- **Example 2:** Didn't profile entire app
- **Example 3:** Didn't audit all 500 functions after critical found
- **Example 4:** Didn't reverse VS Code's React code
- **Example 5:** Didn't disassemble WebGPU implementation
- **Example 6:** Didn't read any code

### Time savings
Average investigation time **with ripREA:** 1-3 minutes  
Typical time **without strategy:** 5-15 minutes  
**Speedup:** 3-5× faster to answer

### The ripREA advantage

Not faster tools—**smarter routing**.

The tools are the same (from leanrea). ripREA just tells you:
1. Which tool for this goal
2. How deep to go
3. When to stop

That's the difference.
