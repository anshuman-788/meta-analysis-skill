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
and reports a full narrative analysis per category using the rules defined in the doc.

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
**Q2:** "Full path to your Google service account JSON file? Example: ~/.credentials/my-service-account.json"
**Q3:** "Paste the URL of your rules Google Doc — or type 'skip' to use built-in default rules."

Save config:
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
os.makedirs(os.path.dirname(path), exist_ok=True)
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

creds = service_account.Credentials.from_service_account_file(
    cfg["creds_path"],
    scopes=[
        "https://www.googleapis.com/auth/spreadsheets.readonly",
        "https://www.googleapis.com/auth/documents.readonly"
    ]
)

# Fetch rules doc
rules_text = None
if cfg.get("doc_id"):
    docs_svc = build("docs", "v1", credentials=creds)
    doc = docs_svc.documents().get(documentId=cfg["doc_id"]).execute()
    parts = []
    for elem in doc.get("body", {}).get("content", []):
        para = elem.get("paragraph")
        if para:
            for pe in para.get("elements", []):
                tr = pe.get("textRun")
                if tr:
                    parts.append(tr.get("content", ""))
    rules_text = "".join(parts).strip()

# Load last run snapshot
last_run = None
if os.path.exists(last_run_path):
    with open(last_run_path) as f:
        last_run = json.load(f)

print("RULES:", rules_text or "USE_DEFAULTS")
print("LAST_RUN:", json.dumps(last_run) if last_run else "NONE")
```

Keep `rules_text` and `last_run` for Steps 3 and 4.

---

## Step 2 — Fetch Sheet, Compute Efficient Frontier and CTR Decay

Run this single script. It returns a JSON blob with monthly/weekly/daily data,
efficient frontier, ₹ impact, and a CTR decay flag per category.

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
svc = build("sheets", "v4", credentials=creds)

header_text = svc.spreadsheets().values().get(
    spreadsheetId=cfg["sheet_id"], range="Cat View!1:1"
).execute().get("values", [[]])[0]

rows = svc.spreadsheets().values().get(
    spreadsheetId=cfg["sheet_id"], range="Cat View",
    valueRenderOption="UNFORMATTED_VALUE"
).execute().get("values", [])

col_labels   = header_text[2:]
monthly_cols = list(range(0, 4))
weekly_cols  = list(range(4, 8))
daily_cols   = list(range(8, len(col_labels)))

def sf(v):
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
    vals += [""] * (len(col_labels) - len(vals))
    data[(current_cat, metric)] = vals

# Find last filled daily column using Overall Revenue as probe
rev_row = data.get(("Overall", "Revenue"), [])
last_filled = daily_cols[0] if daily_cols else 8
for i in daily_cols:
    if i < len(rev_row) and sf(rev_row[i]) and sf(rev_row[i]) > 0:
        last_filled = i

filled_daily = list(range(daily_cols[0], last_filled + 1))
last30 = list(range(max(daily_cols[0], last_filled - 29), last_filled + 1))
last7  = list(range(max(daily_cols[0], last_filled - 6),  last_filled + 1))
last10 = list(range(max(daily_cols[0], last_filled - 9),  last_filled + 1))

mlabels = [col_labels[i] for i in monthly_cols if i < len(col_labels)]
wlabels = [col_labels[i] for i in weekly_cols  if i < len(col_labels)]

categories = list(dict.fromkeys(k[0] for k in data.keys()))
key_metrics = [
    "Total Spends", "Perf Spends", "Revenue", "Orders",
    "ROAS Perf", "Perf CVR", "CTR Perf excl AN",
    "CPM Perf", "CPS Perf excl AN", "AN Percentage", "% of AN Sessions"
]

def gv(cat, met, idx):
    vals = data.get((cat, met), [])
    return [sf(vals[i]) if i < len(vals) else None for i in idx]

def smooth3(lst):
    """3-day moving average to reduce noise."""
    out = [None] * len(lst)
    for i in range(2, len(lst)):
        vs = [v for v in lst[i-2:i+1] if v is not None]
        out[i] = sum(vs)/len(vs) if vs else None
    return out

output = {
    "col_labels": {
        "monthly":      mlabels,
        "weekly":       wlabels,
        "last7_labels": [col_labels[i] for i in last7  if i < len(col_labels)],
        "last30_labels":[col_labels[i] for i in last30 if i < len(col_labels)],
    },
    "categories": {}
}

for cat in categories:
    is_ov  = (cat == "Overall")
    sp_key = "Total Spends" if is_ov else "Perf Spends"
    an_key = "AN Percentage" if is_ov else "% of AN Sessions"

    cd = {}
    for met in key_metrics:
        cd[met] = {
            "monthly": gv(cat, met, monthly_cols),
            "weekly":  gv(cat, met, weekly_cols),
            "last7":   gv(cat, met, last7),
            "last30":  gv(cat, met, last30),
        }

    # ── Efficient Frontier ────────────────────────────────────────────────────
    roas_m = cd["ROAS Perf"]["monthly"]
    sp_m   = cd[sp_key]["monthly"]
    roas_w = cd["ROAS Perf"]["weekly"]
    sp_w   = cd[sp_key]["weekly"]

    best_roas, best_spend, best_label = 0, None, "unknown"
    for i, (r, s) in enumerate(zip(roas_m, sp_m)):
        if r and r > best_roas:
            best_roas, best_spend = r, s
            best_label = mlabels[i] if i < len(mlabels) else f"M{i}"
    for i, (r, s) in enumerate(zip(roas_w, sp_w)):
        if r and r > best_roas:
            best_roas, best_spend = r, s
            best_label = wlabels[i] if i < len(wlabels) else f"W{i}"

    current_roas  = roas_m[2] if len(roas_m) > 2 else None
    current_spend = sp_m[2]   if len(sp_m)   > 2 else None
    rupee_impact  = (best_roas - current_roas) * current_spend \
                    if (best_roas and current_roas and current_spend) else None

    # ── CTR Decay Detection (R16) ─────────────────────────────────────────────
    # Check last 10 days for consecutive declining CTR (3-day smoothed)
    ctr_d10 = gv(cat, "CTR Perf excl AN", last10)
    an_d10  = gv(cat, an_key, last10)
    ctr_sm  = smooth3(ctr_d10)
    an_sm   = smooth3(an_d10)

    # Count consecutive declining days from the most recent end
    decay_run = 0
    for i in range(len(ctr_sm)-1, 0, -1):
        a, b = ctr_sm[i-1], ctr_sm[i]
        if a is not None and b is not None and b < a:
            decay_run += 1
        else:
            break

    # Current AN% trend over same window (is it already rising?)
    an_start = next((v for v in an_sm if v is not None), None)
    an_end   = next((v for v in reversed(an_sm) if v is not None), None)
    an_rising = (an_end > an_start * 1.1) if (an_start and an_end) else False

    ctr_decay_flag = None
    if decay_run >= 7:
        ctr_decay_flag = {
            "consecutive_days": decay_run,
            "an_already_rising": an_rising,
            "warning": "R16: CTR falling for 7+ consecutive days — AN% spike likely in next 3–10 days"
        }

    cd["_meta"] = {
        "frontier":     {"roas": best_roas, "spend": best_spend, "label": best_label},
        "current":      {"roas": current_roas, "spend": current_spend},
        "rupee_impact": rupee_impact,
        "ctr_decay":    ctr_decay_flag,
    }
    output["categories"][cat] = cd

print(json.dumps(output))
```

