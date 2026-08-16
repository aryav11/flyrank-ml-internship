# Capstone Report — Content Refresh Prioritization

- **Author:** Aryav Deep
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/aryav11/flyrank-ml-internship
- **Date:** 2026-08-16

## 1. Problem framing

### Decision

Which webpages should be prioritized for content-refresh review based on search performance and engagement signals?

The unit of analysis is one webpage/content item for a client. The output is a ranked review queue with four practical action categories: **Refresh Immediately, Review Soon, Monitor, and No Action**.

A content or SEO team can use the ranking to decide which pages should receive human review first. The cost of a wrong recommendation is wasted review or editing effort, or overlooking a page that may deserve attention.

Machine learning helps by combining multiple search, traffic, engagement, content, and freshness signals into a consistent ranking signal rather than relying only on one rule or manually reviewing the entire inventory.

This project is **decision support**. It does not claim that any individual signal causes decline, that refreshing a page will improve future performance, or that the model predicts Google's ranking algorithm.

## 2. Data safety

The analysis uses the anonymized FlyRank content-refresh dataset at `data/raw/content_refresh_anonymized.csv`.

### Dataset

- **30,000 rows**
- **44 source columns**
- **30,000 unique content items**
- **32 unique clients**

The dataset contains webpage-level search, traffic, engagement, content, freshness, and trend information. Most performance variables summarize trailing 90-day windows, with selected last-30-day and previous-30-day measures.

### Features and exclusions

The Logistic Regression model uses 28 non-identifier, non-target features covering search context, content characteristics, performance, engagement, traffic, and freshness signals.

The following fields are deliberately excluded from the feature vector:

- `content_id` — identifier only.
- `client_id` — used for grouped validation only, never as a model feature.
- `trend_direction` — directly determines the observed target.
- `trend_pct` — outcome-related trend information and therefore excluded from prediction.
- `is_declining_label` — the target itself.
- `provider_used` and `model_used` — metadata with substantial missingness and no direct role in the model.

The earlier leakage audit also identified overlapping/future fields and excluded them from the feature vector. The direct forbidden-field check passed.

### Public safety

The dataset is anonymized. No client names, private queries, credentials, private URLs, or personally identifying information are used in this report. The project uses public-safe language such as observed, measured, directional, and decision-support.

### Important temporal limitation

The current model is a **retrospective observed-period ranking experiment**, not a fully future-safe forecasting model. Some performance features are measured over windows that can overlap the observed trend outcome. This is disclosed rather than treated as proof of causal or future prediction.

## 3. Baseline

The transparent ML-07 rule-based baseline prioritizes pages using observable performance conditions involving impressions, CTR, and average position.

For the grouped capstone evaluation, baseline thresholds are learned from the training portion only and then applied to the held-out test clients. This keeps the comparison aligned with the model evaluation and avoids defining the baseline from the held-out labels.

The baseline is a fair comparison because it represents the earlier non-ML approach and is evaluated on the same held-out clients and the same ranking metrics as Logistic Regression.

| Method | Precision@20 | Precision@50 | Test Base Rate |
|---|---:|---:|---:|
| ML-07 Rule Baseline | 0.55 | 0.48 | 0.511 |
| Logistic Regression | **1.00** | **1.00** | 0.511 |

The model improves the recorded ranking metrics by **+0.45** at Precision@20 and **+0.52** at Precision@50 on this fixed grouped evaluation.

## 4. Model / analysis

### Method

The model is **Logistic Regression**. It is suitable as a first model because the target is a binary observed outcome and the resulting probability score can be used to rank webpages for review.

The model pipeline uses:

1. Median imputation for missing numeric feature values.
2. Standard scaling.
3. Logistic Regression with `max_iter=1000` and `random_state=42`.

The target is:

`is_declining_label = 1` when the observed `trend_direction` is `down`; otherwise 0.

The model is used to rank pages associated with an observed declining trend. It is not presented as a causal or future-performance forecast.

### Feature groups

The 28 model features cover:

- Search volume and competition
- CPC
- Content word and character counts
- 90-day impressions, clicks, pageviews, sessions, users, engaged sessions, AI sessions, and scroll events
- Search position and CTR
- Engagement and traffic ratios
- Content age and freshness
- Content type and search-intent/context fields
- Prior-window performance measures where included by the feature design

Identifiers and direct outcome-derived fields are excluded.

## 5. Evaluation

### Split design

The final evaluation uses a **client-grouped holdout split** with `GroupShuffleSplit`, `test_size=0.20`, and `random_state=42`.

| Split | Rows | Clients |
|---|---:|---:|
| Training | 23,837 | 25 |
| Testing | 6,163 | 7 |
| Client overlap | 0 | — |

No client appears in both training and testing. This is more conservative than allowing pages from the same client in both sets.

### Ranking metrics

| Method | Precision@20 | Precision@50 | Base Rate |
|---|---:|---:|---:|
| ML-07 Rule Baseline | 0.55 | 0.48 | 0.511 |
| Logistic Regression | **1.00** | **1.00** | 0.511 |

Additional Logistic Regression metrics on the same grouped test set:

| Metric | Value |
|---|---:|
| ROC AUC | 0.850 |
| Average Precision | 0.873 |
| Accuracy | 0.756 |
| Precision | 0.807 |
| Recall | 0.687 |
| F1 | 0.742 |

