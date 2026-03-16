# 🔀 Streaming MTA — Multi-Touch Attribution for a Brazilian Streaming Platform

> **Multi-touch attribution pipeline** comparing last-touch, linear, and Shapley value allocation across the subscriber acquisition funnel — with an honest discussion of MTA's structural limitations and its relationship to MMM.

---

## Overview

This project implements and compares three attribution methodologies applied to a subscription streaming business. Beyond the mechanics of each model, the core contribution is a **critical framework for understanding what MTA can and cannot answer** — and how it complements, rather than replaces, Media Mix Modeling for budget decisions.

**The central tension in streaming attribution:** A subscriber's journey from first ad exposure to active plan activation often spans multiple weeks, devices, and channels. Last-touch attribution systematically over-credits performance channels (Google, Meta) that appear at conversion while ignoring the upper-funnel channels (TV, YouTube, influencer) that drove initial awareness. Shapley value allocation corrects for this — but introduces its own assumptions that practitioners need to understand.

---

## Key Features

- **Model Comparison Framework** — Last-touch, linear, and Shapley value attribution computed on the same journey dataset, with side-by-side credit distribution
- **Shapley Value Attribution** — Game-theoretic fair allocation across channel coalitions, computed via exact and approximate (Monte Carlo) methods
- **Markov Chain Attribution** — Removal-effect model estimating each channel's contribution to conversion probability
- **MTA vs. MMM Discussion** — Structured analysis of where each approach is valid, where each breaks down, and how to use them together
- **Funnel Position Analysis** — Credit allocation segmented by funnel stage (awareness / consideration / conversion) per channel

---

## Model Architecture

```
Raw Conversion Journeys (session-level touchpoint sequences)
                    │
                    ▼
         Journey Preprocessing
    (deduplication, time windowing, device stitching)
                    │
          ┌─────────┼──────────┐
          ▼         ▼          ▼
     Last-Touch   Linear    Shapley Value
     Attribution  Attribution  (coalition marginal contributions)
          │         │          │
          └─────────┴──────────┘
                    │
                    ▼
          Markov Chain Model
     (transition matrix + removal effects)
                    │
                    ▼
     Comparative Attribution Report
     (credit share per channel, funnel position, conversion rate)
```

---

## Dataset

The dataset is **fully synthetic**, generated to reflect realistic conversion journey patterns of a Brazilian SVOD platform:

| Plan | Avg. Touchpoints to Convert | Typical Journey Length | Primary Entry Channel |
|---|---|---|---|
| Família | 4.2 | 18 days | TV → Meta → Direct |
| Premiere (Football) | 2.8 | 9 days | Meta → Google → Direct |
| Telecine (Movies) | 3.5 | 14 days | YouTube → Google → Direct |
| HBO Max bundle | 2.1 | 6 days | TikTok → Direct |

Channels modeled: **Meta Ads, Google Ads, YouTube, TikTok, TV/OOH, Email, Organic/Direct, Influencer**

Journey data includes: channel, touchpoint timestamp, device, content category exposed, and conversion outcome (plan activated / abandoned).

---

## Attribution Models Compared

### 1. Last-Touch Attribution
Assigns 100% of conversion credit to the final touchpoint before plan activation.

**Bias:** Systematically over-credits direct and branded search. Penalizes awareness channels that initiate the journey but never appear last.

### 2. Linear Attribution
Distributes credit equally across all touchpoints in the journey.

**Bias:** Treats all touchpoints as equally valuable regardless of position, timing, or channel role. Ignores the structural difference between a first awareness impression and a retargeting click at intent moment.

### 3. Shapley Value Attribution
Computes each channel's **marginal contribution** averaged across all possible orderings of the channel coalition — borrowing from cooperative game theory.

```
φᵢ = Σ [|S|!(n−|S|−1)!/n!] × [v(S ∪ {i}) − v(S)]
     S ⊆ N\{i}
```

Where `v(S)` is the conversion rate of journey subset `S`. Channels are credited fairly for what they add, not where they appear.

**Limitation:** Requires sufficient data per channel coalition to estimate `v(S)` reliably. Sparse journeys or rare channel combinations produce unstable estimates.

### 4. Markov Chain Attribution
Models the conversion funnel as a Markov chain over channel states. Each channel's credit is its **removal effect**: the drop in overall conversion probability when that channel is removed from all journeys.

```
removal_effect(channel) = 1 − (conversion_rate without channel /
                               overall_conversion_rate)
```

**Advantage over Shapley:** Computationally tractable for large journey datasets. Captures sequential dependencies between channels — not just coalition membership.

---

## Results Snapshot

| Channel | Last-Touch | Linear | Shapley | Markov |
|---|---|---|---|---|
| Meta Ads | 38% | 24% | 21% | 22% |
| Google Ads | 31% | 19% | 16% | 17% |
| YouTube | 5% | 14% | 18% | 16% |
| TikTok | 8% | 12% | 14% | 13% |
| TV/OOH | 1% | 11% | 13% | 14% |
| Email | 9% | 10% | 9% | 9% |
| Organic/Direct | 8% | 10% | 9% | 9% |

> **Key finding:** Last-touch over-credits Meta and Google by ~15–17pp versus Shapley. TV and YouTube are systematically undervalued under position-based models — a critical distortion when these channels drive upper-funnel volume that enables retargeting to work at all.

---

## MTA vs. MMM — Structured Comparison

This project explicitly addresses the boundary between attribution and media mix modeling, a distinction that is often blurred in practice.

