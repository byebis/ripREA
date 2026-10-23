# ripREA Quick Start

Get started with ripREA in 60 seconds.

## Installation

```sh
# Recommended: Install both
npx skills add aicodeking/leanrea    # Tools
npx skills add byebis/ripREA         # Strategy

# Or manual
git clone https://github.com/byebis/ripREA
mkdir -p ~/.claude/skills && cp -R ripREA ~/.claude/skills/
```

## Basic Usage

Just prefix your request with `/ripREA`:

```
/ripREA How does the search work in this app?
```

The agent will automatically:
1. Identify goal type (FEATURE)
2. Apply appropriate strategy
3. Stop at the right depth

## The 6 Goal Types

Remember these when asking questions:

| Your Question | Goal Type | What You Get |
|---------------|-----------|--------------|
| "How does X work?" | **FEATURE** | Data flow trace |
| "Why does X fail?" | **DEBUG** | Root cause |
| "I want to clone X" | **REBUILD** | Behavior model |
| "Is X safe/vulnerable?" | **SECURITY** | Threat assessment |
| "What changed in v2?" | **COMPARE** | Delta only |
| "What version/deps?" | **EXTRACT** | Metadata only |

## Quick Examples

### Feature Investigation
```
/ripREA How does the export feature work in this Electron app?

→ Agent identifies FEATURE goal
→ L1: Searches for 'export' in IPC channels (10s)
→ L2: Traces handler to implementation (30s)  
→ L3: Reads export logic (1-2m)
→ STOPS: Complete flow documented
```

### Debugging
```
/ripREA Why does it crash when opening settings?

→ Agent identifies DEBUG goal
→ Finds error in crash log
→ Traces to null dereference
→ Finds uninitialized variable
→ STOPS: Root cause identified
```

### Security Audit
```
/ripREA Is this Electron app safe?

→ Agent identifies SECURITY goal
→ Checks contextIsolation, nodeIntegration
→ Finds critical vuln in one window
→ STOPS: Critical finding documented
```

### Rebuilding
```
/ripREA Clone the command palette from VS Code

→ Agent identifies REBUILD goal
→ Tests behavior (fuzzy search, recency)
→ Identifies algorithm (fuzzy + boost)
→ Implements without reading VS Code's code
→ STOPS: Functional parity achieved
```

## Key Concepts

### Progressive Depth
Don't read everything. Climb only as high as needed:

```
L1 (Surface)    → strings, APIs, entry points
L2 (Boundaries) → component map, data flow  
L3 (Logic)      → implementation details
L4 (Deep)       → disassembly (rare)
```

**Most questions stop at L1 or L2.**

### Stop Conditions
Know when to stop:

1. ✓ Question answered
2. ✓ Boundary hit ("calls library X")
3. ✓ 3+ attempts, no progress (re-route)

### Efficiency Rules
From leanrea, still apply:

- Work on copy (`cp -c`)
- Batch probes (one shell call with `;`)
- Never `cat` minified files (use `rg -o` to slice)
- Use `rg -uu` for extracted trees
- Mark claims: seen/inferred/unknown

## Common Patterns

### "How does X work?"
→ FEATURE strategy → L1→L2→L3 → stop when flow complete

### "Why does X break?"
→ DEBUG strategy → reproduce → trace back → root cause

### "Clone X"
→ REBUILD strategy → black-box behavior → implement → verify

### "Is X safe?"
→ SECURITY strategy → surface → boundaries → impact

### "What changed?"
→ COMPARE strategy → diff inventories → selective code diff

### "What version?"
→ EXTRACT strategy → one-shot metadata → done

## Tips

### 1. Be specific with goal
❌ "Check out this app"  
✓ "How does search work?" (FEATURE)  
✓ "Why does it crash?" (DEBUG)  
✓ "Is it safe?" (SECURITY)

### 2. Let agent stop early
Don't push for more depth if answer is complete.

### 3. Use with leanrea
ripREA strategy + leanrea tools = best results

### 4. Trust the boundaries
"It calls library X" is a valid answer. Don't force deeper.

## When It Helps Most

✅ Ambiguous requests ("understand how X works")  
✅ Time pressure (need answer fast)  
✅ Risk of over-investigation (large codebase)  
✅ Unknown depth (not sure how deep to go)

## When It Helps Less

⚠️ Precise tool requests ("extract this asar")  
⚠️ Expert users (already know the strategy)  
⚠️ Simple metadata ("get version")

## Troubleshooting

**Agent reads too much code?**
→ Remind: "Use ripREA strategy, stop at appropriate depth"

**Agent stops too early?**
→ Specify: "Go to L3 (logic level)" or "I need implementation details"

**Wrong goal type identified?**
→ Correct: "This is a DEBUG task, not FEATURE"

**Need more depth?**
→ Ask: "Go deeper, previous answer incomplete"

## Next Steps

- Read [WALKTHROUGH.md](WALKTHROUGH.md) for a measured, reproducible run on the gh CLI
- Read [EXAMPLES.md](EXAMPLES.md) for detailed walkthroughs
- See [COMPARISON.md](COMPARISON.md) for before/after analysis  
- Check [VALUE.md](VALUE.md) for understanding the benefits
- Reference [SKILL.md](SKILL.md) for complete strategy guide

## TL;DR

```
1. Install: npx skills add byebis/ripREA
2. Use: /ripREA <your question>
3. Agent: identifies goal → applies strategy → stops at right depth
4. You: get answer faster with less waste
```

That's it. Start using ripREA now.
