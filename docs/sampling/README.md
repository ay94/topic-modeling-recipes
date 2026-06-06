# Sampling & Extrapolation

How to handle datasets too large to model in full — and the critical problem that arises when a topic model trained on a sample is applied to the full corpus.

> **Cross-references:**
> - Core pipeline context: [`docs/workflow/`](../workflow/)
> - Outlier mitigation: [`docs/outliers/`](../outliers/)

---

## The Problem

When working with large datasets, it is common practice to train the topic model on a sample and then apply the trained model to the full corpus. This works well in many contexts, but with UMAP + HDBSCAN it introduces a silent failure mode: **topic definitions shift significantly when the model is applied to data outside its training distribution**.

The failure is silent because initial evaluation on the training sample shows high performance. The degradation only becomes apparent when the model is applied to the full dataset — at which point analyst-defined topic descriptions may no longer match what the model is actually assigning to those topics. A topic initially defined as covering one specific narrative can expand to absorb a substantially different set of messages in the broader corpus.

**On the training sample** — topics are compact and well-separated in the UMAP space:

![Topics on training sample](figures/training-sample.png)

**On the full corpus** — the same topics have expanded dramatically and their boundaries have collapsed into each other:

![Topic drift on full corpus](figures/full-corpus-drift.png)

The shift between these two states is the extrapolation problem.

In practice this has been observed as a significant issue on large datasets and a lesser but still present issue on moderately large ones. It is one of the most common silent failures in applied topic modelling.

---

## Why It Happens

### UMAP's extrapolation limitation

UMAP learns a reduced embedding space from the training data, preserving the density structure of that specific sample. When new data points are projected into this learned space, UMAP must place them based on a latent structure it never saw — it was not designed for extrapolation.

Two UMAP parameters make this especially sensitive:

**`n_components`** — the dimensionality of the reduced space. A value chosen to capture the structure of a 100K training sample may dramatically underfit a 1M document corpus. The latent space is too simple to represent the broader data, causing messages with different content to collapse into the same region.

**`n_neighbors`** — controls the balance between local and global structure. Lower values preserve fine-grained local relationships in the training data but generalise poorly to unseen data that introduces new local structures. Higher values emphasise global structure and may generalise better to diverse larger datasets, but can obscure important local distinctions.

### HDBSCAN's fixed-cluster prediction

HDBSCAN is not designed for online prediction. When applied to new data points, it fixes the condensed tree learned during training and determines where each new point falls within that existing structure. This means:

1. Clusters are defined entirely by the training sample's density structure
2. New data points that fall in regions of the space that were sparse during training get assigned to whatever cluster boundary they happen to be nearest — regardless of whether that assignment is semantically meaningful
3. If the full corpus has substantially different density patterns from the training sample (which it typically does, being much larger), the cluster assignments for new points can be systematically wrong

The combined effect: UMAP places new points in an oversimplified latent space, and HDBSCAN assigns them to clusters that were calibrated for a different data distribution.

---

## Solutions

### Solution 1 — Keyword Filtering (recommended)

Reduce the full dataset before training, so the model is trained and applied to data of comparable size and distribution. Two variants:

**Keyword-based approach:**
1. Randomly sample a small subset of the full dataset
2. Annotate to identify the relevant discussions of interest
3. Extract keywords that represent those discussions
4. Use those keywords to retrieve all matching documents from the full corpus
5. Train and evaluate the topic model on this filtered dataset

**Topic-model-based approach:**
1. Train a preliminary topic model on a sample
2. Identify which topics contain relevant discussions
3. Extract representative keywords from those topics
4. Discard the preliminary model — use only the keywords
5. Filter the full corpus using the keywords
6. Train a new topic model on the filtered corpus

The keyword filtering approach is the safer option in most cases. By narrowing the search space to documents that are relevant, the training and application data are drawn from the same distribution, eliminating the extrapolation problem.

---

### Solution 2 — Incremental Transformation

> **Status:** Proposed — requires further testing before production use.

Rather than applying the trained model to the full corpus in one pass, divide the unseen data into batches approximately the same size as the training data and transform each batch separately.

For example: a model trained on 100K documents applied to 1M documents would process 10 batches of 100K. Each batch is transformed independently, keeping the input size consistent with what the model was trained on.

**Sampling within batches:** The training sample must itself be representative. If the full corpus is divided into chunks, sample from each chunk proportionally until reaching the training cap — rather than sampling all training data from one region of the corpus.

**Tradeoff:** Incremental transformation adds computational and maintenance overhead. It does not fully eliminate the extrapolation problem — it reduces it by keeping batch sizes closer to training size — but it does not guarantee that each batch has the same density structure as the training data.

---

## Decision Guide

```mermaid
flowchart TD
    A([Large dataset]) --> B{Can full corpus\nfit in training?}
    B -->|Yes| C["Train on full corpus\n— no extrapolation risk"]
    B -->|No| D{Is there a clear\nrelevance filter?}
    D -->|Yes| E["Keyword filtering\n— reduce corpus first\nthen train"]
    D -->|No| F{Batch processing\nfeasible?}
    F -->|Yes| G["Incremental transformation\n— batch ≈ training size"]
    F -->|No| H["Sample + train\n⚠️ Validate carefully\non held-out full-corpus sample"]

    C --> Z([Proceed])
    E --> Z
    G --> Z
    H --> I{Topic definitions\nstable on full corpus?}
    I -->|Yes| Z
    I -->|No| E

    classDef process fill:#ffffff,stroke:#1B1A18,stroke-width:1.5px,color:#1B1A18
    classDef decision fill:#C8A876,stroke:#1B1A18,stroke-width:1.5px,color:#1B1A18,font-weight:bold
    classDef terminal fill:#1B1A18,stroke:#1B1A18,color:#ffffff,stroke-width:1.5px
    classDef warning fill:#F4EFE5,stroke:#1B1A18,stroke-width:1.5px,color:#1B1A18

    class C,E,G process
    class B,D,F,I decision
    class A,Z terminal
    class H warning
```

---

## Checklist

- [ ] Dataset size assessed relative to computational budget before choosing a training strategy
- [ ] If training on a sample: extrapolation risk acknowledged and a mitigation strategy chosen
- [ ] If using keyword filtering: keywords validated as representative before filtering the full corpus
- [ ] If using incremental transformation: batch size matched to training corpus size
- [ ] Topic definitions validated on a held-out sample of the full corpus before final annotation

---

## References

- [UMAP transform documentation](https://umap-learn.readthedocs.io/en/latest/transform.html)
- [HDBSCAN prediction tutorial](https://hdbscan.readthedocs.io/en/latest/prediction_tutorial.html)
