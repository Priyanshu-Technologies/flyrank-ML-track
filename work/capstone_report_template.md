# Capstone Report — Refresh/Content Opportunity Scoring

- **Author:** Priyanshu Sharma
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Priyanshu-Technologies/flyrank-ML-track
- **Date:** 6 September 2026

> Copy this file to `work/capstone_report.md` and fill it in as you build. The eight
> sections mirror the Pass / Needs-Work rubric axes, so nothing here is optional.

## 1. Problem framing


This project supports the decision of which content pages should be prioritized for human review.

The unit of analysis is one content page for one client. The output is a ranked review queue with a model score, reason code, and suggested action.

A human editor can use the ranking to decide whether a page should be considered for refresh, expansion, protection, pruning, or monitoring.

The cost of a false positive is spending limited editorial review time on a page that does not need attention. The cost of a false negative is failing to review a page that may represent a meaningful content opportunity or performance concern.

Data and ML help by combining multiple observable page-level signals into a consistent ranking rather than relying on a single hand-written rule. The model is intended to support human decision-making rather than replace editorial judgment.

## 2. Data safety

The analysis uses the anonymized FlyRank content-refresh starter dataset containing page-level search, engagement, visibility, and content signals.

The target/proxy is derived from `trend_direction`, where the observed `down` category represents the declining label used for evaluation.

The following fields were deliberately excluded from model features:

- `trend_direction` — used to construct the observed label, so using it as a feature would leak the target.
- `trend_pct` — derived from the same trend information and therefore excluded.
- `content_id` — pseudonymous identifier; used only to identify rows.
- `client_id` — used for grouped validation, never as a model feature.
- Any future-window outcome — unavailable at the decision point and therefore excluded.

The analysis does not use client names, domains, URLs, private queries, credentials, or other client-identifying information.

The model uses only observable signals available in the supplied dataset. The leakage audit in Week 6 checked label-derived fields, identifiers, and obvious future-window features.

## 3. Baseline

The first benchmark was a transparent rule-based score created in Week 4.

A page received a positive baseline score when it met both of these conditions:

- `days_since_last_update >= 180`
- `impressions_90d >= 500`

The baseline score was the page's `impressions_90d` value when both conditions were satisfied, and zero otherwise.

Pages receiving a positive score were assigned the reason code `stale_visible_page` and the action `refresh_review`. Other pages received `not_prioritized` and `monitor`.

This is a fair baseline because it is simple, transparent, reproducible, and based only on observable signals available before the decision. It gives the learned model a meaningful benchmark rather than comparing it against an arbitrary score.

The baseline and Logistic Regression model were evaluated using the same Precision@50 metric and the same held-out client-grouped test set.

## 4. Model / analysis


I used Logistic Regression as the learned ranking model because the task is to prioritize pages associated with an observed declining label while keeping the first learned model relatively simple and interpretable.

The model uses the predicted probability of the declining label as its ranking score. Pages are ordered from highest to lowest score and evaluated using Precision@50.

### Feature list

The model uses these observable page-level features:

- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `content_age_days`
- `days_since_last_update`
- `impressions_90d`
- `clicks_90d`
- `pageviews_90d`
- `sessions_90d`
- `users_90d`
- `engaged_sessions_90d`
- `scroll_events_90d`
- `days_with_impressions`
- `days_with_sessions`
- `impressions_last_30d`
- `clicks_last_30d`
- `sessions_last_30d`
- `impressions_prev_30d`
- `clicks_prev_30d`
- `sessions_prev_30d`
- `ctr`
- `avg_position`
- `engagement_rate`
- `scroll_rate`
- `ai_traffic_pct`

`trend_direction` and `trend_pct` were intentionally left out because they are used to define the observed declining label and would therefore create label leakage.

`content_id` and `client_id` were also excluded as model features. `client_id` was retained only for grouped validation.

The target is the observed proxy `is_declining_label`, defined as 1 when `trend_direction == "down"` and 0 otherwise.

The model is therefore learning a ranking associated with an observed label in the available dataset; it is not being trained on a future refresh outcome.

## 5. Evaluation


The primary evaluation uses a client-grouped train/test split. All pages belonging to a client remain entirely within either the training set or the held-out test set.

This design is more appropriate for testing generalization to unseen clients than a random page-level split because pages from the same client can share characteristics.

The evaluation metric is Precision@50 because the intended use is to prioritize a small review queue. The test-set base rate is reported alongside Precision@50 so that the ranking result can be interpreted against the prevalence of the observed declining label.

The Logistic Regression model and the Week-4 transparent baseline are evaluated on the same held-out test clients and using the same Precision@50 metric.

