# Outlier Mitigation

> **Status:** Coming soon

## What this covers

HDBSCAN typically assigns 40–60% of data to the -1 (noise) cluster. These are not noise — they are fringe documents on the periphery of topics. This section covers how to handle them.

**Core ideas:**
- Soft clustering — reassigns -1 messages using HDBSCAN membership vectors, which encode the probability of a point belonging to each cluster based on density reachability
- KNN classification — using annotated cluster exemplars to label the full corpus including outliers (see [`semantic-knn`](https://github.com/ay94/semantic-knn))
- When to use which — soft clustering for smaller datasets with clear cluster structure; KNN for large-scale extrapolation where cluster definitions are already established

**Key finding:** Fringe documents predominantly associate with one core topic rather than multiple — making them analytically useful rather than genuinely ambiguous.
