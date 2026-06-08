# Outlier Mitigation

In the BERTopic workflow, HDBSCAN assigns a **-1 label** to messages that do not fall within any dense cluster region. These are not necessarily noise — they often represent fringe discussions that sit at the periphery of core topics and take the same shape as the full data. In practice, 40–60% of messages in a typical dataset fall into the -1 cluster, making effective outlier handling a significant methodological concern.

This document covers:
- How the -1 cluster forms and what it represents
- Two mitigation approaches investigated: soft clustering and k-means
- Findings and trade-offs from applying each approach

> **Cross-references:**
> - Core pipeline and HDBSCAN parameters: [`docs/workflow/`](../workflow/)
> - Annotation methodology: [`docs/annotation/`](../annotation/)

---

## The -1 Cluster: What It Is

![Figure 1: 2D UMAP distribution of core topics, -1 messages, and all data combined](umap-distribution.png)

*Figure 1: 2D UMAP distribution showing (A) core topics, (B) -1 messages, and (C) all data combined. The -1 messages span the full shape of the data rather than forming an isolated region — they are distributed across the entire embedding space, interspersed with and surrounding the core clusters.*

This spatial property is important: the -1 cluster is not a separate or peripheral region. It takes the same shape as the full dataset, which means -1 messages are not simply noise confined to the edges — they are fringe discussions that sit between, around, and within the structure of the core topics.

---

## Background

### How HDBSCAN Works

HDBSCAN is a density-based clustering algorithm designed to identify clusters of varying shapes and sizes within a dataset. Unlike traditional clustering algorithms that assume clusters to be spherical, this assumption breaks down when data forms clusters in more complex or irregular shapes — it is like trying to fit everything into a circle, even when the real groups are more diverse. HDBSCAN leverages minimum spanning trees and density reachability to uncover clusters in a more flexible manner. A minimum spanning tree can be thought of as the best way to connect all the points with the shortest possible lines, forming a tree structure that reveals how points naturally connect. Density reachability is about understanding how densely packed certain areas of the data are and finding clusters based on that density, rather than insisting on specific geometric shapes.

The algorithm proceeds in four steps:

**1. Minimum Spanning Tree (MST)**
HDBSCAN constructs an MST over the dataset — a tree-like structure connecting all data points with the minimum possible total edge length. This reveals the underlying connectivity between points without imposing a fixed shape.

**2. Single Linkage Clustering**
Clusters are formed by progressively joining points with the shortest distance between them. HDBSCAN employs varied distance metrics across different points for robustness, rather than applying a single global threshold.

**3. Condensed Dendrogram**
The resulting hierarchy is represented as a dendrogram. A condensed version is created by removing branches that do not significantly contribute to the clustering structure — this condensation aids in identifying stable, meaningful clusters rather than transient groupings.

**4. Stability**
HDBSCAN emphasises stability: areas in the condensed tree where branches persist across the hierarchy are treated as robust clusters. This makes the algorithm resilient to noise and ensures that identified topics consistently appear across different levels of analysis.

---

### The Membership Vector

Every message in HDBSCAN receives a **membership vector** — a set of scores indicating its strength of association with each identified cluster. These scores are derived from the stability of cluster membership across the hierarchy, not from a simple distance-to-centroid calculation.

**Nature of the vector:**
- Scores reflect how consistently a point is grouped with the same cluster as the hierarchy is traversed. Higher stability → higher score.
- A point's raw scores do not necessarily sum to one. When normalised, the membership vector can be interpreted as a probability distribution where scores sum to 1 across all clusters for a given point.
- Unlike hard clustering, a point may have significant scores across multiple clusters, capturing the ambiguity or overlap common in real-world datasets.

**What stability means:**
- The score assigned to a data point for a given cluster reflects how often it is grouped together with others in that cluster across different levels of the hierarchy.
- The stability of the cluster itself is also factored in — if a cluster persists across hierarchy levels, the points within it receive higher scores.
- Points deep inside a dense cluster receive high scores for that cluster; points at the boundary carry meaningful scores for multiple clusters.

**Soft clustering:**
The membership vector is what enables soft clustering — a point exhibiting characteristics of multiple clusters simultaneously can be assigned to more than one, capturing fringe membership rather than forcing a binary decision.

---

### How the -1 Cluster Forms

The -1 cluster arises from HDBSCAN's density threshold: points that do not meet the minimum density requirements to belong to any identified cluster are labelled -1. Five factors govern this:

