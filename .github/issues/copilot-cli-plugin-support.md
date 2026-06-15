# Add GitHub Copilot CLI plugin support

## Summary

Add a Copilot-native plugin layer for Ponytail so GitHub Copilot CLI users can install Ponytail as a real plugin with commands, skills, and hooks instead of only relying on `AGENTS.md` and `.github/copilot-instructions.md`.

This should be a thin adapter PR, not a rewrite of Ponytail. The repo already has the core behavior in `skills/`, project instruction fallbacks in `AGENTS.md` and `.github/copilot-instructions.md`, Claude marketplace metadata in `.claude-plugin/`, and lifecycle hook scripts in `hooks/`. The Copilot work should package those existing assets using Copilot's plugin conventions.

## Background

Maintainer context from discussion:

> Copilot CLI already reads instruction files, so Ponytail currently works at the instruction tier through `AGENTS.md` and `.github/copilot-instructions.md`.
>
> What is missing is a real Copilot plugin that supports commands and hooks, comparable to the Claude plugin experience.

GitHub Copilot CLI supports installable plugins. A plugin must include a `plugin.json` manifest and can include agents, skills, hooks, command directories, and MCP configuration.

Relevant GitHub docs:

- Creating plugins: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating
- Finding and installing plugins: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing
- Creating plugin marketplaces: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace
- Plugin reference: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference

## Current repo state

Existing assets to reuse:

- `.claude-plugin/plugin.json`: Claude plugin metadata.
- `.claude-plugin/marketplace.json`: Claude marketplace metadata.
- `.codex-plugin/plugin.json`: Codex plugin metadata.
- `hooks/hooks.json`: Claude-shaped lifecycle hooks using `SessionStart`, `UserPromptSubmit`, `CLAUDE_PLUGIN_ROOT`, and `commandWindows`.
- `hooks/ponytail-activate.js`: startup activation hook that writes active mode state and emits Ponytail rules.
- `hooks/ponytail-mode-tracker.js`: prompt hook that tracks `/ponytail` mode switches.
- `hooks/ponytail-runtime.js`: runtime state/output helper; currently distinguishes Codex by `PLUGIN_DATA` and otherwise falls back to Claude paths.
- `hooks/ponytail-config.js`: default mode and config resolver.
- `hooks/ponytail-instructions.js`: shared instruction builder used by multiple adapters.
- `commands/ponytail.toml`, `commands/ponytail-review.toml`, `commands/ponytail-audit.toml`: existing command definitions.
- `skills/ponytail/SKILL.md`, `skills/ponytail-review/SKILL.md`, `skills/ponytail-audit/SKILL.md`, `skills/ponytail-help/SKILL.md`: portable skill definitions.
- `AGENTS.md` and `.github/copilot-instructions.md`: current Copilot fallback instruction path.
- `docs/agent-portability.md`: documents current Copilot CLI support as instruction-tier only.

## Proposed implementation

### 1. Add a Copilot plugin manifest

Add `.github/plugin/plugin.json`.

Use a dedicated `.github/plugin/` location instead of overloading `.claude-plugin/plugin.json`. GitHub's plugin reference lists `.github/plugin/plugin.json` as a valid plugin manifest location, and this keeps the Claude adapter untouched.

Proposed starting point:

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

Notes:

- `commands`, `skills`, and `hooks` are valid component path fields in the Copilot CLI plugin reference.
- Use the repo root paths so the manifest points at existing assets instead of duplicating behavior.
- Keep the version aligned with the existing plugin metadata unless the maintainer wants a release bump.

### 2. Add a Copilot marketplace manifest

Add `.github/plugin/marketplace.json`.

GitHub's plugin reference lists `.github/plugin/marketplace.json` as a valid marketplace manifest location. This enables the marketplace install flow:

```bash
copilot plugin marketplace add DietrichGebert/ponytail
copilot plugin install ponytail@ponytail
```

Equivalent interactive TUI flow:

```text
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```

Proposed starting point:

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

### 3. Add Copilot-shaped hooks config

Add `hooks/copilot-hooks.json` rather than changing `hooks/hooks.json`.

Reason: `hooks/hooks.json` is currently Claude-shaped and includes Claude-specific fields and environment variables. A Copilot-specific file avoids breaking Claude and keeps adapter behavior explicit.

