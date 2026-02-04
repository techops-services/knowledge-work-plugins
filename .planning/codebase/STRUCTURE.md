# Codebase Structure

**Analysis Date:** 2026-02-04

## Directory Layout

```
knowledge-work-plugins/
├── .claude-plugin/
│   ├── marketplace.json           # Registry of all plugins in repo
│   └── plugin.json               # Marketplace-level plugin metadata
├── .planning/
│   └── codebase/                 # Analysis documents (this directory)
├── .git/                         # Git repository metadata
├── README.md                     # Main repository documentation
├── LICENSE                       # Apache 2.0 license
│
├── productivity/                 # Task & calendar management plugin
│   ├── .claude-plugin/
│   │   └── plugin.json          # Plugin manifest
│   ├── .mcp.json                # MCP server connections (Slack, Notion, Asana, etc.)
│   ├── commands/
│   │   ├── start.md             # /start command workflow
│   │   └── update.md            # /update command workflow
│   ├── skills/
│   │   ├── dashboard.html       # HTML-rendered dashboard for task view
│   │   ├── memory-management/
│   │   │   └── SKILL.md         # Context retention and preferences skill
│   │   └── task-management/
│   │       └── SKILL.md         # Daily planning and task orchestration skill
│   ├── README.md                # Plugin-specific documentation
│   ├── CONNECTORS.md            # Tool category mappings
│   └── LICENSE
│
├── sales/                        # Sales pipeline and outreach plugin
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── .mcp.json                # HubSpot, Close, Clay, ZoomInfo, Notion, Jira, Fireflies, MS365
│   ├── commands/
│   │   ├── call-summary.md
│   │   ├── forecast.md
│   │   └── pipeline-review.md
│   ├── skills/
│   │   ├── account-research/
│   │   │   └── SKILL.md
│   │   ├── call-prep/
│   │   │   └── SKILL.md
│   │   ├── competitive-intelligence/
│   │   │   └── SKILL.md
│   │   ├── create-an-asset/
│   │   │   ├── README.md
│   │   │   ├── QUICKREF.md
│   │   │   └── SKILL.md
│   │   ├── daily-briefing/
│   │   │   └── SKILL.md
│   │   └── draft-outreach/
│   │       └── SKILL.md
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── customer-support/            # Support ticket & KB management plugin
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── .mcp.json                # Slack, Intercom, HubSpot, Guru, Jira, Notion, MS365
│   ├── commands/
│   │   ├── triage.md            # /triage command (ticket categorization)
│   │   ├── escalate.md          # /escalate command (package for engineering)
│   │   ├── research.md          # /research command (multi-source investigation)
│   │   ├── draft-response.md    # /draft-response command (compose customer reply)
│   │   └── kb-article.md        # /kb-article command (create help docs)
│   ├── skills/
│   │   ├── ticket-triage/
│   │   │   └── SKILL.md         # Category taxonomy, priority framework (P1-P4)
│   │   ├── customer-research/
│   │   │   └── SKILL.md         # Multi-source research methodology
│   │   ├── response-drafting/
│   │   │   └── SKILL.md         # Communication templates and tone guidelines
│   │   ├── escalation/
│   │   │   └── SKILL.md         # Escalation tiers and structured format
│   │   └── knowledge-management/
│   │       └── SKILL.md         # KB article standards and searchability
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── product-management/          # Roadmap and spec writing plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # Linear, Asana, Monday, ClickUp, Jira, Notion, Figma, Amplitude, Pendo, Intercom, Fireflies
│   ├── commands/
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── marketing/                   # Content and campaign planning plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # Canva, Figma, HubSpot, Amplitude, Notion, Ahrefs, SimilarWeb, Klaviyo
│   ├── commands/
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── legal/                       # Contract review and compliance plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # Box, Egnyte, Jira, MS365
│   ├── commands/
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── finance/                     # Journal entries and financial statements plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # Snowflake, Databricks, BigQuery, Slack, MS365
│   ├── commands/
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── data/                        # SQL queries and data visualization plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # Snowflake, Databricks, BigQuery, Hex, Amplitude, Jira
│   ├── commands/
│   │   ├── write-query.md       # SQL writing and execution
│   │   ├── analyze.md           # Statistical analysis command
│   │   └── build-dashboard.md   # Visualization command
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── enterprise-search/           # Cross-tool search plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # Slack, Notion, Guru, Jira, Asana, MS365
│   ├── commands/
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
├── bio-research/                # Life sciences research plugin
│   ├── .claude-plugin/
│   ├── .mcp.json                # PubMed, BioRender, bioRxiv, ClinicalTrials.gov, ChEMBL, Synapse, Wiley, Owkin, Open Targets, Benchling
│   ├── commands/
│   ├── skills/
│   ├── README.md
│   ├── CONNECTORS.md
│   └── LICENSE
│
└── cowork-plugin-management/    # Plugin customization meta-plugin
    ├── .claude-plugin/
    │   └── plugin.json
    ├── skills/
    │   └── cowork-plugin-customizer/
    │       ├── SKILL.md                      # Main customization workflow
    │       ├── examples/
    │       │   └── customized-mcp.json       # Example of customized MCP config
    │       └── references/
    │           ├── mcp-servers.md            # MCP discovery workflow and registry
    │           └── search-strategies.md      # Query patterns for knowledge MCPs
    ├── README.md
    └── LICENSE
```