---

## Step 3 — Full Analysis: Apply Rules and Write Per-Category Narrative

**This step produces the actual report. Every category gets a complete written analysis — not just
data tables. Use the computed data from Step 2 and the rules from Step 1.**

### Compute for each category

For each category, derive the following before writing:

**Trend directions** (use second-half vs first-half average of each window, flag if >5% change):
- D/D 7-day: Revenue, Spends, ROAS, CVR, CTR excl AN, CPM, AN%, CPS
- W/W: latest week vs prior week % change for Revenue, Spends, ROAS
- MTD DRR: current month total ÷ days elapsed vs prior month total ÷ days in month
- M/M: direction across last 3 complete monthly columns

**ROAS signal classification** (always do this first):
- ROAS up + spend up → GENUINE SCALE (R1)
- ROAS up + spend down → EFFICIENCY CEILING (R3) — do not call this a scale signal
- ROAS up + spend flat → GENUINE SCALE, spend flat (R1)
- ROAS declining → apply R9 and R10

**Creative fatigue vs audience exhaustion** (whenever ROAS is declining):
- CPM stable or falling + CTR falling → CREATIVE FATIGUE → recommend creative refresh
- CPM rising + CTR flat or falling → AUDIENCE EXHAUSTION → recommend audience expansion or spend reduction
- State the diagnosis explicitly by name

**CTR decay check** (R16):
- Pull `_meta.ctr_decay` from computed data
- If decay_run ≥ 7: flag as "CTR falling [N] consecutive days — AN% spike likely incoming"
- If AN is already rising alongside: flag as "AN% already responding to CTR decay"
- State this as a proactive warning — act now during the CTR decay window, not after the AN spike

**Efficient frontier** (R13 — always state):
- Best ROAS, spend at that period, period label
- Current ROAS and spend
- ₹ impact = (best ROAS − current ROAS) × current monthly spend

**vs last run** (R15 — if last_run.json was loaded):
- Compare ROAS, Revenue, Spend, AN%, CTR to prior run values
- Flag anything moved >10%

---

### Report format — write this block for EVERY category

Overall first. Then all other categories ranked by ₹ impact (highest first). New launches last.

