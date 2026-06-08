# Outlier Mitigation

In the BERTopic workflow, HDBSCAN assigns a **-1 label** to messages that do not fall within any dense cluster region. These are not necessarily noise — they often represent fringe discussions that sit at the periphery of core topics and take the same shape as the full data. In practice, 40–60% of messages in a typical dataset fall into the -1 cluster, making effective outlier handling a significant methodological concern.

This document covers:
- How the -1 cluster forms and what it represents
- Two mitigation approaches investigated: soft clustering and k-means
- Findings and trade-offs from applying each approach

> **Cross-references:**
> - Core pipeline and HDBSCAN parameters: [`docs/workflow/`](../workflow/)
> - Annotation methodology: [`docs/annotation/`](../annotation/)
> - Implementation: [`multilingual-topic-modeling`](https://github.com/ay94/multilingual-topic-modeling) — `multilingual_topic/outlier_mitigation.py` (`SoftReclusterer`, `StaticReducer`, `StaticClusterer`)
> - Demo notebook: [`notebooks/outlier_mitigation_demo.ipynb`](https://github.com/ay94/multilingual-topic-modeling/blob/main/notebooks/outlier_mitigation_demo.ipynb)

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
HDBSCAN constructs an MST over the dataset — a tree-like structure connecting all data points with the minimum possible total edge length (the sum of the lengths of all potential edges in a graph). This reveals the underlying connectivity between points without imposing a fixed shape.

**2. Single Linkage Clustering**
Clusters are formed by progressively joining points with the shortest distance between them. HDBSCAN employs varied distance metrics across different points for robustness, rather than applying a single global threshold.

**3. Condensed Dendrogram**
The resulting hierarchy is represented as a dendrogram, and a condensed version of this dendrogram is created by removing branches that do not significantly contribute to the clustering structure. This condensation aids in identifying stable clusters by stripping away transient or insignificant groupings.

**4. Stability**
HDBSCAN emphasises the stability of clusters by identifying areas in the condensed tree where branches persist. This ensures robustness against noise and identifies clusters that consistently appear across different perspectives on the data.

---

### The Membership Vector

The membership vector in HDBSCAN is a component that provides information about the association of each data point with the identified clusters. For each point, the membership vector is essentially a set of scores indicating the probability or strength of affiliation with each cluster. This vector reflects the density-based nature of HDBSCAN. It is worth noting that the membership scores represent the strength of association between a data point and different clusters — these scores are not probabilities in the traditional sense; they indicate how well a point fits into each cluster.

**1. Nature of the vector**
- The probability assignment in HDBSCAN is derived from the stability of a point's cluster membership across different levels of the cluster hierarchy.
- The probability assigned to a data point reflects the stability of its cluster membership. Points that are consistently part of the same cluster across different levels of the hierarchy are assigned higher probabilities.
- Stability is assessed by considering how often a point is grouped together with others in the same cluster as we move up and down the hierarchy. If a point consistently belongs to the same cluster, it is deemed more stable, and its probability of belonging to that cluster is higher.
- The stability of a cluster is also taken into account. If a cluster persists across different levels of the hierarchy, the points within that cluster are assigned higher probabilities.
- The output is a set of probabilities for each data point indicating its likelihood of belonging to different clusters. However, these probabilities do not necessarily sum to one for each data point.

**2. Probability distribution**
The membership vector, when normalised, can be interpreted as a probability distribution. Each element represents the likelihood of the corresponding point belonging to a specific cluster. The sum of probabilities across all clusters for a single point is equal to 1.

**3. Granular cluster assignments**
Unlike traditional clustering methods that assign each point to a single cluster, the membership vector allows for more granular assignments. A point may have significant membership scores across multiple clusters, capturing the ambiguity or overlap often present in real-world datasets.

**4. Soft clustering**
This concept of soft clustering, facilitated by the membership vector, accommodates scenarios where a point may exhibit characteristics of multiple clusters simultaneously — which is the basis of the soft clustering mitigation approach described below.

---

### How the -1 Cluster Forms

In HDBSCAN, the -1 cluster represents outliers or data points that do not fit well into any identified clusters. The formation of the -1 cluster is influenced by the density-based approach of HDBSCAN, which inherently distinguishes between core points, border points, and outliers. Five factors govern how points end up in -1:

1. **Density reachability threshold** — HDBSCAN uses a density reachability threshold to determine whether a point is part of a dense region. Points that do not meet this threshold are considered outliers and are assigned to the -1 cluster.
2. **Outliers in sparse regions** — points situated in sparse regions of the dataset, where the density is below the defined threshold, are more likely to be labelled as outliers. These regions may represent areas with lower data density, where traditional clustering algorithms might struggle to identify meaningful clusters.
3. **Connectivity to core points** — the connectivity of a point to core points within the minimum spanning tree plays a role. If a point does not have sufficient connections to core points, it is more likely to be designated as an outlier.
4. **Flexibility in cluster shapes** — the flexibility of HDBSCAN to identify clusters of various shapes means that outliers are not solely determined by geometric considerations. Instead, the algorithm adapts to local density variations, making it effective in capturing outliers in complex datasets.
5. **Influence of parameters** — the formation of the -1 cluster is also influenced by the parameters set during execution, such as the minimum cluster size and other density-related parameters. Adjusting these parameters can impact the identification of outliers: a larger minimum cluster size produces more -1 messages; a smaller one allows smaller clusters to form, reducing -1 but potentially fragmenting meaningful topics.

---

## Mitigation Approaches

The clustering approach in BERTopic classifies topics into two types: **core topics** (dense areas identified by HDBSCAN) and the **fringe topic (-1)**, encompassing messages that don't fall within or close to any dense areas. Each message is assigned a membership vector indicating a probability distribution over all topics. Two approaches are investigated to mitigate the presence of -1:

1. **Soft clustering** — utilises the membership vector to assign -1 messages to one or multiple topics, up to 5 topics, based on specific criteria (discussed in the next section).
2. **K-means:**
   - Unlike density-based approaches, k-means assigns each message to one of the specified buckets (identified by k).
   - HDBSCAN is initially applied to identify the number of dense areas. Given that the -1 topic often reflects the structure of the entire data (as shown in Figure 1), k-means is applied solely to the -1 cluster, using the same number of dense areas discovered by HDBSCAN as k.

**Sampling and analysis strategy (both approaches):**
- Sample from the topic breakdown in each output
- Annotate the resulting topics
- Analyse relationships between fringe topics and core topics to determine whether fringe topics are genuinely on the periphery between various cores and assess the utility of outliers in this setup
- For k-means: conduct a comparative analysis against the soft clustering output to reveal whether k-means provides a distinct perspective on the data

**Dataset:** A domain-specific economic dataset was used for this investigation. This dataset was selected because of familiarity with the domain, enabling domain expert analysis of the results.

---

## Soft Clustering

### Approach

The soft clustering approach is designed to leverage the **distribution** of membership vectors to assign messages labelled as -1 to one or more core topics based on specific criteria. The `SoftReclusterer` is applied once BERTopic is trained and membership vectors are extracted. Implementation: [`multilingual_topic/outlier_mitigation.py`](https://github.com/ay94/multilingual-topic-modeling/blob/main/multilingual_topic/outlier_mitigation.py). It is governed by four parameters:

- **`min_membership_score`** — Minimum membership score required for a point to be considered part of a cluster.
- **`max_core_clusters`** — Maximum number of core clusters to consider for a point. For instance, a point can only be assigned to up to 5 core clusters to be considered a fringe of those 5.
- **`min_fringe_cluster_size`** — Minimum number of points in a fringe cluster to validate it as a valid fringe cluster. If there are fewer than 5 messages, it is considered an outlier.
- **`threshold_ratio`** — The ratio used to determine the number of clusters a message can be a part of:
  - This parameter is used to determine the maximum membership score of a cluster to be considered a fringe cluster for a given point.
  - When `method='ratio'`, the threshold is calculated by multiplying the maximum value in the membership vector by `threshold_ratio`.
  - For example, if the maximum value in the vector is 0.6 and `threshold_ratio` is 0.5, the threshold is 0.3 — any cluster with a membership score ≥ 0.3 is considered a fringe cluster for that point.
  - This ratio is useful to ensure the maximum is not very close to the next closest distribution (i.e. when variability is high).

Once BERTopic is trained and the membership vectors are extracted, the `SoftReclusterer` is utilised to provide the fringe breakdown. It takes into account the parameters above to determine the fringe clusters. Overall, this approach offers an overview of the reassignment of messages labelled as -1 to core topics based on the distribution of membership vectors and specific criteria.

---

### Observations

As discussed, this section covers three types of observations:

- Overall observations of the approach
- Observations about the fringe topic
- Observations about the outlier cluster

#### General

1. **Relevance within topics**
   - a. Fringe topic discussions are generally centred around one of the core topics or span across multiple topics.
   - b. There have been no instances where fringe topics contain messages completely outside the scope of the identified topics.

2. **Obscurity and information content**
   - a. Some messages within fringe topics are occasionally obscure or lack sufficient information.
   - b. Despite potential obscurity, domain experts can easily capture and recognise that.

3. **Proximity to most probable topic**
   - a. Messages that are close to the fringe often align with the most probable topic rather than exhibiting balanced membership across several.

4. **Persisting challenges**
   - a. The current approach still faces challenges, particularly in terms of the chosen ratio and hyperparameters.
   - b. The identified ratio and hyperparameters have led to the creation of numerous outliers.
   - c. Further tweaking of these parameters is necessary for improved results.

5. **Increased control over hyperparameters**
   - a. Despite the persistence of challenges, the current approach offers more control over hyperparameters compared to adjusting UMAP or HDBSCAN parameters directly.
   - b. This increased control eases the refinement of the results and increases the inclusion of the data.

#### Fringe Topics

The following are the observations identified based on the fringe cluster analysis:

1. **Association with core topics**
   - a. A consistent pattern emerged where fringe topics were predominantly associated with one of the core topics identified by HDBSCAN. It was rare to find fringe topics that did not exhibit a clear association with a specific core topic.

2. **Homogeneity and core topic relationship**
   - a. The homogeneity within fringe topics was reasonably satisfactory. Messages within a fringe topic typically shared common themes and context, indicating the algorithm successfully grouped relevant discussions.
   - b. Further analysis revealed that fringe topics were often distinctive from core topics. While core topics encapsulated more central and widely discussed themes, fringe topics delved into specific nuances or subtopics related to a particular core.

3. **Multiplicity of core topic associations**
   - a. Contrary to expectations, fringe topics were not exclusively tied to multiple core topics. The majority demonstrated a more straightforward relationship, primarily aligning with one core topic. Qualitatively, domain experts can observe the multiplicity that the membership scores alone do not fully capture.
   - b. Instances where a fringe topic spanned multiple core topics were infrequent, suggesting that -1 is normally a fringe of one topic rather than multiple.

4. **Differentiation from outliers**
   - a. The analysis provided clarity on the differentiation between true fringe topics and outliers. Outliers are a collection of discussions without a clear theme.
   - b. Fringe topics may represent elements of the discourse that are less conventional or more subjective, adding a layer of diversity to the overall analysis.

#### Outlier Cluster

The residual outlier cluster — messages that remain -1 after soft clustering — is a mixed bag rather than a thematically coherent group. These messages cover a wide range of subjects with uncommon or unusual combinations of keywords and contextual information that do not fit neatly into any core cluster. They may represent genuinely ambiguous discussions, highly specific sub-topics without sufficient mass to form their own cluster, or messages whose content is too sparse to classify reliably. The parameter choice directly determines how large this residual cluster is.

---

### Pros

1. **Methodological control**
   - a. This approach provides more control over the outlier cluster because working with membership vectors is far more interpretable compared to adjusting UMAP or HDBSCAN parameters directly.

2. **Analytical usefulness**
   - a. The incorporation of fringe topics can add to the analytical framework because analysts are often interested in interdisciplinary topics that discuss various aspects.
   - b. It also captures more nuanced granular analysis which can potentially improve recall: as discussed, -1 is not always noise — it can be a fringe discussion that may or may not be of interest.

---

### Cons

1. **Methodological complexity**
   - a. Despite providing more control, the method still requires parameter tweaking, as the soft clusterer generates a significant proportion of outliers.
   - b. Rapid evaluation of hyperparameters is crucial to avoid repetitive annotation efforts and minimise overhead.

2. **Annotation complexity**
   - a. The addition of fringe topics increases the number of topics an annotator must go through, necessitating additional steps in the annotation process.

---

## K-Means Comparison

K-means was applied to the messages assigned to the -1 cluster, with k set equal to the number of topics identified by HDBSCAN. The rationale: since the -1 cluster takes the same shape as the full data (Figure 1), applying k-means with the same number of groups as HDBSCAN found provides a direct structural comparison of what each approach discovers within the outlier space. The objective is to compare and contrast the findings of k-means with the results obtained from the soft clustering approach, particularly focusing on common and distinct topics. Additionally, the aim is to determine whether k-means presents a unique perspective on the data in comparison to the fringe and outlier clusters.

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

K-means applied to the -1 cluster contributes complementary insights rather than duplicating soft clustering findings. Commonalities indicate robustness in identifying broad thematic groupings within the outlier space. Unique k-means findings suggest it recovers patterns that remain in the soft clustering residual outlier cluster (probably corresponding to the outliers in the soft clusterer) — the two approaches are more complementary than redundant.

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
