---
name: apps-script-dashboard
description: Build Google Apps Script web app dashboards with filters, KPIs, tables, and charts. Use when creating interactive dashboards backed by Google Sheets data.
---

# Build Apps Script Dashboard

## When to Use

- User asks to create a dashboard from Google Sheets data
- Need interactive filters, KPIs, paginated tables
- Building web app with `HtmlService`

## Architecture Pattern

```
Google Sheet (data source)
    ↓
Apps Script (Code.gs)
    ├── doGet() → serves HTML
    ├── getData() → returns filtered data
    ├── getFilterOptions() → returns filter values
    └── calculateKPIs() → returns aggregated metrics
    ↓
HTML/CSS/JS (DashboardUI.html)
    ├── Filter bar with multi-select dropdowns
    ├── KPI cards
    ├── Data table with pagination
    └── Charts (via external libraries)
```

## Step-by-Step Build Process

### 1. Prepare Data Sheet

Ensure source sheet has:
- Header row with column names
- Clean data (no merged cells)
- Filter columns prefixed with `_` for clarity (e.g., `_GEO`, `_SEGMENT`)

### 2. Create Apps Script Project

In Google Sheet: Extensions → Apps Script

### 3. Implement Code.gs

Use template from `templates/Code.gs`:

```javascript
// Configuration
const CONFIG = {
  DATA_SHEET: 'Data',
  ROWS_PER_PAGE: 100,
};

// Web app entry point
function doGet() {
  return HtmlService.createHtmlOutputFromFile('DashboardUI')
    .setTitle('Dashboard')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

// Load data with optional caching
function loadData() {
  const cache = CacheService.getScriptCache();
  const cached = cache.get('data');
  if (cached) return JSON.parse(cached);
  
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(CONFIG.DATA_SHEET);
  const data = sheet.getDataRange().getValues();
  
  // Convert to objects
  const headers = data[0];
  const rows = data.slice(1).map(row => {
    const obj = {};
    headers.forEach((h, i) => obj[h] = row[i]);
    return obj;
  });
  
  try {
    cache.put('data', JSON.stringify(rows), 21600); // 6 hours
  } catch (e) {
    console.log('Data too large for cache');
  }
  
  return rows;
}
```

### 4. Implement DashboardUI.html

Use template from `templates/DashboardUI.html`:

Key components:
- CSS variables for consistent styling
- Filter bar with multi-select dropdowns
- `google.script.run` for server calls
- Client-side filtering for responsiveness

### 5. Add Charts (Optional)

See `apps-script-visualizations` skill for chart integration.

### 6. Configure appsscript.json Manifest (CRITICAL for Web Apps)

**When deploying as a web app that accesses spreadsheets by ID**, you MUST configure OAuth scopes in the manifest. Without this, you'll get:

```
Error: You do not have permission to call SpreadsheetApp.openById
```

**Steps:**
1. In Apps Script editor, click **gear icon** (Project Settings)
2. Check: "Show 'appsscript.json' manifest file in editor"
3. Click `appsscript.json` in file list and add scopes:

```json
{
  "timeZone": "America/New_York",
  "dependencies": {},
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/script.container.ui",
    "https://www.googleapis.com/auth/userinfo.email"
  ],
  "webapp": {
    "executeAs": "USER_DEPLOYING",
    "access": "ANYONE"
  }
}
```

**When is this required?**
- Spreadsheet created programmatically (MCP, API, Python script)
- Script added to sheet after creation (not via Extensions > Apps Script in that sheet)
- Using `SpreadsheetApp.openById()` instead of `getActiveSpreadsheet()`
- Deploying as standalone web app (not container-bound)

**When is this NOT required?**
- Script created via Extensions > Apps Script from within the target sheet
- Container-bound scripts accessing only their parent spreadsheet

### 7. Deploy workflow — ask first, then recommend

**Before scaffolding a new Apps Script web app (or establishing a local copy), ask the user:**

> Do you want a **clasp** local project (repo folder ↔ Google sync), or **browser-only** in script.google.com?

Give an **opinionated recommendation** based on deployment type, then follow their choice.

| Situation | Recommend | Why |
|-----------|-----------|-----|
| Shared web app (`/exec`), will iterate, lives in a git repo | **clasp** | Local edit in Cursor, `clasp push`, then `clasp deploy -i <id>` for versioned updates |
| Production / team dashboard with versioned deployments | **clasp** | Redeploys are repeatable from the terminal; code stays reviewable |
| One-off throwaway, tiny experiment, or user refuses local setup | **Browser-only** | Faster; no Node/clasp setup |
| Container-bound script only (Extensions → Apps Script on a sheet), no standalone web app | **Browser-only** (or clasp only if they want git) | Often never leaves the sheet editor |

#### Head (`/dev`) vs versioned (`/exec`)

| | Test (Head) | Production |
|--|-------------|------------|
| URL | ends in `/dev` | ends in `/exec` |
| Updates | latest saved code | needs new version / `clasp deploy` |
| Who can open | script **editors** only | per deployment access settings |

Never share `/dev` as the team link. Shared dashboards use versioned `/exec` + redeploy after code changes.

#### If user chooses clasp

Prerequisite: Node.js + `@google/clasp` (`npm i -g @google/clasp`), then `clasp login` once.

1. Create a local folder in the repo (e.g. `webapp/` or `dashboards/<name>/`).
2. Add `appsscript.json` (manifest with required OAuth scopes — see §6).
3. Add `.gs` / `.html` sources in that folder.
4. Link to Google:
   - New project: `clasp create --type webapp --title "…" --rootDir .`
   - Existing project: `clasp clone <scriptId>` or write `.clasp.json` with `scriptId` + `rootDir`.
5. Sync: `clasp push` (upload) / `clasp pull` (download).
6. First shareable URL: Deploy → New deployment → Web app (or `clasp deploy`), copy the `/exec` URL.
7. Later updates to the **same** URL: `clasp push` then `clasp deploy -i <DEPLOYMENT_ID> -d "note"`.

#### If user chooses browser-only

1. Deploy → New deployment → Web app
2. Execute as: Me (user deploying)
3. Who has access: Anyone (if using email whitelisting in code)
4. Copy the `/exec` URL
5. **Re-authorize when prompted** — Google will ask for spreadsheet access
6. Later code changes still need Manage deployments → Edit → New version (Head `/dev` is not a shared production link)

## Email Whitelisting (Access Control)

Instead of managing access through Google's deployment settings (which requires redeployment every time you add/remove users), implement email whitelisting in the script. This allows:
- Deploy once with broad access ("Anyone" or "Anyone with Google account")
- Control who can access via code or a Whitelist sheet
- No redeployment needed when access changes

### CRITICAL: userinfo.email Scope Required

**If you see `email: unknown` or empty email in access denied messages**, the script is missing the `userinfo.email` OAuth scope.

When using whitelisting with "Execute as: Me" deployment:
- The script runs with the deployer's credentials (can access spreadsheet)
- `Session.getActiveUser().getEmail()` returns the **visitor's** email (not deployer's)
- But ONLY if `https://www.googleapis.com/auth/userinfo.email` is declared in `appsscript.json`

Without this scope, email detection fails silently and returns empty/unknown. This scope is already shown in the Step 6 `appsscript.json` example above — do not omit it.

### CRITICAL: Web App Context

**In standalone web apps, `getActiveSpreadsheet()` returns NULL!**

Always use `openById()` with an explicit spreadsheet ID:

```javascript
// ============================================================
// SPREADSHEET ACCESS - CRITICAL FOR WEB APPS
// ============================================================
const SPREADSHEET_ID = 'your-spreadsheet-id-here';  // <-- REQUIRED

function getSpreadsheet() {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    if (ss) return ss;
  } catch (e) {}
  return SpreadsheetApp.openById(SPREADSHEET_ID);
}
```

### Implementation Pattern

Add to Code.gs:

```javascript
// ============================================================
// SPREADSHEET ACCESS - REQUIRED FOR WEB APPS
// ============================================================
const SPREADSHEET_ID = 'your-spreadsheet-id-here';

function getSpreadsheet() {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    if (ss) return ss;
  } catch (e) {}
  return SpreadsheetApp.openById(SPREADSHEET_ID);
}

// ============================================================
// EMAIL WHITELISTING - Access Control
// ============================================================
const ACCESS_CONFIG = {
  ENABLED: true,
  ALLOWED_DOMAIN: '',             // e.g., '@redhat.com' - leave empty to disable domain check
  ALLOWED_EMAILS: [               // Hardcoded fallback emails
    'admin@example.com',
  ],
  USE_WHITELIST_SHEET: true       // Recommended: manage via 'Whitelist' sheet tab
};

function checkAccess() {
  if (!ACCESS_CONFIG.ENABLED) {
    return { authorized: true, email: '', reason: 'Access control disabled' };
  }
  
  const email = Session.getActiveUser().getEmail();
  
  if (!email) {
    return { authorized: false, email: '', reason: 'Could not determine user email' };
  }
  
  // Check domain (if configured)
  if (ACCESS_CONFIG.ALLOWED_DOMAIN && email.endsWith(ACCESS_CONFIG.ALLOWED_DOMAIN)) {
    return { authorized: true, email: email, reason: 'Domain authorized' };
  }
  
  // Check hardcoded emails (fallback)
  if (ACCESS_CONFIG.ALLOWED_EMAILS.map(e => e.toLowerCase()).includes(email.toLowerCase())) {
    return { authorized: true, email: email, reason: 'Email in allowed list' };
  }
  
  // Check whitelist sheet (recommended for easy management)
  if (ACCESS_CONFIG.USE_WHITELIST_SHEET) {
    const whitelistEmails = getWhitelistFromSheet();
    if (whitelistEmails.includes(email.toLowerCase())) {
      return { authorized: true, email: email, reason: 'Email in Whitelist sheet' };
    }
  }
  
  return { authorized: false, email: email, reason: 'Email not authorized' };
}

function getWhitelistFromSheet() {
  try {
    // CRITICAL: Use getSpreadsheet(), NOT getActiveSpreadsheet()
    const ss = getSpreadsheet();
    const sheet = ss.getSheetByName('Whitelist');
    if (!sheet) return [];
    const data = sheet.getDataRange().getValues();
    // Skip header row (row 0), extract emails from column A
    return data.slice(1)
      .map(row => String(row[0] || '').trim().toLowerCase())
      .filter(e => e.includes('@'));
  } catch (e) {
    console.log('Whitelist error:', e);
    return [];
  }
}
```

### Update doGet() to Check Access

```javascript
function doGet() {
  const access = checkAccess();
  
  if (!access.authorized) {
    // Include debug info to help troubleshoot access issues
    return HtmlService.createHtmlOutput(
      `<html><body style="font-family:sans-serif;text-align:center;padding:50px;">
        <h1 style="color:#EE0000;">Access Denied</h1>
        <p><strong>Reason:</strong> ${access.reason}</p>
        <p><strong>Your email:</strong> ${access.email || 'Could not detect'}</p>
        <hr style="margin:30px 0;">
        <p style="color:#666;">Contact the dashboard administrator to request access.</p>
        <p style="color:#999;font-size:12px;">
          Debug: Domain=${ACCESS_CONFIG.ALLOWED_DOMAIN || 'none'}, 
          Sheet=${ACCESS_CONFIG.USE_WHITELIST_SHEET ? 'enabled' : 'disabled'}
        </p>
      </body></html>`
    ).setTitle('Access Denied');
  }
  
  return HtmlService.createHtmlOutputFromFile('DashboardUI')
    .setTitle('Dashboard');
}
```

### Whitelist Sheet Setup

If using `USE_WHITELIST_SHEET: true`:

1. Create a sheet tab called **"Whitelist"**
2. **Row 1 = Header** (required - code skips row 1)
3. Column A: Email addresses
4. Column B: Notes (optional)

Example:
| Email | Notes |
|-------|-------|
| user1@example.com | Added 2026-05-25 |
| user2@example.com | External partner |

### Helper: Create/Manage Whitelist Sheet

Add this function to create the Whitelist sheet with proper formatting:

```javascript
function manageWhitelist() {
  const ss = getSpreadsheet();
  let sheet = ss.getSheetByName('Whitelist');
  
  if (!sheet) {
    sheet = ss.insertSheet('Whitelist');
    sheet.getRange('A1:B1').setValues([['Email', 'Notes']]);
    sheet.getRange('A1:B1')
      .setBackground('#EE0000')
      .setFontColor('#FFFFFF')
      .setFontWeight('bold');
    sheet.setColumnWidth(1, 250);
    sheet.setColumnWidth(2, 200);
    SpreadsheetApp.getUi().alert('Whitelist sheet created. Add emails to column A (skip header row).');
  } else {
    SpreadsheetApp.getUi().alert('Whitelist sheet already exists.');
  }
}

// Add menu item
function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('Dashboard')
    .addItem('Manage Whitelist', 'manageWhitelist')
    .addToUi();
}
```

### When to Use Each Option

| Scenario | Configuration |
|----------|---------------|
| Internal only (single domain) | `ALLOWED_DOMAIN: '@company.com'` |
| Internal + few external | Domain + `ALLOWED_EMAILS` list |
| Many external users | `USE_WHITELIST_SHEET: true` |
| Testing (no restriction) | `ENABLED: false` |

---

## Advanced Access Control (PropertiesService + Google Groups)

For production dashboards with sensitive data, use this more robust approach:

- **PropertiesService storage** - No external spreadsheet to manage permissions for
- **Google Groups support** - Access auto-updates when users join/leave groups
- **Permanent admins** - Hardcoded admins can't accidentally lock themselves out
- **Admin UI** - Manage access from within the dashboard (no code editing)
- **Server-side security** - Access check runs before any content is sent to browser

### Architecture

```
User visits web app URL
        |
        v
doGet() -> Session.getActiveUser().getEmail()
        |
        v
checkUserAccess(email) checks:
  1. Hardcoded ADMIN_EMAILS array
  2. Stored EMAIL entries (PropertiesService)
  3. Stored GROUP entries (GroupsApp membership)
        |
    +-- GRANTED --> Render dashboard (Index.html)
    |               + inject isAdmin flag
    |               + show Settings button if admin
    |
    +-- DENIED --> Render AccessDenied.html
                   + show user's email
                   + show contact for requesting access
```

### Storage

- **Engine**: `PropertiesService.getScriptProperties()` (built-in key-value store)
- **Key**: `access_entries`
- **Value**: JSON array of `{ type: "EMAIL"|"GROUP", value: "user@domain.com", description: "optional" }`
- **Limits**: 9KB per property (hundreds of entries)
- **Persistence**: Survives redeployments and code pushes

### Step 1: Create AccessControl.js

