---
name: google-sheets-management
description: Manage Google Sheets via MCP tools and Python scripts. Expand columns/rows, update cells, sync data. Use when working with Google Sheets, encountering grid limit errors, adding columns to sheets, or performing bulk updates.
---

# Google Sheets Management

## Quick Reference

### MCP Tools (user-gsheets)

| Tool | Use For | Limitation |
|------|---------|------------|
| `list_spreadsheets` | Find spreadsheet IDs | - |
| `list_sheets` | Get sheet names and IDs | - |
| `get_sheet_data` | Read cell data | Large sheets may timeout |
| `update_cells` | Write to single range | Cannot expand grid |
| `batch_update_cells` | Write to multiple ranges | Cannot expand grid |
| `create_spreadsheet` | New spreadsheet | - |
| `create_sheet` | New tab in spreadsheet | - |
| `share_spreadsheet` | Set permissions | - |

### Critical Limitations

- **MCP cannot expand grid** → Use `scripts/expand_sheet.py` first
- **MCP payload limit ~100 rows** → Use `scripts/fast_upload.py` for large data

## Fast Bulk Upload (Recommended for Large Data)

For uploading CSV files or large datasets, use the direct API script instead of MCP:

```bash
# If using project with venv:
source .venv/bin/activate
python scripts/fast_upload_gsheets.py

# Or using the generic skill script:
python ~/.cursor/skills/google-sheets-management/scripts/fast_upload.py
```

**Note:** If you encounter cryptography library errors, use a virtual environment:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib
```

**Features:**
- Uploads 5000+ rows per API call
- Auto-expands sheet grid if needed
- OAuth2 authentication
- Handles multiple sheets in one run

**Configuration (edit the script):**
```python
SPREADSHEET_ID = 'your-spreadsheet-id'
UPLOADS = [
    ('file1.csv', 'Sheet1'),
    ('file2.csv', 'Sheet2'),
]
```

## Expanding Columns/Rows

### Step 1: Get Sheet ID (numeric)

```bash
python scripts/expand_sheet.py \
  --spreadsheet-id "1abc..." \
  --list-sheets
```

### Step 2: Expand Dimensions

```bash
# Add 5 columns to sheet with numeric ID 123456789
python scripts/expand_sheet.py \
  --spreadsheet-id "1abc..." \
  --sheet-id 123456789 \
  --add-columns 5

# Add 100 rows
python scripts/expand_sheet.py \
  --spreadsheet-id "1abc..." \
  --sheet-id 123456789 \
  --add-rows 100
```

### Step 3: Update Cells via MCP

After expanding, use MCP `update_cells` or `batch_update_cells`.

## Python Helper Functions

### Get Sheet ID by Name

```python
def get_sheet_id(service, spreadsheet_id, sheet_name):
    """Get numeric sheetId for a named sheet tab."""
    spreadsheet = service.spreadsheets().get(spreadsheetId=spreadsheet_id).execute()
    for sheet in spreadsheet['sheets']:
        if sheet['properties']['title'] == sheet_name:
            return sheet['properties']['sheetId']
    return None
```

### Get Last Row with Data

```python
def get_last_row(service, spreadsheet_id, sheet_name):
    """Get the last row number with data (for append operations)."""
    result = service.spreadsheets().values().get(
        spreadsheetId=spreadsheet_id,
        range=f"{sheet_name}!A:A"
    ).execute()
    values = result.get('values', [])
    return len(values)
```

### Column Number to Letter Conversion

```python
def col_letter(n):
    """Convert column number (1-based) to letter (A, B, ..., Z, AA, AB, ...)."""
    result = ""
    while n > 0:
        n, remainder = divmod(n - 1, 26)
        result = chr(65 + remainder) + result
    return result

