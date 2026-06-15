---
name: Add GitHub Copilot CLI plugin support
about: Track implementation of a Copilot-native Ponytail plugin
labels: enhancement
---

Add a Copilot-native plugin layer for Ponytail so GitHub Copilot CLI users can install Ponytail as a real plugin with commands, skills, and hooks instead of only relying on `AGENTS.md` and `.github/copilot-instructions.md`.

This should be a thin adapter PR, not a rewrite of Ponytail. The repo already has the core behavior in `skills/`, project instruction fallbacks in `AGENTS.md` and `.github/copilot-instructions.md`, Claude marketplace metadata in `.claude-plugin/`, Codex metadata in `.codex-plugin/`, and lifecycle hook scripts in `hooks/`.

Relevant docs:

- Creating plugins: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating
- Installing plugins: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing
- Creating marketplaces: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace
- Plugin reference: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference

## Proposed implementation

1. Add `.github/plugin/plugin.json` with Ponytail metadata and component paths for `commands/`, `skills/`, and `hooks/copilot-hooks.json`.

2. Add `.github/plugin/marketplace.json` so users can run:

```bash
copilot plugin marketplace add DietrichGebert/ponytail
copilot plugin install ponytail@ponytail
```

And inside Copilot CLI:

```text
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```

3. Add `hooks/copilot-hooks.json` instead of mutating `hooks/hooks.json`, because the current hook config is Claude-shaped and uses `SessionStart`, `UserPromptSubmit`, `CLAUDE_PLUGIN_ROOT`, `commandWindows`, and `timeout`.

4. Add Copilot wrapper scripts:

```text
hooks/ponytail-copilot-activate.js
hooks/ponytail-copilot-mode-tracker.js
```

These should reuse existing shared behavior from `hooks/ponytail-config.js` and `hooks/ponytail-instructions.js`, but should not write state to `~/.claude`.

5. Store Copilot runtime state under `COPILOT_PLUGIN_DATA` when available. The Copilot plugin reference says plugin data is available through `${COPILOT_PLUGIN_DATA}` and points to a persistent writable directory unique to the installed plugin.

Suggested runtime path precedence:

```text
COPILOT_PLUGIN_DATA -> PLUGIN_DATA -> CLAUDE_CONFIG_DIR / ~/.claude
```

6. Verify whether Copilot CLI can load the existing `commands/*.toml` files. If not, add a Copilot-specific command adapter directory, for example:

```text
.github/copilot-commands/
  ponytail.md
  ponytail-review.md
  ponytail-audit.md
  ponytail-help.md
```

Then point the Copilot manifest at that directory.

7. Update `README.md` so GitHub Copilot CLI has both paths documented:

- Full plugin install via marketplace.
- Fallback instruction-only mode via `AGENTS.md`, `.github/copilot-instructions.md`, or `~/.copilot/copilot-instructions.md`.

8. Update `docs/agent-portability.md` so GitHub Copilot CLI is no longer described only as instruction-tier once this plugin path exists.

## Candidate `.github/plugin/plugin.json`

```json
{
  "name": "ponytail",
  "description": "Lazy senior dev mode. Forces the simplest, shortest solution that actually works: YAGNI, stdlib first, no unrequested abstractions.",
  "version": "4.3.0",
  "author": {
    "name": "Dietrich Gebert",
    "url": "https://github.com/DietrichGebert"
  },
  "homepage": "https://github.com/DietrichGebert/ponytail",
  "repository": "https://github.com/DietrichGebert/ponytail",
  "license": "MIT",
  "keywords": ["yagni", "minimalism", "code-review", "productivity"],
  "commands": "commands/",
  "skills": "skills/",
  "hooks": "hooks/copilot-hooks.json"
}
```

## Candidate `.github/plugin/marketplace.json`

```json
{
  "name": "ponytail",
  "description": "Lazy senior dev mode for AI agents. The best code is the code you never wrote.",
  "owner": {
    "name": "Dietrich Gebert",
    "url": "https://github.com/DietrichGebert"
  },
  "plugins": [
    {
      "name": "ponytail",
      "description": "Forces the laziest solution that works. YAGNI, stdlib first, one line over fifty.",
      "source": "./",
      "category": "productivity",
      "tags": ["yagni", "minimalism", "code-review", "productivity"],
      "commands": "commands/",
      "skills": "skills/",
      "hooks": "hooks/copilot-hooks.json"
    }
  ]
}
```

## Validation

```bash
node -e "JSON.parse(require('fs').readFileSync('.github/plugin/plugin.json', 'utf8'))"
node -e "JSON.parse(require('fs').readFileSync('.github/plugin/marketplace.json', 'utf8'))"
node -e "JSON.parse(require('fs').readFileSync('hooks/copilot-hooks.json', 'utf8'))"
node scripts/check-rule-copies.js
```

Local Copilot CLI validation:

```bash
copilot plugin install .
copilot plugin list
copilot
```

Inside Copilot CLI:

```text
/plugin list
/skills list
/ponytail
/ponytail lite
/ponytail ultra
/ponytail off
/ponytail-review
/ponytail-audit
```

Note: Copilot caches installed plugin components, so reinstall the local plugin after changes:

```bash
copilot plugin install .
```

## Acceptance criteria

- [ ] `.github/plugin/plugin.json` exists and declares Ponytail metadata plus `commands`, `skills`, and `hooks` component paths.
- [ ] `.github/plugin/marketplace.json` exists and supports marketplace install.
- [ ] Copilot-specific hooks are added without breaking Claude hooks.
- [ ] Copilot runtime state uses `COPILOT_PLUGIN_DATA` when available, not `~/.claude`.
- [ ] `/ponytail`, `/ponytail lite`, `/ponytail full`, `/ponytail ultra`, and `/ponytail off` work in Copilot CLI.
- [ ] `/ponytail-review` and `/ponytail-audit` are available in Copilot CLI.
- [ ] `skills/` remain the source of truth.
- [ ] `AGENTS.md` and `.github/copilot-instructions.md` remain fallback instruction-only paths.
- [ ] README documents both full plugin install and fallback instruction-only mode.
- [ ] `docs/agent-portability.md` documents the new Copilot CLI plugin path.
- [ ] JSON manifests parse cleanly.
- [ ] `node scripts/check-rule-copies.js` passes.
- [ ] Existing Claude, Codex, OpenCode, and Gemini support is not regressed.

## Non-goals

- Do not rewrite Ponytail's core instructions.
- Do not replace the Claude plugin manifest.
- Do not remove `AGENTS.md` or `.github/copilot-instructions.md`.
- Do not add a package dependency unless Copilot validation requires it.
- Do not claim same-turn mode injection from `userPromptSubmitted` unless verified against Copilot hook behavior.
