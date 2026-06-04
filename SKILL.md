---
name: meta-analysis
description: >
  Reads a Cat View Google Sheet (Meta Ads category performance data) and a rules Google Doc,
  then produces a structured analysis across all categories and Overall using the rules defined
  in the doc. On first run, walks the user through a one-time setup to connect their sheet,
  credentials, and rules doc.
  Use when the user says "run meta analysis", "analyse meta ads", "check meta performance",
  "how are our meta ads doing", or any request to review Meta advertising category data.
  Accepts an optional category name to focus on a single category (e.g. /meta-analysis DSK).
---

# Meta Ads Category Analysis

Reads a Cat View Google Sheet and a rules Google Doc, computes trends across 4 time horizons,
and reports per-category findings using the rules the user has defined.

---

## Step 0 — First-Run Setup

Before doing anything else, check whether a config file exists at
`~/.claude/skills/meta-analysis/config.json`.

Run this check:
```bash
cat ~/.claude/skills/meta-analysis/config.json 2>/dev/null || echo "NO_CONFIG"
```

**If config exists:** load it and skip to Step 1.

**If NO_CONFIG:** run the setup wizard below.

### Setup Wizard

Tell the user:
> "This is the first time you're running /meta-analysis. I need three things to get started —
> your Cat View Google Sheet, your service account credentials file, and your rules Google Doc."

Then ask the following questions **one at a time** using AskUserQuestion:

**Q1 — Sheet URL**
"Paste the URL of your Cat View Google Sheet (the tab should be named 'Cat View')."
Free-text input.

**Q2 — Service Account Credentials**
"What is the full path to your Google service account JSON file?
This account needs read access to both the sheet and the rules doc.
Example: ~/.credentials/my-service-account.json"
Free-text input.

**Q3 — Rules Doc URL**
"Paste the URL of your rules Google Doc.
This is where your analysis rules are written — the skill will read and apply them on every run.
If you don't have one yet, type 'skip' and the skill will use a set of built-in default rules."
Free-text input.

After collecting answers, extract the Sheet ID from the Sheet URL
(format: `https://docs.google.com/spreadsheets/d/{SHEET_ID}/...`)
and the Doc ID from the Doc URL
(format: `https://docs.google.com/document/d/{DOC_ID}/...`).

Save config:
```bash
python3 - <<'PYEOF'
import json, os, re

sheet_url  = "SHEET_URL_PLACEHOLDER"
creds_path = "CREDS_PATH_PLACEHOLDER"
doc_url    = "DOC_URL_PLACEHOLDER"

def extract_id(url, pattern):
    m = re.search(pattern, url)
    return m.group(1) if m else url.strip()

sheet_id = extract_id(sheet_url, r'/spreadsheets/d/([a-zA-Z0-9_-]+)')
doc_id   = extract_id(doc_url,   r'/document/d/([a-zA-Z0-9_-]+)') if doc_url.lower() != 'skip' else None

config = {
    "sheet_id":   sheet_id,
    "creds_path": os.path.expanduser(creds_path.strip()),
    "doc_id":     doc_id
}

path = os.path.expanduser("~/.claude/skills/meta-analysis/config.json")
with open(path, "w") as f:
    json.dump(config, f, indent=2)
print(json.dumps(config))
PYEOF
```

Confirm to the user that setup is complete and proceed with Step 1.

---

## Step 1 — Fetch Rules

Load the config, then fetch the rules from the user's Google Doc (if `doc_id` is set).