# Examples:
# col_letter(1)  → 'A'
# col_letter(26) → 'Z'
# col_letter(27) → 'AA'
# col_letter(52) → 'AZ'
```

### Expand Sheet Grid (Alternative Method)

Using `updateSheetProperties` instead of `appendDimension`:

```python
def expand_sheet_grid(service, spreadsheet_id, sheet_name, rows_needed, cols_needed):
    """Expand sheet to accommodate data, with buffer."""
    sheet_id = get_sheet_id(service, spreadsheet_id, sheet_name)
    if sheet_id is None:
        raise ValueError(f"Sheet '{sheet_name}' not found")
    
    requests = [{
        'updateSheetProperties': {
            'properties': {
                'sheetId': sheet_id,
                'gridProperties': {
                    'rowCount': rows_needed + 100,      # Add buffer
                    'columnCount': max(cols_needed + 5, 26)
                }
            },
            'fields': 'gridProperties.rowCount,gridProperties.columnCount'
        }
    }]
    
    service.spreadsheets().batchUpdate(
        spreadsheetId=spreadsheet_id,
        body={'requests': requests}
    ).execute()
```

### Clear Sheet with Keep Header Option

```python
def clear_sheet(service, spreadsheet_id, sheet_name, keep_header=False):
    """Clear sheet data, optionally preserving header row."""
    range_str = f"{sheet_name}!A2:ZZ" if keep_header else f"{sheet_name}!A1:ZZ"
    
    service.spreadsheets().values().clear(
        spreadsheetId=spreadsheet_id,
        range=range_str,
        body={}
    ).execute()
```

---

## Common Workflows

### Adding New Columns to Existing Sheet

1. **List sheets to get numeric IDs**:
   ```bash
   python scripts/expand_sheet.py --spreadsheet-id "..." --list-sheets
   ```

2. **Expand each target sheet**:
   ```bash
   python scripts/expand_sheet.py --spreadsheet-id "..." --sheet-id <ID> --add-columns 2
   ```

3. **Write headers and data via MCP**:
   Use `batch_update_cells` with:
   - `spreadsheet_id`: The spreadsheet ID
   - `sheet`: Sheet name (string)
   - `ranges`: `{"AA1:AB1": [["Header1", "Header2"]], "AA2:AB100": [[val1, val2], ...]}`

### Bulk Data Push with Source Record ID Matching

1. Load lookup data from CSV (e.g., `master_data.csv`)
2. Read existing sheet data via MCP `get_sheet_data`
3. Find `Source_Record_ID` or `ID` column
4. Build update ranges matching IDs to row positions
5. Use `batch_update_cells` to write (max ~500 ranges per call)

### Batch Upload with Row Batching

For large datasets, batch rows to avoid API limits:

```python
def upload_to_sheet(service, spreadsheet_id, sheet_name, data):
    """Upload data with batching for large datasets."""
    total_rows = len(data)
    total_cols = max(len(row) for row in data) if data else 0
    
    # Clear existing data first
    clear_sheet(service, spreadsheet_id, sheet_name)
    
    # Expand grid if needed (default is 1000 rows x 26 cols)
    if total_rows > 1000 or total_cols > 26:
        expand_sheet_grid(service, spreadsheet_id, sheet_name, total_rows, total_cols)
    
    # Upload in batches of 5000 rows (API limit is ~10MB per request)
    BATCH_SIZE = 5000
    
    for i in range(0, total_rows, BATCH_SIZE):
        batch = data[i:i + BATCH_SIZE]
        start_row = i + 1  # 1-indexed
        range_name = f"'{sheet_name}'!A{start_row}"
        
        service.spreadsheets().values().update(
            spreadsheetId=spreadsheet_id,
            range=range_name,
            valueInputOption='RAW',
            body={'values': batch}
        ).execute()
        
        print(f"  Uploaded rows {i+1}-{min(i+BATCH_SIZE, total_rows)}")
```

### Clear vs Append Mode

Support both fresh upload and incremental append:

```python
import argparse

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--clear', action='store_true', help='Clear sheet before upload')
    parser.add_argument('--append', action='store_true', help='Append to existing data')
    args = parser.parse_args()
    
    if args.clear and args.append:
        raise ValueError("Cannot use both --clear and --append")
    
    service = get_sheets_service()
    
    if args.append:
        # Append mode: find last row and write after it
        last_row = get_last_row(service, SPREADSHEET_ID, 'Data')
        push_data(service, SPREADSHEET_ID, 'Data', data,
                  start_row=last_row + 1, include_header=False)
    else:
        # Clear mode (default): clear and write from row 1
        clear_sheet(service, SPREADSHEET_ID, 'Data')
        push_data(service, SPREADSHEET_ID, 'Data', data,
                  start_row=1, include_header=True)
