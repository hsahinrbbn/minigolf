# Mini Golf Dashboard - Score Entry System

A simple QR code-based score entry system that saves data to SharePoint for use with Ribbon Analytics dashboards.

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐     ┌──────────────┐
│  QR Code    │────▶│  HTML Score Page │────▶│ Power Automate  │────▶│  SharePoint  │
│  (per hole) │     │  (hosted on SP)  │     │  HTTP Trigger   │     │  Excel/CSV   │
└─────────────┘     └──────────────────┘     └─────────────────┘     └──────────────┘
                                                                            │
                                                                            ▼
                                                                     ┌──────────────┐
                                                                     │ RA Dashboard │
                                                                     │  (Live Mode) │
                                                                     └──────────────┘
```

## Files Included

| File | Purpose |
|------|---------|
| `minigolf-score.html` | Mobile-friendly score entry page |
| `generate-qrcodes.ps1` | PowerShell script to generate QR codes for all holes |
| `PowerAutomate-Flow.md` | Step-by-step instructions for Power Automate setup |

---

## Setup Instructions

### Step 1: Create SharePoint Excel File

1. Go to your **PMBI SharePoint site**
2. Navigate to or create a folder: `Documents/MiniGolf/`
3. Create a new Excel file: `MiniGolfScores.xlsx`
4. Add these column headers in Row 1:

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| Timestamp | Player | Hole | Par | Strokes | Score | Date | Time |

5. Format as a Table (Ctrl+T) and name it `Scores`
6. Save and close

---

### Step 2: Create Power Automate Flow

1. Go to [Power Automate](https://make.powerautomate.com)
2. Click **+ Create** → **Instant cloud flow**
3. Name: `MiniGolf Score Entry`
4. Trigger: **When an HTTP request is received**
5. Click **Create**

#### Configure the HTTP Trigger:

In the trigger, paste this JSON schema:
```json
{
    "type": "object",
    "properties": {
        "Timestamp": { "type": "string" },
        "Player": { "type": "string" },
        "Hole": { "type": "integer" },
        "Par": { "type": "integer" },
        "Strokes": { "type": "integer" },
        "Score": { "type": "integer" },
        "Date": { "type": "string" },
        "Time": { "type": "string" }
    }
}
```

#### Add Action: Add a row into a table

1. Click **+ New step**
2. Search for **Excel Online (Business)**
3. Select **Add a row into a table**
4. Configure:
   - **Location**: Your SharePoint site (e.g., `PMBI`)
   - **Document Library**: `Documents`
   - **File**: `/MiniGolf/MiniGolfScores.xlsx`
   - **Table**: `Scores`
   - Map each column to the corresponding dynamic content from the trigger

#### Add Response Action (Optional but recommended):

1. Click **+ New step**
2. Search for **Response**
3. Configure:
   - **Status Code**: `200`
   - **Body**: `{"status": "success"}`

5. **Save** the flow
6. **Copy the HTTP POST URL** from the trigger (you'll need this for the HTML page)

---

### Step 3: Update HTML Page

1. Open `minigolf-score.html`
2. Find this line near the top of the `<script>` section:
   ```javascript
   const POWER_AUTOMATE_URL = 'YOUR_POWER_AUTOMATE_HTTP_TRIGGER_URL';
   ```
3. Replace with your actual Power Automate URL:
   ```javascript
   const POWER_AUTOMATE_URL = 'https://prod-xx.westus.logic.azure.com:443/workflows/...';
   ```
4. Save the file

---

### Step 4: Host the HTML Page on SharePoint

1. Go to your **PMBI SharePoint site**
2. Navigate to **Site Pages** or create a folder in Documents
3. Upload `minigolf-score.html`
4. Get the direct URL to the file (right-click → Copy link)
5. Test by opening: `https://your-site/path/minigolf-score.html?hole=1`

---

### Step 5: Generate QR Codes

1. Open `generate-qrcodes.ps1`
2. Update the `$BaseUrl` variable with your SharePoint page URL:
   ```powershell
   $BaseUrl = "https://ribboncommunications.sharepoint.com/sites/PMBI/Shared%20Documents/MiniGolf/minigolf-score.html"
   ```
3. Run the script:
   ```powershell
   .\generate-qrcodes.ps1
   ```
4. QR codes will be saved to `.\QRCodes\` folder
5. Open `PrintSheet.html` in a browser to print QR code signs

---

## Data Format

Each score submission creates a row in Excel:

| Timestamp | Player | Hole | Par | Strokes | Score | Date | Time |
|-----------|--------|------|-----|---------|-------|------|------|
| 2026-04-16T14:30:00.000Z | Team Alpha | 5 | 3 | 2 | -1 | 4/16/2026 | 14:30:00 |
| 2026-04-16T14:32:15.000Z | Team Beta | 5 | 3 | 4 | 1 | 4/16/2026 | 14:32:15 |

- **Score** = Strokes - Par (negative is under par, positive is over par)

---

## Connecting to Ribbon Analytics

### Option A: Direct Excel Connection
If RA supports Excel/SharePoint data sources, point directly to `MiniGolfScores.xlsx`

### Option B: Export to CSV
Add a scheduled Power Automate flow to export Excel data to CSV:
1. Trigger: Recurrence (every 5 minutes)
2. Action: List rows in table
3. Action: Create CSV file in SharePoint

### Option C: Database
Modify the Power Automate flow to also write to a SQL database that RA can query.

---

## Customization

### Change Par Values
Edit the `HOLE_PARS` object in `minigolf-score.html`:
```javascript
const HOLE_PARS = {
    1: 3, 2: 2, 3: 3, 4: 4, 5: 3,
    // ... customize for your course
};
```

### Change Number of Holes
Edit `$NumberOfHoles` in `generate-qrcodes.ps1`:
```powershell
$NumberOfHoles = 9  # For a 9-hole course
```

### Add Team/Session ID
Modify the HTML form to include additional fields like Session ID or Event Name.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| QR codes not generating | Check internet connection; Google Charts API requires internet |
| Score not submitting | Check browser console (F12); verify Power Automate URL is correct |
| CORS error | Ensure Power Automate response includes proper headers |
| Data not appearing in Excel | Check Power Automate run history for errors |

---

## Files Location Summary

```
SharePoint (PMBI site)
├── Documents/
│   └── MiniGolf/
│       ├── MiniGolfScores.xlsx    ← Score data
│       └── minigolf-score.html    ← Score entry page (optional location)
└── Site Pages/
    └── minigolf-score.html        ← Score entry page (alternative location)

Local (for setup)
├── minigolf-score.html
├── generate-qrcodes.ps1
├── README.md
└── QRCodes/
    ├── Hole_01.png
    ├── Hole_02.png
    ├── ...
    └── PrintSheet.html
```
