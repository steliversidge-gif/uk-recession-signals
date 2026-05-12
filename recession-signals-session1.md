# UK Recession Signals — Session 1 Plan

## Focus

Foundation. Get the project skeleton in place, pull the headline insolvency data, and produce your first time series visualisation of the data you'll be analysing for the next several months.

This session is about setting up properly. No analysis yet, no statistical methods. The point is to lay foundations that won't need ripping up in three months when you've added more data sources and want to refactor.

## What You'll Cover

### Part 1 — Project Skeleton (~30 mins)

**Create the folder structure:**

```
recession-signals/
├── data/
│   ├── raw/
│   │   ├── insolvency/
│   │   ├── macro/
│   │   └── behavioural/
│   ├── processed/
│   └── marts/
├── src/
│   ├── ingest/
│   ├── transform/
│   └── analysis/
├── notebooks/
├── outputs/
│   ├── charts/
│   └── tables/
├── dashboard/
└── README.md
```

You can do this in PowerShell or via VS Code's file explorer. PowerShell version:

```powershell
mkdir recession-signals
cd recession-signals
mkdir data\raw\insolvency, data\raw\macro, data\raw\behavioural, data\processed, data\marts
mkdir src\ingest, src\transform, src\analysis
mkdir notebooks, outputs\charts, outputs\tables, dashboard
```

**Initialise git:**

```
git init
```

**Create a comprehensive `.gitignore` immediately** — learn from the WoW project pain. Include:

```
# Python
__pycache__/
*.pyc
*.pyo
.venv/
venv/
env/

# Data — never commit raw or large processed files
data/raw/
*.csv.zip
*.xlsx
data/processed/*.csv
data/marts/*.csv

# Jupyter
.ipynb_checkpoints/

# IDE
.vscode/
.idea/

# Environment
.env
*.env

# OS
.DS_Store
Thumbs.db
```

The data exclusions matter. CSVs from ONS can run to tens of MB. The whole point of the consolidation pipeline is that anyone cloning the repo can rebuild the processed data from source — they don't need it pre-built.

**Create a placeholder README:**

```markdown
# UK Recession Signals

A leading indicator framework analysing UK insolvency patterns and 
discretionary consumption to identify pre-recessionary sector signals.

## Status

In active development. See workflow document for full project plan.

## Setup

[To be added once dependencies are stable]
```

**Initial commit:**

```
git config user.email "steliversidge@gmail.com"
git config user.name "Stephen Liversidge"
git add .
git commit -m "Initial project skeleton"
```

