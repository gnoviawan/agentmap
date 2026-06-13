# agentmap

**The repo map your coding agent is _forced_ to use.**

A queryable, ranked code-relationship map for TypeScript/JavaScript repos — personalized
PageRank importance, Aider-style symbol ranking, a token-budgeted digest, and a single
`--any` router (file → symbol → feature → live git-grep) — wired straight into the agent
loop so it actually gets used, not just published.

<!-- badges (placeholder — wire up once published) -->
[![npm](https://img.shields.io/npm/v/@raymondchins/agentmap)](https://www.npmjs.com/package/@raymondchins/agentmap)
[![CI](https://img.shields.io/badge/CI-pending-lightgrey)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-green)](./LICENSE)
[![node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](#)

> One file, one dependency (`ts-morph`). No vector DB, no embedding API, no server.
> `npx @raymondchins/agentmap --any <query>` and you have a ranked answer.

---

## Why it's different

Most "repo context" tools are one-shot: they pack the repository (or a slice of it) into a
prompt and stop there. agentmap is a **queryable, ranked, and self-refreshing** map that an
agent can interrogate flag-by-flag — and, crucially, is **wired into the agent loop** via a
post-commit auto-refresh and a `PreToolUse` hook that nudges the agent to use the map
*before* it falls back to serial grep.

| | **agentmap** | Aider repo map | RepoMapper | Repomix | code2prompt |
| --- | --- | --- | --- | --- | --- |
| **Ranking algorithm** | Personalized PageRank (file + symbol graphs) | PageRank (graph ranking) | Importance heuristics | None (file order) | None (file order) |
| **Languages** | TS/JS (via ts-morph) | Many (tree-sitter) | Many (tree-sitter) | Language-agnostic (text) | Language-agnostic (text) |
| **Token-budget output** | Yes — `--map [--tokens N]` ranked digest | Yes (built into Aider's context) | Partial | Yes (size caps) | Yes (templates/caps) |
| **Agent-loop integration** | **Yes — post-commit auto-refresh + PreToolUse hook** | In-process (Aider only) | No | No | No |
| **Dependencies** | `ts-morph` only | Python + tree-sitter stack | Python + tree-sitter | Node | Rust binary |
| **Install** | `npx @raymondchins/agentmap` | `pip install aider` | `pip install` | `npx`/global | `cargo`/binary |

What that table is **not** claiming: agentmap is TS/JS-only (the others are multi-language),
and it's a **file-level import graph**, not a full call-site/reference resolver (see
[Scope & limitations](#scope--limitations)). The differentiators are narrow and honest:
**(1)** the `--any` router, and **(2)** the agent-loop wiring. Everything else is table stakes.

---

## Quickstart

No install needed:

```bash
npx @raymondchins/agentmap --any <query>
```

…or run it directly from a checkout:

```bash
node repomap.mjs --any <query>
```

The first run builds and caches the map to `.claude/repomap.json` (add it to
`.gitignore`). Subsequent runs serve the cache when the tree is clean and `HEAD` is
unchanged, and silently rebuild from disk when there are uncommitted `.ts/.tsx/.js/...`
edits — so queries always reflect your in-flight work.

Run with no flag to build + print a one-line summary:

```
$ node repomap.mjs
repomap: 78 files | 3 features | top hub: app/bari/types.ts (deg 25, pr 0.148412)
```

---

## The `--any` router

One flag, no flag-picking. `--any <query>` resolves your query through a cascade and
returns the first layer that hits:

```
--any <query>
   │
   ├─ 1. FILE     exact path → unique basename → unique substring
   ├─ 2. SYMBOL   exported name contains the query (across all files)
   ├─ 3. FEATURE  app/-router feature name contains the query
   └─ 4. CONTENT  live `git grep` (tracked + untracked) — never stale
```

Layers 1–3 read the cached structural map (fast, ranked). Layer 4 is a **live disk read**
via `git grep -F`, so raw strings, copy, Tailwind classes, and config values the structural
graph never indexes still resolve instead of coming up empty.

**File hit** (query resolved to a file → full block):

```
$ node repomap.mjs --any validation
[structure:file] lib/validation.ts  (pr 0.049669)
exports (16): isPayloadTooLarge(FunctionDeclaration), isAllowedOrigin(FunctionDeclaration), …
imports (0): —
dependents (13): app/api/leads/route.ts, app/api/dashboard/login/route.ts, app/api/dashboard/logout/route.ts, …
```

**Content fallback** (no file/symbol/feature match → live git-grep):

```
$ node repomap.mjs --any scoring
[content] 28 lines:
CLAUDE.md:82:    scoring.ts           — pillar scoring + recommendations [508 LOC]
app/bari/bari-flow.tsx:12:import { computeStage1Result, computeStage2Result, getMultiSelect, getRecommendations } from "./scoring"
app/bari/scoring.ts:1:// BARI v2 scoring engine.
docs/ARCHITECTURE.md:55:- **`lib/wib.ts`** — Core business logic (Pillar reveal, scoring, report generation).
…
```

---

## Commands

Every snippet below is **verbatim output** from running agentmap against a real 78-file
Next.js 16 + Supabase repo.

### `--any <q>` — the router (file → symbol → feature → live content)

See [The `--any` router](#the---any-router) above. Default first move for any
"where/what/who" question.

### `--find <q>` — reuse-before-rebuild symbol search

Find every exported symbol whose name contains the query. Use it before writing a new util
or component to check what already exists.

```
$ node repomap.mjs --find rateLimit
find "rateLimit": 10 match
  lib/ratelimit.ts → rateLimitGenerate (FunctionDeclaration)
  lib/ratelimit.ts → rateLimitLeads (FunctionDeclaration)
  lib/ratelimit.ts → rateLimitPdf (FunctionDeclaration)
  lib/ratelimit.ts → rateLimitEmail (FunctionDeclaration)
  lib/ratelimit.ts → RateLimitResult (TypeAliasDeclaration)
  …
```

### `--relates <path>` — blast radius + transitive relevance

The file's own block (exports / imports / direct dependents) **plus** a random-walk
relevance list (personalized PageRank on the bidirectional import graph) — the files most
related to the target, transitively, not just its direct importers.

```
$ node repomap.mjs --relates lib/validation.ts
relates: lib/validation.ts  (pr 0.049669)
exports (16): isPayloadTooLarge(FunctionDeclaration), …
imports (0): —
dependents (13): app/api/leads/route.ts, app/api/dashboard/login/route.ts, …
related (random-walk relevance):
  lib/supabase.ts (0.0751)
  lib/ratelimit.ts (0.0504)
  app/bari/types.ts (0.0453)
  app/api/report/generate/route.ts (0.0302)
  …
```

### `--feature <name>` — files that make up a feature

Resolves a Next.js `app/`-router feature to its file set, plus the external files that
depend on it.

```
$ node repomap.mjs --feature bari
feature "bari": 21 files
  app/bari/bari-flow.tsx
  app/bari/scoring.ts
  app/bari/types.ts
  …
external dependents (7): app/dashboard/(admin)/leads/[id]/page.tsx, lib/server/send-bari-report.tsx, app/api/report/generate/route.ts, …
```

### `--features` — list features by size

```
$ node repomap.mjs --features
features (3):
  bari (21 files)
  dashboard (15 files)
  api (14 files)
```

### `--hubs` — most important files (PageRank)

The files that matter most, ranked by PageRank importance (raw dependent degree shown
alongside).

```
$ node repomap.mjs --hubs
repomap: 78 files (sha 5fbc953)
hubs (PageRank importance):
  app/bari/types.ts (deg 25, pr 0.148412)
  lib/validation.ts (deg 13, pr 0.049669)
  lib/supabase.ts (deg 17, pr 0.04257)
  lib/wib.ts (deg 8, pr 0.028502)
  lib/ratelimit.ts (deg 9, pr 0.025418)
  …
```

### `--symbols [N]` — top ranked symbols (Aider-style)

The most important individual symbols across the repo, ranked by the identifier graph
(defaults to 30).

```
$ node repomap.mjs --symbols 10
top 10 ranked symbols (Aider-style):
  0.034335  app/bari/types.ts → LocaleString (TypeAliasDeclaration)
  0.027014  lib/supabase.ts → getSupabaseAdmin (FunctionDeclaration)
  0.02365   app/bari/types.ts → Question (TypeAliasDeclaration)
  0.015032  lib/events.ts → EventType (TypeAliasDeclaration)
  …
```

### `--map [--tokens N] [--focus <path>]` — token-budgeted ranked digest

The token-budgeted digest (Aider's killer feature): a ranked, files-and-symbols summary
that fits a token budget. Default budget is 8192 (1024 with `--focus`). `--focus <path>`
personalizes the ranking toward a file you're working on.

```
$ node repomap.mjs --map --tokens 400
# repomap (78 files, sha 5fbc953) — focus: global, budget ~400 tok

app/bari/types.ts:
  LocaleString (TypeAliasDeclaration)
  Question (TypeAliasDeclaration)
  …

lib/supabase.ts:
  getSupabaseAdmin (FunctionDeclaration)
  funnelStageRank (FunctionDeclaration)
  …

# ~387 tokens (33 files shown)
```

Focused on a working file — the ranking re-centers on what `scoring.ts` actually touches:

```
$ node repomap.mjs --map --focus app/bari/scoring.ts --tokens 350
# repomap (78 files, sha 5fbc953) — focus: app/bari/scoring.ts, budget ~350 tok

app/bari/types.ts:
  Question (TypeAliasDeclaration)
  FullPillarScore (TypeAliasDeclaration)
  …
app/bari/questions.ts:
  ALL_STAGE1_QUESTIONS (VariableDeclaration)
  QUESTIONS (VariableDeclaration)
  …

# ~303 tokens (41 files shown)
```

### `--print` — full map as JSON

Dumps the cached map (`hubs`, `features`, `rankedSymbols`, `files`) as one JSON object —
for piping into other tools.

```
$ node repomap.mjs --print | jq '.hubs[0]'
"app/bari/types.ts (deg 25, pr 0.148412)"
```

---

## The agent loop (the actual point)

A repo map only helps if the agent uses it. agentmap ships two hooks (in [`./hooks/`](./hooks/))
that close the loop: the map refreshes itself after every commit, and the agent gets nudged
to query the map before it serial-greps.

### 1. Auto-refresh on commit

[`hooks/post-commit`](./hooks/post-commit) rebuilds `.claude/repomap.json` after each
commit, detached + silenced so it never slows the commit. It skips during
rebase/merge/cherry-pick and no-ops if Node is missing.

```bash
# from your repo root
cp hooks/post-commit .git/hooks/post-commit
chmod +x .git/hooks/post-commit
```

The hook auto-locates the builder: a local `repomap.mjs`, then `scripts/repomap.mjs`, then
the installed `agentmap` binary, then `npx @raymondchins/agentmap`.

### 2. Force the agent to use it — `PreToolUse` hook

[`hooks/repomap-nudge.mjs`](./hooks/repomap-nudge.mjs) is a **non-blocking** `PreToolUse(Grep)`
hook for Claude Code. When a `Grep` looks like a dependency / who-imports / component-usage /
reuse search, it injects a reminder steering the agent to `agentmap --any` first. It never
denies the grep, and stays silent for raw-string / Tailwind-class / lowercase-HTML-tag
sweeps — so it's high-signal, not nagging.

Wire it up in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Grep",
        "hooks": [
          { "type": "command", "command": "node ./hooks/repomap-nudge.mjs" }
        ]
      }
    ]
  }
}
```

That's the "forced to use it" in the tagline: the map stays current on its own, and the
agent is steered to it the moment it reaches for a dependency-shaped grep.

---

## Scope & limitations

Honesty first — this is deliberately a small, sharp tool, not a universal code-graph.

- **TS/JS only, by design.** Built on `ts-morph`. No Python, Go, Rust, etc. If your repo
  isn't TypeScript/JavaScript, use a tree-sitter-based tool instead.
- **File-level import graph, not a full reference graph.** Edges come from static
  `import` / re-export declarations and the named symbols crossing them. It does **not**
  do call-site or full reference resolution — `--relates` tells you which files import a
  module, not every line that calls a given function.
- **PageRank + symbol ranking are real and implemented** (damping 0.85, deterministic
  power iteration; personalized variants for `--relates` and `--map --focus`). The symbol
  ranking is a faithful port of Aider's identifier-graph approach (credit:
  [Aider](https://github.com/Aider-AI/aider), Apache-2.0).
- **Feature detection assumes the Next.js `app/` router.** `--feature` / `--features`
  derive features from the first real route segment under `app/` (or `src/app/`), skipping
  route groups `(...)`, dynamic `[...]`, and parallel `@...` segments. Repos without an
  `app/` directory simply report zero features — every other command still works.
- **Token counts are estimates** (`chars / 4`), not a real BPE tokenizer. Treat
  `--map`/`--tokens` budgets as approximate (±10%).
- The PreToolUse hook is **Claude Code-specific** (it speaks Claude Code's hook JSON). The
  post-commit hook is generic git.

---

## Benchmark

Against a real **78-file Next.js 16 + Supabase repo**:

- `agentmap --map` digest: **~891 tokens** vs. **~182,200 tokens** to dump the same source
  → **99.5% context reduction**.
- Cold build (parse + PageRank + symbol graph): **~1.2s**. Warm cached query (`--hubs`,
  clean tree): **~0.2s**.

Caveat: that 99.5% is *ranked-digest vs. full-source-dump* (the worst-case baseline), and
it measures context efficiency, **not** end-to-end retrieval accuracy — there's no
"did the agent fix the bug faster" eval yet.

Full methodology, commands, and caveats: **[`./benchmark/RESULTS.md`](./benchmark/RESULTS.md)**.

---

## Contributing

Issues and PRs welcome. High-value directions:

- An end-to-end retrieval/accuracy eval (the benchmark is context-efficiency only today).
- A real tokenizer behind the `--map` budget.
- Hardening feature detection for non-`app/`-router layouts.

Keep the dependency footprint minimal — `ts-morph` is the only runtime dep, and that's a
feature.

## License

[MIT](./LICENSE). Symbol-ranking algorithm credit: [Aider](https://github.com/Aider-AI/aider) (Apache-2.0).
