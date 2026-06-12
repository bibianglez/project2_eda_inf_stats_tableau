# Vanguard A/B Test – Project 2

Analysis of a Vanguard digital onboarding experiment comparing a new UI (Test) against the existing one (Control), across the full client funnel from `start` to `confirm`.

---

## Data sources

| File | Description |
|---|---|
| `df_final_web_data_pt_1/2.txt` | Raw clickstream logs (755,405 rows combined) |
| `df_final_experiment_clients.txt` | Experiment group assignment per `client_id` (`Test` / `Control`) |
| `df_alldata.csv` | Client-level enriched table: group, completion flag, errors, age, balance, gender, tenure |
| `funnel_by_step.csv` | Step-level reach counts and rates per group |
| `errors_by_transition.csv` | Transition-level error counts and rates per group |

---

## Pipeline – `Project2_final_clean.ipynb`

### 1. Load & concatenate
- `df_web_1` (343,141 rows) + `df_web_2` (412,264 rows) → `df_final_1_2` (755,405 rows)

### 2. Clean
- Column names → lowercase, spaces replaced with `_`
- `date_time` converted to `datetime`

### 3. Remove consecutive duplicate steps
- Within each `visit_id` (sorted by `date_time`), rows where `process_step` matches the immediately previous or next step are dropped
- 144,747 rows removed → 610,658 remaining

### 4. Merge with experiment clients → `df_merged_all`
- Inner join on `client_id` keeps only clients assigned to the experiment
- Result: **365,546 rows**, **67,734 unique clients**

### 5. Funnel analysis

| Metric | Value |
|---|---|
| Clients with more than 1 `visit_id` | 15,679 |
| Clients with full process (start → confirm) | 38,851 |
| Stopped at START | 5,461 |
| Stopped at STEP 1 | 4,689 |
| Stopped at STEP 2 | 3,983 |
| Stopped at STEP 3 | 9,008 |

**Completed process — Test vs Control:**

| Group | Completed | Did not complete |
|---|---|---|
| Test | 14,925 | 11,007 |
| Control | 12,771 | 9,755 |

### 6. Pivot table (`df_time`)
One row per `visit_id` with columns: `client_id`, `visitor_id`, `visit_id`, timestamp per step (`start` → `confirm`), and elapsed time between consecutive steps in seconds:

- `time_start_step1`
- `time_step1_step2`
- `time_step3_step3` *(step 2 → step 3)*
- `time_step3_confitm` *(step 3 → confirm)*
- `confirmation` (boolean)

### 7. Completion time insights (completers only)

**Mean time per step (seconds):**

| Transition | Control | Test |
|---|---|---|
| Start → Step 1 | 38.72 s | 25.58 s |
| Step 1 → Step 2 | 33.30 s | 35.59 s |
| Step 2 → Step 3 | 85.48 s | 79.81 s |
| Step 3 → Confirm | 127.74 s | 102.68 s |

Test clients complete the process faster overall, with the biggest time saving at Step 3 → Confirm.

**T-test results per step (Test vs Control):**

| Step | p-value | Significant? |
|---|---|---|
| Start → Step 1 | 9.7e-09 | Yes |
| Step 1 → Step 2 | 0.159 | No |
| Step 2 → Step 3 | 2.3e-05 | Yes |
| Step 3 → Confirm | 2.0e-43 | Yes |

---

## Statistical tests – `stats_claire.ipynb`

Runs independently from the clean CSVs. Sample sizes: **Control n = 22,934 | Test n = 25,388**.

### 1. Overall completion-rate z-test

| Group | Completion rate |
|---|---|
| Control | 55.56% |
| Test | 60.43% |

z = 10.836 · p ≈ 0.0000 → **statistically significant**. The new UI meaningfully increases the share of clients who finish the process.

### 2. Per-step reach-rate z-tests

| Step | Control rate | Test rate | z | p |
|---|---|---|---|---|
| step_1 | 79.09% | 86.64% | 22.10 | ~0 |
| step_2 | 70.34% | 76.82% | 16.16 | ~0 |
| step_3 | 64.49% | 70.28% | 13.57 | ~0 |
| confirm | 55.56% | 60.43% | 10.84 | ~0 |

