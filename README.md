# Cursor Agent Skills

A collection of skills for Cursor AI agents to help with Google Apps Script dashboards, Google Sheets management, and conversation context management.

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

### For Cursor Users

1. Copy the skill folders to your Cursor skills directory:
   ```bash
   cp -r apps-script-dashboard ~/.cursor/skills/
   cp -r apps-script-visualizations ~/.cursor/skills/
   cp -r google-sheets-management ~/.cursor/skills/
   cp -r context-handoff ~/.cursor/skills/
   ```

2. The skills will automatically be available in your Cursor agent sessions.

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

Skills are automatically loaded by Cursor agents when relevant. You can also explicitly reference them:

```
@apps-script-dashboard build me a dashboard for this data
```

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
