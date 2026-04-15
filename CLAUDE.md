# CLAUDE.md

Guidance for Claude (and other AI assistants) working in this repository.

## What this repo is

This is **Knowledge Work Plugins** — a marketplace of role-specific plugins for [Claude Cowork](https://claude.com/product/cowork) and [Claude Code](https://claude.com/product/claude-code). Each plugin turns Claude into a specialist for a particular job function (sales, finance, engineering, etc.) by bundling skills, slash commands, and MCP connectors.

There is **no build system, no code, no tests**. The entire repo is markdown and JSON files. Changes are made by editing these files directly — no compilation, no package installs, no CI steps for developers.

## Repository layout

```
knowledge-work-plugins/
├── .claude-plugin/
│   └── marketplace.json          # Registers every plugin in the marketplace
├── README.md                      # User-facing overview
├── CLAUDE.md                      # This file
├── <plugin-name>/                 # One directory per plugin (see list below)
│   ├── .claude-plugin/
│   │   └── plugin.json            # Plugin manifest (name, version, description)
│   ├── .mcp.json                  # Pre-configured MCP server connections
│   ├── README.md                  # Plugin-specific documentation
│   ├── CONNECTORS.md              # (Optional) Explains ~~placeholder conventions
│   ├── commands/
│   │   └── <command>.md           # Slash commands (user-invoked workflows)
│   └── skills/
│       └── <skill-name>/
│           ├── SKILL.md           # Domain knowledge (auto-triggered)
│           ├── references/        # Progressive-disclosure detail docs
│           └── examples/          # Working examples
└── partner-built/                 # Plugins authored by partners
    ├── slack/                     # Slack (by Salesforce)
    ├── apollo/                    # Apollo.io
    ├── common-room/               # Common Room
    └── brand-voice/               # Tribe AI
```

### First-party plugins (authored by Anthropic)

`bio-research`, `cowork-plugin-management`, `customer-support`, `data`, `design`, `engineering`, `enterprise-search`, `finance`, `human-resources`, `legal`, `marketing`, `operations`, `product-management`, `productivity`, `sales`.

### Partner-built plugins

Live under `partner-built/`. These are owned by external authors (Salesforce, Apollo.io, Common Room, Tribe AI) — be conservative about editing partner content.

### Special plugin: `cowork-plugin-management`

This plugin contains the authoritative reference for how plugins should be built:

- `cowork-plugin-management/skills/create-cowork-plugin/SKILL.md` — the canonical guide to scaffolding a new plugin.
- `cowork-plugin-management/skills/create-cowork-plugin/references/component-schemas.md` — exact frontmatter and format for every component type (commands, skills, agents, hooks, MCP servers, CONNECTORS.md).
- `cowork-plugin-management/skills/cowork-plugin-customizer/SKILL.md` — guide to customizing an existing plugin for a specific organization.

**When in doubt about plugin structure, consult these files — they are the source of truth.**

## Plugin structure conventions

Every plugin (first-party and partner) follows the same skeleton. When adding or editing one, match the existing pattern.

### `plugin.json`

Located at `<plugin>/.claude-plugin/plugin.json`. Minimal shape:

```json
{
  "name": "plugin-name",
  "version": "1.1.0",
  "description": "One-sentence summary of what the plugin does.",
  "author": { "name": "Anthropic" }
}
```

- `name` must be kebab-case and match the directory name.
- `version` uses semver — bump MINOR when adding new functionality (the current first-party plugins are on `1.1.0`).
- Keep `description` aligned with the entry in `.claude-plugin/marketplace.json`.

### `.mcp.json`

Declares pre-configured MCP server connections. Most first-party plugins use hosted HTTP servers:

```json
{
  "mcpServers": {
    "slack": { "type": "http", "url": "https://mcp.slack.com/mcp" },
    "hubspot": { "type": "http", "url": "https://mcp.hubspot.com/anthropic" }
  }
}
```

Use `${CLAUDE_PLUGIN_ROOT}` for any intra-plugin path references — **never hardcode absolute paths**.

### Commands (`commands/*.md`)

Slash commands the user explicitly invokes (e.g. `/sales:call-summary`). Frontmatter:

```markdown
---
description: Short one-liner shown in /help (under 60 chars)
argument-hint: "<call notes or transcript>"
allowed-tools: Read, Grep, Bash(git:*)
---

# /command-name

Body contains instructions FOR Claude to follow — not documentation the user reads.
Use $ARGUMENTS for the full argument string, $1/$2/... for positional args,
@$1 to include a referenced file, and `!`backticks`` for inline bash.
```

Key rules:
- **Commands are directives to Claude**, written in the imperative.
- File name becomes the command name (kebab-case).
- Reference `CONNECTORS.md` in the body if the command uses `~~placeholders`.

### Skills (`skills/<name>/SKILL.md`)

Domain knowledge Claude loads automatically when a user query matches the trigger phrases. Frontmatter:

```markdown
---
name: call-prep
description: Prepare for a sales call with account context, attendee research, and suggested agenda. Trigger with "prep me for my call with [company]", "I'm meeting with [company] prep me", "call prep [company]", or "get me ready for [meeting]".
---

# Call Prep

Imperative body content. Use "Parse the config", not "You should parse the config".
```

Key rules:
- **Description must be third-person** and include specific quoted trigger phrases so the skill reliably activates.
- Keep `SKILL.md` body under ~3,000 words (ideally 1,500–2,000). Move deep detail into `references/` for progressive disclosure.
- Working examples go in `examples/`; utility scripts in `scripts/`.
- Directory name, skill `name`, and trigger phrases should stay consistent.

### `CONNECTORS.md` and `~~` placeholders

Most first-party plugins are designed to be **tool-agnostic** — they describe workflows in terms of tool categories rather than specific products. This is expressed with `~~category` placeholders:

```markdown
Check ~~project tracker for open tickets assigned to the user.
Post a summary to ~~chat in the team channel.
Pull account history from ~~CRM.
```

Each plugin that uses placeholders ships a `CONNECTORS.md` mapping placeholders to concrete server options (included + alternatives). When you edit or add placeholders, **keep `CONNECTORS.md` in sync**.

The `cowork-plugin-customizer` skill is what replaces `~~` placeholders with real tool names for a specific customer — so the placeholders should remain in the committed files.

### Marketplace registration

Every plugin must have a corresponding entry in `.claude-plugin/marketplace.json`:

```json
{
  "name": "sales",
  "source": "./sales",
  "description": "Prospect, craft outreach, and build deal strategy faster..."
}
```

When adding a new plugin, **always update `marketplace.json`** — otherwise it will not be discoverable. Partner plugins include an extra `author` field.

## Writing-style conventions

These appear consistently across the existing plugins; match them when contributing:

1. **Standalone + supercharged framing.** Every command and skill should work without any MCP connectors (using web search, pasted input, or user answers) and get richer when tools are connected. Most skills document this with an ASCII box diagram near the top:
   ```
   ┌─────────────────────────────────────────────────────────────────┐
   │  ALWAYS (works standalone)                                       │
   │  ✓ ...                                                          │
   ├─────────────────────────────────────────────────────────────────┤
   │  SUPERCHARGED (when you connect your tools)                      │
   │  + ...                                                          │
   └─────────────────────────────────────────────────────────────────┘
   ```
2. **Imperative voice in skill bodies.** "Parse the transcript", not "You should parse the transcript".
3. **Third-person skill descriptions** in frontmatter, with explicit trigger phrases in quotes.
4. **Commands are instructions for Claude**, not prose for the user.
5. **Output formats are shown as fenced markdown templates** with `[placeholders]` the model fills in.
6. **Progressive disclosure**: concise top-level explanations, with deeper detail in `references/` subfolders loaded on demand.
7. **No emojis** unless the existing plugin already uses them (most don't). ASCII checkmarks (`✓`, `+`) inside the diagrams are fine.

## Common workflows

### Adding a new plugin

1. Create `<plugin-name>/` at the repo root with kebab-case name.
2. Add `<plugin-name>/.claude-plugin/plugin.json` (see schema above).
3. Add `<plugin-name>/.mcp.json` if the plugin needs MCP connectors.
4. Add `commands/` and/or `skills/` directories with the components.
5. Add `<plugin-name>/README.md` summarizing commands, skills, and connectors.
6. If tool-agnostic, add `<plugin-name>/CONNECTORS.md`.
7. **Register the plugin in `.claude-plugin/marketplace.json`.**
8. Update the top-level `README.md` marketplace table.
9. Validate with: `claude plugin validate <plugin-name>/.claude-plugin/plugin.json`

Use the `cowork-plugin-management/skills/create-cowork-plugin/SKILL.md` guide for the full five-phase process (Discovery → Planning → Design → Implementation → Review).

### Adding a skill to an existing plugin

1. Create `skills/<skill-name>/SKILL.md` with proper frontmatter.
2. Include clear trigger phrases in the description.
3. Add deeper reference docs under `skills/<skill-name>/references/` if the body would exceed ~3,000 words.
4. Update the plugin's `README.md` skills table.

### Adding a command to an existing plugin

1. Create `commands/<command-name>.md` with frontmatter (`description`, `argument-hint`, optionally `allowed-tools`).
2. Write the body as instructions Claude should follow when invoked.
3. Update the plugin's `README.md` commands table.

### Editing an existing plugin

- Edit the markdown/JSON directly — there is no build step.
- Bump `version` in `plugin.json` for user-facing changes.
- If you change MCP servers, update both `.mcp.json` and the README / `CONNECTORS.md`.
- Keep the plugin's `README.md` table of commands/skills in sync.

## Things to avoid

- **Don't hardcode absolute paths** — use `${CLAUDE_PLUGIN_ROOT}` for intra-plugin references.
- **Don't replace `~~placeholders` with specific tool names** in committed files. That substitution is part of per-customer customization, not the default plugin.
- **Don't rename plugin directories or `name` fields** without also updating `marketplace.json` and every cross-reference.
- **Don't create new files unless necessary.** Prefer extending an existing skill or command. The whole repo's value proposition is small, focused components.
- **Don't add code, build tooling, or package managers.** Plugins are pure markdown + JSON by design.
- **Don't edit `partner-built/*`** unless the change is clearly scoped to fixing a partner-owned plugin and is requested.
- **Don't add emojis** to skills/commands unless matching an existing convention.

## Quick reference: component locations

| Component        | Lives at                                          | Format                        |
| ---------------- | ------------------------------------------------- | ----------------------------- |
| Plugin manifest  | `<plugin>/.claude-plugin/plugin.json`             | JSON                          |
| MCP connectors   | `<plugin>/.mcp.json`                              | JSON                          |
| Slash commands   | `<plugin>/commands/<name>.md`                     | Markdown + YAML frontmatter   |
| Skills           | `<plugin>/skills/<name>/SKILL.md`                 | Markdown + YAML frontmatter   |
| Skill references | `<plugin>/skills/<name>/references/*.md`          | Markdown                      |
| Connector map    | `<plugin>/CONNECTORS.md`                          | Markdown                      |
| Marketplace      | `.claude-plugin/marketplace.json`                 | JSON (must include new plugins) |

For the full schema of every component type — including agents and hooks, which are uncommon in this repo — see `cowork-plugin-management/skills/create-cowork-plugin/references/component-schemas.md`.
