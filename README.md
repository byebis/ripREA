# ripREA

A reverse-engineering skill for coding agents: the fastest local technique for each target, plus the decisions that keep an investigation from ballooning.

A tool collection answers "which command". It says little about "how deep", and an agent that picks a depth by accident usually picks too much: it reads every symbol when one `strings` grep would do, or disassembles a native module whose export list already told the story. The opposite failure is just as common. The agent greps once, finds the entry point, and stops one hop before the answer.

## Why one file

A catalogue without decisions causes over-investigation. A strategy without commands causes under-investigation: the agent knows it should trace a boundary, but not the command that reaches it. Kept as separate documents, the two drift. The steps guess at commands, the catalogue knows nothing about depth, and shared rules get copied until they disagree.

Here each strategy step names the router row that executes it, and shared rules appear once.

## What's inside

**Routing.** Six goals, each with a deliverable and a stop condition:

| Goal | Question | Deliverable | Stop when |
|---|---|---|---|
| FEATURE | How does X work? | Flow trace UI/API → effect | Every hop evidenced |
| DEBUG | Why does X fail? | Trigger → root-cause chain, minimal fix | Cause explains the symptom |
| REBUILD | Clone X | Behaviour contract + parity checks | Functional parity |
| SECURITY | What are the risks? | Findings by severity, each with a trigger path | Threat model done |
| COMPARE | v1 → v2? | Delta table | Delta documented |
| EXTRACT | Versions, deps? | Metadata table + command per field | All fields found |

**One depth ladder.** Surface (strings, exports, imports) → boundaries (component and data-flow map) → logic (implementation, pseudocode) → deep (disassembly or decompilation, roughly ten times the cost, needs the user's go-ahead).

**Six strategies**, each with steps, branch logic for the usual symptoms (`crash with stack`, `wrong output`, `silent failure`, `cannot execute`), a stop condition and a worked example.

**The router: 24 targets.** Electron, including ASAR integrity and fuses; web bundles; Hermes and bytenode; Mach-O; native code logic; ELF and PE; Go; Rust; .NET; Android APKs; Flutter; Unity IL2CPP; packaged Python; Node SEA; Java; iOS IPAs; macOS packages and Windows installers; WASM; SQLite; plists and asset catalogs; pcaps and HARs; firmware; websites; running processes.

**A first move.** One batched reconnaissance call each for Mach-O, Electron and Go, ready to paste.

**The rest.** Speed patterns (literal-pool string tracing, import maps, inventory diffs before code diffs), an escalation ladder (minified → sourcemap, stripped → symbols, packed → memory dump, obfuscated → decrypt routine), safety rules, and a report contract that marks every claim seen, inferred or unknown.

## Size

**18,066 bytes, 2,827 words, 175 lines, 4,982 tokens** (tiktoken cl100k). The catalogue and the routing that makes it usable, in one file.

Measured 2026-11-11 with `wc -c -w -l` and tiktoken; reproduce with `python -c "import tiktoken;print(len(tiktoken.get_encoding('cl100k_base').encode(open('SKILL.md').read())))"`. Other tokenizers differ slightly. CI fails above 20,000 bytes or 5,500 tokens, so every addition needs a matching cut or a deliberate budget bump.

## An example

**Request:** "How does the search feature in Notion work?"

```
Goal: FEATURE. Ladder: L1 → L2 → L3.
1. rg 'search'          → 'search:query' IPC channel        (L1, seconds)
2. handler trace        → SearchIndex class → .node addon   (L2, seconds)
3. nm SearchIndex.node  → fulltext_search export            (L3, seconds)
STOP: UI → IPC → native full-text search. Answer cited, no disassembly.
```

That trace shows the shape of the answer; it is not a recording. The recorded run is [WALKTHROUGH.md](WALKTHROUGH.md): four tasks on real binaries (three gh CLI releases and Joplin 3.7.21), with the exact commands, input checksums and timings, so the run can be repeated instead of trusted.

## Install

```sh
npx skills add byebis/ripREA
```

Or by hand:

```sh
git clone https://github.com/byebis/ripREA
mkdir -p ~/.claude/skills && cp -R ripREA ~/.claude/skills/
```

## Use

```
/ripREA How does the export feature in Figma work?
```

The agent identifies the goal, climbs only as far as the deliverable needs, and reports with evidence. If it picks the wrong route, name the right one; the taxonomy is small enough to correct in a sentence.

## Documentation

- [SKILL.md](SKILL.md) is the skill: routing, strategies, router, rules.
- [EXAMPLES.md](EXAMPLES.md) has six worked examples, one per goal.
- [WALKTHROUGH.md](WALKTHROUGH.md) is the measured run on gh CLI and Joplin.

## Extending ripREA

1. Add the goal to the routing table in `SKILL.md`.
2. Add its strategy: steps, branches, stop condition, one example.
3. Add any new target rows to the router.

Pull requests are welcome for new goals (AUDIT, MIGRATE, OPTIMIZE), domain specialisations (IoT firmware, mobile apps, malware) and additional targets. CI enforces the size budget, so additions need a matching cut or a justified bump in `.github/workflows/checks.yml`.

## License

MIT
