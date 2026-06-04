# Layered Topic Modeling

> **Status:** Coming soon

## What this covers

A single-pass topic model often produces topics that are too coarse or too heterogeneous for analytical use. Layered topic modeling applies successive modelling rounds to progressively refine structure.

**Core ideas:**
- Layer 1 — global model across the full corpus to surface broad thematic structure
- Layer 2 — per-theme models on subsets identified in Layer 1, surfacing granular sub-topics
- Layer 3 — deep models for specific dense or complex narrative clusters requiring further unpacking
- Cross-layer annotation schema — a unified scheme bridging layers to ensure consistency

**When to use it:** When the corpus contains distinct narrative communities (e.g. different ideological groups discussing the same event), or when initial clusters are too broad to be analytically useful.

**Relationship to contrastive learning:** SetFit can be used between layers to steer the embedding space toward analytically meaningful cluster boundaries — see [`multilingual-topic-modeling`](https://github.com/ay94/multilingual-topic-modeling).