```

### JSON Data to Sheets

Convert aggregated JSON to sheet format:

```python
import json

def push_json_data(service, spreadsheet_id, sheet_name, json_path):
    """Push JSON data file to sheet."""
    with open(json_path, 'r') as f:
        data = json.load(f)
    
    columns = data['columns']
    records = data['data']
    
    # Build rows: header + data
    rows = [columns]  # Header row
    for record in records:
        row = [record.get(col, '') for col in columns]
        rows.append(row)
    
    upload_to_sheet(service, spreadsheet_id, sheet_name, rows)
```

### Push Filter Cache (Dropdown Values)

Store unique filter values for dashboard dropdowns:

```python
def push_filter_cache(service, spreadsheet_id, sheet_name, filter_cache):
    """Push filter values to a cache sheet for fast dropdown loading.
    
    filter_cache format: {'GEO': ['EMEA', 'NA', 'APAC'], 'Segment': ['Enterprise', 'SMB']}
    """
    rows = [['Filter', 'Value']]  # Header
    
    for filter_name, values in filter_cache.items():
        for value in values:
            rows.append([filter_name, value])
    
    clear_sheet(service, spreadsheet_id, sheet_name)
    
    service.spreadsheets().values().update(
        spreadsheetId=spreadsheet_id,
        range=f"{sheet_name}!A1:B{len(rows)}",
        valueInputOption='RAW',
        body={'values': rows}
    ).execute()
```

---

## Multi-File Upload Workflow

For dashboards with multiple data sheets:

```bash
# Step 1: Process source data
python scripts/process_data.py source.csv

# Step 2: Upload all sheets
python scripts/upload_all.py --clear
```

```python
# upload_all.py
UPLOADS = [
    ('output/summary.csv', 'Summary'),
    ('output/details.csv', 'Details'),
    ('output/filters.csv', 'FilterCache'),
]

def main():
    service = get_sheets_service()
    
    for csv_file, sheet_name in UPLOADS:
        if os.path.exists(csv_file):
            data = load_csv(csv_file)
            upload_to_sheet(service, SPREADSHEET_ID, sheet_name, data)
            print(f"✓ Uploaded {csv_file} → {sheet_name}")
        else:
            print(f"✗ Missing: {csv_file}")
```

## MCP Tool Schemas

### batch_update_cells

```json
{
  "spreadsheet_id": "string",
  "sheet": "string (name, not ID)",
  "ranges": {
    "A1:B2": [["val1", "val2"], ["val3", "val4"]],
    "D5": [["single value"]]
  }
}
```

### get_sheet_data

```json
{
  "spreadsheet_id": "string",
  "range": "SheetName!A1:Z1000"
}
```

## Authentication

### OAuth (User Tokens)

For user-facing applications or personal scripts:

```python
TOKEN_FILE = './token.json'  # Or use environment variable
SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request

creds = Credentials.from_authorized_user_file(TOKEN_FILE, SCOPES)
if creds.expired:
    creds.refresh(Request())
```

### Service Account (For Bots/Automation)

For unattended scripts, CI/CD, or server-to-server communication:

1. Create service account in Google Cloud Console (IAM & Admin → Service Accounts)
2. Download JSON key file
3. **CRITICAL**: Share spreadsheet with the `client_email` from the JSON file

```python
from google.oauth2.service_account import Credentials
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

creds = Credentials.from_service_account_file(
    'service-account-key.json', 
    scopes=SCOPES
)
service = build('sheets', 'v4', credentials=creds)
```

### Using gspread Library (Simpler API)

```bash
pip install gspread
```

```python
import gspread

gc = gspread.service_account(filename='service-account-key.json')
sh = gc.open("My Spreadsheet")
data = sh.sheet1.get_all_records()
```

---

## API Quotas and Rate Limiting

### Default Quotas

| Quota Type | Limit |
|------------|-------|
| Read requests per minute (project) | 300 |
| Read requests per minute (per user) | 60 |
| Write requests per minute (project) | 300 |
| Write requests per minute (per user) | 60 |
| Request processing timeout | 180 seconds |
| Payload size recommendation | 2 MB max |

### Exponential Backoff Pattern

```python
import time
import random
from googleapiclient.errors import HttpError

