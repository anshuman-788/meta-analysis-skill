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

Check whether a config file exists:
```bash
cat ~/.claude/skills/meta-analysis/config.json 2>/dev/null || echo "NO_CONFIG"
```

**If config exists:** load it and skip to Step 1.

**If NO_CONFIG:** tell the user:
> "This is the first time you're running /meta-analysis. I need three things to get started —
> your Cat View Google Sheet, your service account credentials file, and your rules Google Doc."

Ask the following using AskUserQuestion, one at a time:

**Q1:** "Paste the URL of your Cat View Google Sheet (the tab must be named 'Cat View')."
**Q2:** "Full path to your Google service account JSON file? (Needs read access to both sheet and doc.) Example: ~/.credentials/my-service-account.json"
**Q3:** "Paste the URL of your rules Google Doc — or type 'skip' to use built-in default rules."

Extract Sheet ID and Doc ID from URLs, then save:
```bash
python3 - <<'PYEOF'
import json, os, re
sheet_url  = "SHEET_URL"
creds_path = "CREDS_PATH"
doc_url    = "DOC_URL"
def extract_id(url, pat):
    m = re.search(pat, url)
    return m.group(1) if m else url.strip()
config = {
    "sheet_id":   extract_id(sheet_url,  r'/spreadsheets/d/([a-zA-Z0-9_-]+)'),
    "creds_path": os.path.expanduser(creds_path.strip()),
    "doc_id":     extract_id(doc_url, r'/document/d/([a-zA-Z0-9_-]+)') if doc_url.lower() != 'skip' else None
}
path = os.path.expanduser("~/.claude/skills/meta-analysis/config.json")
with open(path, "w") as f:
    json.dump(config, f, indent=2)
print(json.dumps(config))
PYEOF
```

---

## Step 1 — Load Config, Rules, and Last Run

```python
from googleapiclient.discovery import build
from google.oauth2 import service_account
import json, os

config_path   = os.path.expanduser("~/.claude/skills/meta-analysis/config.json")
last_run_path = os.path.expanduser("~/.claude/skills/meta-analysis/last_run.json")

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

# Fetch rules doc
rules_text = None
if DOC_ID:
    docs_svc = build("docs", "v1", credentials=creds)
    doc = docs_svc.documents().get(documentId=DOC_ID).execute()
    parts = []
    for elem in doc.get("body", {}).get("content", []):
        para = elem.get("paragraph")
        if para:
            for pe in para.get("elements", []):
                tr = pe.get("textRun")
                if tr:
                    parts.append(tr.get("content", ""))
    rules_text = "".join(parts).strip()

# Load last run if it exists
last_run = None
if os.path.exists(last_run_path):
    with open(last_run_path) as f:
        last_run = json.load(f)

print("RULES:", rules_text or "USE_DEFAULTS")
print("LAST_RUN_EXISTS:", last_run is not None)
```

Keep `rules_text` and `last_run` in memory for Steps 3 and 4.

---

## Step 2 — Fetch & Parse Sheet, Compute Efficient Frontier

