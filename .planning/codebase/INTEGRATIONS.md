# External Integrations

**Analysis Date:** 2026-02-04

## APIs & External Services

**Communication & Collaboration:**
- Slack - Chat and messaging integration
  - MCP Endpoint: `https://mcp.slack.com/mcp`
  - Used by: sales, marketing, customer-support, product-management, productivity, data, legal, enterprise-search
  - Capabilities: Message posting, channel access, workspace data

- Microsoft 365 - Email, calendar, collaboration
  - MCP Endpoint: `https://microsoft365.mcp.claude.com/mcp`
  - Used by: sales, customer-support, product-management, productivity, legal, finance, data, enterprise-search
  - Capabilities: Email, calendar access, document collaboration, Teams integration

**CRM & Sales:**
- HubSpot - CRM platform and sales data management
  - MCP Endpoint: `https://mcp.hubspot.com/anthropic`
  - Used by: sales, customer-support, marketing
  - Capabilities: Deal pipeline, contact records, company data

- Close - Sales CRM
  - MCP Endpoint: `https://mcp.close.com/mcp`
  - Used by: sales
  - Capabilities: Deal and contact management

- Clay - Data enrichment and prospecting
  - MCP Endpoint: `https://api.clay.com/v3/mcp`
  - Used by: sales
  - Capabilities: Prospect research, company enrichment

- ZoomInfo - B2B data and intelligence
  - MCP Endpoint: `https://mcp.zoominfo.com/mcp`
  - Used by: sales
  - Capabilities: Company and contact intelligence

**Project Management & Work Tracking:**
- Atlassian (Jira/Confluence) - Project tracking and wiki
  - MCP Endpoint: `https://mcp.atlassian.com/v1/mcp`
  - Used by: sales, customer-support, product-management, data, legal, enterprise-search
  - Capabilities: Issue tracking, documentation, wikis

- Linear - Issue tracking
  - MCP Endpoint: `https://mcp.linear.app/mcp`
  - Used by: product-management, productivity
  - Capabilities: Issue management, roadmap tracking

- Asana - Task and project management
  - MCP Endpoint: `https://mcp.asana.com/v2/mcp`
  - Used by: productivity, product-management, enterprise-search
  - Capabilities: Task lists, project planning, team collaboration

- Monday.com - Flexible work OS
  - MCP Endpoint: `https://mcp.monday.com/mcp`
  - Used by: productivity, product-management, sales
  - Capabilities: Custom workflows, task management, team views

- ClickUp - Unified work platform
  - MCP Endpoint: `https://mcp.clickup.com/mcp`
  - Used by: productivity, product-management
  - Capabilities: Task management, time tracking, collaboration

**Knowledge Management:**
- Notion - Document database and wiki
  - MCP Endpoint: `https://mcp.notion.com/mcp`
  - Used by: sales, customer-support, product-management, marketing, productivity, enterprise-search
  - Capabilities: Document access, database queries, content management

- Guru - Knowledge base platform
  - MCP Endpoint: `https://mcp.api.getguru.com/mcp`
  - Used by: customer-support, enterprise-search
  - Capabilities: Knowledge article search, content discovery

**File Storage & Management:**
- Box - Enterprise file management
  - MCP Endpoint: `https://mcp.box.com`
  - Used by: legal
  - Capabilities: File access, document management

- Egnyte - Cloud file platform
  - MCP Endpoint: `https://mcp-server.egnyte.com/mcp`
  - Used by: legal
  - Capabilities: Secure file access, document collaboration

**Customer Support:**
- Intercom - Customer communication platform
  - MCP Endpoint: `https://mcp.intercom.com/mcp`
  - Used by: customer-support, product-management
  - Capabilities: Ticket management, customer conversation history

**Design & Content Creation:**
- Figma - Design platform
  - MCP Endpoint: `https://mcp.figma.com/mcp`
  - Used by: product-management, marketing
  - Capabilities: Design file access, component library

- Canva - Design tool
  - MCP Endpoint: `https://mcp.canva.com/mcp`
  - Used by: marketing
  - Capabilities: Template access, design creation

**Analytics & Business Intelligence:**
- Amplitude - Product analytics
  - MCP Endpoint: `https://mcp.amplitude.com/mcp`
  - Used by: product-management, marketing, data
  - Capabilities: Event data, user cohorts, funnel analysis

- Pendo - Product experience platform
  - MCP Endpoint: `https://app.pendo.io/mcp/v0/shttp`
  - Used by: product-management
  - Capabilities: Feature usage, user feedback, analytics

**Marketing & SEO:**
- Ahrefs - SEO and backlink analysis
  - MCP Endpoint: `https://api.ahrefs.com/mcp/mcp`
  - Used by: marketing
  - Capabilities: SEO metrics, competitor analysis, backlink data

- SimilarWeb - Market intelligence
  - MCP Endpoint: `https://mcp.similarweb.com`
  - Used by: marketing
  - Capabilities: Website traffic analysis, competitive benchmarking

- Klaviyo - Email marketing platform
  - MCP Endpoint: `https://mcp.klaviyo.com/mcp`
  - Used by: marketing
  - Capabilities: Email campaign data, subscriber lists

**Meeting & Conversation Intelligence:**
- Fireflies - Meeting transcription and notes
  - MCP Endpoint: `https://api.fireflies.ai/mcp`
  - Used by: sales, product-management
  - Capabilities: Meeting transcripts, action item extraction

## Data Storage