```javascript
/**
 * AccessControl.js - Access Control using PropertiesService
 * 
 * Supports: individual emails, Google Groups, permanent admins
 * Storage: PropertiesService (no external sheets required)
 */

// ============================================================
// CONFIGURATION - UPDATE THESE FOR YOUR DEPLOYMENT
// ============================================================

// Permanent admins - always have access, cannot be deleted from UI
var ADMIN_EMAILS = [
  'your-email@domain.com'  // <-- REPLACE with actual admin email
];

// Contact shown on Access Denied page
var ACCESS_CONTACT = {
  name: 'Your Name',           // <-- REPLACE
  email: 'your-email@domain.com'  // <-- REPLACE
};

// PropertiesService key for stored entries
var ACCESS_PROP_KEY = 'access_entries';

// ============================================================
// ACCESS CHECK
// ============================================================

/**
 * Check whether the given email is allowed to access the dashboard.
 * Priority: hardcoded admins > stored emails > stored groups.
 */
function checkUserAccess(userEmail) {
  if (!userEmail) return false;
  userEmail = userEmail.toLowerCase().trim();
  
  // 1. Hardcoded admins
  for (var i = 0; i < ADMIN_EMAILS.length; i++) {
    if (ADMIN_EMAILS[i].toLowerCase() === userEmail) {
      console.log('Access granted (admin): ' + userEmail);
      return true;
    }
  }
  
  // 2. Stored entries (emails and groups)
  var entries = getStoredEntries();
  for (var j = 0; j < entries.length; j++) {
    var entry = entries[j];
    if (entry.type === 'EMAIL' && entry.value.toLowerCase() === userEmail) {
      console.log('Access granted (email): ' + userEmail);
      return true;
    }
    if (entry.type === 'GROUP' && isUserInGroup(userEmail, entry.value)) {
      console.log('Access granted (group ' + entry.value + '): ' + userEmail);
      return true;
    }
  }
  
  console.log('Access denied: ' + userEmail);
  return false;
}

/**
 * Check if a user is a member of a Google Group.
 */
function isUserInGroup(userEmail, groupEmail) {
  try {
    var group = GroupsApp.getGroupByEmail(groupEmail);
    return group.hasUser(userEmail);
  } catch (e) {
    console.error('Group check failed for ' + groupEmail + ': ' + e.message);
    return false;
  }
}

/**
 * Check if the current user is a hardcoded admin.
 */
function isCurrentUserAdmin() {
  var email = Session.getActiveUser().getEmail().toLowerCase().trim();
  for (var i = 0; i < ADMIN_EMAILS.length; i++) {
    if (ADMIN_EMAILS[i].toLowerCase() === email) return true;
  }
  return false;
}

// ============================================================
// PROPERTIES STORAGE
// ============================================================

function getStoredEntries() {
  try {
    var props = PropertiesService.getScriptProperties();
    var json = props.getProperty(ACCESS_PROP_KEY);
    if (!json) return [];
    return JSON.parse(json);
  } catch (e) {
    console.error('Failed to read access entries:', e);
    return [];
  }
}

function saveStoredEntries(entries) {
  var props = PropertiesService.getScriptProperties();
  props.setProperty(ACCESS_PROP_KEY, JSON.stringify(entries));
}

// ============================================================
// CLIENT-CALLABLE FUNCTIONS (Settings UI)
// ============================================================

/**
 * Get all access entries for display in the Settings modal.
 */
function getAccessEntries() {
  if (!isCurrentUserAdmin()) {
    return { status: 'error', message: 'Only admins can view access settings.' };
  }
  
  var stored = getStoredEntries();
  var entries = [];
  
  // Add hardcoded admins first (non-deletable)
  for (var i = 0; i < ADMIN_EMAILS.length; i++) {
    entries.push({
      id: 'admin_' + i,
      type: 'ADMIN',
      value: ADMIN_EMAILS[i],
      description: 'Permanent admin (hardcoded)',
      deletable: false
    });
  }
  
  // Add stored entries (deletable)
  for (var j = 0; j < stored.length; j++) {
    entries.push({
      id: 'stored_' + j,
      type: stored[j].type,
      value: stored[j].value,
      description: stored[j].description || '',
      deletable: true
    });
  }
  
  return {
    status: 'success',
    entries: entries,
    currentUser: Session.getActiveUser().getEmail().toLowerCase()
  };
}

/**
 * Add a new access entry.
 */
function addAccessEntry(type, value, description) {
  if (!isCurrentUserAdmin()) {
    return { status: 'error', message: 'Only admins can manage access.' };
  }
  
  type = (type || '').toString().toUpperCase().trim();
  value = (value || '').toString().toLowerCase().trim();
  description = (description || '').toString().trim();
  
  if (!type || !value) {
    return { status: 'error', message: 'Type and email/group are required.' };
  }
  if (['EMAIL', 'GROUP'].indexOf(type) === -1) {
    return { status: 'error', message: 'Type must be EMAIL or GROUP.' };
  }
  if (value.indexOf('@') === -1) {
    return { status: 'error', message: 'Invalid email format.' };
  }
  
  // Check for duplicates
  var entries = getStoredEntries();
  for (var j = 0; j < entries.length; j++) {
    if (entries[j].value.toLowerCase() === value) {
      return { status: 'error', message: 'This email/group is already in the access list.' };
    }
  }
  
  entries.push({ type: type, value: value, description: description });
  saveStoredEntries(entries);
  return { status: 'success', message: 'Entry added successfully.' };
}

/**
 * Delete a stored access entry by index.
 */
function deleteAccessEntry(index) {
  if (!isCurrentUserAdmin()) {
    return { status: 'error', message: 'Only admins can manage access.' };
  }
  
  var entries = getStoredEntries();
  if (index < 0 || index >= entries.length) {
    return { status: 'error', message: 'Invalid entry index.' };
  }
  
  entries.splice(index, 1);
  saveStoredEntries(entries);
  return { status: 'success', message: 'Entry deleted successfully.' };
}

/**
 * Returns contact info for the Access Denied page.
 */
function getAccessDeniedInfo() {
  return {
    contactName: ACCESS_CONTACT.name,
    contactEmail: ACCESS_CONTACT.email,
    userEmail: Session.getActiveUser().getEmail() || 'Unknown'
  };
}
```

### Step 2: Modify doGet() Entry Point

```javascript
function doGet(e) {
  var userEmail = Session.getActiveUser().getEmail();
  
  // Access gate - check BEFORE rendering anything
  if (!checkUserAccess(userEmail)) {
    return HtmlService.createTemplateFromFile('AccessDenied')
      .evaluate()
      .setTitle('Access Restricted')
      .addMetaTag('viewport', 'width=device-width, initial-scale=1');
  }
  
  // User is authorized - render dashboard
  var template = HtmlService.createTemplateFromFile('Index');
  template.userEmail = userEmail;
  template.isAdmin = isCurrentUserAdmin();
  
  return template
    .evaluate()
    .setTitle('Your Dashboard Name')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}
```

### Step 3: Create AccessDenied.html

A standalone page shown to unauthorized users (no dashboard code exposed):

```html
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: #151515;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      color: #e0e0e0;
    }
    .container {
      text-align: center;
      padding: 48px 40px;
      max-width: 520px;
      background: #1f1f1f;
      border: 1px solid #383838;
      border-radius: 12px;
    }
    .lock-icon {
      width: 64px; height: 64px;
      margin: 0 auto 24px;
      background: rgba(238, 0, 0, 0.1);
      border: 1px solid rgba(238, 0, 0, 0.25);
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 28px;
    }
    h1 { font-size: 24px; font-weight: 600; color: #ffffff; margin-bottom: 12px; }
    .subtitle { font-size: 15px; color: #707070; line-height: 1.6; margin-bottom: 28px; }
    .user-email-box {
      background: rgba(238, 0, 0, 0.08);
      border: 1px solid rgba(238, 0, 0, 0.2);
      border-radius: 8px;
      padding: 10px 20px;
      display: inline-block;
      font-family: monospace;
      font-size: 13px;
      color: #ee0000;
      margin-bottom: 28px;
    }
    .contact-card {
      display: inline-flex;
      align-items: center;
      gap: 12px;
      background: #292929;
      border: 1px solid #383838;
      border-radius: 8px;
      padding: 12px 20px;
    }
    .contact-avatar {
      width: 36px; height: 36px;
      background: #ee0000;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-weight: 600;
      font-size: 14px;
      color: #ffffff;
    }
    .contact-info { text-align: left; }
    .contact-name { font-size: 14px; font-weight: 600; color: #ffffff; }
    .contact-email a { font-size: 13px; color: #707070; text-decoration: none; }
    .contact-email a:hover { color: #ee0000; }
  </style>
</head>
<body>
  <div class="container">
    <div class="lock-icon">&#x1F512;</div>
    <h1>Access Restricted</h1>
    <p class="subtitle">
      You don't have permission to view this dashboard.<br>
      Your current account is:
    </p>
    <div class="user-email-box" id="user-email">Loading...</div>
    <p style="font-size: 13px; color: #707070; margin-bottom: 8px;">
      To request access, contact:
    </p>
    <div class="contact-card">
      <div class="contact-avatar" id="contact-initials">--</div>
      <div class="contact-info">
        <div class="contact-name" id="contact-name">Loading...</div>
        <div class="contact-email">
          <a href="#" id="contact-email-link">Loading...</a>
        </div>
      </div>
    </div>
  </div>
  <script>
    google.script.run
      .withSuccessHandler(function(info) {
        document.getElementById('user-email').textContent = info.userEmail || 'Unknown';
        var name = info.contactName || 'Admin';
        var email = info.contactEmail || '';
        var parts = name.split(' ');
        var initials = parts.length >= 2 
          ? parts[0].charAt(0) + parts[1].charAt(0) 
          : name.substring(0, 2);
        document.getElementById('contact-initials').textContent = initials.toUpperCase();
        document.getElementById('contact-name').textContent = name;
        document.getElementById('contact-email-link').textContent = email;
        document.getElementById('contact-email-link').href = 'mailto:' + email;
      })
      .getAccessDeniedInfo();
  </script>
</body>
</html>
```

### Comparison: Simple vs Advanced Access Control

