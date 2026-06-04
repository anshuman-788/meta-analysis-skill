# meta-analysis — Meta Ads Category Analysis Skill

A Claude Code skill that reads your Meta Ads **Cat View** Google Sheet, applies your own rules
from a Google Doc, and produces a structured performance analysis across all categories.

## What it does

- Analyses every category in your Cat View sheet across **4 time horizons**: day-on-day (last 7
  days), week-on-week (last 4 weeks), MTD run rate, and month-on-month (last 3 months)
- Reads your rules from a Google Doc — your rules, your logic, applied automatically
- Flags trends, diagnoses funnel breakdowns (CTR, CVR, CPM, CPS, AN%), and suggests actions
- Never recommends pausing — always suggests what to improve

## Installation

```bash
npx skills add https://github.com/anshuman-788/meta-analysis-skill
```

Then in Claude Code:

```
/meta-analysis
```

On first run, the skill will ask you for:
1. Your Cat View Google Sheet URL
2. Your service account credentials file path
3. Your rules Google Doc URL (optional — built-in defaults are used if skipped)

## Required: Sheet Format

Your Google Sheet must have a tab named **Cat View** in this structure:

| Col | Content |
|-----|---------|
| A | Category name (filled only on first row of each category group) |
| B | Metric name |
| C–F | Monthly aggregates (e.g. Mar 2026, Apr 2026, May 2026, Jun 2026 MTD) |
| G–J | Weekly aggregates (e.g. W4May, W11May, W18May, W25May) |
| K+ | Daily data — one column per calendar day |

**Metrics expected per category** (one row each):
`Total Spends`, `Perf Spends`, `Revenue`, `Orders`, `ROAS Perf`, `Perf CVR`,
`CTR Perf excl AN`, `CPM Perf`, `CPS Perf excl AN`, `AN Percentage` / `% of AN Sessions`

An Overall category row should be present at the top with account-level totals.

## Required: Google Service Account

The service account needs **read access** to both the Sheet and the rules Doc.

1. Create a service account in Google Cloud Console
2. Download the JSON key file
3. Share your Sheet (and Doc) with the service account email as a Viewer

## Optional: Rules Google Doc

Write your analysis rules in plain English in a Google Doc and share it with the service account.
The skill reads the doc on every run and applies whatever rules it finds there.

If you skip this, the skill uses built-in default rules covering standard Meta Ads patterns
(scale signals, CTR/CVR decay, AN% thresholds, ROAS decline diagnosis).

## Usage

```
/meta-analysis              # analyse all categories
/meta-analysis DSK          # analyse one category + Overall
/meta-analysis Makeup       # same for any category name
```

## Reconfigure

To reset the configuration and re-run setup:

```bash
rm ~/.claude/skills/meta-analysis/config.json
```

Then run `/meta-analysis` again.

## License

MIT
