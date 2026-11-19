# FinGraph AI: Build Order and Status

Rules for this document
- Status values: **Not started / In progress / Done (user-confirmed) / Verified**.
- Nothing is marked Done or Verified unless the team confirmed it.
- Update this file at the end of each work session and keep it in the repo.

Last updated: 2026-11-19

---

## 1. Build order

### Phase A: Data foundation (everything depends on this)

| Step | Task | Output | Depends on | Status |
|---|---|---|---|---|
| A0 | Profile IBM HI-Small (rows, labels, patterns, accounts file) | Profiling numbers | none | Done (user ran the scripts) |
| A1 | Schema v1 (`01_schema_v1.sql`) | Empty tables in all schemas | A0 | Done (user ran it; not yet tested with data) |
| A2 | **IBM loader script**: files to `core` + `truth` | Loaded IBM data | A1 | Not started |
| A3 | Run `02_post_load.sql` (FKs, indexes, checks) | Validation output matches expected numbers | A2 | Not started |
| A4 | `pg_dump` backup of the loaded database | Backup file shared with team | A3 | Not started |
| A5 | **Synthetic overlay**: account profile, devices, IPs | `syn` tables filled | A3 | Not started |
| A6 | **Injected scenarios**: shared identity, shell company, dormant abuse, smurfing | Rows with `source='synthetic'` in `core` and `truth` | A3, A5 | Not started |
| A7 | Overlay leakage check (pattern vs normal accounts look alike on overlay columns) | Check results | A5, A6 | Not started |
| A8 | Freeze the dataset (final `pg_dump`, version tag) | Frozen hybrid dataset | A7 | Not started |

### Phase B: Work that can start after A4 (parallel)

| Step | Task | Needs | Status |
|---|---|---|---|
| B1 | Repo, Git workflow, `.env` handling, Docker for local Postgres/Neo4j | none (can start now) | Not started |
| B2 | Neo4j schema + build from PostgreSQL (IBM part first, overlay edges after A8) | A4 | Not started |
| B3 | Tabular features (transaction and account level), as-of rules | A4 | Not started |
| B4 | Transaction-graph features (degree, cycles, fan-in/out), as-of rules | B2 | Not started |
| B5 | Express backend skeleton (accounts, transactions, dashboard endpoints) | A4 | Not started |
| B6 | React shell (layout, navigation, design system) | none | Not started |

### Phase C: After the dataset is frozen (A8)

| Step | Task | Needs | Status |
|---|---|---|---|
| C1 | Overlay features (device/IP sharing) and final feature sets | A8, B3, B4 | Not started |
| C2 | Train models: tabular, + transaction graph, + overlay (transaction and account) | C1 | Not started |
| C3 | Group-B checks, payment-format ablation, evaluation | C2 | Not started |
| C4 | SHAP (top-10 per account, flagged transactions) and write scores to PostgreSQL | C2 | Not started |
| C5 | Suspicious network discovery into `analytics.*` | B2, A8 | Not started |

### Phase D: Application

| Step | Task | Needs | Status |
|---|---|---|---|
| D1 | Dashboard, Account Search, Account Investigation tabs 1-3 | B5, B6, C4 | Not started |
| D2 | Network graph tab | B2, B5 | Not started |
| D3 | FastAPI service + chat engine (intent-based retrieval) | C4, B2 | Not started |
| D4 | Account chat and Universal chat in the UI | D3 | Not started |
| D5 | Case management | B5 | Not started |
| D6 | PDF reports | D1, D3 | Not started |
| D7 | Integration testing, polish, documentation, viva prep | all | Not started |

---

## 2. Decision log

| # | Decision | Status |
|---|---|---|
| 1 | Base data: IBM AMLworld **HI-Small** + accounts file + our synthetic overlay | Confirmed |
| 2 | Modelling window: Sep 1-10, 2022; Sep 11-18 tail kept in DB/graph but excluded from ML | Confirmed |
| 3 | Account label tiers: `pattern` (positive), `laundering_only` (excluded from training and primary evaluation, checked separately), `none` | Confirmed |
| 4 | Risk bands: Low / Medium / High via score thresholds | Confirmed |
| 5 | SHAP precomputed for all accounts and flagged transactions | Confirmed (account SHAP narrowed to top-10, see 8) |
| 6 | Static dataset, batch pipeline, no live inference or live graph updates | Proposed (accepted in discussion) |
| 7 | Chat = intent-based retrieval with parameterized queries, no free-form Cypher | Proposed (accepted in discussion) |
| 8 | `txn_id` = CSV row number (synthetic from 10,000,001); `account_id` = `<bank_id>-<account_number>`; labels in `truth` schema; positives-only transaction label table; account SHAP top-10; currency-rate table; no login system | Proposed (in schema v1, not individually confirmed) |
| 9 | Compute features locally, store as Parquet; only scores/SHAP go to PostgreSQL | Proposed |
| 10 | LLM: Gemini Flash primary, Groq fallback, behind a provider wrapper | Proposed (check current free-tier limits) |
| 11 | Injected scenario volume: about 25-30 per family (about 100-120 total) | Proposed |
| 12 | Model comparison in three levels: tabular / + transaction graph / + overlay | Proposed |

---

## 3. Open questions

1. Which Aiven plan and storage are you actually on (free = 1 GB; paid reported as 8 GB)? Does the full data live on Aiven or in local Postgres?
2. Are all members running the same Postgres version locally?
3. Is Docker available on every machine, or do we use native installs?
4. Do you want the `feat` schema kept, or replaced by Parquet files (decision 9)?

---

## 4. Known risks to keep in mind

- Payment format is a shortcut: ACH holds about 87% of laundering rows. Keep it, add an ablation.
- Normal activity stops after Sep 10; never use raw date as a feature.
- Overlay can leak if devices/IPs correlate with labels; step A7 exists for that.
- Aiven free tier powers off when idle and has 1 CPU / 1 GB RAM; heavy work runs locally.
- Cross-currency amounts must go through `core.currency_rates`, never summed raw.
