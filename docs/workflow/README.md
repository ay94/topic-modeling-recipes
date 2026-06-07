# Core Workflow

The generic topic modelling pipeline — from raw text to a topic theme map.

---

## Pipeline Steps

### 1. Embedding

Each message is passed through a sentence transformer to produce a contextualised vector representation — an embedding — that captures its semantic meaning. Messages that are similar in meaning will have embeddings that are close together in high-dimensional space.

### 2. Dimensionality reduction — UMAP

The high-dimensional embeddings are reduced to a lower-dimensional space using UMAP. This makes the data tractable for clustering while preserving the neighbourhood structure of the original space — messages that were semantically close remain close after reduction.

### 3. Clustering — HDBSCAN

HDBSCAN is applied to the reduced embeddings to form topic clusters. It identifies dense regions of the space as topics and assigns a `-1` label to messages that do not belong to any dense region. Unlike k-means, HDBSCAN does not require the number of topics to be specified in advance — it determines this from the data's own density structure.

### 4. Topic representation — BERTopic

These three steps are typically implemented via **BERTopic**, a Python library that wraps embedding, UMAP, and HDBSCAN and adds topic representation and visualisation features. BERTopic uses a variant of TF-IDF called **c-TF-IDF** to identify the most significant keywords for each topic — words that are frequent within the topic but rare across the rest of the corpus.

A notable limitation: BERTopic's built-in visualisations are based on c-TF-IDF keyword vectors rather than the original message embeddings. This means the visual layout reflects keyword distributions, not the full semantic structure of the embedding space.

### 5. Thematic allocation — topic theme map

Once topics are formed, the annotation phase begins. A sample of messages is drawn from each topic, reviewed, and used to produce a description. Topics are then organised into sub-themes and overarching themes, producing a **topic theme map** that assigns each topic — and its corresponding messages — to a specific position in the thematic hierarchy.

The topic theme map is the output used for analysis and evaluation. How the sample is drawn, how descriptions are produced, and how topics are judged as homogeneous or heterogeneous is governed by the annotation scheme used. See [`docs/annotation/`](../annotation/) for the granular annotation scheme.

---

## Key Parameters

| Component | Key parameters |
|---|---|
| Sentence transformer | Model choice — multilingual vs monolingual, general vs domain-specific |
| UMAP | `n_components`, `n_neighbors`, `min_dist` |
| HDBSCAN | `min_cluster_size`, `min_samples`, `cluster_selection_method` |

Parameter tuning decisions and their implications are covered in [`docs/sampling/`](../sampling/).

---

## Cross-references

| Topic | Where to look |
|---|---|
| Data collection before the pipeline runs | [`docs/data-collection/`](../data-collection/) |
| Large datasets and the UMAP extrapolation problem | [`docs/sampling/`](../sampling/) |
| The `-1` outlier cluster | [`docs/outliers/`](../outliers/) |
| Annotation and topic theme map construction | [`docs/annotation/`](../annotation/) |
| Heterogeneous topics and iterative refinement | [`docs/layered/`](../layered/) |
| Steering the embedding space toward analytical distinctions | [`docs/guided/`](../guided/) |
| Evaluating thematic allocation | [`docs/evaluation/`](../evaluation/) |
