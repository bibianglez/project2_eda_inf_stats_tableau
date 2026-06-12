# Project 2 – EDA, Inferential Statistics & Tableau

## Overview

This project analyses a **digital onboarding funnel experiment** (A/B test) run by a financial services company. The goal is to understand how users navigate through a multi-step online process and whether a new UI design (Test group) leads to faster or higher completion rates compared to the original (Control group).

The analysis covers data ingestion and cleaning, exploratory data analysis (EDA), funnel drop-off analysis, and inferential statistics (t-tests) to compare the two experiment groups.

---

## Dataset

Three raw files are loaded directly from GitHub:

| File | Description |
|---|---|
| `df_final_experiment_clients.txt` | Client metadata including group assignment (`Test` / `Control`) |
| `df_final_web_data_pt_1.txt` | Web session data — part 1 |
| `df_final_web_data_pt_2.txt` | Web session data — part 2 |

The web data files are concatenated into a single dataframe before processing.

---

## Notebook Structure

### 1. Import, Load and Concatenate Web Data
- Loads all three datasets from raw URLs.
- Concatenates the two web data files into one unified dataframe.

### 2. Clean `df_final_1_2`
- Standardises column names (lowercase, underscores).
- Converts `date_time` to datetime format.

### 3. Remove Consecutive Duplicate Steps
- Sorts events by `visit_id` and `date_time`.
- Removes rows where a user recorded the same `process_step` consecutively within a session, avoiding distorted time calculations.

### 4. Merge with Client Data
- Inner joins the web session data with the client experiment file on `client_id`.
- Only clients present in both datasets are kept.

### 5. Funnel Analysis
The funnel consists of five steps: `start → step_1 → step_2 → step_3 → confirm`.

| Section | What it answers |
|---|---|
| 5.1 | How many clients had more than one visit? |
| 5.2 | How many clients completed the full process? |
| 5.3 | Pivot table with one timestamp per step per visit |
| 5.3 | Time elapsed between each consecutive step |
| 5.3 | Clients who stopped at `start` |
| 5.4–5.6 | Clients who stopped at `step_1`, `step_2`, `step_3` + mean/median time spent at each |
| 5.7 | Completed clients broken down by Test vs Control group |

### 6. Completion Time Insights: Test vs Control
- Compares mean time spent at each step between the Control and Test groups.

### 7. Inferential Statistics
- Runs **Welch's t-tests** (unequal variance) for each step transition to determine whether time differences between groups are statistically significant.
- Produces a summary table exported as `df_times_report.csv` with columns: step, mean (Control), mean (Test), difference, p-value, and significance flag (α = 5%).

---

## Output

| File | Description |
|---|---|
| `df_times_report.csv` | Summary table of mean times and t-test results per step |

---

## Libraries Used

- `pandas` – data manipulation
- `numpy` – numerical operations
- `scipy.stats` – statistical testing (Welch's t-test)

---

## How to Run

1. Clone or download the repository.
2. Open `Project2_final_clean.ipynb` in Jupyter Notebook or JupyterLab.
3. Run all cells in order. No local data files are needed — datasets are fetched directly from GitHub.

```bash
pip install pandas numpy scipy
jupyter notebook Project2_final_clean.ipynb
```

---

## Key Findings

- The notebook identifies how many users drop off at each funnel step and how long they spend before abandoning.
- Statistical tests determine whether the Test UI meaningfully changes the time users spend between steps.
- Results are flagged as significant or not at the 5% significance level (p < 0.05).
