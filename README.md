# Where do online shoppers drop out, and what would I test to fix it?

Product analytics on Google's GA4 sample data (Google Merchandise Store, Nov 2020 to Jan 2021), built with SQL, dbt, BigQuery and Looker Studio (formerly Data Studio).

**[Live dashboard](https://datastudio.google.com/s/ijUBLawOYaM)** · [Experiment plan](experiment_plan.md) · [Exploration notes](notes.md)

## Key findings (1 Nov 2020 to 31 Jan 2021)

1. **The biggest leak is the first step.** Only 19.7% of sessions that viewed a product added it to the cart (15,173 of 77,020). Every later step converts at 39% to 74%.
2. **Device is not the problem.** Full-funnel conversion is 0.82% on mobile, 0.77% on tablet and 0.77% on desktop.
3. **Few new users come back.** Of new users in each November weekly cohort, only 4.3% to 5.7% returned the following week, and under 2% by week 4.

Overall: 360,129 sessions, 4,848 orders, 1.35% conversion, $362,165 revenue.

## Recommendation

Test showing delivery cost and expected delivery date next to the Add to cart button.

- Primary metric: add-to-cart rate (baseline 19.7%)
- Guardrails: overall conversion, revenue per session, checkout-to-purchase rate
- About 6,600 sessions per group to detect a 10% relative lift (5% significance, 80% power), so 3 full weeks at current traffic

Full design, decision rule and risks: [experiment_plan.md](experiment_plan.md)

## Funnel

| Step | Sessions | Step conversion |
|---|---|---|
| 1 Viewed item | 77,020 | |
| 2 Added to cart | 15,173 | 19.7% |
| 3 Began checkout | 5,959 | 39.3% |
| 4 Payment info | 3,858 | 64.7% |
| 5 Purchased | 2,848 | 73.8% |

![Dashboard](images/dashboard.png)

## How I built it

Raw GA4 events → staging → sessions → marts → dashboard, all in dbt on BigQuery.

| Model | One row per | Purpose |
|---|---|---|
| `stg_ga4__events` | event | Clean names, real dates, session ID pulled from nested event_params |
| `int_sessions` | session | 1/0 flag for each funnel step, revenue |
| `dim_users` | user | First visit, cohort week, new-user flag |
| `fct_daily_kpis` | day | Sessions, orders, conversion, revenue |
| `fct_funnel` / `fct_funnel_steps` | day × device × channel | Funnel counts, and the same in long format for charts |
| `fct_retention_weekly` | cohort × week | Weekly retention of new users |

Tests: `unique` and `not_null` on session, user and date keys.

![Lineage](images/lineage.png)

Design choices:
- A session is `user_pseudo_id` + `ga_session_id`, as session IDs alone repeat across users.
- The funnel counts a session at a step only if it also reached every earlier step, so the funnel always narrows. Step order within the session is not checked.
- Conversion on the dashboard is total orders divided by total sessions, not an average of daily rates.
- The device chart shows a different rate: sessions that completed every funnel step divided by all sessions (2,848 of 360,129, about 0.8%). Both rates are correct; they answer different questions.

## How to run

1. Clone the repo and install dbt: `pip install dbt-bigquery`
2. Set up a BigQuery profile in `~/.dbt/profiles.yml` pointing at your own GCP project. Source data: `bigquery-public-data.ga4_obfuscated_sample_ecommerce`
3. Run `dbt build` to build all models and run the tests
4. Run `dbt docs generate` then `dbt docs serve` to view the lineage graph

## Checks and limitations

- **Reconciliation:** total sessions match across `fct_daily_kpis` and `fct_funnel` (360,129).
- **Funnel coverage:** the strict funnel captures 2,848 of the 4,848 orders (it misses 41.3%), because some purchases have no earlier funnel events in this dataset. Funnel rates describe sessions that went through every step.
- **Obfuscated data:** Google scrambled parts of this sample, so some values are missing or masked.
- **First-touch channel:** `traffic_source` is the channel that first brought the user, not the channel of each visit.
- **Right-censoring:** later cohorts have fewer weeks of retention data because the data ends on 31 Jan 2021.
- **Season:** the period includes the Christmas peak, so results may not hold year-round.
- **Transferability:** the same staging → sessions → marts pattern applies to any business's event data (GA4, ads or CRM), which is where most growth questions start.