```
## [Category Name]   [₹ impact vs frontier if >0]

**Trend Summary:** [One sentence covering the dominant trend across all horizons]

**Efficient Frontier:** ROAS [X] @ [₹Y/month] ([period]) | Current: ROAS [X] @ [₹Y/month]

**Time Horizon Breakdown:**
- D/D (last 7 days): Revenue [↑/↓/~], Spends [↑/↓/~], ROAS [↑/↓/~] | CVR [dir], CTR [dir], CPM [dir], AN% [dir]
- W/W (latest vs prior week): Revenue [+X%], Spends [+X%], ROAS [+X%]
- MTD DRR: Revenue [₹X/day] vs [prior month] [₹X/day]  |  Spend DRR [₹X/day] vs [₹X/day]
- M/M (last 3 months): Revenue [dir and ₹ range], ROAS [dir and range]

**ROAS Signal:** [GENUINE SCALE / EFFICIENCY CEILING / DECLINING — state which and one reason]

**Diagnosis:** [CREATIVE FATIGUE / AUDIENCE EXHAUSTION / MIXED / N/A — state which and the metrics that led to this conclusion]

**CTR Decay Warning:** [If R16 triggered: "CTR falling X days — AN% spike likely in 3–10 days. Act now: refresh creative or exclude AN placements." Else: "None"]

**Rules Triggered:**
- R[X]: [one-line reason]
- R[Y]: [one-line reason]

**Suggested Action:** [Specific, actionable, never "pause". Reference the efficient frontier where relevant — e.g. "reduce spend toward ₹X/month, the frontier level."]

**Vs Last Run ([date] or "No prior run"):** [Key metric movements since last analysis. "ROAS: 1.08 → 1.26 (+17%)" style. Or "No prior run — next run will show deltas."]
```

For **new launches** (fewer than 20 days of spend data):
- Label as NEW LAUNCH
- Report direction of Revenue and ROAS only (improving / declining / mixed)
- State latest daily ROAS
- Skip all rules and suggested actions

---

## Step 4 — Save Snapshot

After writing the report, save a compact snapshot for next run's R15 comparison:

```python
import json, os
from datetime import date

last_run_path = os.path.expanduser("~/.claude/skills/meta-analysis/last_run.json")

# `output` is the full parsed data from Step 2
snapshot = {"run_date": date.today().isoformat(), "categories": {}}

for cat, cd in output["categories"].items():
    is_ov = (cat == "Overall")
    def m2(met):
        v = cd.get(met, {}).get("monthly", [])
        return v[2] if len(v) > 2 else None
    snapshot["categories"][cat] = {
        "roas":    m2("ROAS Perf"),
        "revenue": m2("Revenue"),
        "spend":   m2("Total Spends") if is_ov else m2("Perf Spends"),
        "an_pct":  m2("AN Percentage") if is_ov else m2("% of AN Sessions"),
        "ctr":     m2("CTR Perf excl AN"),
        "cvr":     m2("Perf CVR"),
    }

with open(last_run_path, "w") as f:
    json.dump(snapshot, f, indent=2)
```

---

## Step 5 — Overall Summary

Write this after all per-category sections.

Rank the top 3–5 findings by ₹ impact (from `_meta.rupee_impact`). State the ₹ figure for each.
Close with a single bolded line: **Most urgent action this week: [one action].**

---

## Arguments

If a category name or code is passed as the skill argument, only analyse that category + Overall.
Accepted: Overall, DSK, DFK, Makeup, Champi, 99, MRK, Haldi, Detan, Triple Tint.

---

## Default Rules

Used only when no rules doc is provided.

R1  — Scale signal: Spends up + ROAS up. Confirm spend is not falling (else R3).
R2  — Early warning: CTR excl AN or CVR declining even when ROAS holds → leading indicator.
R3  — Efficiency ceiling: ROAS up + spend down → not a scale signal. State efficient spend level.
R4  — Spend up, ROAS holding: compare funnel metrics to best historical period.
R5  — Revenue declining + spend up: diagnose with R10. Compare to efficient frontier.
R6  — Revenue declining + spend flat: check CPM, CPS, AN% for saturation.
R7  — Revenue holding + spend up: flag silent ROAS decline. Diagnose with R10.
R8  — Revenue holding + spend flat: assess scalability via CPM, CPS, AN%, CTR.
R9  — ROAS decline: compare to efficient frontier. Identify which funnel metric changed. Never pause.
R10 — Creative fatigue vs audience exhaustion: CPM down + CTR down = fatigue. CPM up + CTR flat/down = exhaustion.
R11 — AN% overall >20%: flag immediately.
R12 — AN% category: only flag if ROAS also worsened. If ROAS was better at high AN%, note as positive.
R13 — Efficient frontier: always compute and display for every category.
R14 — Volume-weighted summary: rank by ₹ impact = (best ROAS − current ROAS) × monthly spend.
R15 — Vs last run: compare to last_run.json if it exists.
R16 — CTR decay → AN% early warning: if CTR excl AN has been falling for 7+ consecutive days
      (3-day smoothed), flag it. Meta is likely to increase AN% in the next 3–10 days as a
      delivery relief valve. Act during the CTR decay window — refresh creative or proactively
      exclude AN placements — before the AN spike arrives. This pattern is confirmed across
      DSK, DFK, 99, Champi, Triple Tint, and Makeup. It does not reliably apply to new or
      small-scale campaigns still in Meta's learning phase.