Proposed starting point:

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "command": "node \"${PLUGIN_ROOT}/hooks/ponytail-copilot-activate.js\"",
        "timeoutSec": 5
      }
    ],
    "userPromptSubmitted": [
      {
        "type": "command",
        "command": "node \"${PLUGIN_ROOT}/hooks/ponytail-copilot-mode-tracker.js\"",
        "timeoutSec": 5
      }
    ]
  }
}
```

Validate the exact lifecycle names and output schema against the current Copilot hooks reference during implementation. The important design point is not to reuse Claude's `SessionStart`/`UserPromptSubmit` config directly unless Copilot accepts it.

### 4. Add Copilot runtime wrappers

Add:

```text
hooks/ponytail-copilot-activate.js
hooks/ponytail-copilot-mode-tracker.js
```

These should reuse shared logic from:

- `hooks/ponytail-config.js`
- `hooks/ponytail-instructions.js`
- `hooks/ponytail-runtime.js`, if generalized safely

But they should not write Copilot state into `~/.claude`.

Copilot plugin-specific runtime data should use `COPILOT_PLUGIN_DATA` when available. The GitHub plugin reference says plugin data is available through `${COPILOT_PLUGIN_DATA}` and points to a persistent writable directory unique to each installed plugin. Use that for `.ponytail-active` rather than Claude paths.

Suggested behavior:

- `sessionStart`
  - Read default mode using the existing Ponytail config resolver.
  - If mode is `off`, clear Copilot plugin state and emit no rules.
  - Otherwise write `.ponytail-active` into `COPILOT_PLUGIN_DATA` and emit the active Ponytail instruction body using `getPonytailInstructions(mode)`.
- `userPromptSubmitted`
  - Parse prompt JSON from stdin.
  - Track `/ponytail`, `/ponytail lite`, `/ponytail full`, `/ponytail ultra`, `/ponytail off`, `stop ponytail`, and `normal mode`.
  - Persist state under `COPILOT_PLUGIN_DATA`.
  - Do not assume this hook can inject same-turn model instructions unless Copilot's hook reference guarantees that behavior.

### 5. Generalize runtime state carefully

Current `hooks/ponytail-runtime.js` checks `PLUGIN_DATA` to detect Codex and otherwise writes to Claude config paths.

That needs either:

- a Copilot-specific runtime helper, or
- a small generalization that resolves state path by host:

Suggested precedence:

```text
COPILOT_PLUGIN_DATA -> PLUGIN_DATA -> CLAUDE_CONFIG_DIR / ~/.claude
```

Do not use `~/.copilot` as the primary state path when `COPILOT_PLUGIN_DATA` is available. Global user config and plugin runtime state should not be mixed.

### 6. Verify commands format support

The existing commands are `.toml` files:

- `commands/ponytail.toml`
- `commands/ponytail-review.toml`
- `commands/ponytail-audit.toml`

The Copilot plugin reference supports `commands` as a component path field, but the implementation should smoke-test whether Copilot CLI accepts this exact TOML command format.

If Copilot does not load the existing TOML commands, add a Copilot-specific command adapter directory instead of changing the existing files blindly. For example:

```text
.github/copilot-commands/
  ponytail.md
  ponytail-review.md
  ponytail-audit.md
  ponytail-help.md
```

Then point the Copilot manifest at that directory:

```json
"commands": ".github/copilot-commands/"
```

Acceptance should be based on actual Copilot CLI loading behavior, not just manifest shape.

### 7. Update docs

Update `README.md` install docs.

Current README says Copilot CLI reads `AGENTS.md` and `.github/copilot-instructions.md`, or users can copy rules to `~/.copilot/copilot-instructions.md`. That remains true as the fallback path, but it undersells the new plugin behavior.

Proposed README structure:

```md
### GitHub Copilot CLI

Full plugin install:

```bash
copilot plugin marketplace add DietrichGebert/ponytail
copilot plugin install ponytail@ponytail
```

Or from inside Copilot CLI:

```text
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```

Fallback instruction-only mode: Copilot CLI also reads `AGENTS.md` and `.github/copilot-instructions.md` in a project. To run Ponytail rules globally without the plugin, copy the rules into `~/.copilot/copilot-instructions.md`. Instruction-only mode does not provide `/ponytail` mode switching or hooks.
```

Also update `docs/agent-portability.md`:

- Change GitHub Copilot CLI from instruction-tier only to full plugin support.
- Keep a note that `AGENTS.md` and `.github/copilot-instructions.md` remain the fallback.

### 8. Add validation checks

At minimum:

```bash
node -e "JSON.parse(require('fs').readFileSync('.github/plugin/plugin.json', 'utf8'))"
node -e "JSON.parse(require('fs').readFileSync('.github/plugin/marketplace.json', 'utf8'))"
node -e "JSON.parse(require('fs').readFileSync('hooks/copilot-hooks.json', 'utf8'))"
node scripts/check-rule-copies.js
```

Copilot CLI local install validation:

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

After local changes, reinstall the local plugin because Copilot caches installed plugin components:

```bash
copilot plugin install .
```

## Acceptance criteria

- [ ] `.github/plugin/plugin.json` exists and declares Ponytail metadata plus `commands`, `skills`, and `hooks` component paths.
- [ ] `.github/plugin/marketplace.json` exists and supports `copilot plugin marketplace add DietrichGebert/ponytail` followed by `copilot plugin install ponytail@ponytail`.
- [ ] Copilot-specific hooks are added without breaking the existing Claude hook config.
- [ ] Copilot hook runtime stores state in `COPILOT_PLUGIN_DATA` when available, not `~/.claude`.
- [ ] `/ponytail`, `/ponytail lite`, `/ponytail full`, `/ponytail ultra`, and `/ponytail off` work in Copilot CLI.
- [ ] `/ponytail-review` and `/ponytail-audit` are available in Copilot CLI.
- [ ] `skills/` continue to be reused as the source of truth.
- [ ] `AGENTS.md` and `.github/copilot-instructions.md` remain the fallback instruction-only path.
- [ ] `README.md` documents both full plugin install and fallback instruction-only mode.
- [ ] `docs/agent-portability.md` documents GitHub Copilot CLI as plugin-supported, not instruction-tier only.
- [ ] JSON manifests parse cleanly.
- [ ] `node scripts/check-rule-copies.js` still passes.
- [ ] Existing Claude/Codex/OpenCode/Gemini support is not regressed.

## Non-goals

- Do not rewrite Ponytail's core instructions.
- Do not replace the Claude plugin manifest.
- Do not remove `AGENTS.md` or `.github/copilot-instructions.md`.
- Do not add a new package manager dependency unless required by Copilot CLI validation.
- Do not claim same-turn mode injection from `userPromptSubmitted` unless verified against Copilot's hook behavior.

## PR title suggestion

`Add GitHub Copilot CLI plugin support`