Test reaches every step at a significantly higher rate.

### 3. Per-transition error-rate z-tests

An error is defined as a client going backwards in the funnel (regression pattern).

| Transition | Control rate | Test rate | z | p | Significant? |
|---|---|---|---|---|---|
| start → step_1 | 6.64% | 14.88% | 28.93 | 0.000 | Yes — Test has more errors here |
| step_1 → step_2 | 3.60% | 4.98% | 8.62 | 0.000 | Yes — Test has more errors here |
| step_2 → step_3 | 7.55% | 5.17% | -7.40 | 0.000 | Yes — Test has fewer errors here |
| step_3 → confirm | ~0% | ~0% | 0.95 | 0.342 | No difference |

### 4. Within-group chi-square (funnel homogeneity)

Tests whether drop-off is uniform across funnel steps within each group.

| Group | chi² | p | Result |
|---|---|---|---|
| Control | 1456.59 | ~0 | Drop-off is uneven across steps |
| Test | 347.43 | ~0 | Drop-off is uneven across steps |

Both groups show that some transitions lose disproportionately more users (not uniform drop-off).

### 5–7. Age-stratified tests (completion rate, error rate, mean error count)

Clients split into 5 age quintiles (17–31 / 31–42 / 42–53 / 53–62 / 62–96).

**Completion rate by age quintile:** Test outperforms Control in every age band (all p < 0.001). The largest lift is in the youngest group (+7.6 pp).

**Error rate by age quintile:** Test has higher error rates than Control in every age band (all p ≈ 0). Error rates increase with age for Test users, suggesting older clients struggle more with the new UI.

**Mean error count by age quintile:** Consistent with the above — Test clients make more errors on average in every age group.

### 8. Completion rate by balance quintile

Test outperforms Control across all balance quintiles (statistically significant). Lift is present regardless of client wealth.

### 9. Numeric correlation matrix

Notable correlations in `df_alldata`:
- `clnt_tenure_yr` ↔ `clnt_tenure_mnth`: r ≈ 1.00 (same variable, different unit)
- `calls_6_mnth` ↔ `logons_6_mnth`: r = 0.995 (highly collinear)
- `made_error` ↔ `n_errors`: r = 0.971
- `clnt_age` ↔ `completion_time`: r = 0.186 (older clients take longer)
- `completed` ↔ `clnt_age`: r = -0.092 (older clients complete less)

### 10. Gender effects

Overall, men complete at a higher rate than women (58.8% vs 55.7%, p < 0.0001).

Both genders benefit from the Test UI:

| Gender | Control | Test | Lift |
|---|---|---|---|
| Female | 53.3% | 57.9% | +4.6 pp |
| Male | 55.7% | 61.7% | +6.1 pp |

Both results are statistically significant (p < 0.0001). The Test UI improves completion across genders, with a slightly larger effect for men.

---

## Key takeaways

1. **The new UI works**: Test clients complete the process at a 4.9 pp higher rate (60.4% vs 55.6%), a highly significant result.
2. **Faster completion**: Test users complete the full funnel ~36 seconds faster (median), with the biggest saving at the final step.
3. **More early errors**: Test introduces more backward navigation at start → step_1 and step_1 → step_2, but less at step_2 → step_3. This may reflect users exploring the new interface before settling in.
4. **Universal improvement**: The Test UI outperforms Control across all age groups, balance levels, and genders — suggesting the improvement is robust, not driven by a single segment.
5. **Older users need attention**: Completion rates and error rates worsen with age, especially under the new UI. This could be an area for targeted UX improvements.


## Deliverables 
Trello - https://trello.com/b/iGfwxcrc/2nd-project


Slides - https://docs.google.com/presentation/d/18_78xmEKy1o-nXPmY-QG6UUnT9iPaaoeHN4B-LedHyM/edit?usp=sharing



Tableau - https://public.tableau.com/app/profile/claire.leyden.nalpas/viz/tableau_17811029025610/Findingsbycondition?publish=yes