**Data Warehouses:**
- Snowflake - Cloud data warehouse
  - MCP Endpoint: Empty/not configured (placeholder)
  - Used by: finance, data
  - Client/ORM: SQL via MCP
  - Status: Configured but endpoint URL pending

- Databricks - Data and AI platform
  - MCP Endpoint: Empty/not configured (placeholder)
  - Used by: finance, data
  - Client/ORM: SQL via MCP
  - Status: Configured but endpoint URL pending

- BigQuery - Google Cloud data warehouse
  - MCP Endpoint: `https://bigquery.googleapis.com/mcp`
  - Used by: finance, data
  - Client/ORM: SQL via MCP
  - Status: Active

**Analytics/Notebooks:**
- Hex - Collaborative notebooks and dashboards
  - MCP Endpoint: `https://app.hex.tech/mcp`
  - Used by: data
  - Capabilities: SQL notebooks, data exploration, visualization

**File Storage:**
- Local filesystem - Supported for CSV/Excel uploads
  - Used by: data, finance
  - Format support: CSV, Excel, JSON

**Caching:**
- None detected - No caching infrastructure configured

## Authentication & Identity

**Auth Provider:**
- Custom via MCP token/API key configuration
  - Implementation: OAuth 2.0 or API keys configured in `.claude/settings.local.json`
  - No centralized auth system - Each MCP server authenticates independently
  - Example: `sales/.claude/settings.local.json` for user personalization and API credentials

## Monitoring & Observability

**Error Tracking:**
- Not detected - No centralized error tracking service configured

**Logs:**
- Console output or local files within Claude/Cowork environment
- No external logging service detected

## CI/CD & Deployment

**Hosting:**
- Anthropic Cowork (primary deployment platform)
- Claude Code CLI (secondary installation method)
- GitHub repository (source control and distribution)

**Distribution:**
- Claude plugin marketplace: `claude plugin marketplace add anthropics/knowledge-work-plugins`
- Per-plugin installation: `claude plugin install [plugin-name]@knowledge-work-plugins`

**CI Pipeline:**
- None detected - File-based plugins with no build pipeline

## Environment Configuration

**Required env vars:**
- None detected in root configuration
- Individual plugins may require MCP server authentication tokens:
  - `SALESFORCE_API_KEY` (if configured)
  - `HUBSPOT_API_KEY` (if configured)
  - Similar patterns for each MCP server

**Secrets location:**
- `.claude/settings.local.json` - Local configuration file (not committed to git)
- Environment variables or MCP server configuration
- Cowork/Claude Code credential storage

## Webhooks & Callbacks

**Incoming:**
- None detected - Plugins do not expose webhook endpoints

**Outgoing:**
- MCP HTTP calls to external services (unidirectional)
- No callback/webhook pattern observed

## Scientific & Research Tools (bio-research plugin)

**Literature & Knowledge:**
- PubMed - Biomedical literature search
  - MCP Endpoint: `https://pubmed.mcp.claude.com/mcp`
  - Capabilities: Literature search, abstract retrieval

- bioRxiv - Preprint repository
  - MCP Endpoint: `https://mcp.deepsense.ai/biorxiv/mcp`
  - Capabilities: Preprint search, research discovery

- Wiley Scholar Gateway - Journal access
  - MCP Endpoint: `https://connector.scholargateway.ai/mcp`
  - Capabilities: Journal article access, publication browsing

- Open Targets - Drug target discovery
  - MCP Endpoint: `https://mcp.platform.opentargets.org/mcp`
  - Capabilities: Target prioritization, genetics data

- ChEMBL - Chemical database
  - MCP Endpoint: `https://mcp.deepsense.ai/chembl/mcp`
  - Capabilities: Bioactive compound search, drug discovery

**Research Platforms:**
- BioRender - Scientific illustration
  - MCP Endpoint: `https://mcp.services.biorender.com/mcp`
  - Capabilities: Figure creation, scientific visualization

- Synapse (Sage Bionetworks) - Research data management
  - MCP Endpoint: `https://mcp.synapse.org/mcp`
  - Capabilities: Collaborative data storage, version control

- Owkin - AI for biology
  - MCP Endpoint: `https://mcp.k.owkin.com/mcp`
  - Capabilities: Histopathology analysis, drug discovery AI

- Benchling - Lab data management
  - MCP Endpoint: Empty/not configured
  - Capabilities: DNA sequences, experimental data, lab workflows
  - Status: Placeholder - MCP URL not yet configured

**Clinical & Regulatory:**
- ClinicalTrials.gov - Clinical trial registry
  - MCP Endpoint: `https://mcp.deepsense.ai/clinical_trials/mcp`
  - Capabilities: Clinical trial search, protocol information

**Optional/Binary MCP Servers (requires separate installation):**
- 10X Genomics txg-mcp - Genomics analysis
  - Binary: GitHub release download required
  - Capabilities: Cloud analysis data, sequencing workflows

- ToolUniverse - AI tools database
  - Binary: GitHub release download required
  - Source: Harvard MIMS
  - Capabilities: Scientific tool discovery, AI tool catalog

## Bioinformatics Execution

**Pipeline Framework:**
- Nextflow - Workflow orchestration
  - Configuration: `bio-research/skills/nextflow-development/scripts/config/`
  - Supported pipelines: rnaseq, sarek, atacseq (nf-core)

**Analysis Tools (referenced in skills):**
- scvi-tools - Deep learning for single-cell omics
- Nextflow pipelines - RNA-seq, variant calling, ATAC-seq
- Local file support - .h5ad, .h5, CSV, Excel, TXT, PDF

---

*Integration audit: 2026-02-04*
