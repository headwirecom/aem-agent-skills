# AEM Agent Skills

AI agent skills for Adobe Experience Manager development. These skills augment AI coding agents with AEM-specific knowledge to help them generate better code, review templates, and troubleshoot components.

## Skills

### `aem-component-development`

Helpful context for building AEM components. Covers component node structure (`cq:Component`), Touch UI dialogs, Sling Models, client libraries, edit configuration, and content templates.

**Triggers on:** Creating/modifying components, dialogs, Sling Models, clientlibs, extending Core Components, or mentions of `cq:Component`, `cq:dialog`, `componentGroup`, Granite UI.

| Reference | Description |
|-----------|-------------|
| `references/dialogs.md` | Granite UI dialog structure, 10+ field types, validation, conditional visibility, design dialogs |
| `references/sling-models.md` | `@Model` annotations, injection annotations, Interface+Impl pattern, exporters, unit testing with AEM Mocks |
| `references/clientlibs.md` | Client library structure, categories, `allowProxy`, dependencies vs. embed, debugging |

### `htl-scripting`

Reference material for AEM's HTML Template Language. Covers expression syntax, `data-sly-*` block statements, XSS context handling, and common patterns.

**Triggers on:** Creating/modifying `.html` component scripts, fixing XSS context errors, wiring Sling Models via `data-sly-use`, or mentions of HTL, Sightly, `data-sly-*`.

| Reference | Description |
|-----------|-------------|
| `references/xss-contexts.md` | Display contexts with automatic detection rules |
| `references/global-objects.md` | Sling/AEM global bindings, Use-API provider priority, parameter passing |

## Repository Structure

```
skills/
├── aem-component-development/
│   ├── SKILL.md
│   └── references/
│       ├── clientlibs.md
│       ├── dialogs.md
│       └── sling-models.md
└── htl-scripting/
    ├── SKILL.md
    └── references/
        ├── global-objects.md
        └── xss-contexts.md
```

## Installation

### Quick install via [skills.sh](https://skills.sh)

```bash
npx skills add headwirecom/aem-agent-skills
```

This installs all skills and configures them for your AI agent automatically. Works with Claude Code, Cursor, Cline, Windsurf, Copilot, and [many more](https://skills.sh).

### Manual install

Copy the skills you need into your agent's skills directory:

```bash
# Claude Code
cp -r skills/aem-component-development ~/.claude/skills/
cp -r skills/htl-scripting ~/.claude/skills/
```

The agent will automatically detect and activate skills based on the `description` field in each `SKILL.md` frontmatter.
