# agentmap — 30-second tour

Real captures from running agentmap against a **78-file Next.js 16 repo** (App Router + Supabase). Every block below is verbatim CLI output — no hand-editing beyond trimming a couple of long lists to ~10 lines (marked with `…`).

---

### `--hubs` — what matters most, ranked by PageRank

```
$ node repomap.mjs --hubs
repomap: 78 files (sha 5fbc953)
hubs (PageRank importance):
  app/bari/types.ts (deg 25, pr 0.148412)
  lib/validation.ts (deg 13, pr 0.049669)
  lib/supabase.ts (deg 17, pr 0.04257)
  lib/wib.ts (deg 8, pr 0.028502)
  lib/ratelimit.ts (deg 9, pr 0.025418)
  lib/events.ts (deg 5, pr 0.023544)
  app/icons.tsx (deg 3, pr 0.019074)
  app/use-idle.ts (deg 1, pr 0.015993)
  app/preloader.tsx (deg 1, pr 0.01457)
  lib/dashboard-auth-edge.ts (deg 1, pr 0.01457)
  …
```

*The 15 most central files by importance, not just raw import count — note `types.ts` (deg 25) outranks everything, telling you where to look first when onboarding.*

---

### `--symbols 8` — the 8 highest-value symbols in the codebase

```
$ node repomap.mjs --symbols 8
top 8 ranked symbols (Aider-style):
  0.034335  app/bari/types.ts → LocaleString (TypeAliasDeclaration)
  0.027014  lib/supabase.ts → getSupabaseAdmin (FunctionDeclaration)
  0.02365  app/bari/types.ts → Question (TypeAliasDeclaration)
  0.015032  lib/events.ts → EventType (TypeAliasDeclaration)
  0.013059  app/icons.tsx → ArrowRight (FunctionDeclaration)
  0.011595  app/bari/types.ts → Locale (TypeAliasDeclaration)
  0.0114  app/bari/types.ts → BariFlags (TypeAliasDeclaration)
  0.011028  app/bari/types.ts → PILLAR_LABEL (VariableDeclaration)
```

*Aider-style identifier-graph ranking surfaces the exact exports the rest of the repo leans on — `getSupabaseAdmin` is the #2 symbol overall, so it's load-bearing.*

---

### `--map --tokens 400` — a token-budgeted repo digest you can paste into a prompt

```
$ node repomap.mjs --map --tokens 400
# repomap (78 files, sha 5fbc953) — focus: global, budget ~400 tok

app/bari/types.ts:
  LocaleString (TypeAliasDeclaration)
  Question (TypeAliasDeclaration)
  Locale (TypeAliasDeclaration)
  BariFlags (TypeAliasDeclaration)
  PILLAR_LABEL (VariableDeclaration)
  PILLAR_ORDER (VariableDeclaration)
  Stage2Result (TypeAliasDeclaration)
  ReportNarrative (TypeAliasDeclaration)

lib/supabase.ts:
  getSupabaseAdmin (FunctionDeclaration)
  FunnelStage (TypeAliasDeclaration)
  FUNNEL_STAGES (VariableDeclaration)
  funnelStageRank (FunctionDeclaration)

lib/events.ts:
  EventType (TypeAliasDeclaration)
  eventStage (FunctionDeclaration)
  EVENT_TYPE_SET (VariableDeclaration)
  …
# ~387 tokens (33 files shown)
```

*Fits the whole repo's most important symbols into a hard ~400-token budget (it packed 33 files into 387 tokens) — drop it into any agent's context as a cheap map instead of dumping files.*

---

### `--relates lib/supabase.ts` — blast radius + transitively related files

```
$ node repomap.mjs --relates lib/supabase.ts
relates: lib/supabase.ts  (pr 0.04257)
exports (5): getSupabaseAdmin(FunctionDeclaration), funnelStageRank(FunctionDeclaration), maxFunnelStage(FunctionDeclaration), FUNNEL_STAGES(VariableDeclaration), FunnelStage(TypeAliasDeclaration)
imports (0): —
dependents (17): lib/events-server.ts, lib/server/send-bari-report.tsx, app/api/leads/route.ts, app/dashboard/(admin)/page.tsx, app/api/dashboard/promos/route.ts, app/api/funnel/event/route.ts, app/api/funnel/report/route.ts, app/api/funnel/stage2/route.ts, app/api/promo/redeem/route.ts, app/api/promo/validate/route.ts, app/api/report/generate/route.ts, app/dashboard/(admin)/analytics/page.tsx, app/dashboard/(admin)/leads/page.tsx, app/dashboard/(admin)/promos/page.tsx, app/api/dashboard/leads/csv/route.ts, app/api/dashboard/promos/[id]/route.ts, app/dashboard/(admin)/leads/[id]/page.tsx
related (random-walk relevance):
  lib/validation.ts (0.0575)
  app/bari/types.ts (0.0426)
  lib/wib.ts (0.0412)
  lib/ratelimit.ts (0.0391)
  app/dashboard/(admin)/leads/[id]/page.tsx (0.0280)
  lib/server/send-bari-report.tsx (0.0268)
  app/api/report/generate/route.ts (0.0266)
  app/api/promo/redeem/route.ts (0.0238)
  app/api/promo/validate/route.ts (0.0238)
  app/dashboard/(admin)/analytics/page.tsx (0.0227)
```

*Before you touch a file: exactly the 17 dependents that break if you change it, plus a random-walk relevance list of files that are related transitively (not just direct importers).*

---

### `--any supabase` — one router that auto-resolves file vs. symbol vs. feature vs. live grep

```
$ node repomap.mjs --any supabase
[structure:file] lib/supabase.ts  (pr 0.04257)
exports (5): getSupabaseAdmin(FunctionDeclaration), funnelStageRank(FunctionDeclaration), maxFunnelStage(FunctionDeclaration), FUNNEL_STAGES(VariableDeclaration), FunnelStage(TypeAliasDeclaration)
imports (0): —
dependents (17): lib/events-server.ts, lib/server/send-bari-report.tsx, app/api/leads/route.ts, app/dashboard/(admin)/page.tsx, app/api/dashboard/promos/route.ts, app/api/funnel/event/route.ts, app/api/funnel/report/route.ts, app/api/funnel/stage2/route.ts, app/api/promo/redeem/route.ts, app/api/promo/validate/route.ts, app/api/report/generate/route.ts, app/dashboard/(admin)/analytics/page.tsx, app/dashboard/(admin)/leads/page.tsx, app/dashboard/(admin)/promos/page.tsx, app/api/dashboard/leads/csv/route.ts, app/api/dashboard/promos/[id]/route.ts, app/dashboard/(admin)/leads/[id]/page.tsx
```

*One command, no flag-picking: `supabase` resolved to a file and returned its full block. If it had been a symbol or feature name it would route there instead, and if nothing matched it falls back to a live `git grep` so string/copy lookups never come up empty.*