The Week-6 audit also compared a random row-level split with the client-grouped split. In that run, the difference in Precision@50 was 0.0. Therefore, this comparison did not show a measurable validation gap in this particular evaluation. This result should not be interpreted as proof that the model will generalize to every future client or dataset.

### Error analysis

The model's errors consist of false positives and false negatives relative to the observed declining label.

False positives are pages ranked highly by the model that are not labeled as declining. These could consume review capacity unnecessarily.

False negatives are pages with the observed declining label that receive lower model scores. These represent pages that the ranking may fail to prioritize.

The error review is therefore treated as a decision-support check: a high-ranked page still requires human inspection, and a lower-ranked page is not automatically considered unimportant.

The evaluation measures association with the observed label. It does not establish future performance, causality, or the effect of refreshing a page.

## 6. Interpretation

The Logistic Regression model combines multiple observable page-level signals into a directional ranking score.

The coefficient analysis provides an interpretable view of which features are most strongly associated with the model's predicted probability of the observed declining label. Positive coefficients increase the model's predicted probability, while negative coefficients decrease it, holding the other modeled features constant.

The strongest coefficients should be interpreted as predictive associations within this dataset rather than as causal effects.

The model also shows that a high ranking does not necessarily mean that a page requires a refresh. Some highly ranked pages may not actually be poor content candidates when inspected by a human. For example, a page may already satisfy search intent, have strategic importance, or require monitoring rather than a content change.

### Surprises and negative results

The Week-6 validation audit found a 0.0 Precision@50 difference between the random row-level split and the client-grouped split in the evaluation run. Therefore, this audit did not reveal a measurable performance gap between those two validation designs.

This is a useful negative result rather than evidence that the model is universally robust. It means only that the two tested split designs produced the same measured Precision@50 in this run.

The model should therefore be interpreted as a directional ranking system for review prioritization. Its results are useful for identifying pages that deserve attention, but they do not demonstrate that any individual feature causes decline or that refreshing a ranked page will improve its future performance.

## 7. Recommendation

The recommended use of the model is as a ranked decision-support queue for content editors and SEO teams.

### Ranked actions

Pages should be reviewed in descending order of the model score. The ranking provides a consistent way to allocate limited review capacity.

The suggested action framework is:

1. **Refresh review** — investigate high-priority pages for possible content updates.
2. **Expansion review** — consider whether the page has a meaningful content-depth opportunity.
3. **Protection review** — consider preserving pages that already perform strongly and could be harmed by unnecessary changes.
4. **Monitor** — continue observing lower-priority pages when there is not enough evidence for immediate intervention.
5. **Manual investigation** — use additional context when a page does not fit a clear action category.

### Reason codes

The queue provides reason codes to make the ranking easier to interpret.

`stale_visible_page` indicates that a page is sufficiently old and has meaningful observed search visibility.

`high_visibility_page` indicates that a page has strong observed search visibility and therefore deserves human attention before an unnecessary change is made.

`model_priority` indicates that the page received a high model ranking without matching one of the simpler rule-based reason categories.

Reason codes are explanations for review priority, not guarantees about the correct editorial action.

### How an editor could use the queue

A FlyRank editor could start with the highest-ranked pages, inspect the reason code and supporting metrics, and then apply editorial and strategic judgment.

The editor should verify search intent, content quality, freshness, observed performance, and strategic importance before choosing an action.

### Confidence and limits

Confidence in an individual recommendation is limited because the current target is an observed declining proxy rather than a future refresh outcome.

The model provides directional evidence for prioritization, not a guarantee of future performance.

The output should not be used to automatically publish, rewrite, delete, redirect, or otherwise make irreversible content changes.

The strongest practical recommendation is therefore to use the model to decide **what to review first**, while keeping the final content decision with a human.

## 8. Reproducibility
The project is designed so that the analysis can be re-run from a fresh clone of the repository.

### Repository

Repository:

`https://github.com/Priyanshu-Technologies/flyrank-ML-track`

The analysis notebooks are stored under `work/notebooks/`.

The capstone report is stored at:

`work/capstone_report.md`

Generated queues and other non-source artifacts are written to `work/outputs/` according to the repository's data-handling rules.

### Environment

The modeling work uses Python with the following main packages:

- pandas
- numpy
- scikit-learn
- DuckDB for warehouse-based analysis where required

The project uses a dedicated Python virtual environment during development.

The modeling experiments use a fixed random seed of `42` for reproducibility.

### Re-running the analysis

From a fresh clone:

```bash
git clone https://github.com/Priyanshu-Technologies/flyrank-ML-track.git
cd flyrank-ML-track

python3 -m venv .venv
source .venv/bin/activate

pip install pandas numpy scikit-learn duckdb ipykernel


---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
