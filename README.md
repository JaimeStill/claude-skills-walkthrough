# Claude Skills Walkthrough

## References

- [Claude Code Docs](https://code.claude.com/docs/en/overview)
  - [Claude Code Settings](https://code.claude.com/docs/en/settings)
  - [Memory](https://code.claude.com/docs/en/memory)
  - [Extend Claude with Skills](https://code.claude.com/docs/en/skills)
  - [Create Plugins](https://code.claude.com/docs/en/plugins)
  - [Create and Distribute a Plugin Marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
- [TAU Marketplace Repo](https://github.com/tailored-agentic-units/tau-marketplace)
- [Herald Repo](https://github.com/JaimeStill/herald)
- [Omarchy Skill](https://github.com/basecamp/omarchy/blob/dev/default/omarchy-skill/SKILL.md)

## 1. Marketplace Commands

Register a marketplace so Claude Code knows where to find plugins:

```bash
claude plugin marketplace add tailored-agentic-units/tau-marketplace
```

This points Claude Code at the repo's `.claude-plugin/marketplace.json` — the root manifest that declares all available plugins.

---

## 2. Installing Plugins

### From the UI

1. Run `/plugins` in Claude Code
2. The marketplace appears with its available plugins listed
3. Select a plugin to install it

### From the CLI

> [!INFO] You can use the `-s` flag to explicitly set the scope of the installed plugin (defaults to `user` scope).

```bash
claude plugin install dev-workflow@tau-marketplace
claude plugin install github-cli@tau-marketplace
claude plugin install go-patterns@tau-marketplace
claude plugin install iterative-dev@tau-marketplace
claude plugin install project-management@tau-marketplace
```

### Enabling in a Project

Add the installed skills to `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Skill(dev-workflow:dev-workflow)",
      "Skill(github-cli:github-cli)",
      "Skill(go-patterns:go-patterns)",
      "Skill(iterative-dev:iterative-dev)",
      "Skill(project-management:project-management)"
    ]
  }
}
```

The format is `Skill(plugin-name:skill-name)`.

### Updating & Removing

```bash
claude plugin marketplace update
claude plugin update dev-workflow@tau-marketplace
claude plugin remove dev-workflow@tau-marketplace
claude plugin marketplace remove tau-marketplace
```

---

## 3. Skill Anatomy & Contextual Optimization

### The Context Loading Hierarchy

Skills are designed around progressive context loading. Claude doesn't load everything at once — it loads only what the current task requires.

> [!TIP] You can organize the skill sub-directory hierarchy however you want; this is just the convention that makes the most sense to me in terms of differentiating contextually distinct skill infrastructure.

```
CLAUDE.md              Always loaded. Project entry point.
  └─ SKILL.md          Loaded when triggers match. Contextual index.
       ├─ commands/     Loaded on demand. Sub-command workflows.
       └─ references/   Loaded on demand. Detailed lookup material.
```

**CLAUDE.md** — Project-level entry point. Claude reads this first on every session. Contains project identity, conventions, gotchas, and pointers to relevant skills. Never duplicates skill content — just directs Claude to the right skill for a given concern.

> Herald's `CLAUDE.md` includes lines like *"Use the `frontend-design` skill when planning web client view interfaces"* and *"API Cartographer maintenance is an AI responsibility"* — enough for Claude to know when to load those skills, without embedding the skill content itself.

**SKILL.md** — Contextual index for a capability domain. Loaded when conversational triggers match the `description` field in the frontmatter. Contains "when to use" rules, concept definitions, convention tables, and a routing table that points into subdirectories. Provides enough context to orient and decide which sub-file to load next.

**commands/** — Self-contained sub-command workflows. Each file is a complete procedure (e.g., `bootstrap.md`, `init.md`, `review.md`). The SKILL.md routes to the correct command based on the user's argument. This extends a single skill into multiple distinct operations without loading all of them at once.

**references/** — Detailed lookup material. Patterns, templates, API specifics, GraphQL queries. Loaded on-demand when the active command or conversation needs that specific knowledge.

### Why This Matters

A skill with 5 commands and 10 reference files doesn't load all 15 files into context. Claude loads the SKILL.md to orient, then the 1-2 specific files the current task requires. This keeps the context window focused and avoids wasting tokens on irrelevant material.

**Concrete example from Herald:** The `web-development` skill's SKILL.md has a reference table pointing to `references/components.md`, `references/css.md`, `references/services.md`, `references/router.md`, `references/build.md`, etc. When Claude is working on a new component, it loads `components.md` and `css.md` — not the router, build system, or Go integration references.

### Skill Frontmatter

Every SKILL.md starts with YAML frontmatter:

```yaml
---
name: iterative-dev
argument-hint: "[bootstrap | init <issue-number> | review]"
description: >
  Lightweight, plan-mode-driven iterative development workflow.
  Use when the user asks to "bootstrap a project", "start a session",
  "init session", "review project", or any iterative development activity.
  Triggers: bootstrap, init, session, iterative, review project, start development.
allowed-tools:
  - "Bash(gh issue list*)"
  - "Bash(gh issue view*)"
user-invocable: true
---
```

| Field | Purpose |
|-------|---------|
| `name` | Identifier used in permissions and slash commands |
| `description` | Trigger mechanism — Claude loads the skill when conversation matches these keywords |
| `argument-hint` | Advertises available sub-commands to the user |
| `allowed-tools` | Tool permissions the skill needs (optional) |
| `user-invocable` | Whether the user can trigger it directly via slash command |

### Sub-Commands as Skill Multipliers

The `argument-hint` field advertises sub-commands. A single skill handles multiple distinct workflows behind one trigger surface:

```
/iterative-dev bootstrap     → commands/bootstrap.md (scaffold a new project)
/iterative-dev init 42       → commands/init.md (start a dev session from issue #42)
/iterative-dev review        → commands/review.md (analyze project alignment)
```

The SKILL.md contains a routing table that maps arguments to command files. Claude reads the SKILL.md, identifies the sub-command, then loads only that command file.

### Parameter Validation & Inverse Prompts

Skills can specify required and optional parameters for each sub-command, and Claude will validate them before executing. When a required value is missing or invalid, Claude generates what I call an **inverse prompt** — a context-aware request back to the user that explains what's needed and why.

Example from `dev-workflow`:

```yaml
argument-hint: "[concept | phase | objective <issue-number> | task <issue-number> | review | release <version>]"
```

The SKILL.md defines what metadata each sub-command needs:

```markdown
### Task Execution
1. **Issue number** — required, from argument. Read the issue to bootstrap session context.

### Release
1. **Version** — required, from argument (e.g., `/dev-workflow release v0.2.0`).
   Must follow semantic versioning.
```

If a user runs `/dev-workflow task` without the issue number, Claude doesn't fail or guess — it responds with an inverse prompt explaining that task execution requires a GitHub issue number to bootstrap context, along with what that context will be used for. The user supplies the missing value and the command proceeds.

This inverts the usual CLI pattern. A traditional tool prints a usage error and exits. A skill treats missing parameters as a conversation — Claude knows what's required, why it's required, and can ask for it with full context about what happens next.

### Skills as Loose Specification

Writing skills is effectively just a loose means of programming a specification. You're not writing code that executes — you're writing a specification that describes:

- **When to activate** (the `description` and trigger keywords)
- **What parameters are needed** (the `argument-hint` and validation rules)
- **What procedures to follow** (the command files)
- **What reference knowledge applies** (the references files)
- **What safety rules govern behavior** (the "critical" sections)

Claude interprets that specification and executes it. The looseness is the feature — you get structured, deterministic behavior where it matters (required parameters, safety rules, file locations) combined with Claude's judgment where it helps (adapting to unexpected requests, composing from first principles, surfacing edge cases). A well-written skill reads like a contract between the author and the agent: "here's the domain, here's the invariants, here's what you can assume — now solve the user's problem."

### Skill File Organization

```
skills/<skill-name>/
├── SKILL.md                 # Contextual index — triggers, concepts, routing table
├── commands/                # Sub-command workflows (loaded by argument)
│   ├── bootstrap.md
│   ├── init.md
│   └── review.md
└── references/              # Lookup material (loaded on demand)
    ├── components.md
    ├── css.md
    ├── services.md
    └── templates.md
```

---

## 4. Permissions

### Skill Permissions

Two naming patterns depending on where the skill lives:

| Type | Format | Example |
|------|--------|---------|
| Marketplace plugin | `Skill(plugin:skill)` | `Skill(dev-workflow:dev-workflow)` |
| Project-local | `Skill(name)` | `Skill(api-cartographer)` |

### Tool Permissions

Skills that invoke CLI tools need corresponding `Bash()` grants to execute without explicit permissions:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh auth status*)",
      "Bash(gh issue list*)",
      "Bash(gh issue view*)",
      "Bash(gh project field-list*)",
      "Bash(gh project item-list*)",
      "Bash(gh project list*)",
      "Bash(gh project view*)"
    ]
  }
}
```

### Deny Rules

Prevent Claude from reading sensitive files:

```json
{
  "permissions": {
    "deny": ["Read(secrets.json)"]
  }
}
```

---

## 5. Project-Local Skills

### Location & Structure

Project-local skills live in the repository itself:

```
.claude/
├── CLAUDE.md                          # Points Claude to relevant skills
├── settings.json                      # Permissions for local skills
└── skills/
    └── <skill-name>/
        ├── SKILL.md                   # Skill definition
        ├── commands/                  # Sub-command workflows (optional)
        └── references/                # Reference material (optional)
```

Permission format uses just the name (no plugin prefix):

```json
{
  "permissions": {
    "allow": ["Skill(api-cartographer)"]
  }
}
```

### Contrast with Marketplace Skills

| Aspect | Marketplace Plugin | Project-Local |
|--------|-------------------|---------------|
| Location | `plugins/<name>/skills/<name>/` | `.claude/skills/<name>/` |
| Distribution | Installed via marketplace | Lives in the repo |
| Versioning | `plugin.json` with semver + CHANGELOG | None — versioned with the repo |
| Permission format | `Skill(plugin:skill)` | `Skill(name)` |
| Scope | Shared across projects | Scoped to one repo |

### Meta Example: tau-marketplace-dev

The `tau-marketplace` repo has its own project-local skill for managing itself:

```
.claude/skills/tau-marketplace-dev/
├── SKILL.md                           # Triggers on "update skills", "release", "bump version"
└── commands/
    ├── update.md                      # Branch, plan, execute, version bump, CHANGELOG, PR
    └── release.md                     # Tag and push releases after PR merge
```

This skill is not a marketplace plugin — it's a project-local skill that orchestrates the marketplace development workflow. It uses the same anatomy (frontmatter, commands subdirectory) but lives in `.claude/skills/` and is permissioned as `Skill(tau-marketplace-dev)`.

### Herald's Project-Local Skills

Herald defines three project-local skills alongside its marketplace plugin installations:

**api-cartographer** — AI-maintained API endpoint documentation. Generates and updates `_project/api/` specs with curl examples and Kulala `.http` test files whenever HTTP handlers change. Has a `references/templates.md` for endpoint documentation patterns.

**web-development** — Lit component architecture reference. SKILL.md provides the overview (three-tier hierarchy, naming conventions, anti-patterns), with seven reference files covering components, CSS, services, state, router, build, and lifecycles. Claude loads only the references relevant to the current task.

**playwright-cli** — Browser automation. SKILL.md covers core commands, with reference files for request mocking, running code, session management, storage state, test generation, tracing, and video recording.

---

## 6. Marketplace Repo Structure

```
tau-marketplace/
├── .claude-plugin/
│   └── marketplace.json               # Root manifest — lists all plugins
├── .claude/
│   └── skills/
│       └── tau-marketplace-dev/        # Project-local skill for repo maintenance
├── .github/
│   └── workflows/
│       └── release.yml                 # Tag-triggered release automation
├── plugins/
│   ├── dev-workflow/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json            # Name, version, description
│   │   ├── CHANGELOG.md               # Versioned headings: ## [v0.1.0]
│   │   └── skills/
│   │       └── dev-workflow/
│   │           ├── SKILL.md
│   │           └── commands/
│   ├── github-cli/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/
│   │       └── github-cli/
│   │           ├── SKILL.md
│   │           └── references/
│   ├── go-patterns/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/go-patterns/SKILL.md
│   ├── iterative-dev/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── CHANGELOG.md
│   │   └── skills/
│   │       └── iterative-dev/
│   │           ├── SKILL.md
│   │           └── commands/
│   └── project-management/
│       ├── .claude-plugin/plugin.json
│       ├── CHANGELOG.md
│       └── skills/
│           └── project-management/
│               ├── SKILL.md
│               └── references/
└── README.md
```

### Key Files

**`.claude-plugin/marketplace.json`** — Root manifest declaring all available plugins:

```json
{
  "name": "tau-marketplace",
  "owner": { "name": "tailored-agentic-units" },
  "plugins": [
    {
      "name": "dev-workflow",
      "source": "./plugins/dev-workflow",
      "description": "Structured development sessions"
    }
  ]
}
```

**`plugins/<name>/.claude-plugin/plugin.json`** — Per-plugin metadata:

```json
{
  "name": "iterative-dev",
  "description": "Lightweight iterative development sessions with issue-driven lifecycle",
  "version": "0.1.2"
}
```

**`plugins/<name>/CHANGELOG.md`** — Versioned entries with bracketed headings (required by `parse-changelog`):

```markdown
# Changelog

## [v0.1.2]
- Remove "Set Issue Status" step from init command

## [v0.1.1]
- Enumerate AI-owned surfaces in Role Boundaries

## [v0.1.0]
- Initial release as standalone plugin
```

---

## 7. CI/CD with GitHub Actions

### Tag Convention

Each plugin is tagged independently:

```
<plugin-name>/v<major>.<minor>.<patch>
```

Examples: `dev-workflow/v0.1.0`, `iterative-dev/v0.1.2`

### Release Workflow

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags:
      - '*/v*'

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Determine plugin from tag
        id: plugin
        run: |
          TAG="${GITHUB_REF_NAME}"
          PREFIX="${TAG%/v*}"
          echo "prefix=$PREFIX/" >> "$GITHUB_OUTPUT"
          echo "changelog=plugins/$PREFIX/CHANGELOG.md" >> "$GITHUB_OUTPUT"

      - name: Create GitHub Release
        uses: taiki-e/create-gh-release-action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          prefix: ${{ steps.plugin.outputs.prefix }}
          changelog: ${{ steps.plugin.outputs.changelog }}
```

### How It Works

1. **Tag push** matching `*/v*` triggers the workflow
2. **Extract plugin prefix** — strips `/v*` from the tag (e.g., `dev-workflow/v0.1.2` → `dev-workflow`)
3. **Locate CHANGELOG** — maps to `plugins/dev-workflow/CHANGELOG.md`
4. **Create GitHub Release** — `taiki-e/create-gh-release-action` uses `parse-changelog` to extract the matching version section from the CHANGELOG and creates the release with those notes

### Release Flow

```bash
# After a PR is merged to main
git checkout main
git pull origin main

# Tag the affected plugin(s)
git tag iterative-dev/v0.1.2
git push origin iterative-dev/v0.1.2

# Verify
gh release view iterative-dev/v0.1.2
```

Multiple plugins updated in one PR get separate tags:

```bash
git tag dev-workflow/v0.1.1
git tag github-cli/v0.1.1
git push origin dev-workflow/v0.1.1
git push origin github-cli/v0.1.1
```

---

## 8. System-Level Skills

Skills aren't limited to code projects. A third scope exists beyond marketplace plugins and project-local skills: **system-level skills** that live in your user home directory and apply globally across all Claude Code sessions.

### The Three Skill Scopes

| Scope | Location | Applies To |
|-------|----------|------------|
| Marketplace plugin | Installed via `claude plugin install` | Any project that enables it in `settings.json` |
| Project-local | `.claude/skills/` in the repo | That repository only |
| System-level | `~/.claude/skills/` in home directory | Every Claude Code session on the machine |

### Omarchy: A Pre-Installed System Skill

[Omarchy](https://omarchy.org/) is an opinionated Arch Linux distribution built on Hyprland. It ships with a Claude Code skill pre-installed at `~/.claude/skills/omarchy/SKILL.md` that teaches Claude how to safely customize the Linux desktop environment.

The skill follows the same anatomy as any other skill — frontmatter with triggers, a "when to use" section, safety rules, reference tables — but its domain is the operating system itself rather than a codebase:

- **Triggers:** Hyprland, waybar, keybindings, themes, wallpaper, terminal config, monitors
- **Safety rules:** Read `~/.local/share/omarchy/` for reference, but never edit it (changes are lost on update). All customization goes in `~/.config/`.
- **Command discovery:** `compgen -c | grep omarchy` surfaces ~145 system commands
- **Decision framework:** Routes requests to the right pattern (stock command, config edit, custom theme, hooks, package install)

### Demo: Generating Theme Backgrounds

System skills turn Claude Code into an interface for your entire environment. Here's a prompt that combines the Omarchy skill's theme knowledge with image generation:

```
/omarchy

I don't particularly care for the background images that come with the pre-installed themes. For every pre-installed theme, I'd like to replace the default wallpaper with a single solid-color image that uses that theme's own background color as a flat fill. Name the file 0.default.png so it sorts first and becomes the theme's default wallpaper.
```

What happens behind the scenes:

1. `/omarchy` triggers the system skill
2. Claude reads the skill and understands theme structure: stock themes live read-only in `~/.local/share/omarchy/themes/<name>/` with a `colors.toml` containin the `/.config/omarchy/backgrounds/<name>/`.
3. Claude reads the stock themes from `~/.local/share/omarchy/themes/` to understand the file layout
4. For each theme, Claude extracts the background color from `colors.toml`, uses ImageMagick to generate a single solid-color PNG, and writes it as `0.default.png` into `~/.config/omarchy/backgrounds/<theme>/` - the `0.` prefix guarantees it sorts first in `omarchy-theme-bg-next`'s cycle and becomes the default.

The skill didn't need to anticipate this specific request. It provided enough context about theme structure, safe file locations, and system conventions for Claude to compose a solution from first principles. That's the value of a well-structured skill — it teaches Claude the domain rather than scripting individual tasks.

To see the full Claude Code session processing this prompt, refer to [Omarchy Backgrounds Session](./omarchy-backgrounds-session.md).
