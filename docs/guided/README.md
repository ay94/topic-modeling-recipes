# Guided Topic Modelling

How to steer the embedding space toward analytically meaningful cluster boundaries — rather than accepting whatever structure emerges from the data.

> **Cross-references:**
> - Core pipeline context: [`docs/workflow/`](../workflow/)
> - Layered modelling: [`docs/layered/`](../layered/)

---

## The Problem

Traditional sentence transformers are trained for sentence similarity or paraphrasing tasks. They produce embeddings that reflect general semantic similarity — but general semantic similarity is not always what an analysis needs. Two messages can be semantically close (same topic, similar vocabulary) while being analytically distinct, or semantically distant while belonging to the same analytical category.

When HDBSCAN clusters these embeddings, it identifies density-based structure in the learned space. That structure reflects the model's training objective, not the analyst's framework. The result is clusters that are coherent from the model's perspective but misaligned with the analysis: merged clusters that should be separate, noisy clusters that absorb off-topic content, or a flat structure that fails to reflect a known thematic hierarchy.

Guided topic modelling addresses this by adjusting the embedding space itself — before clustering — so that the model's notion of similarity better matches the analyst's.

---

## The Approach

The core mechanism is contrastive learning: fine-tuning a sentence transformer using explicit positive and negative labels derived from cluster output.

Positive pairs are messages that belong together analytically. Negative pairs are messages that should be separated. The model is trained to bring positives closer together in the embedding space and push negatives apart. This reshapes the space so that subsequent UMAP reduction and HDBSCAN clustering produce boundaries that reflect the intended analytical distinctions rather than the model's default ones.

In practice this is implemented via **SetFit** — a few-shot fine-tuning approach that requires only a small number of labelled examples (typically 8–16 per class) drawn from existing cluster output. No large labelled dataset is required; the cluster output itself provides the training signal.

---

## When to Use It

### Untangle cluttered clusters

HDBSCAN identifies a cluster that, on inspection, contains two or more analytically distinct narratives. The messages are semantically close enough that the model cannot separate them, but an analyst can clearly distinguish them.

Label a sample of messages from the merged cluster as positive (the target narrative) or negative (everything else). Fine-tune on these pairs. The adjusted embedding space separates the narratives into distinct regions, which HDBSCAN can then identify as separate clusters in the next modelling round.

### Clean noisy clusters

A cluster contains a core narrative but absorbs a significant volume of off-topic content — messages that are loosely associated with the theme but analytically irrelevant. The `-1` outlier rate is acceptable overall but the cluster's internal precision is low.

Label the core content as positive and the noise as negative. Fine-tuning tightens the cluster's boundary in the embedding space, making it denser and more coherent. HDBSCAN produces a smaller, cleaner cluster with a higher proportion of genuinely relevant content.

### Force analytic semantic structure

The analysis has a predefined thematic framework — categories agreed with a client, a classification scheme from a prior study, or a set of hypotheses to test against the data. The goal is not to discover what emerges but to assess how the data distributes across known categories.

Label examples from each category. Fine-tuning orientates the embedding space toward these distinctions. Topic modelling then surfaces clusters that map onto the predefined framework rather than onto whatever organic structure the data contains.

---

## Practical Notes

**Few-shot is sufficient.** SetFit is designed for low-resource settings. 8 positive and 8 negative examples per class are typically enough to produce meaningful embedding adjustment. The cluster output from a prior modelling run is the natural source for these examples — no separate annotation effort is needed.

**This fits between layers.** Guided modelling is most naturally used between topic model layers: run a first-pass model, identify a cluster that needs refinement, fine-tune on examples from that cluster, rerun the model on the sub-corpus with adjusted embeddings. See [`docs/layered/`](../layered/) for the full layered approach.

**It does not replace parameter tuning.** If a cluster is cluttered because UMAP `n_components` is too low or HDBSCAN `min_cluster_size` is too large, parameter tuning will resolve it more efficiently. Guided modelling is for cases where parameter changes do not help — where the issue is in the embedding space rather than the clustering configuration.

**Performance.** In production use on a 10-fold cross-validation:

| Metric | Mean | Min | Max |
|---|---|---|---|
| Accuracy | 0.861 | 0.816 | 0.931 |
| Precision | 0.857 | 0.828 | 0.924 |
| Recall | 0.855 | 0.816 | 0.928 |
| F1 | 0.854 | 0.818 | 0.926 |

Human validation agreement on held-out data: **0.90 accuracy**.

---

## Checklist

- [ ] First-pass topic model run and clusters inspected
- [ ] Problem type identified: cluttered / noisy / predefined framework
- [ ] Positive and negative examples sampled from cluster output (8–16 per class minimum)
- [ ] SetFit fine-tuning run and embedding space validated before rerunning topic model
- [ ] Post-refinement clusters inspected and compared to pre-refinement output
- [ ] If used between layers: annotation schema updated to reflect refined cluster boundaries
