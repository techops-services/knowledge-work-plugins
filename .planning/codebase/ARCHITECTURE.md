# Architecture

**Analysis Date:** 2026-02-04

## Pattern Overview

**Overall:** Plugin-Based Agent Architecture for Domain-Specific AI Assistance

**Key Characteristics:**
- **Skill-driven**: Encapsulates domain expertise as markdown-based knowledge documents that Claude draws on automatically
- **Command-oriented**: Explicit user actions exposed as slash commands with structured workflows
- **Tool-agnostic**: Uses MCP (Model Context Protocol) server abstraction to decouple plugin logic from specific external tools
- **Declarative configuration**: No code needed; entire plugins defined through JSON manifests and markdown files
- **Marketplace-native**: Individual plugins published in a marketplace, installed into Claude Code or Cowork, and activated in user sessions

## Layers

**Plugin Layer (User Interface):**
- Purpose: Define user interactions through slash commands and automatic skill activation
- Location: `[plugin-name]/commands/` and `[plugin-name]/skills/`
- Contains: Command workflows (markdown) and skill knowledge documents (markdown)
- Depends on: MCP servers for external tool access; Claude for skill interpretation
- Used by: End users invoking commands or triggering skills contextually

**MCP Integration Layer (Tool Abstraction):**
- Purpose: Abstract away specific tool implementations and provide consistent tool interfaces to Claude
- Location: `[plugin-name]/.mcp.json`
- Contains: HTTP-based MCP server configurations for each tool category (support platform, CRM, chat, etc.)
- Depends on: Distributed MCP servers run independently (e.g., `https://mcp.slack.com/mcp`)
- Used by: Plugin commands and skills when they need to query or update external tools

**Manifest & Metadata Layer (Plugin Configuration):**
- Purpose: Define plugin identity, version, description, and entry points for the Cowork/Claude Code runtime
- Location: `[plugin-name]/.claude-plugin/plugin.json` (individual plugin); `.claude-plugin/marketplace.json` (marketplace)
- Contains: Plugin metadata, version info, author details
- Depends on: File system structure and naming conventions
- Used by: Plugin installer/marketplace to identify installable plugins and dependencies

## Data Flow

**Command Execution Flow:**

1. User invokes slash command (e.g., `/triage "Customer issue here"`)
2. Claude Code / Cowork runtime identifies matching command file in `commands/[command-name].md`
3. Command markdown defines workflow steps that may reference relevant skills
4. Claude reads referenced skill documents from `skills/[skill-name]/SKILL.md`
5. Skills may trigger MCP calls (via `.mcp.json` connections) to external tools
6. Claude synthesizes tool data with skill instructions and generates structured output
7. Output presented to user; user can trigger follow-up commands or next steps

**Skill Activation Flow:**

1. User creates content or asks a question in Cowork/Claude Code session
2. Claude monitors for context matching skill descriptions
3. When relevant, Claude automatically references skill knowledge (no explicit invocation)
4. Skill instructions guide Claude's reasoning for that task
5. If skill requires external data, Claude queries MCP connections
6. Skill knowledge influences Claude's response without user knowing a skill triggered

**Tool Connection Flow:**

1. Plugin declares tool categories in `CONNECTORS.md` (e.g., `~~support platform`, `~~CRM`)
2. `.mcp.json` pre-configures default MCP servers for each category
3. User can override by editing `.mcp.json` or connecting alternative MCP servers via Claude settings
4. When skill/command references a placeholder (e.g., `~~support platform`), Claude queries that MCP connection
5. MCP server responds with available tools and executes queries against external service

**State Management:**
- **No persistent state within plugins**: State exists entirely in external tools (support platforms, project trackers, etc.)
- **Conversation memory**: Cowork/Claude Code maintains session context; users provide context manually or reference tool queries
- **Configuration state**: Persisted in `.mcp.json` file; user-customized by editing connectors or answering prompts in cowork-plugin-customizer

## Key Abstractions

