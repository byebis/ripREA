# Examples

Six worked examples, one per goal. They show the decision process rather than recorded sessions: the targets are real apps, but the conversations are illustrations. The measured run, with real output and timings, is in [WALKTHROUGH.md](WALKTHROUGH.md).

## 1. A feature question: how does Discord share a screen?

**Request:** "How does Discord's screen sharing work?"

That's FEATURE: the user wants the path from clicking a button to pixels leaving the machine.

The surface search looks for the words a screen-sharing feature would need:

```sh
npx -y @electron/asar extract Discord.app/Contents/Resources/app.asar /tmp/discord
rg -uu 'screen.*share|screenshare|SCREEN_SHARE' /tmp/discord
```

That finds a `SCREEN_SHARE_START` IPC channel and uses of `desktopCapturer`. Following the channel into the main process shows the handler calling `desktopCapturer.getSources()`, replying with the source list, and the renderer starting `getUserMedia` with `chromeMediaSourceId`.

At that point the chain is closed: UI → IPC → main process enumeration → renderer capture → WebRTC. There is a native component somewhere, but nothing in the question requires opening it. `nm` on the capture addon would answer a different question ("what does the native side do?"), and the report says so.

**Where it stopped:** after the boundaries step, with the flow complete. Disassembly would have added detail without adding an answer.

## 2. A crash: why does the app freeze on a large file?

**Request:** "Opening a file over 10 MB freezes the UI for five seconds."

DEBUG. First reproduce it, then find the code that produces the symptom. Searching for the open-file path:

```sh
rg -uu 'openFile|readFile|fs\.read' app.asar-extracted/
```

The handler turns out to be `ipcMain.handle('file:open', ...)` calling `fs.readFileSync(path, 'utf-8')` in the main process. That is the answer: a synchronous read on the thread that also drives the window, so the event loop blocks until the file is in memory.

The fix follows from the cause (`fs.promises.readFile`, or move the read to a worker), and there is no reason to profile the rest of the app.

**Where it stopped:** at the root cause. A stack trace or a profiler would have reached the same line with more effort, which is why the skill says to try the cheap trace first.

## 3. A security question: is this Electron app safe to run?

**Request:** "Is this app safe?" That is SECURITY, and the honest first step is scope: static analysis of a local copy, no exploitation. If the app isn't yours, confirm you're allowed to look.

The surface map comes from a handful of patterns that decide most Electron audits:

```sh
rg -uu 'webPreferences|BrowserWindow|nodeIntegration|contextIsolation' app/main.js
npx @electron/fuses read --app App.app
```

Suppose that turns up three windows. The main window has `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`. A settings window has `contextIsolation: false` but still no Node integration. A helper window has `nodeIntegration: true`, `webSecurity: false`, and loads a URL that comes from user input.

That last one is the finding, and it doesn't need a deeper pass to be worth reporting: remote content in a window with Node access is remote code execution. The fuses also allow Node CLI arguments, which is a second, smaller finding. Checking entitlements shows network and user-selected file access, which sets the impact.

The rest of the app does not get audited. Once a critical path is established, reading five hundred more functions changes the report's length, not its conclusion.

**Where it stopped:** after the trust boundaries and capability steps, with two findings and a clear impact statement. The suspected stored XSS in note titles would have died at the same step: checking the code shows `textContent` and an explicit escaping comment.

## 4. A rebuild: clone the command palette

**Request:** "I want a command palette like VS Code's."

REBUILD, and the goal is behaviour, not their React tree.

Start by using the thing. `Cmd+Shift+P` opens an input with a filtered list; typing `file op` matches "File: Open"; typing `flop` also matches it, so the matching is fuzzy rather than prefix; capital letters seem to count more (`FiOp` still matches); recently used commands float to the top. Enter runs, Escape closes.

That behaviour matrix is enough to write the feature: a command registry, a fuzzy scorer with a bonus for capital-letter matches, a recency multiplier, and a virtualised list. Then test it against the original: same queries, same ordering, same edge cases.

The only reason to look at VS Code's source would be to name the algorithm, and even then you'd read about fuzzy matching, not copy their module. Anything beyond that is a different goal (COMPARE, or curiosity).

**Where it stopped:** at functional parity. No bundle extraction, no decompilation, no copied assets.

## 5. A version diff: what changed between two releases?

**Request:** "What's different in 118 compared with 117?"

COMPARE, and the first decision is the size of the gap. One major version means the full inventory: metadata, symbols, strings, and code only where those disagree.

```sh
strings -a chrome-117 | rg -i 'version' > /tmp/v117
strings -a chrome-118 | rg -i 'version' > /tmp/v118
diff -u /tmp/v117 /tmp/v118
```

The metadata diff shows the engine and Blink version bumps. A symbols diff on an unstripped build (or function names recovered from the binary when it is stripped) lists new exports such as GPU and container-query entry points, and strings show the matching feature names and a deprecation notice.

Those three inventories usually answer the question on their own. The new symbols are public APIs with specifications, so disassembling their implementations adds nothing a user would call an answer.

**Where it stopped:** after the strings diff. For a patch release the same route would skip metadata and strings and go straight to the handful of changed symbols; that is the gap rule doing its job.

## 6. A metadata question: what built this binary?

**Request:** "Which Go version was used to build this?"

EXTRACT. One command:

```sh
go version -m ./binary
```

The output gives the toolchain (`go1.21.5`), the module path and version, every dependency with its version, and build settings such as `CGO_ENABLED` and `GOARCH`. That is the answer, formatted as a small table, and the investigation is over.

There is a temptation to keep going, because the binary is right there and `nm` is one command away. The skill's answer is no: this goal has no use for symbols, and reading them would be work with no question behind it.

**Where it stopped:** immediately after the metadata dump. The measured version of this task, on three real gh releases, is the first section of [WALKTHROUGH.md](WALKTHROUGH.md).

## What the six have in common

Every example ends earlier than a curious engineer would stop. That isn't laziness; it's the answer to "what would I do with more information?" being "nothing that changes the report". The route names the deliverable up front, so the investigation has a finish line, and the depth ladder makes the cheapest path to that line explicit.

The other shared trait is honesty about evidence. A configuration flag is seen; the exploitability that follows from it is inferred; the part you didn't trace is unknown. Reports that keep those apart are shorter and more useful than reports that blur them.
