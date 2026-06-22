# Codex Publish Path — Self-Hosted Marketplace Design

**Date:** 2026-06-22
**Status:** Approved (design)
**Author:** DaRealLebron (with Claude Code / Opus 4.8)

## Problem

After the Superpowers → Hyperpowers rename, Claude Code installs with a one-liner because
the repo self-hosts a marketplace (`.claude-plugin/marketplace.json`):

```
/plugin marketplace add DaRealLebron/hyperpowers
/plugin install hyperpowers@hyperpowers
```

Codex has no equivalent. The README tells Codex users to "clone the repository and load the
plugin manually," and the only Codex distribution machinery —
`scripts/sync-to-codex-plugin.sh` plus `tests/codex-plugin-sync/` — **publishes into a repo
the operator does not own** (`prime-radiant-inc/openai-codex-plugins`, the upstream's curated
fork). That tooling is dead weight for an independent fork: it opens PRs against someone else's
marketplace. This is `todo.md` item #3 ("give the Codex sync tooling a Hyperpowers path, or
remove it").

## Goal

Give Codex the same one-liner install Claude Code has, by making `DaRealLebron/hyperpowers`
its **own** Codex marketplace, and remove the redundant external-sync tooling.

Target experience:

```
codex plugin marketplace add DaRealLebron/hyperpowers
codex plugin install hyperpowers
```

## Background: how Codex distribution works (verified)

Verified against OpenAI's Codex docs (`developers.openai.com/codex/plugins`,
`/codex/plugins/build`) and a real OpenAI example repo (`openai/role-specific-plugins`):

- A Git repo becomes a marketplace by committing a manifest at
  **`.agents/plugins/marketplace.json`** (the Codex analog of Claude Code's
  `.claude-plugin/marketplace.json`).
- Users register it with `codex plugin marketplace add <owner/repo>` (GitHub shorthand,
  full git URL, or local path; `@branch`/`#tag` pinning supported), then
  `codex plugin install <plugin-name>`.
- Each `plugins[]` entry **requires** `name`, `source`, `policy` (with both
  `policy.installation` and `policy.authentication`), and `category`.
- `policy.installation` enum: `AVAILABLE` | `INSTALLED_BY_DEFAULT` | `NOT_AVAILABLE`.
- `policy.authentication`: OpenAI's own skills plugins (`role-specific-plugins`) use
  `ON_USE` for plugins that need no credentials — that is the value Hyperpowers uses.
- `source` is a **nested object** `{ "source": "local", "path": "./..." }` — note this differs
  from Claude Code's flat `"source": "./"` string.
- The plugin's `.codex-plugin/plugin.json` must live at the directory named by `source.path`;
  components (`skills/`, `hooks/`) live at that plugin root. Hyperpowers already has
  `.codex-plugin/plugin.json`, `skills/`, `hooks/hooks-codex.json`, and `assets/` at the repo
  root, so `source.path: "./"` reuses the existing tree with no duplication.

## Decision

**Self-host from this repo, root as the plugin.** Chosen over (a) a committed
`plugins/hyperpowers/` subtree and (b) a separate Hyperpowers-owned marketplace repo. Self-host
mirrors how Claude Code already works, needs no second repo, and keeps a single source of truth
(no duplicated skills tree).

### Known risk (documented, not blocking)

OpenAI's published examples all point `source.path` at a `./plugins/<name>` subdirectory, not
the repo root. The spec text permits a root path (`source.path` "must start with `./`" and
"remain inside the marketplace root"), and Claude Code already self-hosts with `source: "./"`,
so root-source is expected to work. It cannot be proven end-to-end on the development machine
(Windows + Git Bash, no `codex` CLI). Therefore:

- In-repo verification proves the manifest is **valid JSON and self-consistent** (its
  `source.path` resolves to a directory containing `.codex-plugin/plugin.json`).
- The live `codex plugin marketplace add` / `codex plugin install` round-trip is a **manual
  verification step** recorded for a machine with the Codex CLI (joins `todo.md` item #2's
  external-CLI checks).
