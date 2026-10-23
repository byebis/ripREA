# Measured walkthrough: ripREA on gh CLI

A reproducible, real run of the ripREA decision layer - no composites, no estimates. Every command below was executed on **2026-10-23** (Linux 6.1.102, x86_64); outputs are trimmed copies of the actual output, and the timings come from `date +%s%N` around each command (single run).

The before/after scenarios in [COMPARISON.md](COMPARISON.md) and [EXAMPLES.md](EXAMPLES.md) are illustrative composites. **This file is the measured one.**

## Setup

```sh
# Three releases of gh (open source: https://github.com/cli/cli)
mkdir bench && cd bench
curl -sSL -o gh_2.63.2.tgz  https://github.com/cli/cli/releases/download/v2.63.2/gh_2.63.2_linux_amd64.tar.gz
curl -sSL -o gh_2.103.0.tgz https://github.com/cli/cli/releases/download/v2.103.0/gh_2.103.0_linux_amd64.tar.gz
curl -sSL -o gh_2.103.1.tgz https://github.com/cli/cli/releases/download/v2.103.1/gh_2.103.1_linux_amd64.tar.gz
tar -xzf gh_2.63.2.tgz && tar -xzf gh_2.103.0.tgz && tar -xzf gh_2.103.1.tgz

# EXTRACT needs any Go toolchain (it only reads build info; nothing is rebuilt)
curl -sSL -o go.tgz https://go.dev/dl/go1.23.4.linux-amd64.tar.gz && tar -xzf go.tgz
export PATH=$PWD/go/bin:$PATH
```

Verify you are looking at the same inputs:

| binary | md5 |
|---|---|
| gh 2.63.2 | `667344610aa7767b71408278123b6168` |
| gh 2.103.0 | `a7927c45b1b2e2e2f6f287e1aa59dc45` |
| gh 2.103.1 | `21af56870f298108fa29457230ed4bf1` |

Tools used: `rg`, binutils (`nm`, `strings`, `comm`), any Go >= 1.18 toolchain. On macOS install coreutils for `comm`.

---

## Task 1 - EXTRACT: "What built gh v2.103.1?"

**Routing:** metadata question → **EXTRACT** → one shot, stop immediately, no code ever.

```sh
file gh_2.103.1_linux_amd64/bin/gh            # route by magic first
go version -m gh_2.103.1_linux_amd64/bin/gh    # one-shot build metadata
```

Actual output (trimmed; the full dump is 174 lines):

```
gh_2.103.1_linux_amd64/bin/gh: go1.27.3
	path	github.com/cli/cli/v2/cmd/gh
	mod	github.com/cli/cli/v2	v2.103.1
	dep	charm.land/bubbles/v2	v2.2.1	h1:Fq1+...
	dep	charm.land/bubbletea/v2	v2.1.0	h1:A9fj...
	dep	charm.land/glamour/v2	v2.0.2-0.20261015065741-010543cac1a0	h1:pnfN...
```

**Deliverable (report contract):**

