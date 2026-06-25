# MultiBank Dashboard — 2026-YTD KPI Verification

**Window:** 2026-01-01 → 2026-06-25 (queried as `>= '2026-01-01' AND < '2026-06-26'` so the current day is included)
**Database:** `analytics_warehouse`
**Verified marketing cost 2026:** $1,562,937 = paid $1,124,997 + adjust $46,602 + offline $391,338

> All queries use **literal dates**, not `SET @var` session variables. An earlier run returned a false `0` because session variables did not carry across query tabs. If you reuse `@cost` below, set and use it in the **same** session.

```sql
-- Reusable total 2026 marketing cost (set + use in the SAME tab)
SET @cost = (SELECT SUM(Cost)  FROM paidmarketing_2024          WHERE Date >= '2026-01-01' AND Date < '2026-06-26')
          + (SELECT SUM(Cost)  FROM adjust                      WHERE Date >= '2026-01-01' AND Date < '2026-06-26')
          + (SELECT SUM(spend) FROM stg_offline_marketing_spend WHERE date >= '2026-01-01' AND date < '2026-06-26');
-- @cost = 1,562,937
```

---

## Verification table

| # | KPI | Card | Verified 2026 | Status | SQL |
|---|-----|------|---------------|--------|-----|
| 1 | NMI | $69.7M | $71.96M | Publishable (matches, timing diff) | `SELECT SUM(NMI) FROM accounts_daily_2026 WHERE DateTime >= '2026-01-01' AND DateTime < '2026-06-26';` |
| 2 | Trading revenue | $54.87M (lifetime) | $39.61M | Publishable as 2026 | `SELECT SUM(total_usd_spreads+total_usd_commission+total_usd_swaps) FROM dm_bv_revenue WHERE report_date >= '2026-01-01' AND report_date < '2026-06-26';` |
| 3 | Account to Funded % | 19.71% | 29.73% | Publishable (2026 converts better) | `SELECT SUM(is_test=0 AND has_ftd=1)/NULLIF(SUM(is_test=0),0) FROM accounts_python WHERE account_creation_time >= '2026-01-01' AND account_creation_time < '2026-06-26';` |
| 4 | Funded accounts | 196,203 (lifetime) | 21,825 | Publishable as 2026 | `SELECT SUM(is_test=0 AND has_ftd=1) FROM accounts_python WHERE account_creation_time >= '2026-01-01' AND account_creation_time < '2026-06-26';` |
| 5 | Accounts opened | 995,393 (lifetime) | 73,407 | Publishable as 2026 | `SELECT COUNT(*) FROM accounts_python WHERE account_creation_time >= '2026-01-01' AND account_creation_time < '2026-06-26';` |
| 6 | CPA | $22.10 | $21.43 | Publishable (near-identical) | `SELECT @cost / NULLIF((SELECT SUM(is_test=0 AND Account_Type IN ('Individual Client','Introducing Broker')) FROM accounts_python WHERE account_creation_time >= '2026-01-01' AND account_creation_time < '2026-06-26'),0);` |
| 7 | CPFA | $111.90 | $71.61 | Publishable (2026 more efficient) | `SELECT @cost / NULLIF((SELECT SUM(is_test=0 AND has_ftd=1) FROM accounts_python WHERE account_creation_time >= '2026-01-01' AND account_creation_time < '2026-06-26'),0);` |
| 8 | Avg account size | $15,074 (lifetime) | $6,582 | Publishable as 2026 (clean grain) | `SELECT SUM(total_deposit_usd)/NULLIF(COUNT(*),0) FROM dm_profile WHERE ftd_dtm >= '2026-01-01' AND ftd_dtm < '2026-06-26';` |
| 9 | Deposits (funded users) | — | $114.06M | New clean figure | `SELECT SUM(total_deposit_usd) FROM dm_profile WHERE ftd_dtm >= '2026-01-01' AND ftd_dtm < '2026-06-26';` |
| 10 | Funded users (FTD 2026) | — | 17,329 | New clean figure | `SELECT COUNT(*) FROM dm_profile WHERE ftd_dtm >= '2026-01-01' AND ftd_dtm < '2026-06-26';` |
| 11 | CTR | 0.996% | 0.569% | CARD IS WRONG | `SELECT SUM(Clicks)/NULLIF(SUM(Impressions),0) FROM paidmarketing_2024 WHERE Date >= '2026-01-01' AND Date < '2026-06-26';` |
| 12 | CPC | $0.65 | $1.50 | CARD IS WRONG | `SELECT SUM(Cost)/NULLIF(SUM(Clicks),0) FROM paidmarketing_2024 WHERE Date >= '2026-01-01' AND Date < '2026-06-26';` |
| 13 | ROI | 149.9% | 2,434% | DO NOT PUBLISH RAW (see note) | `SELECT ((SELECT SUM(total_usd_spreads+total_usd_commission+total_usd_swaps) FROM dm_bv_revenue WHERE report_date >= '2026-01-01' AND report_date < '2026-06-26') - @cost) / NULLIF(@cost,0);` |
| 14 | LPV % | 14.85% | Not computable | BLOCKED — no date in stg_ga_csv | n/a |
| 15 | Bounce Rate | 62.45% | Not computable | BLOCKED — no date in stg_ga_csv | n/a |
| 16 | Real Visitors | target | No actual | Planning value only | n/a (proxy = sessions x bounce, but stg_ga_csv is undated) |
| 17 | Leads | target | No actual | Planning value only | Optional real actual: `SELECT COUNT(*) FROM leads_python WHERE creation_time >= '2026-01-01' AND creation_time < '2026-06-26';` |
| 18 | Live Accounts / CPL / Leads to Accounts | target | No actual | Planning values | Derive from accounts_python + leads_python once defined |

---

## Notes on the judgment-call rows

**ROI 2,434% — mathematically correct, strategically misleading.** $39.6M of 2026 revenue vs only $1.56M of 2026 spend. Most of that revenue comes from clients acquired in prior years who are still trading, so 2026's small ad budget is being credited with revenue earlier spend paid to acquire. Report as "2026 revenue vs 2026 spend — strong but not attributable to 2026 marketing," never as a headline 2,434%.

**CTR & CPC — genuine errors on the card.** 2026 reality: CTR 0.57% (card ~1%), CPC $1.50 (card $0.65). Paid efficiency is materially worse than the card shows. Fix the card.

## Data-quality issues & fixes

1. **Grain duplication (100x deposit error).** `accounts_python.total_deposits_usd` = $675,060/user vs `dm_profile.total_deposit_usd` = $6,582/user. accounts_python is per-account (users hold many), so it double-counts. FIX: source per-customer money metrics from `dm_profile` (user grain); reconcile to the `accounts_daily_2026` ledger.

2. **Undated GA feed.** `stg_ga_csv` has no date column — LPV % and Bounce can't be sliced to 2026. FIX: add a date/period column at ingestion; until then label these "all-time, undated."

3. **Misleading names / moving snapshots.** `paidmarketing_2024` holds 2026 data; `accounts_python` grew 864k→996k rows between pulls. FIX: rename period-neutral (`paidmarketing_all`); stamp snapshots with a load timestamp for reproducibility.

4. **Session-variable fragility.** `SET @start` returned a false 0. FIX: use literal dates (or a date-spine table) in production.

5. **Targets shown as actuals.** Real Visitors, Leads, Live Accounts, CPL, Leads to Accounts have no warehouse source. FIX: wire to real sources (leads_python, accounts_python, dm_profile) or label clearly as targets.