def api_call_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except HttpError as e:
            if e.resp.status == 429:
                wait = (2 ** attempt) + random.uniform(0, 1)
                print(f"Rate limited. Waiting {wait:.1f}s...")
                time.sleep(wait)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Requesting Quota Increases

1. Go to Google Cloud Console → IAM & Admin → Quotas
2. Filter for "Sheets API"
3. Request increase with business justification

---

## Gemini AI Functions (Google Workspace)

### Requirements

- Google Workspace Business/Enterprise OR Google AI Premium subscription
- Desktop browser only

### =AI() Function

```
=AI("Summarize this text", A1)
=AI("Categorize: Electronics, Clothing, Food", B2)
=AI("Translate to Spanish", C1)
```

**Limitations:**
- Text output only
- Cell-level operation (cannot see entire spreadsheet)
- Cannot use embedded functions

### Gemini Side Panel

Access via sparkle icon (✨). Can build spreadsheets, create charts, generate formulas, and run multi-step analysis.

---

## Comprehensive Error Handling

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 400 | Bad Request | Check request format |
| 429 | Too Many Requests | Implement backoff |
| 500 | Internal Server Error | Retry or file bug |
| 503 | Service Unavailable | Reduce complexity, retry |

### 503 Troubleshooting

- Use batchUpdate instead of multiple calls
- Limit concurrent requests to 1/sec/spreadsheet
- Use field masks to reduce response size
- Split large spreadsheets into multiple files
- Limit IMPORTRANGE and complex formulas

### Field Masks (Reduce Response Size)

```python
result = service.spreadsheets().get(
    spreadsheetId=SPREADSHEET_ID,
    fields='sheets.properties.title,sheets.properties.sheetId'
).execute()
```

---

## CRITICAL: Clear Sheets Before Uploading (Schema Changes)

When you modify the data pipeline (add/remove columns), **you must clear the target sheet before uploading**.

### Why

If you previously uploaded 100 columns and now upload 90, the old columns 91-100 remain with stale data. This causes:
- Wrong data displayed
- Column mismatches
- Confusing debugging

### Implementation

Add this to your upload script:

```python
def clear_sheet(service, spreadsheet_id, sheet_name):
    """Clear all data from sheet before uploading new data."""
    try:
        service.spreadsheets().values().clear(
            spreadsheetId=spreadsheet_id,
            range=f'{sheet_name}!A1:ZZ50000',
            body={}
        ).execute()
        print(f"  Cleared {sheet_name}")
    except Exception as e:
        print(f"  Warning: Could not clear {sheet_name}: {e}")

def upload_to_sheet(service, spreadsheet_id, csv_path, sheet_name):
    # ALWAYS clear first
    clear_sheet(service, spreadsheet_id, sheet_name)
    
    # Then upload new data
    # ... existing upload logic ...
```

---

## CRITICAL: Google Sheets Returns All Values as Strings

When you read data from Google Sheets (via MCP or Apps Script), **all values come back as strings**.

### The Problem

```python
# Python after reading from Sheets
data = get_sheet_data(...)
value = data[0]['Total Target Value']  # value = "29597684.41" (string!)
average = value / 10  # TypeError: unsupported operand type(s) for /
```

```javascript
// JavaScript in Apps Script
const value = row['Total Target Value'];  // value = "29597684.41" (string!)
value.toFixed(2)  // TypeError: toFixed is not a function
```

### The Fix

**Always parse before numeric operations:**

```python
# Python
value = float(data[0]['Total Target Value'])
```

```javascript
// JavaScript
const value = parseFloat(row['Total Target Value']);
```

---

## Complete Upload Script Template

Copy and customize this template for new projects:

```python
#!/usr/bin/env python3
"""
Fast Google Sheets Upload Script
Uploads CSV files directly via Sheets API (not MCP).
"""

import os
import csv
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

# ============================================================
# CONFIGURATION - Edit these values
# ============================================================
SPREADSHEET_ID = 'your-spreadsheet-id-here'
TOKEN_FILE = './token.json'  # OAuth token file
SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

# CSV files to upload: (csv_path, sheet_name)
UPLOADS = [
    ('data/summary.csv', 'Summary'),
    ('data/details.csv', 'Details'),
]

# ============================================================
# HELPER FUNCTIONS
# ============================================================

def get_credentials():
    """Load and refresh OAuth credentials."""
    creds = Credentials.from_authorized_user_file(TOKEN_FILE, SCOPES)
    if creds.expired and creds.refresh_token:
        creds.refresh(Request())
        with open(TOKEN_FILE, 'w') as f:
            f.write(creds.to_json())
    return creds

def load_csv(filepath):
    """Load CSV as list of lists."""
    with open(filepath, 'r', encoding='utf-8') as f:
        return list(csv.reader(f))

def get_sheet_id(service, spreadsheet_id, sheet_name):
    """Get numeric sheetId by name."""
    spreadsheet = service.spreadsheets().get(spreadsheetId=spreadsheet_id).execute()
    for sheet in spreadsheet['sheets']:
        if sheet['properties']['title'] == sheet_name:
            return sheet['properties']['sheetId']
    return None

def clear_sheet(service, spreadsheet_id, sheet_name):
    """Clear all data from sheet."""
    try:
        service.spreadsheets().values().clear(
            spreadsheetId=spreadsheet_id,
            range=f"'{sheet_name}'!A:ZZ",
            body={}
        ).execute()
    except Exception as e:
        print(f"  Warning: Could not clear {sheet_name}: {e}")

def expand_sheet(service, spreadsheet_id, sheet_name, rows, cols):
    """Expand sheet grid dimensions."""
    sheet_id = get_sheet_id(service, spreadsheet_id, sheet_name)
    if not sheet_id:
        return
    
    service.spreadsheets().batchUpdate(
        spreadsheetId=spreadsheet_id,
        body={'requests': [{
            'updateSheetProperties': {
                'properties': {
                    'sheetId': sheet_id,
                    'gridProperties': {'rowCount': rows + 100, 'columnCount': max(cols + 5, 26)}
                },
                'fields': 'gridProperties.rowCount,gridProperties.columnCount'
            }
        }]}
    ).execute()

def upload_to_sheet(service, spreadsheet_id, sheet_name, data):
    """Upload data to sheet with batching."""
    total_rows = len(data)
    total_cols = max(len(row) for row in data) if data else 0
    
    print(f"  Uploading {total_rows} rows × {total_cols} cols to '{sheet_name}'...")
    
    clear_sheet(service, spreadsheet_id, sheet_name)
    
    if total_rows > 1000 or total_cols > 26:
        expand_sheet(service, spreadsheet_id, sheet_name, total_rows, total_cols)
    
    BATCH_SIZE = 5000
    for i in range(0, total_rows, BATCH_SIZE):
        batch = data[i:i + BATCH_SIZE]
        service.spreadsheets().values().update(
            spreadsheetId=spreadsheet_id,
            range=f"'{sheet_name}'!A{i + 1}",
            valueInputOption='RAW',
            body={'values': batch}
        ).execute()
        print(f"    Rows {i+1}-{min(i+BATCH_SIZE, total_rows)}")

# ============================================================
# MAIN
# ============================================================

def main():
    print("=" * 50)
    print("Google Sheets Upload")
    print("=" * 50)
    
    creds = get_credentials()
    service = build('sheets', 'v4', credentials=creds)
    
    for csv_file, sheet_name in UPLOADS:
        if os.path.exists(csv_file):
            data = load_csv(csv_file)
            upload_to_sheet(service, SPREADSHEET_ID, sheet_name, data)
        else:
            print(f"  ✗ Missing: {csv_file}")
    
    print("=" * 50)
    print(f"Done! https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}")

if __name__ == '__main__':
    main()
```

---

## Related Files

- Fast bulk upload: [scripts/fast_upload.py](scripts/fast_upload.py) - Direct API for large datasets
- Grid expansion: [scripts/expand_sheet.py](scripts/expand_sheet.py) - Expand rows/columns