**Create a GitHub repo** called `uk-recession-signals` (public, no README, no .gitignore — you've added them locally), then:

```
git remote add origin https://github.com/steliversidge-gif/uk-recession-signals.git
git branch -M main
git push -u origin main
```

You've done all of this before with the WoW tracker. Should be familiar territory.

### Part 2 — Set Up Python Environment (~20 mins)

For this project you'll want a virtual environment. The WoW tracker installed packages globally; you got away with it because the project is small. With this one you'll be installing pandas, statsmodels, plotly, jupyter, scipy, and more — better to keep it isolated.

```
python -m venv .venv
.venv\Scripts\activate
```

You'll see `(.venv)` appear at the start of your prompt. That means you're inside the virtual environment.

Install initial dependencies:

```
pip install pandas matplotlib jupyter
```

Save dependencies to a requirements file:

```
pip freeze > requirements.txt
```

Anyone cloning the repo can recreate your environment with `pip install -r requirements.txt`. That's the standard Python project pattern.

**Important:** add this to `.gitignore` if not already there: `.venv/`. Virtual environments shouldn't be committed — they're recreated from `requirements.txt`.

Commit:

```
git add requirements.txt .gitignore
git commit -m "Add Python environment setup"
git push
```

### Part 3 — Pull the Long-Run Insolvency CSV (~30 mins)

Go to gov.uk's insolvency statistics page. Find the Long-Run Series CSV. Download it manually for now — you'll automate downloads later.

Save it as `data/raw/insolvency/long_run_series.csv`.

Also download the Metadata for Long-Run Series CSV. Save as `data/raw/insolvency/long_run_metadata.csv`.

**Read the metadata first.** This is non-negotiable. Government statistics are full of column names that look obvious but mean something specific. The metadata tells you what each column actually contains, what units it's in, and what the geographical coverage is.

### Part 4 — First Look in Jupyter (~45 mins)

Launch Jupyter:

```
jupyter notebook
```

This opens a browser tab. Create a new notebook in the `notebooks/` folder called `01_initial_inspection.ipynb`.

**Cell 1 — imports:**

```python
import pandas as pd
import matplotlib.pyplot as plt

# Show all columns, don't truncate
pd.set_option('display.max_columns', None)
pd.set_option('display.width', None)
```

**Cell 2 — load the data:**

```python
df = pd.read_csv('../data/raw/insolvency/long_run_series.csv')
df.head()
```

The `../` is because the notebook is one level deep in the `notebooks/` folder.

**Cell 3 — basic inspection:**

```python
print(f"Shape: {df.shape}")
print(f"\nDtypes:\n{df.dtypes}")
print(f"\nNulls per column:\n{df.isnull().sum()}")
```

**Cell 4 — date column handling:**

The date column will probably load as a string. Convert it:

```python
# Adjust column name based on what's actually in your data
df['date'] = pd.to_datetime(df['date_column_name'])
df = df.sort_values('date').reset_index(drop=True)
df.head()
```

**Cell 5 — first plot:**

```python
fig, ax = plt.subplots(figsize=(14, 6))
ax.plot(df['date'], df['total_company_insolvencies'])  # adjust column name
ax.set_title('UK Company Insolvencies — Long-Run Series')
ax.set_xlabel('Date')
ax.set_ylabel('Number of Insolvencies')
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

Look at the chart. You should be able to identify by eye:
- The early 1990s recession spike
- The late 2000s spike (2008 financial crisis aftermath)
- The Covid drop in 2020 (insolvency moratoriums and government support kept the numbers artificially low)
- The post-Covid recovery and rebound

**Cell 6 — annotations on the plot:**

Add vertical lines for known UK recession periods. Research these — UK recessions are dated by ONS, not NBER. Headline UK recessions in your data window:
- Early 1990s: roughly Q3 1990 to Q3 1991
- Great Recession: roughly Q2 2008 to Q2 2009
- Covid: Q1 2020 to Q2 2020 (technical recession, very brief but very deep)
- 2023: H2 2023 (very mild, two consecutive quarters of contraction)

```python
import matplotlib.dates as mdates

recession_periods = [
    ('1990-07-01', '1991-09-30', 'Early 1990s'),
    ('2008-04-01', '2009-06-30', 'Great Recession'),
    ('2020-01-01', '2020-06-30', 'Covid'),
    ('2023-07-01', '2023-12-31', '2023 mild'),
]

fig, ax = plt.subplots(figsize=(14, 6))
ax.plot(df['date'], df['total_company_insolvencies'])

for start, end, label in recession_periods:
    ax.axvspan(pd.to_datetime(start), pd.to_datetime(end), 
               alpha=0.2, color='red', label=label)

ax.set_title('UK Company Insolvencies with Recession Periods')
ax.set_xlabel('Date')
ax.set_ylabel('Number of Insolvencies')
ax.legend(loc='upper left')
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('../outputs/charts/insolvencies_with_recessions.png', dpi=120)
plt.show()
```

This is your first analytical artefact. Save it. Commit it.

### Part 5 — Document What You Learned (~15 mins)

In the notebook, add markdown cells documenting:
- What columns the dataset has
- What the time range is
- Anything weird you noticed (gaps, format changes, methodology revisions)
- Questions to investigate further

This is part of your portfolio. Hiring managers reading the repo can see how you think about data. Leave a clear trail.

**Commit and push:**

```
git add .
git commit -m "Initial inspection of long-run insolvency series"
git push
```

## What You'll Learn

- Setting up a Python project with virtual environments and dependency management
- Why `.gitignore` for data files matters (and how to avoid the WoW project history pain)
- Reading government statistics metadata before assuming column meanings
- Pandas date handling and time series plotting
- Recession dating conventions for the UK
- Notebook discipline — documenting findings as you go

## What's Coming Next

**Session 2 — Macro and Behavioural Indicators:** pull Bank of England base rate, ONS unemployment, ONS CPI, ONS retail sales by category, GfK consumer confidence. Begin building the upstream signals that the canary thesis depends on.

**Session 3 — Star Schema Build:** dim_date with recession phase markers, dim_sector with discretionary classification.

## Notes

- Don't try to do statistical analysis yet. The point is foundations.
- Spend real time looking at the data and the metadata. The questions you can ask later depend on understanding what the data is and isn't.
- Commit early and often. A messy commit history showing real work is better than a polished single commit.
- The virtual environment is non-negotiable for this project. With the variety of statistical packages you'll be installing, isolation matters.