**Plugin:**
- Purpose: Self-contained package of related skills, commands, and tool connectors for a specific job role
- Examples: `sales/`, `customer-support/`, `finance/`
- Pattern: Directory with `.claude-plugin/plugin.json`, `.mcp.json`, `commands/`, `skills/` subdirectories

**Skill:**
- Purpose: Encapsulates domain expertise, best practices, and step-by-step workflows as knowledge Claude draws on automatically
- Examples: `customer-support/skills/ticket-triage/SKILL.md`, `sales/skills/call-prep/SKILL.md`
- Pattern: Markdown file with YAML frontmatter (`name`, `description`, optional `compatibility`); frontmatter triggers Claude to reference skill when relevant

**Command:**
- Purpose: Explicit user action with a structured workflow; invoked via slash syntax (e.g., `/triage`, `/forecast`)
- Examples: `customer-support/commands/triage.md`, `sales/commands/call-summary.md`
- Pattern: Markdown file with YAML frontmatter (`description`, `argument-hint`); describes step-by-step workflow for Claude to execute

**MCP Server Connection:**
- Purpose: Provides standardized interface to an external tool (Slack, Jira, HubSpot, etc.)
- Examples: `https://mcp.slack.com/mcp`, `https://mcp.hubspot.com/anthropic`
- Pattern: HTTP endpoint declared in `.mcp.json` under `mcpServers` object; tool-agnostic category name (e.g., `slack` is an instance of the `chat` category)

**Customization Point (Placeholder):**
- Purpose: Marks generic content that should be replaced with organization-specific values or tool names
- Examples: `~~support platform` (replaced with `Zendesk` or `Intercom`), `~~your-org-channel` (replaced with `#engineering`)
- Pattern: String prefixed with `~~` in markdown or JSON; identified via `grep -rn '~~\w'` for customization audit

## Entry Points

**Cowork Desktop App:**
- Location: Installed via Cowork's plugin marketplace or file system mount
- Triggers: User installs plugin; skills activate automatically in sessions; slash commands available
- Responsibilities: Load plugin files, maintain session context, expose MCP connections, execute commands and activate skills

**Claude Code CLI:**
- Location: `claude plugin install [plugin-name]@knowledge-work-plugins`
- Triggers: Plugin installed; activated on session start
- Responsibilities: Register slash commands in session, make skills available during task execution

**Marketplace Registry:**
- Location: `.claude-plugin/marketplace.json` at repository root
- Triggers: User browses plugin marketplace; orchestrator discovers available plugins
- Responsibilities: Index all plugins in repository; provide metadata for discovery and installation

## Error Handling

**Strategy:** Graceful degradation with fallback guidance

**Patterns:**

1. **Tool connection unavailable**: Skill/command detects MCP connection failure, falls back to requesting user to provide context manually
   - Example: `customer-support/commands/triage.md` includes guidance: "If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](CONNECTORS.md)"

2. **Customization point unresolved**: Plugin operates with `~~`-prefixed placeholder intact, prompts user to customize or provides generic workflow
   - Example: `cowork-plugin-management` skill with fallback: "If knowledge MCPs didn't provide a specific answer, ask"

3. **Workflow ambiguity**: Commands include next-steps prompts to guide user toward desired action
   - Example: In `triage` command output: "Want me to draft a full response to the customer?" / "Should I search for more context?"

4. **Missing required context**: Skills request explicit user input rather than assuming defaults
   - Example: `ticket-triage` skill: "When in doubt, lean toward Bug — it's better to investigate than dismiss"

## Cross-Cutting Concerns

**Logging:** Not implemented within plugins; all logging occurs in external tools (e.g., Slack for chat context, project tracker for issue history)

**Validation:** Embedded in skill knowledge through category taxonomies and priority frameworks
  - Example: `ticket-triage` skill defines category taxonomy with signal words and priority criteria (P1–P4 framework)

**Authentication:** Delegated to MCP servers; handled during user setup when connecting external tools to Claude settings

**Authorization:** Inherited from external tool permissions; Claude respects tool-level access control

**Customization:** Central to design via placeholder pattern; `cowork-plugin-customizer` skill automates replacement workflow

---

*Architecture analysis: 2026-02-04*