1. **Density reachability threshold** — points that do not reach the density threshold for any dense region are assigned to -1, regardless of their proximity to clusters.
2. **Sparse regions** — points in areas where density falls below the threshold are more likely to be labelled outliers. These regions may contain genuine discussions but lack the critical mass to form a recognised cluster.
3. **Connectivity to core points** — a point's connectivity within the MST matters. If it lacks sufficient connections to core points, it is designated as an outlier even if thematically related to a cluster.
4. **Cluster shape flexibility** — because HDBSCAN identifies clusters of arbitrary shape, outlier classification is not purely geometric. The algorithm adapts to local density variations, meaning points can be outliers even when geometrically close to a cluster if local density is low.
5. **Parameter influence** — `min_cluster_size` and related parameters directly control how many points fall into -1. A larger minimum cluster size produces more -1 messages; a smaller one allows smaller clusters to form, reducing -1 but potentially fragmenting meaningful topics.

---

## Mitigation Approaches

Two approaches are investigated to reduce the proportion of -1 messages and recover analytical value from them:

1. **Soft clustering** — uses the HDBSCAN membership vector to assign -1 messages to one or more core topics based on configurable score thresholds.
2. **K-means** — applies k-means clustering to the -1 messages in isolation, using the number of HDBSCAN core topics as k.

**Sampling and analysis strategy (both approaches):**
- Sample from the topic breakdown in each output
- Annotate the resulting topics
- Analyse relationships between fringe topics and core topics
- Determine whether fringe topics sit genuinely at the periphery of cores or represent true noise
- For k-means: conduct a comparative analysis against the soft clustering output to reveal whether k-means provides a distinct perspective on the data

---

## Soft Clustering

### Approach

The soft clustering approach leverages the membership vector to reassign -1 messages to one or more core topics. The `SoftReclusterer` is applied once BERTopic is trained and membership vectors are extracted. It is governed by four parameters:

| Parameter | Description |
|---|---|
| `min_membership_score` | Minimum membership score required for a point to be considered part of a cluster |
| `max_core_clusters` | Maximum number of core clusters a point can be assigned to (e.g. up to 5) |
| `min_fringe_cluster_size` | Minimum number of points required to validate a fringe cluster; below this threshold the group is treated as an outlier |
| `threshold_ratio` | Determines the membership score threshold relative to the maximum score in the vector |

**On `threshold_ratio`:** when `method='ratio'`, the threshold is `max_membership_score × threshold_ratio`. For example, if the maximum score in the vector is 0.6 and the ratio is 0.5, the threshold is 0.3 — any cluster with a score ≥ 0.3 qualifies as a fringe assignment for that point. This prevents assignments when scores are closely bunched, ensuring fringe assignments reflect genuine secondary membership rather than noise in the vector.

The result is a fringe breakdown: -1 messages are reassigned to one or more core topics based on these criteria, with residual messages that do not meet the thresholds remaining as true outliers.

---

### Observations

#### General

1. **Relevance within topics** — fringe topic discussions are generally centred around one or more core topics. No instances were observed where fringe topics contained messages entirely outside the scope of the identified core topics.
2. **Obscurity** — some messages within fringe topics are occasionally obscure or lack sufficient information. Despite this, domain experts can recognise and contextualise them without difficulty.
3. **Proximity to most probable topic** — messages close to the fringe tend to align with their most probable core topic rather than exhibiting balanced membership across several. The primary association dominates even when secondary scores exist.
4. **Persisting parameter challenges** — the chosen ratio and hyperparameters still produce a significant proportion of residual outliers. Further tuning is necessary to balance inclusion and quality.
5. **Increased control** — despite these challenges, working with membership vectors offers more control and interpretability than adjusting UMAP or HDBSCAN parameters directly. The effect of parameter changes is more predictable and auditable.

#### Fringe Topics

1. **Association with core topics** — fringe topics are predominantly associated with one core topic. It was rare to observe fringe topics without a clear primary core association, even when secondary associations existed.
2. **Homogeneity** — messages within a fringe topic typically share common themes and context, indicating the algorithm successfully groups relevant discussions. Fringe topics tend to explore specific nuances or sub-topics of a core rather than mixing unrelated content.
3. **Multiplicity of associations** — contrary to initial expectations, the majority of fringe topics align primarily with one core topic rather than spanning multiple. Multi-core associations are more apparent to domain experts through qualitative review than through the membership scores alone. Instances of genuine multi-core fringe topics were infrequent, suggesting that -1 is normally a fringe of one topic rather than multiple.
4. **Differentiation from outliers** — fringe topics are distinguishable from true outliers: fringe topics have a clear primary core association, while the residual outlier cluster lacks any coherent theme across its messages.

#### Outlier Cluster

The residual outlier cluster — messages that remain -1 after soft clustering — is a mixed bag rather than a thematically coherent group. These messages cover a wide range of subjects with uncommon or unusual combinations of keywords and contextual information that do not fit neatly into any core cluster. They may represent genuinely ambiguous discussions, highly specific sub-topics without sufficient mass to form their own cluster, or messages whose content is too sparse to classify reliably. The parameter choice directly determines how large this residual cluster is.

---

### Pros

