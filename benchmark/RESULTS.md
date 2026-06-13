# agentmap — token-savings benchmark results

**Headline: 84.2% fewer tokens** to perform three common "understand the
codebase" tasks when an agent queries agentmap instead of reading raw files with
`cat` / `grep` / `find`. Measured on a real repo (nalarx), reproducible with one
command. Every number below is captured tool output from this run — no
hand-tuned figures.

## Captured run (`bench.mjs`)

```
agentmap token-savings benchmark
repo: /Users/raymondchins/Desktop/Code/nalarx
env:  node v26.3.0, 78 mapped files, HEAD 5fbc953
est:  tokens = chars / 4

Scenario                                  Baseline tokagentmap tok  Saved %
---------------------------------------------------------------------------
A. Understand file deps (app/bari/types.t         2475         597    75.9%
B. Find symbol (LocaleString)                     1667          22    98.7%
C. Repo overview (tree + cat 3 hub files)         5420         891    83.6%
---------------------------------------------------------------------------
TOTAL                                             9562        1510    84.2%

HEADLINE: 84.2% fewer tokens (9562 -> 1510) across 3 scenarios.
```

Deterministic: three consecutive runs all reported `84.2% (9562 -> 1510)`.

## What each scenario compares

The benchmark contrasts the bytes an agent must pull into context for a task,
via the naive shell approach vs the equivalent single agentmap query.

| # | Task | Baseline (naive agent) | agentmap query |
|---|------|------------------------|----------------|
| A | Understand a file's dependencies | `cat <hub-file>` + `grep -rl <basename> .` (read the file + find who imports it) | `repomap.mjs --any <hub-file>` (exports + imports + dependents, no source body) |
| B | Find where a symbol lives | `grep -rn <symbol> .` (every line that mentions it) | `repomap.mjs --find <symbol>` (definition site(s) only) |
| C | Get a repo overview to start | `find . -name '*.ts*'` (file tree) + `cat` top-3 hub files | `repomap.mjs --map` (token-budgeted ranked symbol digest) |

Targets are **auto-derived from the repo's own map** (top hub file, top-ranked
exported symbol, top-3 hub files for the overview), so the same script runs on
any ts-morph-mappable repo, not just nalarx. On nalarx the auto-picked targets
were: hub file `app/bari/types.ts`, symbol `LocaleString`.

## Reproduce it

```bash
# macOS / Linux. Node >= 18 (tested on v26.3.0). agentmap deps already installed
# (single dep: ts-morph@28). nalarx must be a git repo (it is) for git-grep paths.
eval "$(/opt/homebrew/bin/brew shellenv)"          # macOS PATH for node; skip on Linux
node /Users/raymondchins/Desktop/Code/agentmap/benchmark/bench.mjs \
     /Users/raymondchins/Desktop/Code/nalarx
```

Run it against any other repo by passing a different path (defaults to cwd):

```bash
node /Users/raymondchins/Desktop/Code/agentmap/benchmark/bench.mjs /path/to/repo
```

The script is **zero-dependency** (only `node:child_process` / `node:path`). A
machine-readable `@@JSON@@{...}` footer is appended for CI/scripting.

## Secondary: digest-vs-dump (the `--map` extreme)

Scenario C above uses a *fair* overview baseline (tree + 3 hub files). The
worst-case baseline an agent might hit is dumping the **entire** source into the
prompt. Against that, `--map` looks even better — measured this run:

```bash
cd /Users/raymondchins/Desktop/Code/nalarx
git ls-files '*.ts' '*.tsx' '*.mjs' '*.cjs' '*.js' '*.jsx' | xargs wc -c | tail -1
#   728879 total chars  ≈ 182,220 tokens  (chars/4)
node /Users/raymondchins/Desktop/Code/agentmap/repomap.mjs --map | wc -c
#   3566 chars          ≈ 891 tokens
#   1 - (891 / 182220)  =  99.5% reduction vs full-source dump
```

This 99.5% is the eye-catching number but the *less honest* comparison (no
competent agent dumps the whole repo). The 84.2% three-scenario figure is the
conservative, headline result.

## Build / refresh cost (measured this run)

```bash
ls -l .claude/repomap.json                # 49,685 bytes persisted map (never sent to model)
rm -f .claude/repomap.json
time node repomap.mjs        # cold build (parse + PageRank + symbol graph) → 1.20s real
time node repomap.mjs --hubs # warm (cached map, clean tree)               → 0.24s real
```

## Environment (as captured)

- **node** v26.3.0 ; **ts-morph** 28.0.0 (agentmap's single dependency)
- **repo** `/Users/raymondchins/Desktop/Code/nalarx` @ HEAD `5fbc953`
- **mapped files** 78 (TS/TSX/JS/MJS/CJS that agentmap's ts-morph pass sees)
- **tracked files total** 113; **source files** 77 (`git ls-files`)
- **raw source** 728,879 chars (~182K tok) across the source files
- **agentmap** `/Users/raymondchins/Desktop/Code/agentmap/repomap.mjs`
  (resolved relative to `bench.mjs`; ts-morph resolves from the agentmap dir)

## Honest caveats — read before quoting the number

1. **Token estimate is `chars / 4`.** A rough heuristic (the same one agentmap
   uses internally), not a real BPE tokenizer count. It applies to **both** sides
   of every comparison, so the *ratio* / saved-% is far more robust than the
   absolute token figures. The raw char counts (in the `@@JSON@@` footer) are the
   ground truth; treat token columns as ±10%.

2. **Single repo, single commit.** Results are from one mid-size Next.js app
   (78 files) at one point in time. Savings scale with repo size for scenarios B
   and C (more files to grep / list) and with fan-in for scenario A (more
   importers to find). A tiny repo shows smaller savings; a large monorepo,
   larger. Re-run on your own repo before generalizing.

3. **The grep baseline is deliberately *fair*, not worst-case.** It prunes
   `node_modules` / `.next` / `.git` (`--exclude-dir`) to mirror a competent
   agent. A naive `grep -rn <symbol> .` *without* those excludes hit minified
   bundles and produced ~1.56M baseline tokens for scenario B alone — a dishonest
   ~100% "saving." We rejected that; 84.2% is the prune-the-junk number.

4. **Scenario equivalence is approximate.** agentmap returns *structured*
   answers (definition sites, dependents, ranked digest); the shell baselines
   return *raw* lines/source the agent must still parse. The comparison is "bytes
   into context for the same question," which favors agentmap partly because it
   has already done the parsing — that is the value prop, not a like-for-like
   algorithmic comparison.

5. **What's NOT measured here.** Answer *quality* / completeness, wall-clock
   agent time, and end-to-end "did the agent finish the task faster." This
   benchmark measures context-token volume only. The build-cost section above is
   the one-time amortized price (auto-refreshed on post-commit).

## Files

- benchmark script: `/Users/raymondchins/Desktop/Code/agentmap/benchmark/bench.mjs`
- this file: `/Users/raymondchins/Desktop/Code/agentmap/benchmark/RESULTS.md`
- tool under test: `/Users/raymondchins/Desktop/Code/agentmap/repomap.mjs`