| Dimension | MTA (this project) | MMM (streaming-mmm) |
|---|---|---|
| **Unit of analysis** | Individual conversion journey | Aggregate weekly spend vs. outcomes |
| **What it measures** | Credit allocation across touchpoints | Causal contribution of spend to conversions |
| **Time horizon** | Journey window (days to weeks) | Campaign-level (months to years) |
| **Handles offline media** | ❌ No (journeys are digital only) | ✅ Yes (TV, OOH modeled via spend) |
| **Controls for seasonality** | ❌ No | ✅ Yes (Fourier / trend decomposition) |
| **Incrementality** | ❌ Correlation-based, not causal | ⚠️ Partially (requires experiment validation) |
| **Best use case** | Channel-level creative & audience optimization | Budget allocation, planning, CFO-level reporting |
| **Primary weakness** | Cannot capture cross-device or offline journeys; correlation ≠ causation | Requires 2+ years of data; slow to detect tactical changes |

### When to use each

- **MTA** — Optimizing bid strategies, creative rotation, and audience targeting within a campaign flight. MTA tells you *which ad sequences convert*; it does not tell you whether increasing spend on that channel would generate more conversions.
- **MMM** — Annual and quarterly budget planning, scenario modeling, CFO reporting on marketing ROI. MMM tells you the *aggregate causal effect of spend*; it does not tell you which individual journey sequences drove those conversions.
- **Together** — MTA informs in-flight tactical decisions; MMM governs strategic budget allocation. Neither should be used alone for the question the other is designed to answer.

---

## Project Structure

```
streaming-mta/
│
├── data/
│   ├── generate_journey_data.py         # Synthetic journey generator
│   └── streaming_journeys.csv           # Touchpoint-level journey dataset
│
├── notebooks/
│   ├── 01_eda_journey_analysis.ipynb    # Journey length, channel frequency, funnel analysis
│   ├── 02_last_touch_linear.ipynb       # Rule-based attribution models
│   ├── 03_shapley_attribution.ipynb     # Game-theoretic Shapley computation
│   ├── 04_markov_chain.ipynb            # Transition matrix + removal effects
│   └── 05_model_comparison.ipynb        # Side-by-side results + MTA vs. MMM discussion
│
├── src/
│   ├── journey_preprocessing.py         # Deduplication, windowing, device stitching
│   ├── attribution_models.py            # Last-touch, linear, Shapley, Markov implementations
│   └── comparison.py                    # Attribution comparison reporting
│
├── outputs/
│   ├── attribution_comparison.png
│   ├── markov_transition_matrix.png
│   ├── funnel_position_credit.png
│   └── mta_vs_mmm_summary.png
│
├── requirements.txt
└── README.md
```

---

## Tech Stack

| Layer | Tools |
|---|---|
| Data Generation | Python, NumPy, pandas |
| Attribution Models | Python (custom implementations), itertools |
| Markov Chain | NumPy, custom transition matrix |
| Visualization | matplotlib, seaborn, Plotly, networkx (journey graph) |
| Reporting | pandas, Jupyter |

---

## How to Run

```bash
# 1. Clone the repository
git clone https://github.com/your-username/streaming-mta.git
cd streaming-mta

# 2. Install dependencies
pip install -r requirements.txt

# 3. Generate synthetic journey dataset
python data/generate_journey_data.py

# 4. Run notebooks in order
jupyter notebook notebooks/
```

---

## Methodology Notes

**Journey windowing**
Touchpoints are included in a journey only if they occur within a 30-day lookback window prior to conversion. Touchpoints outside this window are treated as belonging to a prior, independent journey. The 30-day window reflects the typical consideration cycle for streaming subscriptions.

**Shapley computation**
Exact Shapley computation is O(2ⁿ) in the number of channels. For 8 channels, this is tractable (~256 coalitions). For datasets with more channels, a Monte Carlo approximation is implemented (random coalition sampling, convergence tested at 10k samples).

**Device stitching**
Journeys are stitched across devices using subscriber ID where available, and probabilistically using IP + user agent matching where not. This is a known limitation — cross-device journeys where the subscriber is not logged in on all devices will appear as disconnected sequences.

**Identified Limitations**
- MTA is fundamentally correlation-based. A channel appearing consistently before conversion does not mean it caused conversion — it may simply target high-intent audiences who would have converted anyway
- TV and OOH cannot be tracked at the journey level; their contribution in this dataset is approximated via exposed vs. non-exposed cohort matching, not individual-level tracking
- Shapley estimates are unstable for channels with low journey volume — results for low-frequency channels should be interpreted with wider confidence intervals

---

## Roadmap

- [ ] Geo-based incrementality test to validate Shapley estimates for top channels
- [ ] Data-driven attribution via logistic regression on touchpoint sequences (position + recency weighted)
- [ ] Integration with MMM outputs for unified cross-channel reporting view
- [ ] Bootstrap confidence intervals on Shapley estimates

---

## Portfolio Context

This repository is the **third component** of an integrated Marketing Science portfolio for a streaming subscription business:

| Repository | Focus |
|---|---|
| [`streaming-mmm`](../streaming-mmm) | Media Mix Modeling, saturation curves, budget optimization |
| [`streaming-churn-forecasting`](../streaming-churn-forecasting) | Behavioral churn prediction, intervention threshold optimization |
| **`streaming-mta`** *(this repo)* | Multi-touch attribution, model comparison, MTA vs. MMM framework |

All three repositories share the same synthetic platform dataset and subscription plan structure, enabling cross-model comparisons and a unified analytical narrative.

---

## Author

**Bruno** — Technical Architect & Decision Science Lead  
Focused on Growth Analytics, Marketing Mix Modeling, and causal inference for subscription businesses.

[LinkedIn](https://linkedin.com/in/your-profile) • [GitHub](https://github.com/your-username)

---

*Synthetic data. All figures are illustrative and do not represent any real company's performance.*