### Error analysis

At the 0.5 classification threshold, the recorded evaluation has an error rate of approximately **24.4%**. This means the model still makes classification errors even though its top-20 and top-50 ranking precision is very high.

The model's ranking performance should therefore be interpreted together with the base rate, ROC AUC, Average Precision, and classification metrics rather than treating Precision@20 or Precision@50 alone as evidence of a universally strong model.

### Research-claim audit

The project also audits two research findings: differences between growing and declining content, and observed performance differences associated with freshness/maturity. These findings are useful as measured or directional evidence, but observational comparisons do not establish that adding words, changing content age, or refreshing a page causes growth.

## 6. Interpretation

The evaluated Logistic Regression model ranked the observed declining pages more effectively than the ML-07 rule baseline on the fixed client-grouped test split. The recorded Precision@20 and Precision@50 values were both 1.00 for the model, compared with 0.55 and 0.48 for the baseline.

The practical interpretation is not that every high-ranked page needs editing. Instead, the model provides a stronger **review-prioritization signal** on this evaluation.

The workflow turns model scores into a human-review playbook:

| Priority | Action | Meaning |
|---|---|---|
| 1 | **Refresh Immediately** | Highest-priority human review and potential refresh |
| 2 | **Review Soon** | Focused review before deciding whether refresh is justified |
| 3 | **Monitor** | Continue tracking without immediate editing effort |
| 4 | **No Action** | No additional refresh effort based on current evidence |

The action playbook contains all **30,000 webpages** and uses reason codes so that recommendations remain interpretable.

A key limitation is that the observed-period feature windows can overlap the outcome period. Therefore, the result should not be described as a clean future forecast. A future production version would require a genuinely time-aware feature/label design with strict future-window separation.

The recorded 1.00 Precision@20 and Precision@50 should also not be interpreted as universal or guaranteed performance. They describe this dataset and this fixed grouped evaluation.

## 7. Recommendation

### How an editor should use the output

A FlyRank editor or SEO/content team can use the action queue as a **triage tool**:

1. Start with **Refresh Immediately** pages.
2. Inspect the reason codes and supporting search/engagement signals.
3. Review the actual page and business/search intent.
4. Decide whether a refresh, rewrite, expansion, technical investigation, or no change is appropriate.
5. Use **Review Soon** for the next tier.
6. Keep **Monitor** pages under observation.
7. Do not treat **No Action** as proof that a page is healthy; it means no immediate action is recommended by the current evidence.

### Human review and no-go rules

The model must not automatically:

- publish rewritten content,
- delete pages,
- create redirects,
- change metadata without review,
- make client-specific claims,
- or declare that a refresh will improve rankings.

The recommendation is a ranked signal for a human decision.

### Confidence and limits

Confidence is strongest in the narrow sense that the model produced better recorded top-K precision than the ML-07 baseline on the evaluated client-grouped split. Confidence is lower for generalization to future time periods, unseen distributions, or causal refresh impact.

## 8. Reproducibility

The work is organized in the repository under `work/`, while the reference pipeline remains separate. The repository guidance requires the reference pipeline to remain unchanged, datasets not to be committed to Git, fixed random seeds, and public-safe language. fileciteturn40file0L2-L6

### Main notebooks

- `work/notebooks/w04_baseline_score.ipynb` — ML-07 baseline
- `work/notebooks/w05_model.ipynb` — ML-08 modeling
- `work/notebooks/w06_validation_audit.ipynb` — ML-09 validation and research-claim audit
- `work/notebooks/w07_action_playbook.ipynb` — ML-10 ranked action playbook
- `work/notebooks/capstone.ipynb` — ML-11 capstone

ML-05 and ML-06 are excluded from the final internship workflow because those assignments were archived by the company, as reflected in the project workflow.

### Re-run approach

From the repository root, use the project's documented Python environment and dependencies, then execute the notebooks from `work/notebooks/`. The modeling and validation notebooks use `random_state=42` for reproducibility.

The capstone evaluation should be checked for:

- 30,000 source rows and 44 columns.
- 32 unique clients.
- 23,837 grouped training rows and 6,163 grouped testing rows.
- 25 training clients and 7 testing clients.
- Zero client overlap.
- Model and baseline evaluated on the same held-out data.
- Results consistent with the current executable notebook outputs.
- Action playbook containing 30,000 rows.

### Paper and artifacts

The deployed research paper is:

https://aryav11.github.io/flyrank-ml-internship/

The repository stores the exact deployed URL in:

`submission/paper_url.txt`

The capstone notebook produces the model-vs-baseline results and the paper artifacts, including the action-playbook distribution and validation summary. The deployed paper presents the same project as a public-safe decision-support study.

## Claims checklist

- **Observed / measured / directional / decision-support:** yes.
- **Causal claims:** none.
- **Google algorithm prediction claims:** none.
- **Client-identifying details:** none.
- **Base rate reported with ranking metrics:** yes.
- **Model and baseline evaluated on the same grouped test split:** yes.
- **Temporal-overlap limitation disclosed:** yes.
- **Numbers aligned with the executable capstone evaluation:** yes.

## Data credit

Built on the FlyRank ML Internship dataset. Data source: https://flyrank.ai/

This project uses the anonymized, public-safe internship dataset for research and educational purposes and does not expose private client information.
