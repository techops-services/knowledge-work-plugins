# Technology Stack

**Analysis Date:** 2026-02-04

## Languages

**Primary:**
- Markdown - Documentation and skill/command definitions across all plugins
- JSON - Configuration files, plugin manifests, and MCP server definitions

**Secondary:**
- YAML - Bioinformatics pipeline configuration (nextflow, genomics workflows)
- Python - Visualization and analysis scripts (referenced in data skills)
- SQL - Data analysis queries (referenced in data and finance skills)
- Shell - Bioinformatics pipeline execution scripts

## Runtime

**Environment:**
- Claude (Anthropic's language model) - Primary runtime for all plugins via Cowork and Claude Code
- Browser - HTML/JavaScript dashboards execution

**Package Manager:**
- Not applicable - No build system or package manager required
- Plugins are file-based with no compilation or installation process

## Frameworks

**Core:**
- Model Context Protocol (MCP) - HTTP-based tool integration standard
- Cowork - Anthropic's agentic desktop application (primary platform)
- Claude Code - Alternative platform for plugin installation

**Documentation:**
- Markdown for all skill definitions, command specifications, and reference materials

## Key Dependencies

**Critical:**
- MCP Servers (HTTP endpoints) - External integrations for all plugin connectors
- Anthropic Claude Model - Powers all plugin functionality

**Infrastructure:**
- HTTP over MCP Protocol - Communication with external MCP servers
- No server infrastructure required - Stateless, client-side operation

## Configuration

**Environment:**
- `.claude-plugin/plugin.json` - Plugin manifest with name, version, description, author
  - Example: `bio-research/.claude-plugin/plugin.json`
- `.mcp.json` - MCP server URL mappings for external integrations
  - Example: `sales/.mcp.json` with HTTP endpoints for Slack, HubSpot, Close, etc.
- `.claude/settings.local.json` - Optional user settings (personalization, quotas, credentials)
  - Example: `sales/.claude/settings.local.json` for sales team configuration

**Build:**
- No build configuration files detected
- No compilation, bundling, or build pipeline required
- Content is consumed directly by Claude runtime

## Platform Requirements

**Development:**
- Git repository (source control)
- Markdown editor or text editor
- Web browser for testing dashboards (data plugin)

**Production:**
- Cowork desktop application (primary deployment target) or Claude Code CLI
- Active Claude subscription
- Internet connectivity for MCP server endpoints
- Optional: Local configuration files for customization

## Marketplace & Distribution

**Marketplace:**
- `.claude-plugin/marketplace.json` - Centralized plugin registry at root level
  - Contains 11 registered plugins with names, sources, descriptions
  - Installed via Cowork plugin marketplace or Claude Code CLI
  - Format: `claude plugin install [plugin-name]@knowledge-work-plugins`

## Documentation Structure

**Plugin Documentation:**
- `README.md` - Plugin overview, installation, usage examples, workflows
- `CONNECTORS.md` - Tool-agnostic placeholder mapping to MCP server categories
- `.mcp.json` - Active MCP server configurations
- `LICENSE` - Apache 2.0 for each plugin

**Skill Documentation:**
- `SKILL.md` - Skill definition with purpose, steps, and context
- `references/` - Supporting documentation and reference materials
- `QUICKREF.md` - Quick reference guides (example: `create-an-asset/QUICKREF.md`)

**Command Documentation:**
- `commands/[command-name].md` - Slash command specification with purpose and usage

---

*Stack analysis: 2026-02-04*
