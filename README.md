# QoE → LTV Intelligence

Streaming subscriber quality-of-experience analysis, built on a real public dataset with a clearly disclosed modeled layer on top. Includes a churn driver attribution model with a Markov-chain-style removal effect check.

**Live dashboard:** _add your GitHub Pages link here after enabling Pages, see repo settings_

**Dataset source:** [IBM Telco Customer Churn — Kaggle](https://www.kaggle.com/datasets/blastchar/telco-customer-churn)

---

## Repo structure

| File | Description |
|------|-------------|
| `data/qoe_ltv_subscribers.csv` | 7,043 customers, 40 columns |
| `index.html` | Interactive dashboard (open directly, or view via GitHub Pages) |
| `README.md` | This file — full methodology and data dictionary |

---

## Telco Customer Churn QoE Enrichment — Real Base, Modeled QoE Layer

### Provenance (read this first)

This file has two parts, and they are not the same kind of data.

**Columns 1–33 are real.** They are the unmodified IBM Telco Customer Churn dataset — the same file used across dozens of published churn-analysis projects (source: IBM Cognos Analytics sample data, 7,043 California customers, Q3 snapshot). Nothing in this block is invented: demographics, services, Monthly Charges, Total Charges, Churn Label, Churn Value, Churn Score, and CLTV are all as IBM published them.

**Columns 34–40 are my addition, and they are modeled, not measured.** IBM's dataset has no telemetry, no rebuffering logs, no playback data — because IBM is a telecom sample set, not a streaming platform's internal analytics. There is no public dataset that pairs real QoE telemetry with real subscriber LTV. So I built a QoEScore and five related fields as a calibrated latent-variable model, anchored to two real columns already in the data (Churn Score and CLTV), rather than pure noise.

**Formula (streaming-eligible customers only, i.e. Internet Service ≠ No):**

```
QoE_latent = −0.55·z(Churn Score) + 0.30·z(CLTV) + fiber_bonus + noise
QoEScore   = clip(3 + 0.62·QoE_latent, 1, 5)
```

where `fiber_bonus = 0.35` for Fiber optic customers (higher bandwidth ceiling than DSL), and `noise` is Gaussian. Rebuffering ratio, startup time, video/audio quality, and bitrate are then derived as noisy linear functions of QoEScore, in the direction you'd expect (higher QoE → less rebuffering, faster startup, higher bitrate).

**What this means in practice:** any relationship you find between QoEScore and real churn/CLTV is partly a self-fulfilling result of the formula, because I built it to move opposite real Churn Score and with real CLTV. Treat this as a demonstration of a modeling approach — how you'd structure the analysis if you had real telemetry — not as evidence that IBM's telco customers actually experienced these rebuffering rates.

**Say this plainly if anyone asks:** the churn and CLTV numbers are real IBM data; the QoE layer is my own calibrated addition on top of it.

---

### Column reference — Real IBM, unmodified

| Column | Notes |
|--------|-------|
| CustomerID, Count, Country, State, City, Zip Code, Lat Long, Latitude, Longitude | Identifiers & geography |
| Gender, Senior Citizen, Partner, Dependents | Demographics |
| Tenure Months | Months with company |
| Phone Service, Multiple Lines, Internet Service, Online Security, Online Backup, Device Protection, Tech Support, Streaming TV, Streaming Movies | Services subscribed |
| Contract, Paperless Billing, Payment Method | Account terms |
| Monthly Charges, Total Charges | Billing |
| Churn Label, Churn Value | Yes/No and 1/0 churn flag |
| Churn Score | IBM's proprietary 0–100 churn propensity score (SPSS Modeler) |
| CLTV | IBM's proprietary predicted Customer Lifetime Value |
| Churn Reason | Free text, mostly null (only populated for churned customers) |

### Column reference — Modeled (my addition; streaming-eligible rows only, null where Internet Service = No)

| Column | Description |
|--------|-------------|
| QoEScore | Composite Quality of Experience, 1.00–5.00 |
| QoEBucket | Poor (1–2) / Fair (2–3) / Good (3–4) / Excellent (4–5) |
| RebufferingRatioPct | Modeled % of session time spent buffering |
| StartupTimeSec | Modeled seconds to first frame |
| VideoQualityScore, AudioQualityScore | Modeled subscores, 1–5 |
| AvgBitrateMbps | Modeled average delivered bitrate |

---

### What the enriched file actually shows (computed on this file)

- 5,517 of 7,043 customers (78%) subscribe to internet service and are QoE-eligible.
- Real CLTV by modeled QoE bucket: Poor $3,476 · Fair $4,086 · Good $4,630 · Excellent $5,155.
- Real 90-day churn rate by modeled QoE bucket: Poor 65.3% · Fair 49.1% · Good 19.3% · Excellent 3.6%.
- Correlation, modeled QoE vs real CLTV: r = 0.34.
- Correlation, modeled QoE vs real Churn Score: r = −0.56.
- Correlation, modeled Rebuffering vs real Churn Value: r = 0.35.

These are weaker, messier correlations than a fully synthetic dataset would produce — and that's the point: they're constrained by real underlying data, not tuned to a target number.

---

### Driver attribution — churn credit across factors

A natural next question: once you have both real outcomes and a modeled QoE score, which factor actually "deserves" credit for churn? This mirrors multi-touch attribution in marketing (which touchpoint gets credit for a conversion), applied to churn drivers instead of channels.

**Model:** standardized logistic regression, real Churn Value as outcome, six predictors: Contract (two dummies vs. month-to-month baseline), Tenure Months, Internet Service (fiber optic dummy vs. DSL baseline), Monthly Charges, and QoEScore. These six predictors have low mutual correlation (max 0.28), unlike the four QoE sub-columns, making coefficient-based credit-splitting defensible here.

| Factor | Type | Credit (normalized coef) | Direction |
|--------|------|--------------------------|----------|
| QoE Score | Modeled | 31.0% | Higher QoE → less churn |
| Fiber optic connection | Real | 22.0% | Fiber → more churn |
| Tenure months | Real | 19.8% | Longer tenure → less churn |
| Two-year contract | Real | 17.5% | → less churn |
| One-year contract | Real | 8.7% | → less churn |
| Monthly charges | Real | 1.0% | Higher charges → more churn |

**Model AUC: 0.863.**

Removal-effect check (same logic as removing a touchpoint in a Markov chain attribution model):

| Factor removed | AUC without it | Drop |
|----------------|---------------|------|
| QoE Score | 0.807 | −0.057 |
| Tenure | 0.853 | −0.011 |
| Fiber optic | 0.855 | −0.009 |
| Two-year contract | 0.858 | −0.005 |
| One-year contract | 0.860 | −0.003 |
| Monthly charges | 0.863 | 0.000 |

QoE produces by far the largest removal-effect drop — more than 5× the next-largest factor (tenure) — corroborating its top rank in the coefficient-based credit split.

---

### Source

IBM Telco Customer Churn sample dataset — Cognos Analytics sample data, commonly mirrored on Kaggle and GitHub. This copy pulled from a public GitHub mirror and verified against the standard 33-column IBM schema (CLTV, Churn Score, Streaming TV/Movies, etc.).