- **Fallback if Codex rejects root-source:** commit a `plugins/hyperpowers/` subtree and point
  `source.path` at it. This is a contingency, not part of this change.

## Changes

### 1. New manifest — `.agents/plugins/marketplace.json`

```json
{
  "name": "hyperpowers",
  "interface": { "displayName": "Hyperpowers" },
  "plugins": [
    {
      "name": "hyperpowers",
      "source": { "source": "local", "path": "./" },
      "policy": { "installation": "AVAILABLE", "authentication": "ON_USE" },
      "category": "Coding"
    }
  ]
}
```

### 2. Remove the redundant external-sync tooling

- Delete `scripts/sync-to-codex-plugin.sh`.
- Delete `tests/codex-plugin-sync/` (its sole test file).
- Git history preserves both; nothing else executes them (`package.json` has no test-runner
  script that enumerates the suite).

### 3. Update live documentation

- **`README.md`** — replace the Codex CLI "clone + manual load" block with the one-liner
  (`codex plugin marketplace add DaRealLebron/hyperpowers` → `codex plugin install hyperpowers`).
- **`docs/porting-to-a-new-harness.md`** — Codex moves from the "external marketplace fork,
  synced by script" distribution channel to "native plugin marketplace" (alongside Claude Code)
  in both the channel table and the summary table; rework the "if no existing channel fits"
  guidance that currently holds up `sync-to-codex-plugin.sh` as a template to clone.
- **`docs/testing.md`** — drop the `tests/codex-plugin-sync/` line; reflect the new
  `tests/codex-marketplace/` suite.

### 4. Leave frozen / append-only records untouched

- `docs/hyperpowers/specs/*` and `docs/hyperpowers/plans/*` are point-in-time records
  (per `CLAUDE.md`) — their historical references to the sync tooling stay.
- `RELEASE-NOTES.md` is append-only history; the mirror tooling *was* shipped historically.
  Rewriting past release entries would misrepresent what happened. Leave them.

### 5. New verification test — `tests/codex-marketplace/`

A bash test that asserts the observable delta of this change:

- `.agents/plugins/marketplace.json` exists and parses as JSON.
- Top-level `name` is `hyperpowers`; exactly one plugin entry named `hyperpowers`.
- The entry has `policy.installation`, `policy.authentication`, and `category` (Codex
  requires all three).
- `source.source` is `local` and `source.path` resolves (from repo root) to a directory
  containing `.codex-plugin/plugin.json` — i.e. the manifest points at a real plugin root.

Red before the manifest exists, green after. This replaces the deleted `codex-plugin-sync`
test as the Codex distribution verification artifact.

### 6. Resolve `todo.md` item #3

Mark the Codex-sync decision resolved: self-host via `.agents/plugins/marketplace.json` +
removal of the external mirror tooling.

## Out of scope

- Publishing into OpenAI's curated marketplace (the removed tool's old job). Recoverable from
  git history if ever wanted.
- Plugin `version` reset 6.0.2 → 1.0.0 (`todo.md` item #5).
- The end-to-end live `codex` install verification (no Codex CLI on this machine;
  recorded under `todo.md` item #2).

## Verification artifacts

| Command | Observable delta |
|---|---|
| `node -e "JSON.parse(require('fs').readFileSync('.agents/plugins/marketplace.json'))"` | exits 0 — manifest is valid JSON (file did not exist before) |
| `bash tests/codex-marketplace/test-marketplace-manifest.sh` | `PASS` — was absent/red before the manifest existed |
| `bash scripts/lint-fork-customizations.sh` | `49 passed, 0 failed` — unchanged (infrastructure, not an advisory skill behavior) |
| `test ! -f scripts/sync-to-codex-plugin.sh && test ! -d tests/codex-plugin-sync` | both true — the external-sync tooling is gone |
| `grep -n "marketplace add DaRealLebron/hyperpowers" README.md` (Codex section) | the one-liner is present where "clone + manual load" used to be |
