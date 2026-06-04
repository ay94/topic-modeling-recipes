# Topic Modeling Recipes

A collection of practical recipes to unsupervised thematic analysis of large text corpora — covering the full pipeline from preprocessing through annotation, evaluation and large-scale extrapolation.

Built from real-world experience applying topic modeling across multilingual social media and news corpora in research and applied NLP contexts.

## Contents

| Section | Description |
|---|---|
| [Granular Annotation Scheme](docs/annotation/) | Entropy-based, proportional sampling annotation — replacing fixed-sample approaches with theoretically grounded stopping criteria |
| [Sampling & Extrapolation](docs/sampling/) | How to handle large datasets that can't be modelled in full — stratified sampling strategies and the extrapolation problem |
| [Outlier Mitigation](docs/outliers/) | What to do with HDBSCAN's -1 cluster — soft clustering, KNN, and when each is appropriate |
| [Layered Topic Modeling](docs/layered/) | Iterative refinement through successive modelling layers for heterogeneous or complex corpora |
| [Evaluation Framework](docs/evaluation/) | Precision, recall and F1 for thematic allocation — modified blind evaluation and when to skip evaluation entirely |

## Related libraries

- [`multilingual-topic-modeling`](https://github.com/ay94/multilingual-topic-modeling) — BERTopic + SetFit + machine translation pipeline
- [`semantic-knn`](https://github.com/ay94/semantic-knn) — KNN classification via ChromaDB for large-scale corpus labelling

## Status

Work in progress — sections being added iteratively. Each section will include a written guide and worked example notebook.
