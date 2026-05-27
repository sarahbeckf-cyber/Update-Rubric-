# UCSF Health — Medical Leadership Sizing Rubric (Draft)

A single-file, static web app for scoring ambulatory medical leadership roles, built for the UCSF Health Ambulatory Physician Leadership Harmonization workgroup.

This version incorporates the feedback John Fangman gave in the workgroup meeting on the rubric prototype:

- **Academic Department** field added above the cost-center picker; CC list filters by department and is grouped by division
- **Visit volume**, **Clinicians supervised**, and **wRVUs** are **auto-populated from FY25 actuals** when cost centers are selected (no more manual toggling)
- **Patient complexity** simplified to a **3-point scale** (Low / Moderate / High)
- **Strategic growth priority** renamed to **Clinical Enterprise Plan priority** with simplified options (Parity / Aligned / Identified CEP priority)
- Visit volume, clinicians supervised, and wRVUs are weighted ×1.5 as the three primary drivers
- **Tier 1 renamed to "ACMO scope"** with tier-subtitle text removed
- Cost-center list lives in a scrollable container with sticky group headers (fixes the "can't scroll all the way down" issue)

## Data sources

| Field | Source | Vintage |
| --- | --- | --- |
| Cost center → AEMD / Med Dir / Division | `Physician Leader Cost Center Map Master 16.xlsx` | Current master |
| Visit volume | `FY25andFY26_Appt_Count_by_Cost_Center.xlsx` | **FY25 actuals** |
| Clinician count (unique billing providers) | `wRVUs_and_MD_Count_by_Cost_Center.xlsx`, "MD Count" sheet | **FY25 actuals** |
| wRVUs (Nat_Work_RVU) | `wRVUs_and_MD_Count_by_Cost_Center.xlsx`, "wRVUs" sheet | **FY25 actuals** |

> **Why FY25 and not FY26?** Per the dataset note, the FY26 totals in the source file are **not annualized**, so partial-year data would skew the scoring. Switch to FY26 once year-end actuals are available.
>
> **Note on clinician count:** This is a unique-billing-provider headcount (anyone with any activity in the cost center during FY25), not cFTE. Mark flagged in the meeting that a clean cFTE-by-cost-center pull is not currently available; allocation methods give directionally-accurate but imperfect numbers.

## Coverage

- **377 ambulatory cost centers** loaded (Ambulatory BU, Active/Inactive, excluding rows marked for exclusion)
- **268 CCs** have FY25 visit data
- **311 CCs** have FY25 clinician-count data
- **311 CCs** have FY25 wRVU data
- The remaining CCs (admin / overhead / closed / no billing activity) show "—" for the auto-populated metrics; the rubric still works, you'd just have to score the primary drivers manually

## Scoring math

Nine metrics, each scored 1–4 (Patient complexity uses 1 / 2 / 4 because of the 3-point scale; CEP priority uses 1 / 2 / 4 similarly):

- **Primary drivers (×1.5):** Visit volume, Clinicians supervised, wRVUs
- **Standard drivers (×1.0):** Subspecialty complexity, Patient complexity, Number of sites, Practice setting, Adult/Peds mix, CEP priority

**Maximum weighted score:** `3 × (4×1.5) + 6 × (4×1.0) = 18 + 24 = 42`

**Tier bands** (editable in the `TIERS` constant near the top of the JS block):

| Score | Tier | Suggested cFTE |
| --- | --- | --- |
| 0–13 | Tier 4 · Focused MD | 0.10 – 0.20 |
| 14–22 | Tier 3 · Medical Director | 0.20 – 0.40 |
| 23–30 | Tier 2 · AEMD candidate | 0.40 – 0.60 |
| 31–42 | Tier 1 · ACMO scope | 0.60 – 1.00 |

These thresholds are placeholders. The workgroup should calibrate by scoring a handful of known AEMDs (Nina Botto for Derm, Mary Martin for Primary Care, etc.) and adjusting the bands so existing roles land in the expected tier.

**wRVU bands** (per role, summed across selected CCs):

| FY25 wRVUs | Score |
| --- | --- |
| < 10,000 | 1 |
| 10,000 – 30,000 | 2 |
| 30,000 – 75,000 | 3 |
| > 75,000 | 4 |

## Quick start

1. Clone this repo
2. Open `index.html` in any modern browser — that's it
3. To deploy to GitHub Pages: Settings → Pages → Source: `main` / root → live URL in ~30 seconds

No build step, no dependencies, no backend. Everything (including the embedded cost-center data with FY25 actuals) lives in one HTML file.

## Refreshing the data

The 377 cost centers and their FY25 numbers are embedded directly in `index.html` in the `COST_CENTERS` array. To rebuild from updated source files:

```python
import pandas as pd, json

# FY25 visits
appt = pd.read_excel('FY25andFY26_Appt_Count_by_Cost_Center.xlsx')
appt = appt[appt['COST_CENTER_NUM'] != 0]
appt['CCnum'] = appt['COST_CENTER_NUM'].astype(str)
fy25_v = appt[appt['FiscalYear']=='FY25'].set_index('CCnum')['EncounterCount'].to_dict()

# FY25 MD count
md = pd.read_excel('wRVUs_and_MD_Count_by_Cost_Center.xlsx',
                   sheet_name='MD Count', skiprows=2)
md.columns = ['cc','FY25','FY26']
md = md[md['cc'].astype(str).str.match(r'^\d+$')]
md['cc'] = md['cc'].astype(int).astype(str)
fy25_md = md.set_index('cc')['FY25'].to_dict()

# Master map (cascade IDs are 6-digit; data IDs are 7-digit = master + "0")
master = pd.read_excel('Physician Leader Cost Center Map Master 16.xlsx',
                       sheet_name='Physician Leader Cost Center Ma')
amb = master[(master['Primary Business Unit']=='Ambulatory') &
             (master['Exclude from counts'].isna()) &
             (master['Status'].isin(['A','I']))]

records = []
for _, r in amb.iterrows():
    cc6 = str(int(r['DeptID/Cost Center']))
    cc7 = cc6 + '0'
    records.append({
        'i': cc6,
        'n': r['Cost Center Name'],
        'div': r['Executive Division Descr'] or 'Unassigned',
        'aemd': str(r.get('Exec Med Dir Name','') or ''),
        'md':   str(r.get('Med Dir Name','') or ''),
        'v': int(fy25_v.get(cc7)) if pd.notna(fy25_v.get(cc7)) else None,
        'p': int(round(fy25_md.get(cc7))) if pd.notna(fy25_md.get(cc7)) else None,
    })
print(json.dumps(records, separators=(',',':')))
```

Paste the output as the value of `const COST_CENTERS = [...]` in `index.html`.

## Privacy

Everything runs client-side. Saved scores live only in the user's browser (`localStorage`, key `ucsf_rubric_saved_v2`) and never leave the device unless explicitly exported via "Export all to CSV."

## File structure

```
/
├── index.html      ← entire app: HTML + CSS + JS + 377 cost centers with FY25 actuals
├── README.md       ← this file
└── .gitignore
```