```python
from googleapiclient.discovery import build
from google.oauth2 import service_account
import json, os

config_path = os.path.expanduser("~/.claude/skills/meta-analysis/config.json")
with open(config_path) as f:
    cfg = json.load(f)

creds = service_account.Credentials.from_service_account_file(
    cfg["creds_path"], scopes=["https://www.googleapis.com/auth/spreadsheets.readonly"]
)
service = build("sheets", "v4", credentials=creds)

header_text = service.spreadsheets().values().get(
    spreadsheetId=cfg["sheet_id"], range="Cat View!1:1"
).execute().get("values", [[]])[0]

rows = service.spreadsheets().values().get(
    spreadsheetId=cfg["sheet_id"], range="Cat View",
    valueRenderOption="UNFORMATTED_VALUE"
).execute().get("values", [])

col_labels   = header_text[2:]
monthly_cols = list(range(0, 4))
weekly_cols  = list(range(4, 8))
daily_cols   = list(range(8, len(col_labels)))

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

# Find last daily column with actual data
rev_row = data.get(("Overall", "Revenue"), [])
last_filled = 8
for i in daily_cols:
    if i < len(rev_row) and safe_float(rev_row[i]) and safe_float(rev_row[i]) > 0:
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
    cat_data = {}
    for met in key_metrics:
        cat_data[met] = {
            "monthly": get_vals(cat, met, monthly_cols),
            "weekly":  get_vals(cat, met, weekly_cols),
            "last7":   get_vals(cat, met, last7),
            "last30":  get_vals(cat, met, last30),
        }

    # ── Efficient Frontier ──────────────────────────────────────────────────
    # Find the week or month with the best ROAS and the spend level that week
    roas_m = cat_data.get("ROAS Perf", {}).get("monthly", [])
    sp_m   = (cat_data.get("Total Spends", {}) if cat == "Overall"
              else cat_data.get("Perf Spends", {})).get("monthly", [])
    roas_w = cat_data.get("ROAS Perf", {}).get("weekly", [])
    sp_w   = (cat_data.get("Total Spends", {}) if cat == "Overall"
              else cat_data.get("Perf Spends", {})).get("weekly", [])

    best_roas, best_spend, best_label = 0, None, "unknown"
    for i, (r, s) in enumerate(zip(roas_m, sp_m)):
        if r and r > best_roas:
            best_roas, best_spend = r, s
            best_label = output["col_labels"]["monthly"][i] if i < len(output["col_labels"]["monthly"]) else f"M{i}"
    for i, (r, s) in enumerate(zip(roas_w, sp_w)):
        if r and r > best_roas:
            best_roas, best_spend = r, s
            best_label = output["col_labels"]["weekly"][i] if i < len(output["col_labels"]["weekly"]) else f"W{i}"

    # Current ROAS and spend (latest monthly — index 2 = most recent complete month)
    current_roas  = roas_m[2] if len(roas_m) > 2 else None
    current_spend = sp_m[2]   if len(sp_m)   > 2 else None

    # ── ₹ Impact (for volume-weighted summary) ──────────────────────────────
    # (best_roas - current_roas) × current monthly spend = foregone revenue
    rupee_impact = None
    if best_roas and current_roas and current_spend:
        rupee_impact = (best_roas - current_roas) * current_spend

    cat_data["_meta"] = {
        "efficient_frontier": {
            "best_roas":   best_roas,
            "best_spend":  best_spend,
            "best_label":  best_label,
        },
        "current": {
            "roas":  current_roas,
            "spend": current_spend,
        },
        "rupee_impact": rupee_impact,   # positive = efficiency loss vs historical best
    }

    output["categories"][cat] = cat_data

print(json.dumps(output))
```

---

## Step 3 — Analyse Using Rules

Using the parsed JSON from Step 2, apply the rules from the user's doc (or defaults below).

### For each category compute

**Trend directions** across all four time horizons:
- Direction = "up" if second half avg > first half avg by >5%, "down" if <-5%, else "flat"
- For D/D 7-day: also note single-day spikes/drops >20%

**Spend-driven vs genuine ROAS** (critical check before applying R1):
- Compare ROAS direction vs spend direction over the same window
- If ROAS up + spend down → efficiency ceiling (R3), not scale signal (R1)
- If ROAS up + spend flat or up → genuine scale signal (R1)
- State this explicitly in the output

**Creative fatigue vs audience exhaustion** (apply whenever ROAS is declining, R10):
- Check CPM direction and CTR direction together:
  - CPM stable/down + CTR down → creative fatigue → recommend creative refresh
  - CPM up + CTR flat or down → audience exhaustion → recommend audience expansion or spend reduction
- Name the diagnosis explicitly

**Efficient frontier** (R13 — always):
- Pull `_meta.efficient_frontier` from parsed data
- State: "Best efficiency: ROAS [X] at [₹Y/month] spend ([period])"
- State current: "Current: ROAS [X] at [₹Y/month]"
- If current spend > best_spend and ROAS is lower: flag overspend past the efficiency ceiling

**Vs last run** (R15 — if last_run exists):
- Compare current key metrics (ROAS, Revenue, Spend, AN%, CTR) to last_run values for the same category
- Flag anything that has moved >10% since the last run

### 99 Offer contribution check (always):
Compute: 99 Revenue ÷ Overall Revenue for each time horizon.
Target band: 10–15%. Flag if outside.

