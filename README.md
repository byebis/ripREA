# ripREA

**Context-aware reverse engineering: the right tool at the right depth at the right time.**

ripREA extends [leanrea](https://github.com/aicodeking/leanrea) with strategic guidance: goal routing, depth ladders, stop conditions, a report contract and safety rails - **~2.8k tokens** on top of leanrea's ~2.7k (**~5.5k combined**, measured; see [COMPARISON.md](COMPARISON.md)). It answers the question leanrea doesn't: **"When do I use which technique?"**

## The problem with leanrea

leanrea is fast and efficient, but it's a **tool catalog**. It tells you *how* to use `otool` or `jadx`, but not *when* to stop reading strings and start disassembling, or *whether* you need disassembly at all.

Result: agents over-investigate or under-investigate. They either:
- Read every symbol when a single `strings` grep would answer the question (wasted time)
- Give up at the surface when one more layer would reveal the answer (incomplete)

## What ripREA adds

### 1. Goal-first routing
Before touching tools, identify what you're actually trying to do:
- **FEATURE** - How does X work? (trace data flow)
- **DEBUG** - Why does X fail? (find divergence)
- **REBUILD** - Clone X (black-box behavior)
- **SECURITY** - What are X's risks? (attack surface)
- **COMPARE** - What changed? (diff only)
- **EXTRACT** - Versions? (metadata only)

Each goal type has a different **investigation strategy** and **stop condition**.

### 2. Progressive depth ladders
For each goal type, a clear progression:
```
Level 1: Surface (strings, APIs)        → 10s, 1 call
Level 2: Boundaries (components, flow)   → 30s, 1-2 calls
Level 3: Logic (implementation)          → 2-5m, 2-3 calls
Level 4: Deep (disassembly, decompile)   → only if Levels 1-3 insufficient
```

**Decision points** at each level tell you when to stop or go deeper.

### 3. Tactical guidance
- **When to stop:** Answer found, boundary hit, diminishing returns
- **When to go deeper:** Incomplete answer, obfuscation, explicit user request
- **When to switch tools:** Static exhausted → dynamic, disassembly unreadable → decompiler

### 4. Anti-patterns prevention
- ❌ Reading entire disassembly when strings would suffice
- ❌ Giving up at "it's obfuscated" without trying dynamic analysis
- ❌ Full code review when only symbols changed
- ❌ Rebuilding internals when black-box behavior is enough

### 5. Report contract & safety rails
- Every answer ends with its goal's **deliverable** (flow trace, delta table, threat list...), each claim marked **seen / inferred / unknown**
- Safety rules: confirm authorization before SECURITY work, run dynamic analysis only on copies in a VM/snapshot, keep REBUILD black-box (never copy code/assets/names), and treat L4 disassembly as ~10× cost that needs user confirmation

## Comparison

|  | leanrea | ripREA |
|---|---|---|
| **Tool coverage** | Comprehensive | Quick ref + full via leanrea |
| **Efficiency** | High (direct tools) | Same |
| **Strategic guidance** | ❌ Tool catalog only | ✅ Goal → Strategy → Depth |
| **Decision support** | ❌ Agent must guess | ✅ Explicit when/stop rules |
| **Context awareness** | ❌ One-size-fits-all | ✅ Tailored by goal type |
| **Size (measured 2026-10-23)** | 8.2 KB / ~2.7k tokens | 10.7 KB / ~2.8k tokens |

ripREA is **leanrea + a decision layer paid for in context**: loading both (the recommended setup) costs ~5.5k tokens vs ~2.7k for leanrea alone. All tools remain accessible; ripREA adds the **decision logic** that was missing. Reproduce the numbers with `wc -c -w -l SKILL.md` and any tokenizer - see [COMPARISON.md](COMPARISON.md).

## Example: Before and After

**User request:** "How does the search feature in Notion work?"

### Without strategy (typical agent behavior)
```
Agent: "I should reverse engineer Notion"
1. Extract app.asar ✓
2. Read entire main.js (5000 lines) ✗ wasted
3. Find 'search' string ✓
4. Trace to SearchIndex.node ✓
5. Disassemble SearchIndex.node (2000 lines) ✗ wasted
6. Report: "Uses native FTS, probably SQLite"

Time: 5+ minutes, 5 calls, ~7000 lines read
```

### With ripREA strategy
```
Agent: "Goal: FEATURE. Strategy: Progressive depth L1→L2→L3"
1. rg 'search' → 'search:query' IPC channel (L1, 10s)
2. Find handler → traces to SearchIndex class → calls .node (L2, 30s)
3. nm SearchIndex.node → fulltext_search export (L3, 10s)
STOP: UI→IPC→native FTS. Answer complete.

Time: <1 minute, 3 calls, ~100 lines read
```

**Impact (illustrative scenario): same answer, 5× faster, 70× less code read.** The measured, reproducible run - three real tasks on the gh CLI with committed output and timings - is [WALKTHROUGH.md](WALKTHROUGH.md).

## When to use ripREA vs leanrea

**Use leanrea** if:
- You already know what layer you need (e.g., "give me the Go dependencies")
- The task is narrow and tool-specific (e.g., "extract this asar")
- You're an expert who doesn't need decision guidance

**Use ripREA** if:
- The goal is ambiguous ("understand how X works")
- You need to know when to stop investigating
- You want to avoid over-engineering the investigation
- You're collaborating with an LLM agent that tends to over-read or under-read

## Install

### Recommended: Use with leanrea
```sh
npx skills add aicodeking/leanrea
npx skills add byebis/ripREA
```

This gives you full tool coverage (leanrea) + strategic layer (ripREA).

### Standalone (works, but limited tool reference)
```sh
git clone https://github.com/byebis/ripREA
mkdir -p ~/.claude/skills && cp -R ripREA ~/.claude/skills/
```

### Use it
```
/ripREA How does the export feature in Figma work?
```

The agent will:
1. Identify goal type (FEATURE)
2. Apply progressive depth strategy (L1→L2→L3)
3. Stop at appropriate layer (when trace is complete)
4. Report findings with evidence

**For detailed side-by-side comparisons, see [COMPARISON.md](COMPARISON.md)**

## Documentation

- **[QUICKSTART.md](QUICKSTART.md)** - Get started in 60 seconds
- **[SKILL.md](SKILL.md)** - Complete strategy guide (the actual skill file)
- **[EXAMPLES.md](EXAMPLES.md)** - 6 strategy walkthroughs (illustrative)
- **[WALKTHROUGH.md](WALKTHROUGH.md)** - Measured run on the gh CLI: real output, timings, benchmark
- **[COMPARISON.md](COMPARISON.md)** - Side-by-side: with vs without ripREA
- **[VALUE.md](VALUE.md)** - Why ripREA exists and when it matters most

## Extending ripREA

ripREA is designed to be extended. To add support for new goal types or domains:

1. Add the goal type to the taxonomy in `SKILL.md`
2. Define the investigation strategy (phases, decision points, stop condition)
3. Add domain-specific patterns to the tool router if needed

Pull requests welcome for:
- New goal types (AUDIT, MIGRATE, OPTIMIZE, etc.)
- Domain specializations (IoT firmware, mobile apps, malware)
- Additional tactical patterns

## License

MIT (same as leanrea)

## Credits

Built on [leanrea](https://github.com/aicodeking/leanrea) by aicodeking. ripREA adds the strategic layer that makes leanrea's tool mastery actionable in ambiguous scenarios.