## Directory Purposes

**`.claude-plugin/`:**
- Purpose: Plugin manifest and marketplace registry
- Contains: `plugin.json` (individual plugin metadata), `marketplace.json` (registry of all plugins)
- Key files: `plugin.json` defines plugin name, version, description, author

**`commands/`:**
- Purpose: Explicit user-invoked workflows accessible as slash commands
- Contains: One markdown file per command
- Naming pattern: `[command-name].md` (e.g., `triage.md`, `call-summary.md`)
- Frontmatter: `description` (what user sees in help), `argument-hint` (expected input format)

**`skills/`:**
- Purpose: Domain expertise and best practices that Claude uses automatically
- Contains: Named subdirectories, each with `SKILL.md` plus optional supporting files (README.md, QUICKREF.md, examples/, references/)
- Naming pattern: `[skill-name]/SKILL.md` (e.g., `ticket-triage/SKILL.md`)
- Frontmatter: `name` (identifier), `description` (when to use), optional `compatibility` (runtime requirements)

**`.mcp.json`:**
- Purpose: Configure MCP server connections for external tools
- Contains: HTTP endpoints for each tool category (chat, CRM, support platform, etc.)
- Format: JSON object with `mcpServers` key containing tool-name → endpoint mapping
- Optional: Some servers have empty URLs intentionally (e.g., Snowflake, Databricks) for user configuration

**`.planning/codebase/`:**
- Purpose: Analysis and planning documents generated by GSD tools
- Generated: Yes (by `/gsd:map-codebase`)
- Committed: Yes
- Contains: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, CONCERNS.md

## Key File Locations

**Entry Points:**
- `.claude-plugin/marketplace.json`: Root registry; defines all 11 plugins available for installation
- `[plugin-name]/.claude-plugin/plugin.json`: Individual plugin manifest; loaded by installer

**Configuration:**
- `[plugin-name]/.mcp.json`: External tool connections for that plugin; user edits to customize or add tools
- `[plugin-name]/CONNECTORS.md`: Human-readable documentation of tool categories and placeholders

**Core Logic:**
- `[plugin-name]/commands/[command].md`: Workflow implementation for slash commands
- `[plugin-name]/skills/[skill-name]/SKILL.md`: Domain expertise and knowledge; referenced automatically

