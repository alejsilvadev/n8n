#  Time Logger — Chrome Extension + n8n Workflow

Tracks every Chrome tab session, sends it to n8n, and builds a Google Sheet
with raw session logs + per-Teamwork-task time totals (automatically attributed).

---

##  What's Included

```
chrome-extension/
  manifest.json     — Extension config
  background.js     — Core tracking logic (service worker)
  popup.html        — Extension popup UI
  popup.js          — Popup script

n8n-workflow/
  time-logger-workflow.json   — Import this into n8n
```

---

##  Setup — Step by Step

### Step 1 — Create your Google Sheet

1. Create a new Google Sheet
2. Add **two tabs** named exactly:
   - `Raw Sessions`
   - `Task Summary`
3. Copy the Sheet ID from the URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_IS_YOUR_SHEET_ID`**`/edit`

**Raw Sessions columns (Row 1 headers):**
| Date | Start Time | End Time | Duration (seconds) | Duration (HH:MM:SS) | Title | URL | Hostname | Category | Is Teamwork Task | Teamwork Task ID | Trigger Reason | Logged At |

**Task Summary columns (Row 1 headers):**
| Task ID | Task Title | Task URL | Total Seconds | Total Time (HH:MM:SS) | Total Hours | Session Count | First Seen | Last Seen | Attributed Sites | Last Updated |

---

### Step 2 — Import the n8n Workflow

1. Open your n8n instance
2. Click **"Import from file"** and select `time-logger-workflow.json`
3. In the workflow, click each **Google Sheets** node and:
   - Connect your Google account (OAuth2)
   - Replace `YOUR_GOOGLE_SHEET_ID` with your actual Sheet ID
4. Click the **Webhook** node and copy the **Production URL** — it looks like:
   `https://your-n8n.com/webhook/time-logger`
5. **Activate** the workflow (toggle in top right)

---

### Step 3 — Configure the Chrome Extension

1. Open `chrome-extension/background.js`
2. Edit line 6 — paste your n8n webhook URL:
   ```js
   const N8N_WEBHOOK_URL = "https://your-n8n.com/webhook/time-logger";
   ```
3. Edit line 9 — set your Teamwork domain:
   ```js
   const TEAMWORK_DOMAIN = "mycompany.teamwork.com";
   ```
4. Do the same edit in `popup.js` line 1

---

### Step 4 — Load the Extension in Chrome

1. Open Chrome → go to `chrome://extensions`
2. Enable **Developer Mode** (top right toggle)
3. Click **"Load unpacked"**
4. Select the `chrome-extension/` folder
5. The extension icon will appear in your toolbar

---

##  How the Time Attribution Works

```
You visit:  Teamwork Task #100  ← becomes "active task"
Then visit: GitHub (5 min)      → attributed to Task #100
Then visit: Google Docs (3 min) → attributed to Task #100
Then visit: Stack Overflow (2m) → attributed to Task #100
You visit:  Teamwork Task #200  ← NEW active task
Then visit: Figma (10 min)      → attributed to Task #200

Result in Task Summary:
  Task #100 → 10 min  (task page + github + docs + stackoverflow)
  Task #200 → 10 min  (task page + figma)
```

Every time a session ends (tab switch, new URL, window blur), the extension
fires a payload to n8n. n8n logs the raw row, then re-reads all rows and
recalculates the full Task Summary from scratch.

---

##  What You Get in Google Sheets

### Raw Sessions tab
Every single page visit with exact times and durations. Your full browsing
history for work, timestamped to the second.

### Task Summary tab
One row per Teamwork task. Automatically updated after every session.
Shows total time logged, number of sessions, which sites were visited
during that task block, and first/last seen timestamps.

---

##  Customisation

**Add more Teamwork URL patterns** — edit `buildTaskPatterns()` in `background.js`

**Exclude domains from logging** — add to `isIgnoredUrl()` in `background.js`

**Change minimum session length** — edit `MIN_DURATION_MS` in `background.js`
(default: 4000ms = 4 seconds, prevents accidental click noise)

**Add Slack notifications** — add a Slack node after "Get Task Context" node
in n8n, triggered when `is_new_task === true`

---

##  Privacy

All data goes directly from your browser → your n8n instance → your Google Sheet.
Nothing is sent to any third party. You control everything.
