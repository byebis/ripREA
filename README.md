# ripREA

**One skill for reverse engineering: the fastest technique per target, plus the decisions that keep the investigation cheap.**

A single `SKILL.md` answers both questions an agent actually faces - **"which command?"** and **"when do I stop?"** - in ~4.3k tokens: one install, one file, no stack of skills to load.

## Why one file

A tool catalog without decisions produces two failure modes:

- **Over-investigation:** read every symbol when one `strings` grep would answer
- **Under-investigation:** give up at the surface when one more layer holds the answer

A strategy layer without tools produces a third: the agent knows it should trace a boundary, but not the command that reaches it.

Written as separate documents, the catalog and the strategy have to agree by hand: the steps guess at commands, the catalog knows nothing about depth, and shared rules get repeated until they diverge. Here every strategy step names the router row or the command that executes it, and shared rules appear once.

## What's inside

1. **Goal-first routing** - before touching a tool, fix the goal and its deliverable:

| Goal | Question | Deliverable | Stop when |
|---|---|---|---|
| FEATURE | How does X work? | Flow trace UI/API → effect, cited | Every hop evidenced |
| DEBUG | Why does X fail? | Trigger → root-cause chain, minimal fix | Cause explains the symptom |
| REBUILD | Clone X | Behavior contract + parity checks | Functional parity |
| SECURITY | What are the risks? | Findings by severity, each with a trigger path | Threat model done |
| COMPARE | v1 → v2? | Delta table | Delta documented |
| EXTRACT | Versions, deps? | Metadata table + command per field | All fields found |

2. **One depth ladder** - L1 surface (strings, exports, imports) → L2 boundaries (component and data-flow map) → L3 logic (implementation) → L4 deep (disassembly, ~10× the cost, needs the user's go-ahead).

3. **Six strategies** - each with steps, branch logic (`crash with stack` / `wrong output` / `silent failure` / `cannot execute`), a stop condition and a worked example.

4. **Router** - the fastest path per target: Electron, web bundles, Hermes/bytenode, Mach-O, native logic, ELF/PE, Go, Rust, .NET, APK, Flutter, WASM, SQLite, plist/asset catalogs, pcap/HAR, firmware, websites, running processes.

5. **First move** - one batched reconnaissance call for Mach-O, Electron and Go, ready to paste.

6. **Speed patterns, escalation, safety** - literal-pool string tracing, import maps, inventory diffs before code diffs; minified → sourcemap, stripped → symbols, packed → memory dump, obfuscated → decrypt routine; authorization, copies only, black-box rebuilds.

7. **Report contract** - every claim marked **seen** (cited) / **inferred** (reasoning stated) / **unknown** (honest), and no SECURITY finding "confirmed" without a trigger path.

## Size, measured

**15 870 B, 2 472 words, 167 lines, 4 470 tokens** (tiktoken cl100k) - the whole tool catalog plus the routing that makes it actionable, in one file.

*Measured 2026-11-05 with `wc -c -w -l` and tiktoken cl100k; reproduce with `python -c "import tiktoken;print(len(tiktoken.get_encoding('cl100k_base').encode(open('SKILL.md').read())))"`. Tokenizers differ slightly by model. CI fails above 17 500 B / 4 900 tokens, so every addition needs a matching cut.*

## Example: same answer, one third of the reading

**Request:** "How does the search feature in Notion work?"

```
Goal: FEATURE. Ladder: L1 → L2 → L3.
1. rg 'search'          → 'search:query' IPC channel        (L1, seconds)
2. handler trace        → SearchIndex class → .node addon   (L2, seconds)
3. nm SearchIndex.node  → fulltext_search export            (L3, seconds)
STOP: UI → IPC → native full-text search. Answer cited, no disassembly.
```

Without routing, the same agent extracts the asar, reads 5000 lines of `main.js`, finds the string, then disassembles the addon: ~7000 lines read for a trace the ladder reaches in three probes. The measured, reproducible version of that claim - four tasks on real binaries (gh CLI, a Joplin Electron audit) with committed output and timings - is [WALKTHROUGH.md](WALKTHROUGH.md).

## Install

```sh
npx skills add byebis/ripREA
```

Or manually:

```sh
git clone https://github.com/byebis/ripREA
mkdir -p ~/.claude/skills && cp -R ripREA ~/.claude/skills/
```

Then:

```
/ripREA How does the export feature in Figma work?
```

The agent identifies the goal, climbs the ladder only as far as the deliverable needs, and reports with evidence.

## Documentation

- **[QUICKSTART.md](QUICKSTART.md)** - get started in 60 seconds
- **[SKILL.md](SKILL.md)** - the skill itself (routing + strategies + tool router)
- **[EXAMPLES.md](EXAMPLES.md)** - 6 strategy walkthroughs (illustrative)
- **[WALKTHROUGH.md](WALKTHROUGH.md)** - measured run on real binaries (gh CLI, Joplin): output, timings, benchmark
- **[VALUE.md](VALUE.md)** - why the decision layer exists and when it pays off

## Extending ripREA

1. Add the goal type to the routing table in `SKILL.md`
2. Add its strategy (steps, branches, stop condition, one example)
3. Add any new target rows to the router

Pull requests welcome for new goal types (AUDIT, MIGRATE, OPTIMIZE), domain specializations (IoT firmware, mobile apps, malware) and additional targets.

CI enforces the size budget, so additions need a matching cut or a justified budget bump in `.github/workflows/checks.yml`.

## License

MIT.