| Field | Value | Mark |
|---|---|---|
| Toolchain | go1.27.3 | **seen** |
| Module | github.com/cli/cli/v2 v2.103.1 | **seen** |
| Dependencies | 160 | **seen** |
| TUI stack | charm.land/bubbletea v2 family (the move off github.com/charmbracelet/* is **inferred** from Task 2's dep delta) | **seen / inferred** |

**STOP.** Disassembly is not allowed to enter this goal.

**Measured:** 2 commands · 71 ms (file 62 ms + `go version -m` 9 ms) · 174 lines produced · ~6 lines in the answer · 0 lines of code read.

---

## Task 2 - COMPARE: "What changed in gh 2.63.2 → 2.103.1?"

**Routing:** version diff → **COMPARE**. Gap is 40 minor releases → wide → full inventory (steps 1-3); step 4 only if the inventory leaves a question open.

### Step 1 - metadata diff

```sh
go version -m gh_2.63.2_linux_amd64/bin/gh  > old.txt   # 159 lines
go version -m gh_2.103.1_linux_amd64/bin/gh > new.txt   # 174 lines
# delta by dependency name + version (awk on tab fields, then comm)
```

```
go: go1.22.6 → go1.27.3
deps of 160 in new: added=58  removed=43  updated=76
```

```
+ charm.land/bubbletea/v2, charm.land/lipgloss/v2, ...   (TUI stack moved off github.com/charmbracelet/*)
- github.com/charmbracelet/glamour, github.com/charmbracelet/lipgloss, ...
~ github.com/cli/go-gh/v2     v2.11.1 → v2.17.0
~ github.com/docker/cli       v27.1.1+incompatible → v29.7.2+incompatible
```

### Step 2 - symbols → no symbols → switch tools (tactical rule)

```sh
nm gh_2.103.1_linux_amd64/bin/gh
# nm: gh_2.103.1_linux_amd64/bin/gh: no symbols      (release binary is stripped)
```

Stripped Go binaries keep `pclntab` readable through `strings`, so the inventory substitutes for symbols:

```sh
strings -a gh_2.63.2_linux_amd64/bin/gh  | rg -o 'github\.com/cli/cli/v2/pkg/cmd/[a-z0-9-]+' | sort -u > dirs_old
strings -a gh_2.103.1_linux_amd64/bin/gh | rg -o 'github\.com/cli/cli/v2/pkg/cmd/[a-z0-9-]+' | sort -u > dirs_new
comm -13 dirs_old dirs_new    # added command areas
comm -23 dirs_old dirs_new    # removed
```

```
old=31  new=39  added=8  removed=0
github.com/cli/cli/v2/pkg/cmd/accessibility
github.com/cli/cli/v2/pkg/cmd/agent-task
github.com/cli/cli/v2/pkg/cmd/copilot
github.com/cli/cli/v2/pkg/cmd/discussion
github.com/cli/cli/v2/pkg/cmd/licenses
github.com/cli/cli/v2/pkg/cmd/preview
github.com/cli/cli/v2/pkg/cmd/send-telemetry
github.com/cli/cli/v2/pkg/cmd/skills
```

### Step 3 - strings: user-facing domains

```sh
strings -a <bin> | rg -o 'https?://[a-zA-Z0-9._-]+' | sort -u   # then comm against the other binary
```

```
old=45  new=70  added=33  removed=8
https://accessibility.github.com
https://agentskills.io
https://cdn.jsdelivr.net
https://docs.docker.com
https://learn.microsoft.com
...
```

Raw `strings` output glues neighboring literals (`https://githubstatus.comcalled`), so domain deltas are **leads, not findings** - mark them **inferred** until confirmed.

### Answer (deliverable: delta table)

| Delta | Value | Mark |
|---|---|---|
| Toolchain | go1.22.6 → go1.27.3 | **seen** |
| New command areas | accessibility, agent-task, copilot, discussion, licenses, preview, send-telemetry, skills (0 removed) | **seen** |
| Dependency churn | 58 added / 43 removed / 76 updated of 160 | **seen** |
| Notable moves | charmbracelet/* → charm.land/*, go-gh v2.11.1 → v2.17.0, docker v27 → v29 | **seen** |
| New domains | 33 (accessibility.github.com, agentskills.io, ...) | **inferred** |

**STOP:** the inventory explains the capability delta. Disassembling the new code (L4) would add implementation detail, not answers - only go deeper if the user asks.

**Measured:** 8 commands · ~2.1 s compute · 333 metadata lines + inventory sets (31/39 dirs, 45/70 domains) · 4 bullets in the report · 0 lines disassembled.

---

## Task 3 - COMPARE, patch gap: "What changed in 2.103.0 → 2.103.1?"

**Routing:** patch gap → strategy says **steps 2+4 only** (symbols + changed functions); skip metadata and full strings inventories.

```sh
# step 2 (adapted for a stripped binary): command inventory
comm -3 dirs_21030 dirs_21031 | wc -l

# step 4 (adapted): function names from pclntab instead of disassembly
strings -a <bin> | rg '^github\.com/cli/cli/v2(/[a-z0-9._-]+)*\.[A-Z][A-Za-z0-9_]*$' | sort -u
comm -13 funcs_21030 funcs_21031    # added
comm -23 funcs_21030 funcs_21031    # removed
```

```
cmd-dir delta = 0
function names: old=752  new=752  added=0  removed=0
```

The binaries are **not** identical (`a7927c45…` vs `21af5687…`), so the patch changed bodies/constants only.

**STOP (budget rule):** no observable capability delta at inventory depth. L4 disassembly costs ~10× more and is only justified if the user wants exact patch notes - ask first.

**Measured:** 4 commands · ~1.9 s · 0 new symbols, 0 new command areas.

---

## Benchmark summary

| Task | Goal | Gap rule | Commands | Compute | Produced | Answer |
|---|---|---|---|---|---|---|
| 1 | EXTRACT | metadata only | 2 | 71 ms | 174 lines | ~6 lines |
| 2 | COMPARE | wide → full inventory | 8 | 2.1 s | 333 lines + 4 inventories | 4 bullets |
| 3 | COMPARE | patch → steps 2+4 only | 4 | 1.9 s | 2 inventories (752+752 names) | 3 lines |

Honesty notes:

- Single measured run of the **ripREA-routed path** (setup/download excluded), timings via `date +%s%N`, Linux x86_64. Your disk and `strings` speed will differ.
- There is **no measured "without ripREA" baseline here**: the before/after numbers in COMPARISON.md and EXAMPLES.md are illustrative composites, and this file does not pretend otherwise. What *is* measured is which steps the strategy skipped - Task 3 never ran the metadata or full-strings diffs and never reached L4 - and how much output the kept steps produced.
- The batching rule (one shell call with `;` markers) folds each task into 1-3 shell calls; the counts above are unbatched commands so each timing stays attributable.
