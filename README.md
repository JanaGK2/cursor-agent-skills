# Agent Skills for Google Apps Script & Sheets

A collection of Agent Skills for building Google Apps Script dashboards, managing Google Sheets programmatically, and handling long conversation context.

## Compatible Tools

These skills use the open [Agent Skills standard](https://agentskills.io) and work with **16+ AI coding tools**:

| Tool | Developer | Directory |
|------|-----------|-----------|
| **Cursor** | Anysphere | `~/.cursor/skills/` |
| **Claude Code** | Anthropic | `~/.claude/skills/` |
| **Gemini CLI** | Google | `~/.gemini/skills/` |
| **OpenAI Codex** | OpenAI | `~/.codex/skills/` |
| **GitHub Copilot** | GitHub | `.github/skills/` |
| **VS Code** | Microsoft | `.vscode/skills/` |
| **Junie** | JetBrains | `.junie/skills/` |

And more - see [agentskills.io](https://agentskills.io) for the full list.

## Skills Included

### apps-script-dashboard

Build interactive web app dashboards using Google Apps Script and Google Sheets as a backend.

**Features:**
- Step-by-step dashboard build process
- Email whitelisting for access control
- Performance optimization patterns (chunked caching, compact encoding)
- Multi-select filters with bidirectional cascading
- Client-side filtering for responsive UX
- Typeahead search and pagination patterns
- Apps Script quotas and limits reference
- Long-running task patterns (6-minute workaround)
- Installable triggers management
- PropertiesService vs CacheService guidance
- Web app as REST API endpoint

### apps-script-visualizations

Add interactive charts and UI components to Apps Script dashboards.

**Features:**
- Library selection guide (Google Charts, Chart.js, D3.js, Plotly, ApexCharts)
- Responsive chart patterns (window resize handling)
- CSS design system with tokens
- UI components: stat cards, chart grids, pills, progress bars
- Tab navigation, sticky headers, loading overlays
- Expandable text rows, pagination controls

### google-sheets-management

Manage Google Sheets programmatically via Python scripts and the Sheets API.

**Features:**
- MCP tool reference
- Fast bulk upload (5000+ rows per API call)
- Grid expansion helpers
- API quotas and rate limiting (exponential backoff)
- Service account authentication for automation
- Gemini AI functions (=AI() in Sheets)
- Comprehensive error handling (HTTP status codes, 503 troubleshooting)
- Clear vs append mode workflows
- JSON to Sheets conversion

### context-handoff

Manage long conversation context to prevent LLM degradation.

**Features:**
- Signals for when to create a handoff
- Context document template
- Research-backed guidelines

## Installation

### Quick Install (Any Tool)

Copy the skill folders to your tool's skills directory:

```bash
# For Cursor
cp -r apps-script-dashboard ~/.cursor/skills/
cp -r apps-script-visualizations ~/.cursor/skills/
cp -r google-sheets-management ~/.cursor/skills/
cp -r context-handoff ~/.cursor/skills/

# For Claude Code
cp -r apps-script-dashboard ~/.claude/skills/
cp -r apps-script-visualizations ~/.claude/skills/
cp -r google-sheets-management ~/.claude/skills/
cp -r context-handoff ~/.claude/skills/

# For other tools, use the appropriate directory from the table above
```

### Install from GitHub (Cursor 2.4+)

1. Open **Cursor Settings → Rules**
2. Click **Add Rule → Remote Rule (GitHub)**
3. Enter: `https://github.com/JanaGK2/cursor-agent-skills`

Skills will automatically be available in your agent sessions.

### Skill Structure

Each skill follows this structure:
```
skill-name/
├── SKILL.md           # Main skill documentation
├── templates/         # Code templates (optional)
├── scripts/           # Helper scripts (optional)
└── reference/         # Additional reference docs (optional)
```

## Usage

Skills are automatically loaded when relevant. You can also invoke them explicitly:

```
# Cursor / Claude Code
/apps-script-dashboard build me a dashboard for this data

# Or reference as context
@apps-script-dashboard
```

The agent reads the skill instructions and applies the patterns automatically.

## Requirements

- **Google Sheets Management**: Requires Google Cloud OAuth credentials or service account
- **Apps Script Skills**: Requires a Google account with Apps Script access

## License

MIT License - feel free to use, modify, and distribute.

## Attribution

Portions of these skills are modifications based on work created and shared by Google and used according to terms described in the [Creative Commons 4.0 Attribution License](https://creativecommons.org/licenses/by/4.0/).

### Sources

These skills incorporate patterns, best practices, and reference information from the following Google documentation:

- [Google Apps Script Guides](https://developers.google.com/apps-script/guides)
- [HTML Service Best Practices](https://developers.google.com/apps-script/guides/html/best-practices)
- [Apps Script Web Apps](https://developers.google.com/apps-script/guides/web)
- [Installable Triggers](https://developers.google.com/apps-script/guides/triggers/installable)
- [Apps Script Quotas](https://developers.google.com/apps-script/guides/services/quotas)
- [Properties Service](https://developers.google.com/apps-script/guides/properties)
- [Cache Service](https://developers.google.com/apps-script/reference/cache/cache)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Sheets API Quotas](https://developers.google.com/sheets/api/limits)

Code samples from Google documentation are licensed under the [Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).

## Contributing

Contributions welcome! Please ensure any additions follow the existing patterns and include proper documentation and attribution.