### Volume-weighted summary (R14 — for the final Overall Summary):
Rank all category findings by `rupee_impact` descending.
Lead with the highest ₹ impact item. State the ₹ figure.

---

## Step 4 — Save Current Run

After analysis, save key metrics to last_run.json for future comparison:

```python
import json, os
from datetime import date

last_run_path = os.path.expanduser("~/.claude/skills/meta-analysis/last_run.json")

# Build a compact snapshot: category → {roas, revenue, spend, an_pct, ctr} at latest monthly
snapshot = {
    "run_date": date.today().isoformat(),
    "categories": {}
}

# `output` is the parsed data from Step 2
for cat, cd in output["categories"].items():
    def m2(met):
        vals = cd.get(met, {}).get("monthly", [])
        return vals[2] if len(vals) > 2 else None

    snapshot["categories"][cat] = {
        "roas":    m2("ROAS Perf"),
        "revenue": m2("Revenue"),
        "spend":   m2("Total Spends") if cat == "Overall" else m2("Perf Spends"),
        "an_pct":  m2("AN Percentage") if cat == "Overall" else m2("% of AN Sessions"),
        "ctr":     m2("CTR Perf excl AN"),
        "cvr":     m2("Perf CVR"),
    }

with open(last_run_path, "w") as f:
    json.dump(snapshot, f, indent=2)
print("Snapshot saved to last_run.json")
```

---

## Step 5 — Report Format

For each category (Overall first, then ranked by ₹ impact for the rest):

```
## [Category Name]

**Trend Summary:** [one sentence]

**Efficient Frontier:** Best ROAS [X] at [₹Y/month] spend ([period]) | Current: ROAS [X] at [₹Y/month]

**Time Horizon Breakdown:**
- D/D (7 days): Revenue [direction], Spends [direction], ROAS [direction]
- W/W: Revenue [+X%], Spends [+X%], ROAS [+X%] (latest vs prior week)
- MTD DRR: Revenue [₹X/day] vs prior month [₹X/day]
- M/M (last 3 months): Revenue [trend], ROAS [trend]

**ROAS Signal:** [Genuine scale / Efficiency ceiling / Declining — which one and why]

**Diagnosis:** [Creative fatigue / Audience exhaustion / other — state explicitly]

**Rules Triggered:** [R1, R5, R10 etc. with one-line reason each]

**Suggested Action:** [Specific and actionable. Never "pause".]

**Vs Last Run ([date]):** [Key metric changes since prior analysis, or "No prior run"]
```

Skip categories with no data across all time horizons.

**Overall Summary** (last section):
- Rank findings by ₹ impact (R14)
- 3–5 bullet points, highest ₹ impact first
- Single most urgent action as the closing line

---

## Arguments
If a category name is passed as the skill argument, only analyse that category + Overall.

---

## Default Rules

Used only when no rules doc is provided.

**Priority:** Revenue first. CTR/CVR are secondary diagnostics.

R1 — Scale signal: Spends up + ROAS up → flag positive. Confirm spend is not falling (else R3).
R2 — Early warning: CTR excl AN or CVR declining even when ROAS holds → flag as leading indicator.
R3 — Efficiency ceiling: ROAS up + spend down → not a scale signal. State efficient spend level.
R4 — Spend up, ROAS holding: compare funnel metrics to best historical period. Report findings.
R5 — Revenue declining + spend up: diagnose with R10. Compare to efficient frontier.
R6 — Revenue declining + spend flat: check CPM, CPS, AN% for saturation.
R7 — Revenue holding + spend up: flag silent ROAS decline. Diagnose with R10.
R8 — Revenue holding + spend flat: assess scalability via CPM, CPS, AN%, CTR.
R9 — ROAS decline: compare to efficient frontier. Identify which funnel metric changed. Never pause.
R10 — Creative fatigue vs audience exhaustion: CPM down + CTR down = fatigue. CPM up + CTR flat/down = exhaustion.
R11 — AN% overall >20%: flag immediately.
R12 — AN% category: only flag if ROAS also worsened. If ROAS was better at high AN%, note as positive.
R13 — Efficient frontier: always compute and display.
R14 — Volume-weighted summary: rank by ₹ impact = (best ROAS − current ROAS) × monthly spend.
R15 — Vs last run: compare to last_run.json if it exists.
