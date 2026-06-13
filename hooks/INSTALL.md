# agentmap — agent-loop install

The differentiator over "pack the repo into a prompt" tools is **agent-loop
integration**: the map stays fresh on its own (git `post-commit`) and the agent
is *nudged* to use it instead of serial grep (Claude Code `PreToolUse` hook).
Two small, dependency-free files wire both up.

```
hooks/
  repomap-nudge.mjs   PreToolUse(Grep) nudge → "use agentmap --any first"
  post-commit         git hook → rebuilds .claude/repomap.json after each commit
  INSTALL.md          ← you are here
```

Both are pure Node/bash stdlib. The only runtime dependency is `agentmap`
itself (`ts-morph`), used when the map (re)builds.

---

## 0. Prerequisites

- **Node 18+** on PATH.
- **agentmap available in the repo.** Either:
  - drop `repomap.mjs` at the repo root (or `scripts/repomap.mjs`), or
  - install it: `npm i -D agentmap` (then `npx agentmap` works), or
  - install it globally: `npm i -g agentmap` (then `agentmap` works).
- The repo must have a `tsconfig.json` (agentmap reads it to find source files).

Smoke-test it builds:

```bash
node repomap.mjs        # or: npx agentmap
# → repomap: N files | M features | top hub: ...
```

This writes `.claude/repomap.json`. Add it to `.gitignore` (it's a derived
artifact, rebuilt on every commit):

```bash
echo ".claude/repomap.json" >> .gitignore
```

---

## 1. PreToolUse nudge (Claude Code)

Steers `who-imports` / dependency / reuse / `<Component>` greps toward
`agentmap --any` before the agent fans out into serial grep. **Non-blocking** —
it only injects a reminder, never denies the Grep.

### a. Place the hook script

Keep it in the repo so it's version-controlled and path-stable:

```bash
mkdir -p .claude/hooks
cp hooks/repomap-nudge.mjs .claude/hooks/repomap-nudge.mjs
```

### b. Register it in `.claude/settings.json`

Add (or merge) this `hooks` block. The matcher `Grep` runs the hook before
every Grep tool call; the script decides whether to nudge.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/repomap-nudge.mjs\""
          }
        ]
      }
    ]
  }
}
```

`$CLAUDE_PROJECT_DIR` is set by Claude Code to the project root, so the path
resolves no matter the agent's cwd. Restart the session (or `/hooks` →
reload) to pick it up.

### c. Verify

```bash
echo '{"tool_input":{"pattern":"<Heading"}}' | node .claude/hooks/repomap-nudge.mjs
# → {"hookSpecificOutput":{...,"additionalContext":"This Grep looks like ..."}}

echo '{"tool_input":{"pattern":"bg-white"}}'  | node .claude/hooks/repomap-nudge.mjs
# → (no output — raw-string sweeps are left alone)
```

---

## 2. post-commit auto-refresh (git)

Rebuilds the map after each commit so the agent never reads a stale map.
Runs detached + silenced (commit returns instantly) and **skips during
rebase / merge / cherry-pick / bisect / revert** so it doesn't fire on every
replayed commit.

```bash
cp hooks/post-commit .git/hooks/post-commit
chmod +x .git/hooks/post-commit
```

It auto-locates the builder: `./repomap.mjs` → `./scripts/repomap.mjs` →
global `agentmap` → `npx --no-install agentmap`. If none is found it no-ops.

### Verify

```bash
git commit --allow-empty -m "test: agentmap post-commit"
# wait a moment for the background rebuild, then:
git rev-parse --short HEAD
node -e "console.log(require('./.claude/repomap.json').generatedSha)"
# the two SHAs should match
```

> **Husky / shared hooks:** if the repo uses Husky or `core.hooksPath`, append
> the body of `hooks/post-commit` to your existing `post-commit` (e.g.
> `.husky/post-commit`) instead of overwriting `.git/hooks/post-commit`.

---

## 3. One-liner installer (idea)

Drop this as `hooks/install.sh` in your repo (or run inline) to wire both at
once from the repo root:

```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT="$(git rev-parse --show-toplevel)"
HOOKS="$ROOT/hooks"   # where these files live in your repo

# PreToolUse nudge
mkdir -p "$ROOT/.claude/hooks"
cp "$HOOKS/repomap-nudge.mjs" "$ROOT/.claude/hooks/repomap-nudge.mjs"

# Merge the PreToolUse(Grep) hook into .claude/settings.json (needs jq).
SETTINGS="$ROOT/.claude/settings.json"
[ -f "$SETTINGS" ] || echo '{}' > "$SETTINGS"
CMD='node "$CLAUDE_PROJECT_DIR/.claude/hooks/repomap-nudge.mjs"'
jq --arg cmd "$CMD" '
  .hooks.PreToolUse = ((.hooks.PreToolUse // []) + [{
    matcher: "Grep",
    hooks: [{ type: "command", command: $cmd }]
  }])
' "$SETTINGS" > "$SETTINGS.tmp" && mv "$SETTINGS.tmp" "$SETTINGS"

# git post-commit auto-refresh (skip if Husky/core.hooksPath is in use)
cp "$HOOKS/post-commit" "$ROOT/.git/hooks/post-commit"
chmod +x "$ROOT/.git/hooks/post-commit"

# Ignore the derived map + first build
grep -qxF ".claude/repomap.json" "$ROOT/.gitignore" 2>/dev/null \
  || echo ".claude/repomap.json" >> "$ROOT/.gitignore"
( cd "$ROOT" && { node repomap.mjs || npx agentmap; } ) >/dev/null 2>&1 || true

echo "agentmap wired: PreToolUse nudge + post-commit refresh installed."
```

Run it:

```bash
bash hooks/install.sh
```

(The `jq` merge is idempotent-ish but appends — run once. Without `jq`, paste
the snippet from step 1b by hand.)

---

## How they reinforce each other

1. You commit → **post-commit** rebuilds `.claude/repomap.json` in the
   background → the map is always current.
2. The agent reaches for a who-imports / reuse / `<Component>` grep →
   **PreToolUse nudge** fires → the agent runs `agentmap --any <query>` and
   reads the fresh map instead of a slow serial grep.

That loop — fresh map + enforced usage — is the part a static "repo digest"
tool can't reproduce.
