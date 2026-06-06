# Topic Modelling Checklist

A stage-by-stage checklist for starting a topic modelling project. Each item links to the relevant section of this repo for detail.

Work through this in order — the stages are not independent. Decisions made at Stage 1 constrain everything downstream.

---

## Stage 0 — Data Collection
> See: [`docs/data-collection/`](../docs/data-collection/)

- [ ] The data scope is fully defined: seed lists, time frame, platforms, sources
- [ ] The time frame is fixed and will not be extended mid-project
- [ ] No missing or duplicated data — collection bugs are identified and resolved before modelling begins
- [ ] Any filtering applied before collection is documented and agreed

**Why this matters:** Topic models are built on data distribution and density. Adding sources, expanding the time window, or correcting a collection bug after the model has run typically requires rerunning the full pipeline.

---

## Stage 1 — Preprocessing
> See: [`docs/workflow/`](../docs/workflow/)

- [ ] Decide on preprocessing steps appropriate for the data type (social media, news articles, transcripts)
- [ ] Check tokenisation impact — confirm the embedding model's `max_seq_length` relative to document length in your corpus
- [ ] Confirm that documents are divided into the right units (sentences vs posts vs paragraphs)
- [ ] Language handling is decided: multilingual model, translate-first, or language-specific models

---

## Stage 2 — Embedding
> See: [`docs/workflow/`](../docs/workflow/) · [`docs/translation/`](../docs/translation/)

- [ ] Embedding model selected and justified for the domain and language
- [ ] For multilingual data: decision made on source text vs translated text for clustering
- [ ] Embeddings saved — do not recompute during parameter tuning

---

## Stage 3 — Topic Modelling
> See: [`docs/workflow/`](../docs/workflow/) · [`docs/sampling/`](../docs/sampling/) · [`docs/outliers/`](../docs/outliers/)

- [ ] Dataset size assessed — if large, read the [Sampling & Extrapolation guide](../docs/sampling/) before training on a sample
- [ ] UMAP and HDBSCAN parameters tuned iteratively; results saved at each configuration
- [ ] Topic count vs -1 cluster tradeoff assessed and a position agreed
- [ ] Reproducibility: random state fixed; parameter configuration saved alongside model
- [ ] Decision made on whether to address the -1 cluster (soft clustering / k-means / KNN) or retain it as-is

---

## Stage 4 — Annotation
> See: [`docs/annotation/`](../docs/annotation/) · [`docs/layered/`](../docs/layered/) · [`docs/guided/`](../docs/guided/)

- [ ] Annotation approach aligned with project goal: qualitative exploration vs quantitative classification
- [ ] Sampling strategy agreed: fixed-sample or proportional (see [Granular Annotation](../docs/annotation/))
- [ ] Annotator–analyst interaction cadence agreed; inconsistencies in description writing addressed before annotation begins
- [ ] For heterogeneous topics: decision made on multilayered modelling vs classifier-based filtering
- [ ] For predefined analytical categories: consider [Guided Topic Modelling](../docs/guided/)
- [ ] Annotation schema agreed and documented before annotation starts — changing it mid-process requires reannotation

---

## Stage 5 — Evaluation
> See: [`docs/evaluation/`](../docs/evaluation/)

- [ ] Evaluation type agreed: review mode, blind mode, or hybrid
- [ ] Sampling strategy agreed: general or stratified (stratify if specific sub-themes are underrepresented but analytically important)
- [ ] Evaluation level agreed: theme or sub-theme
- [ ] Analysis claims are limited to evaluated levels — do not report sub-theme precision if evaluation was done at theme level only

---

## Stage 6 — Analysis
> See: [`docs/evaluation/`](../docs/evaluation/)

- [ ] All quantitative claims are backed by evaluation results
- [ ] Alignment confirmed between analysis outputs and what the thematic breakdown can actually support
- [ ] Qualitative findings clearly distinguished from quantitative ones in reporting