| Feature | Simple (Whitelist Sheet) | Advanced (PropertiesService) |
|---------|--------------------------|------------------------------|
| Storage | Google Sheet tab | Built-in key-value store |
| Google Groups | No | Yes (auto-sync) |
| Admin UI | No (edit sheet) | Yes (in-dashboard) |
| Permanent admins | No | Yes (can't lock out) |
| External dependency | Requires sheet | Self-contained |
| Best for | Simple internal tools | Production dashboards |

### Security Notes

- Access check runs **server-side** in `doGet()` - cannot be bypassed by browser tools
- All management functions verify `isCurrentUserAdmin()` before executing
- PropertiesService data is encrypted at rest by Google
- Group membership is verified in real-time via GroupsApp API

---

## Specification Compliance

### CRITICAL: Never Remove Specified Features

When implementing a dashboard from user specifications:

1. **List all specified filters** → Verify each exists in source data
2. **If data missing** → Add to data pipeline, DON'T remove filter
3. **Trace back**: Source CSV → Build script → Dashboard sheets → UI → Backend handler
4. **Verify end-to-end** before claiming complete

**Common mistake:** Removing a UI filter because the data isn't in the dashboard sheets.
**Correct approach:** Add the column to `build_dashboard_sheets.py`, regenerate, re-upload.

See rule: `follow-specifications-exactly.mdc`

## Performance Optimization

### For Large Datasets (5000+ rows)

1. **Pre-aggregate data** into summary sheets
2. **Load all data once** to client, filter locally
3. **Use pagination** for detail tables
4. **Cache aggressively** with `CacheService`
5. **Use compact data encoding** (indices instead of strings)
6. **Defer text loading** until user expands a row

### Basic Caching Pattern

```javascript
function getCachedData(key, fetchFn, ttl = 21600) {
  const cache = CacheService.getScriptCache();
  let data = cache.get(key);
  
  if (!data) {
    data = JSON.stringify(fetchFn());
    try {
      cache.put(key, data, ttl);
    } catch (e) {
      // Data too large, skip cache
    }
  }
  
  return JSON.parse(data);
}
```

### Chunked Cache Pattern (For Large Data >100KB)

CacheService has a 100KB limit per key. For large datasets, split into chunks:

```javascript
const CACHE_VERSION = 'v1';  // Bump after data changes to invalidate

function _cacheGet(key) {
  const cache = CacheService.getScriptCache();
  const nStr = cache.get(key + '_n');
  if (!nStr) return null;
  
  const n = parseInt(nStr);
  let json = '';
  for (let i = 0; i < n; i++) {
    const chunk = cache.get(key + '_' + i);
    if (!chunk) return null;  // Partial cache = invalid
    json += chunk;
  }
  
  try { return JSON.parse(json); } 
  catch(e) { return null; }
}

function _cachePut(key, data, ttl) {
  const cache = CacheService.getScriptCache();
  try {
    const json = JSON.stringify(data);
    const CHUNK = 90000;  // Stay under 100KB limit
    const chunks = [];
    for (let i = 0; i < json.length; i += CHUNK) {
      chunks.push(json.slice(i, i + CHUNK));
    }
    cache.put(key + '_n', String(chunks.length), ttl);
    chunks.forEach((c, i) => cache.put(key + '_' + i, c, ttl));
  } catch(e) {
    console.log('Cache put failed:', e);
  }
}

// Usage in main data loader
function getInitialData() {
  const cacheKey = 'dashboard_' + CACHE_VERSION;
  const cached = _cacheGet(cacheKey);
  if (cached) return cached;
  
  const data = _buildAllData();
  _cachePut(cacheKey, data, 480);  // 8 minutes
  return data;
}
```

### Cache Warming (Prevent Slow First Load)

Add a function to pre-warm cache before users access the dashboard:

```javascript
function warmCache() {
  const key = 'dashboard_' + CACHE_VERSION;
  const data = _buildAllData();
  _cachePut(key, data, 480);
  Logger.log('Cache warmed successfully');
}
```

Run `warmCache()` manually after data uploads or set up a time-driven trigger.

### Compact Data Encoding

Reduce JSON payload size by using integer indices instead of repeated strings:

```javascript
function _buildAllData() {
  const rawData = loadRawData();
  
  // Build dimension arrays (unique values)
  const geos = [...new Set(rawData.map(r => r.Geo))].sort();
  const segments = [...new Set(rawData.map(r => r.Segment))].sort();
  
  // Create lookup maps
  const geoIdx = {};
  geos.forEach((v, i) => geoIdx[v] = i);
  const segIdx = {};
  segments.forEach((v, i) => segIdx[v] = i);
  
  // Encode rows as compact arrays
  const accounts = rawData.map(r => [
    r.Name,                    // 0: name (string, keep as-is)
    r.URL,                     // 1: url
    geoIdx[r.Geo] ?? -1,       // 2: geo index (integer!)
    segIdx[r.Segment] ?? -1,   // 3: segment index
    r.Value || 0,              // 4: numeric value
    r.IsActive ? 1 : 0         // 5: boolean as 0/1
  ]);
  
  return {
    dims: { geos, segments },  // Dimension arrays for lookup
    accounts                    // Compact integer-encoded data
  };
}
```

Client-side decoding:

```javascript
// In frontend JavaScript
function decodeRow(row, dims) {
  return {
    name: row[0],
    url: row[1],
    geo: dims.geos[row[2]] || 'Unknown',
    segment: dims.segments[row[3]] || 'Unknown',
    value: row[4],
    isActive: row[5] === 1
  };
}
```

### On-Demand Text Loading

For rows with large text fields, load text only when user expands:

```javascript
// Server-side: lightweight initial load
function getInitialData() {
  // Returns compact data WITHOUT large text fields
  return { accounts: [...], dims: {...} };
}

// Server-side: load text on demand
function getAccountText(recordId) {
  const sheet = getSpreadsheet().getSheetByName('Accounts');
  const data = sheet.getDataRange().getValues();
  
  for (let i = 1; i < data.length; i++) {
    if (String(data[i][0]).trim() === String(recordId).trim()) {
      return {
        description: String(data[i][10] || ''),
        notes: String(data[i][11] || ''),
        analysis: String(data[i][12] || '')
      };
    }
  }
  return { description: '', notes: '', analysis: '' };
}
```

```javascript
// Client-side: load on expand
function expandRow(recordId, containerEl) {
  containerEl.innerHTML = 'Loading...';
  google.script.run
    .withSuccessHandler(text => {
      containerEl.innerHTML = `
        <div class="text-section">
          <div class="lbl">Description</div>
          <p>${text.description || 'N/A'}</p>
        </div>
      `;
    })
    .getAccountText(recordId);
}
```

## Common Patterns

### Column Index Constants (Maintainability)

Define column indices as constants for readable, maintainable code:

```javascript
// Define column indices (0-based) for each sheet
const ACCOUNTS = {
  ID: 0, NAME: 1, URL: 2, GEO: 3, REGION: 4,
  SEGMENT: 5, VALUE: 6, STATUS: 7, NOTES: 8
};

const INITIATIVES = {
  ACCOUNT_ID: 0, NAME: 1, CATEGORY: 2, VALUE: 3, TIMING: 4
};

// Usage - much clearer than magic numbers
const name = row[ACCOUNTS.NAME];
const value = parseFloat(row[ACCOUNTS.VALUE]) || 0;
```

### Multi-Sheet Data Loading

For dashboards that join data from multiple sheets:

```javascript
function _buildAllData() {
  const ss = getSpreadsheet();
  
  // Load primary sheet
  const accountsRaw = ss.getSheetByName('Accounts').getDataRange().getValues();
  
  // Build lookup maps from primary data
  const accountById = {};
  for (let i = 1; i < accountsRaw.length; i++) {
    const id = String(accountsRaw[i][ACCOUNTS.ID]);
    accountById[id] = {
      name: accountsRaw[i][ACCOUNTS.NAME],
      geo: accountsRaw[i][ACCOUNTS.GEO],
      value: parseFloat(accountsRaw[i][ACCOUNTS.VALUE]) || 0
    };
  }
  
  // Load related sheet and join
  const initRaw = ss.getSheetByName('Initiatives').getDataRange().getValues();
  const initiatives = [];
  for (let i = 1; i < initRaw.length; i++) {
    const accountId = String(initRaw[i][INITIATIVES.ACCOUNT_ID]);
    const account = accountById[accountId] || {};
    initiatives.push({
      accountName: account.name || 'Unknown',
      accountGeo: account.geo || '',
      initName: initRaw[i][INITIATIVES.NAME],
      category: initRaw[i][INITIATIVES.CATEGORY],
      value: parseFloat(initRaw[i][INITIATIVES.VALUE]) || 0
    });
  }
  
  return { accounts: Object.values(accountById), initiatives };
}
```

### Multi-Select Filter

```javascript
function getFilterOptions(column, currentSelections) {
  const data = loadData();
  
  // Filter by upstream selections
  let filtered = data;
  // ... apply other filters ...
  
  // Count values
  const counts = {};
  filtered.forEach(row => {
    const val = row[column];
    counts[val] = (counts[val] || 0) + 1;
  });
  
  return Object.entries(counts)
    .map(([value, count]) => ({ value, count }))
    .sort((a, b) => a.value.localeCompare(b.value));
}
```

### Cascading Filter Configuration

Define filters as a configuration array for maintainability:

```javascript
// Server-side configuration — use string IDs that match HTML element suffixes
const FILTERS = [
  { id: 'geo',     name: 'Geo',     column: '_GEO' },
  { id: 'segment', name: 'Segment', column: '_SEGMENT' },
  { id: 'region',  name: 'Region',  column: '_REGION' },
  { id: 'status',  name: 'Status',  column: '_STATUS' },
];

function getDashboardConfig() {
  return {
    filters: FILTERS,
    displayColumns: ['Name', 'Geo', 'Segment', 'Value', 'Status'],
    rowsPerPage: 100
  };
}
```

### Bidirectional Filter Cascading

When selecting a value in any filter, update options in ALL other filters:

```javascript
function getFilterOptions(filterIndex, currentSelections) {
  const { data } = loadData();
  const filter = FILTERS[filterIndex];
  
  // Filter data by ALL OTHER selections (not this filter)
  let filteredData = data;
  FILTERS.forEach((f, i) => {
    if (i === filterIndex) return;  // Skip current filter
    const selected = currentSelections[f.column] || [];
    if (selected.length > 0 && !selected.includes('All')) {
      filteredData = filteredData.filter(row => {
        const rowValues = String(row[f.column] || '').split(';');
        return selected.some(sv => rowValues.includes(sv));
      });
    }
  });
  
  // Count values for THIS filter from filtered data
  const counts = {};
  filteredData.forEach(row => {
    const values = String(row[filter.column] || '').split(';');
    values.forEach(v => {
      if (v.trim()) counts[v.trim()] = (counts[v.trim()] || 0) + 1;
    });
  });
  
  return Object.entries(counts)
    .map(([value, count]) => ({ value, count }))
    .sort((a, b) => a.value.localeCompare(b.value));
}
```

### Client-Side Filtering (Responsive UX)

Load all data once, filter on client for instant responsiveness:

```javascript
// Server-side: load ALL data
function loadAllData() {
  const { data } = loadData();
  const filterColumns = FILTERS.map(f => f.column);
  const displayColumns = ['Name', 'Geo', 'Segment', 'Value'];
  const allColumns = [...new Set([...displayColumns, ...filterColumns])];
  
  return data.map(row => {
    const result = {};
    allColumns.forEach(col => result[col] = row[col]);
    return result;
  });
}
```

```javascript
// Client-side filtering
let allData = [];  // Loaded once

function initDashboard() {
  google.script.run
    .withSuccessHandler(data => {
      allData = data;
      renderFilters();
      applyFiltersAndRender();
    })
    .loadAllData();
}

function applyFiltersAndRender() {
  let filtered = allData;
  
  // Apply each filter
  FILTERS.forEach(filter => {
    const selected = getSelectedValues(filter.column);
    if (selected.length > 0 && !selected.includes('All')) {
      filtered = filtered.filter(row => {
        const rowValues = String(row[filter.column] || '').split(';');
        return selected.some(sv => rowValues.includes(sv));
      });
    }
  });
  
  updateFilterCounts(filtered);  // Update counts in dropdowns
  renderTable(filtered);
  renderKPIs(filtered);
}
```

### Typeahead Search

Add name search with suggestions:

```javascript
// Server-side
function searchNames(query, currentSelections) {
  if (!query || query.length < 2) return [];
  
  let data = loadData().data;
  const queryLower = query.toLowerCase();
  
  // Apply current filter selections
  FILTERS.forEach(filter => {
    const selected = currentSelections[filter.column] || [];
    if (selected.length > 0 && !selected.includes('All')) {
      data = data.filter(row => {
        const rowValues = String(row[filter.column] || '').split(';');
        return selected.some(sv => rowValues.includes(sv));
      });
    }
  });
  
  // Search names
  return data
    .filter(row => String(row['Name'] || '').toLowerCase().includes(queryLower))
    .map(row => row['Name'])
    .slice(0, 10);  // Limit suggestions
}
```

```html
<!-- Client-side typeahead -->
<input type="text" id="name-search" placeholder="Search names..." 
       oninput="debounce(handleNameSearch, 300)(this.value)">
<div id="suggestions"></div>

<script>
function handleNameSearch(query) {
  if (query.length < 2) {
    document.getElementById('suggestions').innerHTML = '';
    return;
  }
  
  google.script.run
    .withSuccessHandler(names => {
      const html = names.map(n => 
        `<div class="suggestion" onclick="selectName('${n}')">${n}</div>`
      ).join('');
      document.getElementById('suggestions').innerHTML = html;
    })
    .searchNames(query, getCurrentSelections());
}

function debounce(fn, delay) {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => fn(...args), delay);
  };
}
</script>
```

### Pagination

```javascript
function applyFilters(selections, page, sortBy) {
  let data = loadData();
  
  // Apply filters
  // ... filtering logic ...
  
  // Sort
  data.sort((a, b) => /* sort logic */);
  
  // Paginate
  const total = data.length;
  const pageSize = CONFIG.ROWS_PER_PAGE;
  const pageData = data.slice((page - 1) * pageSize, page * pageSize);
  
  return {
    data: pageData,
    totalCount: total,
    pageCount: Math.ceil(total / pageSize),
    currentPage: page
  };
}
```

## CRITICAL: Type Coercion from Google Sheets

**Google Sheets returns ALL values as strings**, even numbers.

### The Bug

```javascript
// This FAILS because val is a string like "29597684.41"
function formatValue(val) {
  return '$' + val.toFixed(0);  // TypeError: toFixed is not a function
}
```

### The Fix

**ALWAYS parseFloat() or parseInt() before numeric operations:**

```javascript
function formatValue(val, type) {
  if (type === 'currency') {
    const num = parseFloat(val);  // Convert string to number FIRST
    return isNaN(num) ? '$0' : '$' + num.toFixed(0).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  }
  if (type === 'number') {
    const num = parseFloat(val);
    return isNaN(num) ? '0' : num.toFixed(2);
  }
  return val;
}
```

### Where This Applies

- `renderTable()` functions formatting numeric columns
- Chart data preparation (heatmaps, bars)
- KPI calculations in frontend
- Any sorting by numeric value

---

## Robust Filter Logic (Handle Missing Columns)

When filtering pre-aggregated data, **columns may not exist in all sheets**.

### The Bug

```javascript
// This silently returns 0 rows if 'Region' doesn't exist in data
function applyFilters(data, filters) {
  return data.filter(row => {
    if (filters.region && row['Region'] !== filters.region) return false;  // FAILS
    return true;
  });
}
```

### The Fix

**Check column existence before filtering:**

```javascript
function applyFilters(data, filters) {
  return data.filter(row => {
    // Only apply filter if column EXISTS in this data
    if (filters.region && 'Region' in row && row['Region'] !== filters.region) {
      return false;
    }
    if (filters.segment && 'Segment' in row && row['Segment'] !== filters.segment) {
      return false;
    }
    // ... more filters ...
    return true;
  });
}
```

### Why This Happens

- `accounts_summary` might have Region, Subregion
- `category_summary` might only have Geo, Segment (aggregated without region detail)
- Same filter handler used for multiple sheets → must be robust

---

## Debugging Frontend Issues

### ALWAYS Check Browser Console FIRST

For ANY frontend issue (empty tables, broken charts, UI not rendering):

1. Open browser Developer Tools (F12 or Cmd+Opt+I)
2. Go to Console tab
3. Reproduce the issue
4. Look for red error messages

**Console errors provide exact line numbers and error types**, e.g.:
```
TypeError: val.toFixed is not a function
    at formatValue (DashboardUI.html:282:18)
    at renderAccountsTable (DashboardUI.html:573:37)
```

This immediately identifies:
- Which function failed (`formatValue`)
- Which line (282)
- The call chain (called from `renderAccountsTable`)

### Don't Waste Time On

- Server-side Logger.log (won't show frontend errors)
- Code inspection without error context
- Guessing at the problem

---

## Cache Management

### Clearing Cache After Data Changes

When you change data structure (add columns, modify aggregation), the cache serves stale data.

```javascript
function clearCache() {
  const cache = CacheService.getScriptCache();
  // Clear ALL known cache keys
  cache.removeAll([
    'accounts_data',
    'category_data', 
    'tdp_data',
    'filter_options',
    'kpi_global',
    'initiatives_detail'  // Don't forget detail data!
  ]);
  Logger.log('Cache cleared');
}
```

**Run `clearCache()` from Apps Script editor after ANY data upload.**

### Cache Key Naming Conventions

Use consistent prefixes for cache keys:

| Prefix | Purpose | Example |
|--------|---------|---------|
| `data_` | Raw data from sheets | `data_accounts`, `data_initiatives` |
| `agg_` | Aggregated/summary data | `agg_category`, `agg_geo_segment` |
| `filter_` | Filter options | `filter_options`, `filter_geo` |
| `kpi_` | KPI calculations | `kpi_global`, `kpi_filtered` |

**Benefits:**
- Easy to identify what's cached
- Clear which keys to invalidate after data changes
- Grep-able in code reviews

---

## Pre-Aggregated Sheets: Include ALL Filter Columns

When using pre-aggregated summary sheets for performance, **every filter must have its column in the summary sheet**.

### The Problem

If you have a filter for "Category" but `accounts_summary` doesn't have a Category column, the filter silently does nothing because the filter logic checks `if ('Category' in row)` first.

### The Solution

**List ALL filters → Include ALL columns:**

```
Dashboard Filters          →  accounts_summary Columns Needed
─────────────────────────────────────────────────────────────
Geo, Region, Subregion     →  Geo, Region, Subregion
Industry, IBM Segment      →  Industry, IBM Customer Segment
Global Account, PAI        →  Global Account, Platform Acceleration Initiative
Category, Subcategory      →  Categories, Subcategories (pipe-separated)
Completeness, Confidence   →  Completeness Level, Confidence
Maturity (0-5)             →  maturity_rhel, maturity_containers, etc.
Hyperscaler, Competitor    →  Cloud Providers, Detected Entities (pipe-separated)
```

### Multi-Value Columns

When an account has multiple values (e.g., initiatives in 3 different categories):

```
Categories: "Enterprise Automation|Virtualization|Container Management Platform"
```

Filter logic must handle pipe-separated values:

```javascript
if ('Categories' in row) {
  const cats = (row['Categories'] || '').split('|').map(c => c.trim());
  const hasMatch = filters.category.some(fc => cats.includes(fc));
  if (!hasMatch) return false;
}
```

---

## Pre-Aggregated Sheets: Include ALL Filter Columns

When using pre-aggregated summary sheets for performance, **every filter must have its column in the summary sheet**.

### The Problem

If you have a filter for "Category" but `accounts_summary` doesn't have a Category column, the filter silently does nothing because the filter logic checks `if ('Category' in row)` first.

### The Solution

**List ALL filters → Include ALL columns:**

```
Dashboard Filters          →  accounts_summary Columns Needed
─────────────────────────────────────────────────────────────
Geo, Region, Subregion     →  Geo, Region, Subregion
Industry, IBM Segment      →  Industry, IBM Customer Segment
Global Account, PAI        →  Global Account, Platform Acceleration Initiative
Category, Subcategory      →  Categories, Subcategories (pipe-separated)
Completeness, Confidence   →  Completeness Level, Confidence
Maturity (0-5)             →  maturity_rhel, maturity_containers, etc.
Hyperscaler, Competitor    →  Cloud Providers, Detected Entities (pipe-separated)
```

### Multi-Value Columns

When an account has multiple values (e.g., initiatives in 3 different categories):

```
Categories: "Enterprise Automation|Virtualization|Container Management Platform"
```

Filter logic must handle pipe-separated values:

```javascript
if ('Categories' in row) {
  const cats = (row['Categories'] || '').split('|').map(c => c.trim());
  const hasMatch = filters.category.some(fc => cats.includes(fc));
  if (!hasMatch) return false;
}
```

---

## Apps Script Quotas and Limits

### Execution Limits

| Limit | Consumer (gmail.com) | Google Workspace |
|-------|---------------------|------------------|
| Script runtime | 6 min / execution | 6 min / execution |
| Custom function runtime | 30 sec / execution | 30 sec / execution |
| Simultaneous executions per user | 30 | 30 |
| Simultaneous executions per script | 1,000 | 1,000 |
| Triggers per user per script | 20 | 20 |
| Trigger runtime | 90 min / day | 6 hr / day |
| Properties value size | 9 KB / val | 9 KB / val |
| Properties total storage | 500 KB | 500 KB |

### Web App Concurrency

- **Comfortable concurrency**: ~20-30 concurrent users
- Each `google.script.run` call serializes through the execution queue
- Past ~30 concurrent users, UI responsiveness suffers
- **Mitigation**: Cache heavily, load data client-side, filter locally

### Reserved URL Parameters

**Never use these parameter names** in query strings or POST bodies:
- `c`
- `sid`

Using these causes 405 errors.

---

## Long-Running Task Patterns

### The 6-Minute Problem

Scripts exceeding 6 minutes are terminated with "Exceeded maximum execution time".

### Chunked Processing Pattern

Split work across multiple executions using PropertiesService:

```javascript
function processLargeDataset() {
  const props = PropertiesService.getScriptProperties();
  
  // Get current progress
  const startIndex = parseInt(props.getProperty('processIndex') || '0');
  const data = getAllData();  // Your data source
  
  const BATCH_SIZE = 500;
  const MAX_RUNTIME_MS = 5 * 60 * 1000;  // 5 minutes (leave 1 min buffer)
  const startTime = Date.now();
  
  for (let i = startIndex; i < data.length; i++) {
    // Check if we need to stop
    if (Date.now() - startTime > MAX_RUNTIME_MS) {
      // Save progress and schedule continuation
      props.setProperty('processIndex', String(i));
      scheduleContinuation();
      return { status: 'in_progress', processed: i, total: data.length };
    }
    
    // Process item
    processItem(data[i]);
  }
  
  // Complete - clean up
  props.deleteProperty('processIndex');
  deleteScheduledTrigger();
  return { status: 'complete', processed: data.length };
}

function scheduleContinuation() {
  // Create trigger to continue in 1 minute
  ScriptApp.newTrigger('processLargeDataset')
    .timeBased()
    .after(60 * 1000)  // 1 minute
    .create();
}

function deleteScheduledTrigger() {
  ScriptApp.getProjectTriggers()
    .filter(t => t.getHandlerFunction() === 'processLargeDataset')
    .forEach(t => ScriptApp.deleteTrigger(t));
}
```

---

## Installable Triggers

### Create Triggers Programmatically

```javascript
// Time-based: every 6 hours
ScriptApp.newTrigger('refreshData')
  .timeBased()
  .everyHours(6)
  .create();

// Time-based: daily at 9 AM
ScriptApp.newTrigger('dailyReport')
  .timeBased()
  .atHour(9)
  .everyDays(1)
  .create();

// Time-based: weekly on Monday at 9 AM
ScriptApp.newTrigger('weeklySync')
  .timeBased()
  .onWeekDay(ScriptApp.WeekDay.MONDAY)
  .atHour(9)
  .create();

// Time-based: specific date
ScriptApp.newTrigger('oneTimeTask')
  .timeBased()
  .at(new Date('2026-06-01T10:00:00'))
  .create();

// Spreadsheet event: on edit
ScriptApp.newTrigger('onEditHandler')
  .forSpreadsheet(SpreadsheetApp.getActiveSpreadsheet())
  .onEdit()
  .create();
```

### Delete Triggers

```javascript
// Delete specific trigger by ID
function deleteTriggerById(triggerId) {
  ScriptApp.getProjectTriggers().forEach(trigger => {
    if (trigger.getUniqueId() === triggerId) {
      ScriptApp.deleteTrigger(trigger);
    }
  });
}

// Delete all triggers for a function
function deleteTriggersByFunction(functionName) {
  ScriptApp.getProjectTriggers()
    .filter(t => t.getHandlerFunction() === functionName)
    .forEach(t => ScriptApp.deleteTrigger(t));
}

// Delete ALL project triggers (use carefully)
function deleteAllTriggers() {
  ScriptApp.getProjectTriggers().forEach(t => ScriptApp.deleteTrigger(t));
}
```

### Store Trigger ID for Later Deletion

```javascript
function createTrackedTrigger() {
  const trigger = ScriptApp.newTrigger('myFunction')
    .timeBased()
    .everyHours(1)
    .create();
  
  // Save ID for later management
  PropertiesService.getScriptProperties()
    .setProperty('myTrigger_id', trigger.getUniqueId());
}

function deleteTrackedTrigger() {
  const triggerId = PropertiesService.getScriptProperties()
    .getProperty('myTrigger_id');
  
  if (triggerId) {
    deleteTriggerById(triggerId);
    PropertiesService.getScriptProperties().deleteProperty('myTrigger_id');
  }
}
```

---

## PropertiesService vs CacheService

### When to Use Each

| Feature | CacheService | PropertiesService |
|---------|--------------|-------------------|
| **Max per key** | 100 KB | 9 KB |
| **Total storage** | ~1,000 items | 500 KB |
| **Persistence** | Temporary (max 6 hr) | Permanent |
| **May be evicted** | Yes (any time) | No |
| **Speed** | Faster | Slightly slower |
| **Use case** | Temporary caching | Config, state, progress |

### CacheService: Temporary Data

```javascript
const cache = CacheService.getScriptCache();

// Write (expires in 10 minutes by default)
cache.put('key', JSON.stringify(data));

// Write with custom TTL (max 6 hours = 21600 seconds)
cache.put('key', JSON.stringify(data), 21600);

// Read (returns null if expired/evicted)
const cached = cache.get('key');
if (cached) {
  const data = JSON.parse(cached);
}

// Batch operations (more efficient)
cache.putAll({
  'key1': JSON.stringify(data1),
  'key2': JSON.stringify(data2)
}, 3600);

const results = cache.getAll(['key1', 'key2']);
```

### PropertiesService: Persistent State

```javascript
const props = PropertiesService.getScriptProperties();

// Write
props.setProperty('lastSync', new Date().toISOString());
props.setProperty('config', JSON.stringify({ enabled: true }));

// Read
const lastSync = props.getProperty('lastSync');
const config = JSON.parse(props.getProperty('config') || '{}');

// Delete
props.deleteProperty('lastSync');

// Get all
const allProps = props.getProperties();
```

### Combined Pattern: Cache with Persistent Fallback

```javascript
function getCachedOrPersistent(key, fetchFn) {
  const cache = CacheService.getScriptCache();
  
  // Try cache first (fast path)
  let val = cache.get(key);
  if (val) return JSON.parse(val);
  
  // Try persistent storage (slower but reliable)
  const props = PropertiesService.getScriptProperties();
  val = props.getProperty(key);
  if (val) {
    cache.put(key, val, 600);  // Re-cache for 10 min
    return JSON.parse(val);
  }
  
  // Fetch fresh data
  const data = fetchFn();
  const json = JSON.stringify(data);
  
  cache.put(key, json, 600);
  props.setProperty(key, json);
  
  return data;
}
```

### Prevent Race Conditions with LockService

```javascript
function updateSharedState(key, updateFn) {
  const lock = LockService.getScriptLock();
  
  try {
    lock.waitLock(10000);  // Wait up to 10 seconds
    
    const props = PropertiesService.getScriptProperties();
    const current = JSON.parse(props.getProperty(key) || '{}');
    const updated = updateFn(current);
    props.setProperty(key, JSON.stringify(updated));
    
    return updated;
  } finally {
    lock.releaseLock();
  }
}
```

---

## Web App as REST API

### JSON API Endpoint Pattern

```javascript
function doGet(e) {
  const action = e.parameter.action;
  
  try {
    let result;
    
    switch (action) {
      case 'getData':
        result = { status: 'success', data: getData() };
        break;
      case 'getFilters':
        result = { status: 'success', filters: getFilterOptions() };
        break;
      default:
        result = { status: 'error', message: 'Unknown action' };
    }
    
    return ContentService.createTextOutput(JSON.stringify(result))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: error.message
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

function doPost(e) {
  try {
    const payload = JSON.parse(e.postData.contents);
    const action = payload.action;
    
    let result;
    
    switch (action) {
      case 'saveData':
        result = saveData(payload.data);
        break;
      default:
        result = { status: 'error', message: 'Unknown action' };
    }
    
    return ContentService.createTextOutput(JSON.stringify(result))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: error.message
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

### Event Parameter Structure

```javascript
function doGet(e) {
  // e.queryString: "action=getData&id=123"
  // e.parameter: { action: "getData", id: "123" }  // First value only
  // e.parameters: { action: ["getData"], id: ["123"] }  // All values as arrays
  // e.pathInfo: URL path after /exec (e.g., "users/123")
}

function doPost(e) {
  // e.postData.contents: Raw body string
  // e.postData.type: MIME type (e.g., "application/json")
  // e.contentLength: Body length in bytes
}
```

### IMPORTANT: Redirect Behavior

Apps Script web apps return **302 redirects**. Clients must follow redirects:

```javascript
// JavaScript fetch
fetch(webAppUrl + '?action=getData', {
  method: 'GET',
  redirect: 'follow'  // Default, but be explicit
});

// Python requests
import requests
response = requests.get(web_app_url, params={'action': 'getData'}, allow_redirects=True)
```

---

---

## ⚠️ Filter ID Alignment — Backend and Frontend Must Match

The `FILTERS` array in `Code.gs` defines which columns to filter on. The `id` field in each filter entry **must exactly match** the suffix of `id="filter-<id>"` in the HTML. A mismatch causes all dynamic filter selects to silently fail — no error, just empty dropdowns.

**Root cause pattern:** backend used numeric IDs (`id: 1, 2, 3…`) while HTML expected string IDs (`id="filter-geo"`, etc.) → `getElementById('filter-1')` returns null silently.

### ✅ Correct pattern

**Code.gs:**
```javascript
const FILTERS = [
  { id: 'geo',      label: 'Geo',         column: '_GEO'              },
  { id: 'org',      label: 'Org',         column: '_ORG'              },
  { id: 'role',     label: 'Sales Role',  column: '_SALES_ROLE_SUMM'  },
  { id: 'top_skew', label: 'Top SG SKEW', column: '_TOP_SG_SKEW_BUCKET' },
];
```

**HTML:**
```html
<select multiple class="filter-select" id="filter-geo"      onchange="onFilterChange()"></select>
<select multiple class="filter-select" id="filter-org"      onchange="onFilterChange()"></select>
<select multiple class="filter-select" id="filter-role"     onchange="onFilterChange()"></select>
<select multiple class="filter-select" id="filter-top_skew" onchange="onFilterChange()"></select>
```

**JS population function (must be null-safe):**
```javascript
function populateFilterOptions(filters, filterValues) {
  filters.forEach(f => {
    const el = document.getElementById('filter-' + f.id);
    if (!el) return;  // CRITICAL: skip gracefully if element not found
    // ... populate options
  });
}
```

### Recommendation: hardcode filter elements in HTML

For complex layouts (specific order, mixed search inputs and selects), define all filter elements explicitly in HTML rather than generating them from a JS array. The backend `FILTERS` array then only provides column metadata — not element creation.

---

## ⚠️ Expand/Collapse in Dynamically Rendered Tables

**Always use event delegation + `<span>` — never attach listeners to dynamic elements directly.**

This pattern fails silently in three ways:
1. Inline `onclick` on `<button>` inside dynamically replaced HTML → lost after `innerHTML` replacement
2. `addEventListener` attached after `innerHTML` set → worked once, then lost on next render
3. `<button>` element inside a table cell → browsers sometimes intercept click bubbling

All three produce "click does nothing" with no console error.

### ❌ What breaks

```html
<!-- WRONG: button may intercept clicks; onclick lost on re-render -->
<button onclick="toggleOther('${id}')">Show more ▶</button>
```

```javascript
// WRONG: listener attached to element that gets replaced
table.innerHTML = html;
table.querySelectorAll('.other-btn').forEach(btn =>
  btn.addEventListener('click', toggleOther)
);
```

### ✅ Correct pattern

**Step 1 — Render the expander as a `<span>` with a data attribute:**

```javascript
function renderOtherCell(empId, otherData) {
  return `<span class="other-summary" data-emp="${empId}" style="cursor:pointer;">
    ▶ ${otherData.count} more SGs (avg ${otherData.avg})
  </span>`;
}
```

**Step 2 — Attach ONE delegated listener on `document` (run ONCE at startup, not on each render):**

```javascript
document.addEventListener('click', function(e) {
  const trigger = e.target.closest('.other-summary');
  if (!trigger) return;
  const empId = trigger.dataset.emp;
  toggleOtherExpand(empId, trigger);
});
```

**Step 3 — The toggle function finds or creates the expansion row:**

```javascript
function toggleOtherExpand(empId, trigger) {
  const existing = document.getElementById('other-expand-' + empId);
  if (existing) {
    existing.remove();
    trigger.textContent = trigger.textContent.replace('▼', '▶');
    return;
  }
  const tr = trigger.closest('tr');
  const expandRow = document.createElement('tr');
  expandRow.id = 'other-expand-' + empId;
  expandRow.innerHTML = `<td colspan="99">${buildOtherDetail(empId)}</td>`;
  tr.after(expandRow);
  trigger.textContent = trigger.textContent.replace('▶', '▼');
}
```

### Also fix: no nested `<tbody>` inside a `<tbody>`

HTML does not allow `<tbody>` inside `<tbody>`. If your render loop creates per-group `<tbody>` elements, insert them directly into the `<table>`, not into an outer `<tbody id="table-body">`:

```javascript
// WRONG
document.getElementById('table-body').innerHTML = groups.map(renderGroup).join('');

// CORRECT: insert <tbody> elements directly into <table>
const table = document.getElementById('assoc-table');
table.querySelectorAll('tbody').forEach(b => b.remove());
table.insertAdjacentHTML('beforeend', groups.map(renderGroup).join(''));
```

Each `renderGroup()` should return a full `<tbody>...</tbody>` string (not just rows).

---

## Multi-Tab Dashboard Pattern

When a dashboard needs multiple independent data views (e.g., different roles, different time periods):

### Frontend: per-tab state object

```javascript
function makeTabState() {
  return {
    allData:      null,   // null = not yet loaded
    filteredData: [],
    currentPage:  1,
    sortCol:      'Name',
    sortDir:      'asc',
    // add other UI state as needed
  };
}

// One state entry per tab — must match backend tab keys
const TAB_STATE = {
  summary: makeTabState(),
  ae:      makeTabState(),
  pse:     makeTabState(),
  asa:     makeTabState(),
};

let ACTIVE_TAB = 'summary';

// Always read/write current tab's state through S()
function S() { return TAB_STATE[ACTIVE_TAB]; }
```

### Frontend: lazy loading

```javascript
function switchDashTab(name) {
  ACTIVE_TAB = name;
  ['summary','ae','pse','asa','readme'].forEach(t => {
    document.getElementById('nav-' + t)?.classList.toggle('active', t === name);
  });

  if (name === 'readme') {
    document.getElementById('dashboard-panel').style.display = 'none';
    document.getElementById('readme-panel').style.display = 'block';
    return;
  }
  document.getElementById('dashboard-panel').style.display = 'block';
  document.getElementById('readme-panel').style.display = 'none';

  // Lazy load: only fetch data when tab is first opened
  if (!S().allData) {
    loadData(name);  // calls google.script.run.loadAllData(name)
  } else {
    applyFilters();  // Tab already loaded — just re-render
  }
}
```

### Backend: per-tab sheet mapping + per-tab cache keys

```javascript
const TAB_SHEETS = {
  'summary': 'Aggregated_Summary',
  'ae':      'Aggregated_AE',
  'pse':     'Aggregated_PSE',
  'asa':     'Aggregated_ASA',
};

function loadAllData(tabName) {
  const sheetName = TAB_SHEETS[tabName || 'ae'];
  if (!sheetName) throw new Error('Unknown tab: ' + tabName);
  const { data } = loadSheetData(sheetName, false);
  // ... trim to needed columns and return
}

function loadSheetData(sheetName, forceRefresh) {
  const cacheKey = `data_${CONFIG.CACHE_VERSION}_${sheetName}`;
  // ... chunked CacheService read/write using cacheKey
}
```

**Key principle:** bump `CACHE_VERSION` every time data is re-uploaded so old cache is invalidated.

### Adding a new role/tab (checklist)

1. Process new CSV: `python scripts/process_sg.py "<csv>" <prefix>`
2. Add to `TABS_TO_UPLOAD` in push script
3. Add to `TAB_SHEETS` in `Code.gs`
4. Add to `tabs` array and `tabLabels` in `getDashboardConfig()`
5. Add nav `<button>` to HTML
6. Add to tab-name arrays in `switchDashTab` and `TAB_STATE`
7. Bump `CACHE_VERSION`
8. Re-run push script and deploy new Apps Script version

---

## Multi-Role Data Processing: Prefix Pattern

When a dashboard will eventually cover multiple roles, build the processing script to accept a `prefix` argument from day one — one script handles all roles:

```python
# process_sg.py — accepts (csv_path, prefix)
import sys
from pathlib import Path

def main():
    if len(sys.argv) < 3:
        print("Usage: python process_sg.py <csv_path> <prefix>")
        print("  e.g. python scripts/process_sg.py 'csv_sources/AE Jana v2.csv' ae")
        sys.exit(1)

    csv_path = Path(sys.argv[1])
    prefix   = sys.argv[2].lower()   # e.g. "ae", "pse", "asa"

    data = process(csv_path)
    save_outputs(data, prefix)       # → sg_aggregated_ae_data.csv, etc.

def save_outputs(data, prefix):
    out_dir = Path(__file__).parent
    out_csv   = out_dir / f'sg_aggregated_{prefix}_data.csv'
    out_json  = out_dir / f'sg_aggregated_{prefix}_data.json'
    out_cache = out_dir / f'sg_aggregated_{prefix}_filter_cache.json'
    # ... write files
```

**Usage:**
```bash
python scripts/process_sg.py "csv_sources/AE Jana v2.csv" ae
python scripts/process_sg.py "csv_sources/PSE Jana v1.csv" pse
python scripts/process_sg.py "csv_sources/Acct SA Jana v1.csv" asa
```

**Combining into a summary tab:**
```python
import csv
from pathlib import Path

scripts = Path('scripts')
combined = []
for prefix in ['ae', 'pse', 'asa']:
    combined += list(csv.DictReader(open(scripts / f'sg_aggregated_{prefix}_data.csv')))

cols = list(csv.DictReader(open(scripts / 'sg_aggregated_ae_data.csv')).fieldnames)
with open(scripts / 'sg_aggregated_summary_data.csv', 'w', newline='', encoding='utf-8') as f:
    w = csv.DictWriter(f, fieldnames=cols, extrasaction='ignore')
    w.writeheader()
    w.writerows(combined)
```

**Column name fallback pattern** (handle CSV naming variations across roles):
```python
FILTER_COLUMNS = [
    # (output_col, primary_src_col, fallback_src_col_or_None)
    ('_DIRECT_OVERLAY', 'Direct / Overlay', 'Direct or Overlay'),  # varies by file
    ('_HOME_COUNTRY',   'HOME COUNTRY',      'COUNTRY'),            # varies by file
    ('_ROLE_GROUP',     'SUMMARY_ROLE_PICKLIST', None),
]

for filter_col, src_col, fallback_col in FILTER_COLUMNS:
    val = row.get(src_col, '').strip()
    if (not val or val == '#N/A') and fallback_col:
        val = row.get(fallback_col, '').strip()
```

---

## Reference Templates

- [templates/Code.gs](templates/Code.gs) - Server-side template
- [templates/DashboardUI.html](templates/DashboardUI.html) - Client-side template
- [reference/patterns.md](reference/patterns.md) - Common code patterns

## Related

- `apps-script-visualizations` skill - Adding charts
- `apps-script-safety` rule - Deployment safety
- `apps-script-limits` rule - Quotas and performance
- `frontend-debugging-workflow` rule - Check console first
- `type-coercion-google-sheets` rule - Always parseFloat()