**Customization:**
- `cowork-plugin-management/skills/cowork-plugin-customizer/SKILL.md`: Orchestrates plugin customization workflow
- `cowork-plugin-management/skills/cowork-plugin-customizer/examples/customized-mcp.json`: Template for customized configs
- `cowork-plugin-management/skills/cowork-plugin-customizer/references/`: Query patterns and MCP registry docs

## Naming Conventions

**Files:**
- Commands: kebab-case: `call-summary.md`, `draft-response.md`
- Skills: kebab-case nested: `ticket-triage/SKILL.md`, `response-drafting/SKILL.md`
- Manifests: `plugin.json`, `.mcp.json`, `marketplace.json` (exact)
- Documentation: `README.md`, `CONNECTORS.md`, `LICENSE` (exact)
- Supporting docs: `QUICKREF.md`, `SKILL.md`, `references/`, `examples/` (exact)

**Directories:**
- Plugins: kebab-case at repo root: `customer-support/`, `product-management/`, `bio-research/`
- Skill groups: kebab-case: `ticket-triage/`, `response-drafting/`, `call-prep/`
- Plugin internals: lowercase standard: `.claude-plugin/`, `commands/`, `skills/`, `references/`, `examples/`

**Placeholder tokens in content:**
- Format: `~~category-name` (double-tilde prefix + kebab-case)
- Examples: `~~support platform`, `~~CRM`, `~~your-org-channel`
- Used in: Markdown skill/command files and JSON MCP configs
- Discoverable via: `grep -rn '~~\w' [plugin-dir]`

## Where to Add New Code

**New Plugin:**
- Create directory: `[new-plugin-name]/` at repo root
- Add manifest: `[new-plugin-name]/.claude-plugin/plugin.json` with name, version, description, author
- Add MCP config: `[new-plugin-name]/.mcp.json` with required tool connectors
- Add commands: `[new-plugin-name]/commands/[command-name].md` (YAML frontmatter + workflow)
- Add skills: `[new-plugin-name]/skills/[skill-name]/SKILL.md` (YAML frontmatter + knowledge)
- Add documentation: `[new-plugin-name]/README.md`, `[new-plugin-name]/CONNECTORS.md`
- Register in marketplace: Update `.claude-plugin/marketplace.json` with new plugin entry (name, source, description)

**New Command:**
- File: `[plugin-name]/commands/[new-command-name].md`
- Frontmatter: `description` (user-facing help text), `argument-hint` (input format guidance)
- Content: Markdown describing the workflow; reference relevant skills; include next-steps guidance

**New Skill:**
- Directory: `[plugin-name]/skills/[new-skill-name]/`
- File: `[new-skill-name]/SKILL.md`
- Frontmatter: `name` (short identifier), `description` (trigger conditions), optional `compatibility`
- Content: Markdown with domain expertise, frameworks, templates, step-by-step guidance
- Optional files: `README.md` (overview), `QUICKREF.md` (quick reference), `examples/` (sample outputs), `references/` (deep dives)

**Utility or Shared Content:**
- Shared across plugins: Store in `cowork-plugin-management/skills/` or create a new `shared/` plugin
- Plugin-specific utilities: Place in `[plugin-name]/skills/[utility-name]/`
- External tool references: Document in `[plugin-name]/CONNECTORS.md` with category mappings

## Special Directories

**`.claude-plugin/`:**
- Purpose: Plugin metadata and marketplace registry
- Generated: No
- Committed: Yes

**`.mcp.json`:**
- Purpose: MCP server configuration
- Generated: No (but generated by cowork-plugin-customizer during setup)
- Committed: Yes (default config pre-populated; user overrides locally)

**`examples/` and `references/`:**
- Purpose: Supporting documentation and templates for skills
- Generated: No
- Committed: Yes

**`.git/`:**
- Purpose: Git repository metadata
- Generated: Yes
- Committed: N/A (git internal)

---

*Structure analysis: 2026-02-04*
