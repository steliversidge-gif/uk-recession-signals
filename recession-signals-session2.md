# UK Recession Signals — Session 2 Plan

## Focus

Build the upstream data sources. The canary thesis depends on having clean macro and behavioural indicators alongside the insolvency data. This session is about ingestion discipline — building scripts that turn messy government data into clean tidy CSVs ready for the dimensional model.

This is the unglamorous middle of any data project. Most data work is cleaning, not analysis. Get good at this and the rest of the project moves quickly.

## What You'll Cover

### Part 1 — Establish the Ingestion Pattern (~30 mins)

Before pulling any new source, decide on the convention every ingest script will follow. Consistency matters more than cleverness.

**Standard ingest script structure:**

```python
"""
Ingest [data source name].

Source: [URL]
Frequency: [monthly / quarterly / etc]
Coverage: [time range, geography]
"""

import pandas as pd
from pathlib import Path

RAW_PATH = Path('data/raw/[category]/[filename]')
OUTPUT_PATH = Path('data/processed/[clean_filename].csv')


def load_raw():
    """Load the raw downloaded file."""
    df = pd.read_excel(RAW_PATH, sheet_name='...', skiprows=...)
    return df


def clean(df):
    """Normalise column names, fix dtypes, validate."""
    # Rename columns to snake_case
    # Convert date column
    # Convert numeric columns
    # Drop or fill nulls per source-specific rules
    return df


def validate(df):
    """Check the cleaned data meets expectations."""
    assert df['date'].is_monotonic_increasing, "Dates not sorted"
    assert df['value'].notna().all(), "Nulls in value column"
    # Source-specific checks
    return df


def main():
    df = load_raw()
    df = clean(df)
    df = validate(df)
    df.to_csv(OUTPUT_PATH, index=False)
    print(f"Wrote {len(df)} rows to {OUTPUT_PATH}")


if __name__ == '__main__':
    main()
```

Every ingest script follows this shape. Load, clean, validate, write. Each step is a function so you can test it in isolation.

The output of every ingest script is a tidy two-column CSV: `date, value`. Or for multi-series sources: `date, series_name, value`. Long format, not wide. Your dimensional model can pivot later if needed.

### Part 2 — Bank of England Base Rate (~30 mins)

Easiest source first. BoE publishes the official bank rate as a clean CSV.

**Source:** https://www.bankofengland.co.uk/boeapps/database/Bank-Rate.asp

Download the CSV. Save as `data/raw/macro/boe_base_rate.csv`.

Look at it first. The format may be unusual — BoE provides date and rate columns but the date format and frequency need handling. The rate changes occasionally, so the CSV has one row per change, not one row per month.

For your purposes you need monthly observations. Logic:

1. Take the date-of-change data
2. Resample to month-end frequency, forward-filling the rate (the rate stays at its last value until changed)
3. Save as `data/processed/boe_base_rate.csv` with columns `date, base_rate`

**Apply:** create `src/ingest/boe_base_rate.py` following the standard pattern above. Run it. Verify the output.

**Concepts that come up:**
- Forward-fill resampling
- Why irregular event data needs frequency conversion before joining time series
- The first thing every macro time series teaches you: dates aren't simple

### Part 3 — ONS CPI Inflation (~30 mins)

ONS data is messier than BoE data. CPI is published monthly via the ONS website, available as Excel files with multi-row headers, merged cells, and footer notes.

