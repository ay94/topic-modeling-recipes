# Layered Topic Modelling

Strategies for managing heterogeneous topics and achieving finer thematic granularity through successive rounds of topic modelling.

> **Cross-references:**
> - Core pipeline context: [`docs/workflow/`](../workflow/)
> - Guided topic modelling: [`docs/guided/`](../guided/)
> - Outlier mitigation: [`docs/outliers/`](../outliers/)

---

## The Problem

A single-pass topic model surveys the full data landscape and groups content into broad clusters. Not every topic that emerges will directly correspond to a predefined analytical category — some will be thematically coherent from the model's perspective but analytically heterogeneous, mixing distinct narratives that need to be reported separately. Treating these as a single topic loses analytical resolution; discarding them loses relevant content.

---

## Traditional Filtering Approaches

The conventional response to heterogeneous topics is to apply classifiers after the fact — keyword matching or machine learning-based classifiers that filter thematic data at the topic, theme, or subtheme level depending on the project's focus.

These approaches work but carry two limitations. First, they rely on the preliminary definition of classifiers, which requires the analyst to know in advance what to look for. Second, they can lack the flexibility needed for complex datasets where the relevant distinctions are not cleanly separable by keywords or pre-trained classifiers.

Multilayered topic modelling addresses both limitations by using the data's own structure — rather than external classifiers — to drive the refinement.

---

## The Approach

Multilayered topic modelling applies successive rounds of topic modelling to progressively refine structure. Each layer targets a specific part of the data landscape, producing sub-topics that are then integrated into a unified annotation schema.

### Initial exploration — Layer 1

An initial round of topic modelling surveys the full corpus, identifying broad topics and organising them into preliminary themes. This layer is not expected to produce analytically final clusters — its purpose is to map the data landscape and identify which areas require further investigation.

Topics that are clearly irrelevant are set aside. Topics that are relevant but heterogeneous — semantically broad or analytically mixed — are carried forward to Layer 2.

### Refinement — Layer 2

Each heterogeneous topic from Layer 1 is modelled independently. The messages belonging to that topic are extracted and a new clustering run is applied, typically with adjusted UMAP and HDBSCAN parameters suited to the smaller sub-corpus. This produces sub-topics specific to that topic's content.

This iterative process unpacks complex topics into more manageable and analytically relevant segments — surfacing granular distinctions that a single-pass model cannot resolve.

### Deep refinement — Layer 3 (optional)

For topics that remain too dense or complex after Layer 2, a third layer can be applied to specific sub-topics. This is typically reserved for clusters where two or more analytically distinct narratives are confirmed to be present but cannot be separated at Layer 2 resolution.

### Integration — annotation schema

A unified annotation schema bridges topic information across all layers, ensuring that thematic annotations are consistently applied and interpretable regardless of which layer a topic came from. Developing this schema is the most analytically demanding part of the multilayered approach — it requires deciding how Layer 1 topics, Layer 2 sub-topics, and any Layer 3 deep-topics relate to each other and to the project's reporting categories.

### Layer-specific semantic maps

Each layer maintains its own UMAP semantic map. Mixing coordinates across layers produces misleading visualisations — the UMAP projection from a full-corpus run and a sub-corpus run are not comparable. Keeping maps separate preserves the interpretability of each layer's visual output.

---

## Challenges

**Annotation schema design** — the cross-layer schema must bridge topics across layers without losing analytical coherence. A schema that works for Layer 1 may not accommodate the finer distinctions that emerge at Layer 2, requiring iterative revision as modelling progresses.

**Cross-layer annotation effort** — annotating at multiple layers is substantially more time-consuming than single-pass annotation. Each layer requires its own sampling, description, and categorisation pass, and the results must be reconciled into the unified schema.

**Semantic landscape distinctions** — the UMAP space at each layer reflects a different sub-corpus. Analysts need to interpret each map on its own terms rather than assuming that the same region of the space means the same thing across layers. This requires careful planning and explicit documentation of which layer each semantic map belongs to.

---

## Advantages

**Analytical alignment** — by progressively refining topics, this approach allows for a closer alignment with specific analytical objectives than traditional classifiers can achieve. The distinctions that matter to the analysis emerge from the data itself rather than being imposed by a pre-defined classifier.

**Flexibility** — multilayered topic modelling adapts to the actual complexity of the data. Layers are added where the data requires them, not uniformly across all topics. A corpus with one complex cluster and twenty clean ones needs a second layer on that one cluster, not on all twenty.

**Depth of insight** — successive modelling layers uncover nuanced sub-narratives that a single-pass model would merge into a single undifferentiated topic. This is particularly valuable when the analytical goal is to distinguish between closely related but meaningfully different narratives within the same broad theme.

---

## Relationship to Guided Topic Modelling

SetFit-based contrastive learning can be used between layers to steer the embedding space toward analytically meaningful cluster boundaries before a refinement layer is applied. This is particularly useful when Layer 2 clusters remain entangled and parameter tuning alone cannot separate them. See [`docs/guided/`](../guided/) for the full approach.

---

## Checklist

- [ ] Layer 1 model run and topics inspected
- [ ] Relevant but heterogeneous topics identified for Layer 2
- [ ] Layer 2 run independently per heterogeneous topic with adjusted parameters
- [ ] Layer 2 sub-topics annotated and described
- [ ] Cross-layer annotation schema drafted and validated against both layers
- [ ] Layer-specific semantic maps kept separate and labelled by layer
- [ ] Layer 3 applied where Layer 2 sub-topics remain analytically unresolvable
- [ ] Final annotation schema reconciles all layers into a coherent reporting structure
