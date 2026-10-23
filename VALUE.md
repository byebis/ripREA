# Why ripREA? The Value Proposition

## The Core Problem

leanrea solves **"which tool?"** by providing a comprehensive catalog of RE tools and commands.

But it doesn't solve **"when?"** and **"how deep?"**

### Real agent behavior without strategy

**Task:** "How does the search feature work?"

```
Agent reasoning (implicit):
"I should reverse engineer the app."
↓
Extracts everything
↓
Reads everything it finds
↓
Disassembles everything native
↓
Eventually finds the answer buried in 5000+ lines
```

**Result:** Correct but wasteful. Like using a firehose to fill a glass.

---

## What ripREA Adds

### 1. Goal Taxonomy (6 types)

Before touching tools, identify what you're trying to do:

| Goal | Strategy | Stop Condition |
|------|----------|----------------|
| FEATURE | Trace flow | Flow complete |
| DEBUG | Bisect to cause | Root cause found |
| REBUILD | Black-box behavior | Parity achieved |
| SECURITY | Attack surface | Threat model done |
| COMPARE | Diff inventory | Delta documented |
| EXTRACT | Metadata only | All versions found |

Each goal type needs **different investigation depth**.

### 2. Progressive Depth Ladders

Instead of "read everything," climb only as high as needed:

```
Level 1: SURFACE (strings, APIs)           10s, 1 call
         ↓ stop here if just checking existence
Level 2: BOUNDARIES (components, flow)      30s, 1-2 calls  
         ↓ stop here if mapping architecture
Level 3: LOGIC (implementation)             2-5m, 2-3 calls
         ↓ stop here if rebuilding behavior
Level 4: DEEP (disassembly, decompile)      only if L1-3 insufficient
```

Most questions are answered at L1 or L2.

### 3. Explicit Stop Conditions

**Three stop signals:**
1. **Question answered** - Don't dig for completeness
2. **Boundary hit** - "Calls library X" is a valid answer
3. **Diminishing returns** - 3+ calls, no progress? Wrong layer, re-route

Without these, agents keep digging.

### 4. Branch Logic

Decision trees for common scenarios:

**DEBUG example:**
```
Is it a crash?
  → YES: Start with stack trace, check null/bounds
  → NO: Is it wrong output?
      → YES: Diff actual vs expected, trace divergence
      → NO: Silent failure → dynamic trace (lldb/frida)
```

**REBUILD example:**
```
Is it simple CRUD?
  → YES: Steps 1-2 enough (API + behavior)
  → NO: Is it complex algorithm?
      → YES: Step 3 to identify algo, research separately
      → NO: Proprietary protocol → deeper peek needed
```

This replaces guesswork with clear decision paths.

---

## Real Impact: Side-by-side

### Scenario: "Why does my Electron app crash when opening settings?"

#### Without ripREA
```
Turn 1: Extract app.asar
Turn 2: List all files → 150 files
Turn 3: rg 'setting' → 200+ matches
Turn 4: Read main.js → 3000 lines
Turn 5: Read settings-window.js → 800 lines
Turn 6: Find crash mention in code review comment
Turn 7: Read SettingsController.js → 500 lines
Turn 8: Read preload script → 300 lines
Turn 9: Realize need to check native module
Turn 10: nm settings.node → 80 symbols
Turn 11: Disassemble loadPreferences function
Turn 12: Find null dereference
Turn 13: Trace where prefsCache initialized
Turn 14: Find it's only set if user logged in
Turn 15: Report root cause

Time: 15 turns, ~12 minutes
Lines examined: ~5000+
Result: ✓ Found cause, but exhausting
```

#### With ripREA (DEBUG strategy)
```
Turn 1: Goal=DEBUG. Reproduce: crash on settings tap.
        Crash log → null deref in loadPreferences
Turn 2: Read loadPreferences (from disasm) → accesses prefsCache without check
Turn 3: When is prefsCache init? grep constructor → only if logged in
ROOT CAUSE: prefsCache uninitialized for logged-out users

Time: 3 turns, ~90 seconds  
Lines examined: ~100
Result: ✓ Found cause efficiently
```

**Impact: 80% fewer turns, 98% less reading, 8× faster**

---

## The Size Miracle

```
leanrea:  8.3 KB (1294 words, 57 lines)
ripREA:   8.25 KB (1263 words, 146 lines)
```

**How is ripREA smaller?**

1. Removed tool duplication (points to leanrea instead)
2. Condensed examples (inline, not prose)
3. Dense formatting (tables, not paragraphs)
4. Focused on decisions, not explanations

**ripREA includes:**
- 6 goal type strategies
- 4-level depth ladder with stop conditions  
- Branch logic for common scenarios
- Quick tool reference (10 most common)
- Tactical rules (when to stop/go/switch)
- Meta-adaptation guidance

**All at the same size as the tool catalog alone.**

---

## When Does ripREA Matter Most?

### High Impact Scenarios

1. **Ambiguous requests**
   - "Understand how X works" → Which aspect? How deep?
   - ripREA: Routes to FEATURE, applies depth ladder

2. **Time pressure**
   - Need answer in minutes, not hours
   - ripREA: Optimizes path to minimum viable answer

3. **Over-investigation risk**
   - Agent might read entire codebase
   - ripREA: Stop conditions prevent waste

4. **Wrong-layer problems**
   - Looking at wrong abstraction level
   - ripREA: Depth ladder guides correct progression

### Lower Impact Scenarios

1. **Precise tool requests**
   - "Extract this asar" → leanrea already optimal
   - ripREA: Adds nothing (tool choice is obvious)

2. **Expert users**
   - Already know what layer to check
   - ripREA: Strategy is already implicit

3. **Metadata-only tasks**
   - "Get Go version" → One command
   - ripREA: Both use same command, same result

---

## The Mental Model Shift

### leanrea's mental model
```
User question
    ↓
Agent: "I'll use tools from leanrea"
    ↓
Picks tools (implicitly chooses depth)
    ↓
Reports results
```

**Risk:** Agent might choose wrong depth (too shallow or too deep)

### ripREA's mental model
```
User question
    ↓
Agent: "What's the goal type?"
    ↓
Route to goal strategy (FEATURE/DEBUG/etc)
    ↓
Apply depth ladder with stop conditions
    ↓
Stop when condition met
    ↓
Report results
```

**Benefit:** Explicit routing, defined stopping, minimal waste

---

## The Compound Effect

**Single task:** ripREA saves 2-10 minutes  
**Daily usage:** 10 tasks × 5 min avg = 50 minutes saved  
**Weekly:** 4 hours saved  
**Monthly:** 16 hours saved

**More importantly:**
- Less context window waste (fewer tokens read)
- Less user frustration (faster answers)
- Less agent confusion (clear decision paths)

---

## Recommendation: Use Both

```sh
npx skills add aicodeking/leanrea    # Tool catalog
npx skills add <org>/ripREA          # Strategic layer
```

**Division of labor:**
- **leanrea** = Comprehensive tool reference + speed patterns
- **ripREA** = Goal routing + depth strategy + stop conditions

Together: **Complete RE decision framework**

Alone:
- leanrea: Great tools, no strategy (agent guesses)
- ripREA: Clear strategy, limited tools (quick ref only)

---

## The Bottom Line

**Same size. More value. Zero tradeoff.**

ripREA proves you can add strategic guidance without bloat by:
1. Identifying what actually matters (goal → depth → stop)
2. Removing duplication (tool catalog stays in leanrea)
3. Optimizing format (tables and examples, not prose)

The result: **A decision layer that makes leanrea's tools actionable in ambiguous scenarios.**

That's why ripREA exists.