```python
from googleapiclient.discovery import build
from google.oauth2 import service_account
import json, os

config_path = os.path.expanduser("~/.claude/skills/meta-analysis/config.json")
with open(config_path) as f:
    cfg = json.load(f)

CREDS_FILE = cfg["creds_path"]
SHEET_ID   = cfg["sheet_id"]
DOC_ID     = cfg.get("doc_id")

creds = service_account.Credentials.from_service_account_file(
    CREDS_FILE,
    scopes=[
        "https://www.googleapis.com/auth/spreadsheets.readonly",
        "https://www.googleapis.com/auth/documents.readonly"
    ]
)

rules_text = None
if DOC_ID:
    docs_service = build("docs", "v1", credentials=creds)
    doc = docs_service.documents().get(documentId=DOC_ID).execute()
    parts = []
    for elem in doc.get("body", {}).get("content", []):
        para = elem.get("paragraph")
        if para:
            for pe in para.get("elements", []):
                tr = pe.get("textRun")
                if tr:
                    parts.append(tr.get("content", ""))
    rules_text = "".join(parts).strip()

print("RULES_TEXT:", rules_text or "USE_DEFAULTS")
```

If `rules_text` is returned, use it as the analysis ruleset in Step 3.
If `USE_DEFAULTS`, fall back to the built-in rules in the **Default Rules** section below.

---

## Step 2 — Fetch & Parse the Sheet

```python
from googleapiclient.discovery import build
from google.oauth2 import service_account
import json, os

config_path = os.path.expanduser("~/.claude/skills/meta-analysis/config.json")
with open(config_path) as f:
    cfg = json.load(f)

CREDS_FILE = cfg["creds_path"]
SHEET_ID   = cfg["sheet_id"]

creds = service_account.Credentials.from_service_account_file(
    CREDS_FILE, scopes=["https://www.googleapis.com/auth/spreadsheets.readonly"]
)
service = build("sheets", "v4", credentials=creds)

header_text = service.spreadsheets().values().get(
    spreadsheetId=SHEET_ID, range="Cat View!1:1"
).execute().get("values", [[]])[0]

rows = service.spreadsheets().values().get(
    spreadsheetId=SHEET_ID, range="Cat View",
    valueRenderOption="UNFORMATTED_VALUE"
).execute().get("values", [])

col_labels    = header_text[2:]
monthly_cols  = list(range(0, 4))
weekly_cols   = list(range(4, 8))
daily_cols    = list(range(8, len(col_labels)))

def safe_float(v):
    try: return float(v)
    except: return None

# Fill-down category
data = {}
current_cat = ""
for row in rows[1:]:
    cat_cell = str(row[0]).strip() if row and row[0] else ""
    if cat_cell: current_cat = cat_cell
    metric = str(row[1]).strip() if len(row) > 1 and row[1] else ""
    vals = row[2:] if len(row) > 2 else []
    vals = vals + [""] * (len(col_labels) - len(vals))
    data[(current_cat, metric)] = vals

# Find last daily column with actual data (using Overall Revenue as probe)
rev_row = data.get(("Overall", "Revenue"), [])
last_filled = 8
for i in daily_cols:
    if i < len(rev_row) and safe_float(rev_row[i]) is not None and safe_float(rev_row[i]) > 0:
        last_filled = i

last30 = list(range(max(8, last_filled - 29), last_filled + 1))
last7  = list(range(max(8, last_filled - 6),  last_filled + 1))

categories = list(dict.fromkeys(k[0] for k in data.keys()))
key_metrics = [
    "Total Spends", "Perf Spends", "Revenue", "Orders",
    "ROAS Perf", "Perf CVR", "CTR Perf excl AN",
    "CPM Perf", "CPS Perf excl AN", "AN Percentage", "% of AN Sessions"
]

def get_vals(cat, metric, indices):
    vals = data.get((cat, metric), [])
    return [safe_float(vals[i]) if i < len(vals) else None for i in indices]

output = {
    "col_labels": {
        "monthly":      [col_labels[i] for i in monthly_cols if i < len(col_labels)],
        "weekly":       [col_labels[i] for i in weekly_cols  if i < len(col_labels)],
        "last7_labels": [col_labels[i] for i in last7        if i < len(col_labels)],
        "last30_labels":[col_labels[i] for i in last30       if i < len(col_labels)],
    },
    "categories": {}
}

for cat in categories:
    output["categories"][cat] = {}
    for met in key_metrics:
        output["categories"][cat][met] = {
            "monthly": get_vals(cat, met, monthly_cols),
            "weekly":  get_vals(cat, met, weekly_cols),
            "last7":   get_vals(cat, met, last7),
            "last30":  get_vals(cat, met, last30),
        }

print(json.dumps(output))
```