**Methodological control**
Working with membership vectors is more interpretable than adjusting UMAP or HDBSCAN parameters. The effect of changing `min_membership_score` or `threshold_ratio` is directly observable in the fringe assignments, making iterative refinement more tractable.

**Analytical value of fringe topics**
Fringe topics often capture interdisciplinary discussions that span multiple core areas — content that is frequently valuable precisely because it crosses thematic boundaries. Soft clustering also improves recall: -1 is not always noise. Recovering fringe discussions that genuinely belong near a core increases the proportion of data available for analysis and can potentially improve thematic coverage.

---

### Cons

**Parameter tuning overhead**
The soft clusterer requires careful hyperparameter tuning. Poorly chosen parameters produce a large residual outlier cluster, and rapid evaluation of parameters is essential to avoid repeated annotation effort and minimise overhead.

**Increased annotation load**
Fringe topics increase the total number of topics an annotator must review. Each fringe cluster requires description and assessment, adding steps to the annotation process.

---

## K-Means Comparison

K-means was applied to the -1 cluster in isolation, with k set equal to the number of core topics identified by HDBSCAN. The rationale: since the -1 cluster takes the same shape as the full data (Figure 1), applying k-means with the same number of groups as HDBSCAN found provides a direct structural comparison of what each approach discovers within the outlier space.

### Findings

**Common topics**
Both approaches identified overlapping thematic groupings within the -1 cluster. Topics with broad, widely-discussed themes appeared in both outputs, suggesting robustness in identifying the major patterns present in the outlier space. Where both methods agreed, confidence in the thematic grouping is higher.

**Distinct topics**
Each approach surfaced topics that had no clear equivalent in the other. Some themes identified by soft clustering did not appear in the k-means output, and k-means identified distinct groupings — particularly around narrower sub-topics — that soft clustering left in its residual outlier cluster. This suggests the two approaches are sensitive to different structural properties of the -1 space.

**Similar but different**
Several topics appeared in both outputs but with different emphasis or granularity. K-means tended to produce broader groupings that merged what soft clustering split into distinct fringe topics. Conversely, soft clustering occasionally identified finer-grained associations that k-means subsumed into a larger cluster.

### Observations

- The granularity and specific focus of topics differs between the two approaches even when broad themes overlap.
- Some topics in one approach have broader or narrower counterparts in the other, rather than direct equivalents.
- The diversity of topics within each output reflects the distinct patterns each algorithm is sensitive to: density reachability for soft clustering, geometric distance for k-means.
- K-means results diverge from the soft clustering output in ways that are complementary rather than contradictory — each captures aspects the other misses.

### Conclusion

K-means applied to the -1 cluster contributes complementary insights rather than duplicating soft clustering findings. Commonalities indicate robustness in identifying broad thematic groupings within the outlier space. Unique k-means findings suggest it recovers patterns that remain in the soft clustering residual outlier cluster — the two approaches are more complementary than redundant.

K-means is simpler to apply and requires no membership vector tuning, but provides less interpretable and less nuanced assignments. Soft clustering provides richer analytical structure but demands more tuning effort. Using both in combination — soft clustering for nuanced recovery, k-means for structural validation — is a viable strategy where annotation resource allows.

---

## Summary: Trade-offs

| | Soft clustering | K-means |
|---|---|---|
| Basis | HDBSCAN membership vector scores | Geometric distance to centroid |
| Multi-topic assignment | Yes — up to `max_core_clusters` | No — hard assignment to one cluster |
| Interpretability | High — scores are auditable and parameter effects are observable | Lower — centroid-based assignments are less transparent |
| Parameter sensitivity | High — requires tuning of 4 parameters | Moderate — k is fixed to HDBSCAN topic count |
| Residual outliers | Yes — messages below thresholds remain as outliers | None — all messages are assigned |
| Annotation overhead | Higher — fringe topics add to workload | Moderate |
| Best for | Recovering analytically useful fringe discussions with nuance | Quick secondary view of -1 structure; complementary validation |

---

## Checklist

- [ ] -1 proportion estimated after initial BERTopic run
- [ ] Decision made: soft clustering, k-means, or both
- [ ] **Soft clustering:** `SoftReclusterer` parameters set (`min_membership_score`, `max_core_clusters`, `min_fringe_cluster_size`, `threshold_ratio`)
- [ ] **Soft clustering:** fringe topics sampled and annotated
- [ ] **Soft clustering:** fringe topic associations with core topics reviewed
- [ ] **Soft clustering:** residual outlier cluster reviewed for any recoverable content
- [ ] **K-means:** k set equal to HDBSCAN topic count; applied to -1 cluster only
- [ ] **K-means:** topics sampled and annotated
- [ ] **K-means:** output compared with soft clustering to identify common, distinct, and overlapping themes
- [ ] Final decision: which fringe/k-means topics to include in the analysis
