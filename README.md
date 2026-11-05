# HR Data Warehouse — Student Project

Take raw exports from a (fake) HR system, clean them, load them into a star-schema data warehouse, and answer business questions as a data analyst.

Python 3 and SQLite only, so there's nothing to install. The data is seeded, so every student gets identical numbers.

```bash
python3 1_generate_source.py               # HR system export  -> source/*.csv
python3 2_etl.py                           # CSV -> stg_* -> star schema in hr_warehouse.db
sqlite3 -box hr_warehouse.db < 3_analysis.sql   # the 10 questions (output saved in answers.txt)
```

The dataset's "today" is **2026-06-30**. It covers 400 employees (current staff and leavers) hired from 2018 on, across 6 departments in Amman, Riyadh, Dubai and Cairo.

## 1. Source system (`source/`)

| File | Rows | Grain |
|---|---|---|
| departments.csv | 6 | department |
| jobs.csv | 5 | job title with salary band |
| employees.csv | 400 | employee (current state) |
| salaries.csv | ~1.6k | salary period (from/to) |
| performance_reviews.csv | ~1.2k | employee × year-end review (1–5) |
| attendance.csv | ~59k | employee × working day in 2025 |

**Dirty data on purpose**, so the ETL has something to clean:
- gender coded as `M`, `Male`, `male`, ` m`, `F`, …
- leading and trailing spaces in names
- some emails in UPPERCASE
- dates in `dd/mm/yyyy` in employees.csv (ISO everywhere else)
- empty strings instead of NULL (termination, manager)
- about 0.5% duplicate attendance rows from a "flaky sync job"

## 2. Warehouse (`transform.sql`)

The ETL first loads every CSV as-is into an all-TEXT `stg_*` table. `transform.sql` then builds the star schema:

```
                 dim_date
                     │
dim_department ── fact_salary ────── dim_employee
      │          fact_performance        │
      └───────── fact_attendance ────────┘
                     │
                  dim_job
```

| Table | Grain / notes |
|---|---|
| `dim_date` | one row per day 2018–2026, key `yyyymmdd`, Fri/Sat weekend |
| `dim_department`, `dim_job` | surrogate key + business key |
| `dim_employee` | SCD type 1; cleaned gender/name/email/dates, plus derived `age_band`, `tenure_years`, `is_active`, `manager_name` |
| `fact_salary` | one row per salary period, `is_current` flag |
| `fact_performance` | one row per employee per review year |
| `fact_attendance` | one row per employee per working day, deduplicated, with `is_unplanned_absence` |

`2_etl.py` ends with sanity checks: no uncleaned gender values, no duplicate attendance rows, and no salary rows lost.

## 3. Questions & findings

| # | Question | Finding |
|---|---|---|
| Q1 | Headcount & avg salary per dept | Engineering is the largest team (62) and the best paid (~$3.4k a month). Customer Support is the lowest paid (~$2.1k). |
| Q2 | Yearly attrition rate | Attrition settled at **11–15%** from 2021 on, after 21–27% in the small early years. |
| Q3 | Who loses the most people, and why | **Customer Support has lost 71% of everyone it ever hired**, and Sales 50%. "Better Offer" is the top reason, which points to pay. |
| Q4 | Gender pay gap, same job title | The gap grows with seniority: about 0% for Junior, **13% for Team Lead** and 7% for Manager. |
| Q5 | Top-paid non-manager per dept | The top earner is a Team Lead in most departments. In Sales it's a Senior Associate. |
| Q6 | Does rating drive raises | Yes, and the policy is followed exactly: rating 1 → 0%, 3 → 3%, 5 → 10%. |
| Q7 | Unplanned absence 2025 by quarter | Customer Support has the most unplanned absence (7.3%), then Sales (6.0%). The other departments are around 4–4.6%. |
| Q8 | Do low performers miss more work | Yes. Employees rated 1–2 are unexcused-absent **about 3× more** (4.5% vs 1.5%). |
| Q9 | Paid above the job-band max | 40 active employees are over their band. The bands are stale, or these people are due for promotion. |
| Q10 | Tenure of leavers by reason | Layoffs hit new joiners (21 of 40 within their first year). Performance exits happen later (about 3 years). |

**Story for the final presentation:** Customer Support is low paid, has high absence and loses people to better offers. A pay-band review there, and for women in senior roles, is the clearest action.

## Exercises for students
1. Add a `fact_headcount_monthly` snapshot table, then rewrite Q2 against it.
2. Port `transform.sql` to PostgreSQL or SQL Server and load it with an ETL tool (SSIS, Airflow, dbt).
3. Build a Power BI or Tableau dashboard on the star schema.
4. Ask and answer two new questions of your own.