---

## Step 3 — Analyse & Report

Using the parsed JSON from Step 2 and the rules from Step 1, perform the following for each
category and Overall:

### Time Horizons
For each category, compute and summarise across all four:
- **Day-on-day:** last 7 days — direction (up/down/flat) for Revenue, Spends, ROAS, CVR, CTR excl AN, CPM, AN%
- **Week-on-week:** last 4 weekly columns — latest vs prior week % change for Revenue, Spends, ROAS
- **MTD:** Jun MTD daily run rate vs prior complete month daily run rate (divide monthly total by days in month)
- **Month-on-month:** last 3 complete monthly columns — trend direction and magnitude for Revenue, Spends, ROAS, Orders

### Applying the Rules
Read the rules text fetched from the user's Google Doc (Step 1). Apply each rule stated there to
each category. The rules doc is the authoritative source — follow it exactly.

If `USE_DEFAULTS` (no rules doc provided), apply the **Default Rules** section below.

### Report Format
For each category (Overall first):

```
## [Category Name]

**Trend Summary:** [one sentence]

**Time Horizon Breakdown:**
- D/D (7 days): Revenue [direction], Spends [direction], ROAS [direction]
- W/W: latest vs prior — Revenue [+X%], Spends [+X%], ROAS [+X%]
- MTD DRR: Revenue [₹X/day] vs prior month [₹X/day]
- M/M (last 3 months): Revenue [trend], ROAS [trend]

**Rules Triggered:** [which rules and why]

**Diagnosis:** [which funnel metric is responsible]

**Suggested Action:** [specific and actionable]
```

Skip categories with no data across all time horizons.

End with a **1-paragraph Overall Summary** — top 2–3 findings and the single most urgent action.

### Arguments
If a category name is passed as the skill argument, only analyse that category + Overall.

---

## Default Rules

Used only when no rules doc is provided. These cover standard Meta Ads performance patterns:

**Priority:** Revenue is always the first metric to check. CTR/CVR are secondary diagnostics.

**Scale signal:** If spends are increasing AND ROAS is increasing → positive, can scale.

**Early warning:** If CTR or CVR is decaying even when ROAS appears stable → flag as early warning of future ROAS decline.

**Spends up, ROAS holding:** Compare against best historical period for that category. Report what funnel metrics looked like then vs now.

**Revenue declining + spends up:** ROAS is falling. Diagnose: CTR decay? CVR decay? CPM increase? CPS increase? AN% increase? Suggest what to improve to recover to historical best.

**Revenue declining + spends flat:** Check CPM, CPS, AN% for audience saturation. Report whether headroom to scale exists.

**Revenue holding + spends up:** Flag — ROAS is falling silently. Diagnose.

**Revenue holding + spends flat:** Assess scalability — CPM trend, CPS trend, AN%, CTR.

**ROAS decline:** No fixed floor. Compare to the same category's best historical period. Never recommend pausing. If not improving consistently, recommend creative refresh, offer change, or targeting adjustment.

**AN% — overall:** Flag if overall AN% exceeds 20%.

**AN% — category:** Only flag if ROAS worsened at the same time AN% increased. If ROAS was better at high AN%, that is a positive data point.

**Revenue holding definition:** Flat within a 7-day window.

**New launches:** Any category with fewer than 20 days of spend data — report direction only, skip rules.

**No cross-category comparisons:** Each category is assessed against its own historical trajectory only.