**Source:** ONS Consumer Price Inflation. The headline series is CPIH or CPI (you'll want both for completeness — CPIH includes owner-occupier housing costs, CPI is the older measure).

**Apply:** download the CPI Excel file. Save as `data/raw/macro/ons_cpi.xlsx`. Inspect it in Excel first to understand the layout — which sheet, which rows are headers, which cells are data.

Then in `src/ingest/ons_cpi.py`:

```python
df = pd.read_excel(
    RAW_PATH, 
    sheet_name='1.1',  # adjust based on actual sheet
    skiprows=8,        # adjust based on header rows
    usecols='A:C'      # adjust based on which columns you need
)
```

You'll iterate on `skiprows` and `usecols` until the dataframe loads cleanly. This is normal. ONS doesn't publish for analytical convenience.

The CPI series is a year-on-year percentage change. Document this clearly in your script. CPI inflation rate ≠ CPI index level.

### Part 4 — ONS Unemployment (~30 mins)

Same source pattern as CPI. ONS publishes unemployment monthly.

**Source:** ONS Labour Market Statistics. The headline series is the unemployment rate (16+, ILO measure, seasonally adjusted).

**Apply:** create `src/ingest/ons_unemployment.py`. Output is monthly date and unemployment rate as a percentage.

By this point you'll be familiar with ONS Excel quirks. Each new source teaches you something — different sheet layouts, different missing-value conventions, different header structures.

### Part 5 — ONS Retail Sales by Category (~45 mins)

This is the canary-specific work. Generic retail sales is not what you want — you want the breakdown by category so you can construct a discretionary spending index.

**Source:** ONS Retail Sales Index (RSI). The breakdown by category includes:
- Predominantly food stores
- Predominantly non-food stores
  - Non-specialised stores (department stores)
  - Textile, clothing, footwear stores
  - Household goods stores
  - Other non-food stores
- Non-store retailing (online and mail order)
- Automotive fuel

Each is published as an index value (rebased periodically) and as a year-on-year percentage change.

**Apply:** download the categorical breakdown. This is typically in a different table than the headline retail sales series. Save as `data/raw/behavioural/ons_retail_sales_by_category.xlsx`.

In `src/ingest/ons_retail_sales.py`:
- Load each category as a separate series
- Output in long format: `date, category, value`
- Save as `data/processed/ons_retail_sales_by_category.csv`

This will be the input to your discretionary spending index in a later session. For now just get clean per-category data.

**Concepts:**
- Index series vs growth series
- Why categorical data matters for leading indicator analysis
- Long-format vs wide-format data

### Part 6 — GfK Consumer Confidence (~30 mins)

The flagship behavioural indicator. GfK has published the UK Consumer Confidence Index monthly since 1974. It's the canonical pre-recessionary canary.

**Source:** GfK provides historical data. Some is available freely; some requires registration. ONS also republishes consumer confidence as part of their economic indicators series, which may be easier.

**Apply:** find a clean source of monthly consumer confidence data going back as far as possible. Save the raw file. Build `src/ingest/gfk_consumer_confidence.py`.

Output: `date, consumer_confidence_index`.

The index is a balance score — values above zero indicate net positive sentiment, values below zero indicate net negative sentiment. It moves between roughly +20 and -50 historically. Document this in your script comments.

### Part 7 — Test the Full Ingestion Pipeline (~30 mins)

Now you have five ingest scripts. Run them all from a master script:

```python
# src/ingest/run_all.py
import subprocess
import sys

scripts = [
    'src/ingest/boe_base_rate.py',
    'src/ingest/ons_cpi.py',
    'src/ingest/ons_unemployment.py',
    'src/ingest/ons_retail_sales.py',
    'src/ingest/gfk_consumer_confidence.py',
]

for script in scripts:
    print(f"Running {script}...")
    result = subprocess.run([sys.executable, script])
    if result.returncode != 0:
        print(f"FAILED: {script}")
        sys.exit(1)
    print(f"OK: {script}")

print("All ingestion complete.")
```

Run it:

```
python src/ingest/run_all.py
```

You should see five OKs. Check `data/processed/` for the output files.

**Quick exploration in a notebook:**

Create `notebooks/02_macro_inspection.ipynb`. Load each processed file. Plot each on its own chart. Make sure the data looks sensible:
- Base rate: was 5%+ in early 1990s, near zero from 2009-2021, rising again from 2022
- CPI inflation: roughly 2-3% normally, spiked to 11% in 2022-2023
- Unemployment: was 10%+ in early 1990s, around 4% in late 2010s, rose during Covid then fell
- Retail sales: long-running upward trend with sector-specific cycles
- Consumer confidence: cyclical, dropped sharply in 2008 and again in 2022

If any of these look wrong, your ingest script has a bug. Better to find it now than later when you're trying to fit models.

### Part 8 — Document and Commit (~15 mins)

In the README, add a section about data sources. Each source should be documented:
- Where it comes from (URL)
- Update frequency
- License/attribution requirements
- Any quirks worth knowing

Government statistics often have specific attribution requirements (e.g., "Contains public sector information licensed under the Open Government Licence v3.0"). Add the appropriate attribution in your README.

**Commit and push:**

```
git add .
git commit -m "Add ingestion scripts for macro and behavioural indicators"
git push
```

## What You'll Learn

- Building reusable ingestion scripts following a consistent pattern
- Handling messy government data: ONS Excel files, BoE event-driven data, frequency conversion
- The difference between index levels, percentage changes, and balance scores — and why mixing them up breaks analysis
- Categorical retail sales data as the foundation of discretionary spending analysis
- Pipeline orchestration with a runner script
- Validation as a first-class part of ingestion, not an afterthought

## What's Coming Next

**Session 3 — Star Schema Build:** dim_date with recession phase markers, dim_sector with discretionary classification, dim_macro joining all your cleaned macro series onto the date spine.

**Session 4 — Fact Table and Mart Layer:** transform the insolvency record-level data into the fact table at proper grain. Build the four marts including the leading signal mart.

## Notes

- Government data files change format occasionally. Your ingest scripts will break and need fixing. That's expected. The discipline is fixing the script rather than hand-editing the output.
- The validation step in each script catches issues at the right time — at the source, not three steps downstream when something else fails for a confusing reason.
- ONS data takes longer than you'd think on first contact. Budget more time than feels reasonable for the retail sales work specifically.
- Don't skip the notebook inspection step. Eyeballing each series is how you catch ingestion bugs before they propagate.
- Your output CSVs are in `data/processed/` and excluded from git by your `.gitignore`. That's correct — anyone cloning the repo runs your ingestion scripts to recreate them.
