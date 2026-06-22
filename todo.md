# Hyperpowers — TODO

Follow-ups from the Superpowers → Hyperpowers rename (2026-06-22).

## 1. Reinstall the plugin under the new name
The fork now installs as `hyperpowers@hyperpowers` from `DaRealLebron/hyperpowers`. In Claude Code:
- Remove the old install (and its marketplace): `/plugin` → uninstall `superpowers@superpowers-dev`.
- Add the renamed one: `/plugin marketplace add DaRealLebron/hyperpowers` → `/plugin install hyperpowers@hyperpowers`.

## 2. Run the test suites that need external CLIs
These suites had their identifier strings updated during the rename but could **not be executed** on the rename machine (Windows + Git Bash) because they require external tools. Run them where those tools are available and confirm green:
- `tests/opencode/` — needs the `opencode` CLI
- `tests/claude-code/` — needs the `claude` CLI
- `tests/explicit-skill-requests/` — needs the `claude` CLI
- `tests/antigravity/` — needs `agy`
- `tests/pi/` — re-verify under `npx tsx` in a clean environment

## 3. Give the Codex sync tooling a Hyperpowers path (or remove it) — ✅ RESOLVED (2026-06-22)
Resolved by **self-hosting**: added `.agents/plugins/marketplace.json` so `DaRealLebron/hyperpowers`
is its own Codex marketplace (`codex plugin marketplace add DaRealLebron/hyperpowers` →
`codex plugin install hyperpowers`), and **removed** `scripts/sync-to-codex-plugin.sh` and
`tests/codex-plugin-sync/` (they published to the upstream-owned `prime-radiant-inc/openai-codex-plugins`).
See `docs/hyperpowers/specs/2026-06-22-codex-publish-path-design.md`.
**Verified 2026-06-22 (Codex CLI 0.125.0):** `codex plugin marketplace add DaRealLebron/hyperpowers`
resolves from GitHub and the root-source (`source.path: "./"`) cleanly maps the repo root to the plugin —
the clone exposed `.codex-plugin/plugin.json`, `hooks/hooks-codex.json`, and all 19 skills incl. the
`using-hyperpowers` bootstrap, and registered as a `source_type = "git"` marketplace. The spec's
committed-`plugins/hyperpowers/`-subtree fallback is **not needed**.
**Still open:** the final `/plugins` TUI step (install + enable into a session — cache-copy and bootstrap
auto-trigger) is interactive in 0.125.0 and was not driven non-interactively.

## Other deferred items (from earlier work this cycle)
- **Behavioral evals for the new skills** — the structural lint proves skill text is present, not that agents obey it. Add Drill eval scenarios for `curating-project-memory` (and the BMAD planning-OS skills) once `evals/` is in-tree. See the curation and BMAD design specs under `docs/hyperpowers/specs/`.
- **Optional version reset** — the plugin `version` stayed at `6.0.2` (inherited from upstream). Reset to `1.0.0` if you want to mark the independent Hyperpowers line.
