# Translate-Then-Cluster: Topic Modelling on Non-English Text

## Overview

The objective of this study is to examine the capability of a machine translation tool in accurately translating Arabic text into English, to an extent where the core message of the original text is retained. The translated English is then clustered to find common topics, and those topics are applied back to the Arabic text to assess whether they make sense in the original language. In other words, we are clustering the Arabic text based on the content of its English translation, not its original content.

The **traditional approach** to topic allocation on non-English data: given an analyst studying a multilingual dataset, apply clustering directly to the non-English content using a multilingual representation model, then provide the analyst with the data breakdown.

The **proposed approach**: given a dataset that is not in English, rather than applying clustering to the non-English data using multilingual representation, translate the data into English, apply clustering to the translations, and then provide the analyst with the *original non-English data* broken down by translation-derived topics. The translation is not seen by the analyst — it operates under the hood.

The outcome of this study should show how well the translated English is producing homogeneous topics compared to the standard English approach applied in previous projects.

---

## Methodology

### Stage 1 — Translation model selection

The first stage is to choose a translation model and evaluate its output on a standard parallel corpus. For Arabic, the model `facebook/mbart-large-50-many-to-many-mmt` was selected after testing multiple candidates — most other models produced noisy output with special characters and were unusable. MBART is a many-to-many multilingual model supporting direct translation between 50 language pairs including Arabic and Farsi.

Evaluation was conducted on the OPUS parallel corpus (`opus-2019-12-18.test.txt`, 5,000 sentence pairs). Metrics: METEOR and BERTScore — see [`multilingual-mt/benchmarks/ar/`](https://github.com/ay94/multilingual-mt/tree/main/benchmarks/ar) for full model selection results, metric rationale, and error analysis.

**Key Stage 1 result:** METEOR=0.678, BERTScore=0.964 on OPUS. The high BERTScore indicates strong semantic preservation. METEOR is lower because it penalises paraphrasing and structural variation that is expected from a generative model.

### Stage 2 — Topic modelling on three setups

The second stage applies topic modelling under three parallel setups on the same dataset, then compares the homogeneity of the resulting topics.

**Dataset:** UN Parallel Corpus (`un_pc`) — manually translated UN documents from 1990–2014, covering the six official UN languages. Arabic–English split used. This corpus was chosen because it is a true parallel corpus: each Arabic sentence has a corresponding human-translated English gold standard, enabling direct three-way comparison.

**Three setups:**

| Setup | Content | Role |
|---|---|---|
| 1. Arabic content | Original Arabic text, represented with a multilingual model | Baseline: traditional multilingual clustering |
| 2. English gold standard | Human-translated English (the ideal translation) | Upper bound: best possible translation quality |
| 3. English translation | MBART-translated English | Real scenario under evaluation |

All three topic models use identical BERTopic parameters to ensure the comparison is between the content representations, not the modelling choices.

**Workflow:**
1. Generate translations for the `un_pc` Arabic data using MBART
2. Apply BERTopic to each of the three setups
3. Execute the annotation process (Stage 3)
4. Derive homogeneity findings

### Stage 3 — Annotation and homogeneity assessment

**Topic selection:** For each setup, the top 10 topics (largest, tend to be more heterogeneous) and bottom 5 topics (smallest, tend to be more homogeneous) are selected for manual annotation. A 10% sample of messages is drawn from each selected topic and saved for annotation.

**Annotation process:**
1. Read the messages in the sample
2. For each message, formulate a description of the discussed content (multiple messages may share one description)
3. Define descriptions consistently across annotators
4. Assign the appropriate description to each message

Each topic ends with a single description or a list of descriptions, plus a qualitative homogeneity score.

**Homogeneity measure — entropy:**

Homogeneity is quantified using entropy over the description distribution within each topic:

$$H(X) = -\sum_{x} p(x) \log_2 p(x)$$

where $p(x)$ is the proportion of messages assigned each description within the topic.

Maximum entropy (all descriptions equally probable) is:

$$H_{\max} = \log_2(n)$$

where $n$ is the number of distinct descriptions in the topic.

A topic is deemed **homogeneous** if its entropy falls below 50% of $H_{\max}$. Above that threshold, the topic is **heterogeneous**.

Entropy can also be expressed as a normalised value between 0 and 1:

$$H_{\text{norm}} = \frac{H(X)}{H_{\max}}$$

This allows topics with different numbers of descriptions to be compared on the same scale. A threshold of 0.5 on normalised entropy separates homogeneous from heterogeneous topics.

**Note on threshold calibration:** The 50% threshold is a starting point. It can be calibrated against previously annotated datasets where thematic coherence was assessed qualitatively — compute entropy on those annotations and use the distribution of scores to set an empirically grounded threshold for new work.

**BERTScore with XLM-RoBERTa:** Alongside entropy, BERTScore is computed using `xlm-roberta-base` as the encoder. This multilingual model allows direct comparison of Arabic messages within the same topic, providing a quantitative signal of semantic coherence independent of the annotation labels.

---

## Findings

### Translation model results (Stage 1)

MBART achieved METEOR=0.678 and BERTScore F1=0.964 on the OPUS test set. The gap between the two metrics reflects the generative nature of the model: BERTScore captures that the meaning is preserved, while METEOR penalises the paraphrasing and structural variation that a generative model naturally produces.

The rationale for using both: METEOR serves as the stringent criterion showing how far the translation diverges from the exact reference text. BERTScore aligns more closely with the topic modelling objective — semantic content matters more than surface form.

### Topic model results (Stage 2)

BERTopic topic counts across the three setups:

| Setup | Topics found |
|---|---|
| English translation | 67 |
| English gold standard | 69 |
| Arabic content | 64 |

Topic counts are similar across setups, indicating the translation preserves enough content structure for the topic model to find comparable granularity.

### Homogeneity results (Stage 3)

| Setup | Heterogeneous topics | % of annotated topics |
|---|---|---|
| English translation | 5 out of ~15 annotated | **30%** |
| English gold standard | 1 out of ~15 annotated | 6% |
| Arabic content | 1 out of ~15 annotated | 6% |

The translation setup produces more heterogeneous topics than either the English gold standard or the Arabic content directly. The English and Arabic models perform equivalently — suggesting that with high-quality multilingual representation, direct Arabic clustering is as good as English gold-standard clustering.

The translation setup introduces a meaningful homogeneity penalty: approximately 1 in 3 topics is heterogeneous, compared to 1 in 15 for the other setups. This is the primary finding: **translate-then-cluster works, but at a homogeneity cost relative to ideal conditions.**

### Shared topic discussions (Table 1)

Despite the homogeneity difference, all three setups identify overlapping thematic content:

| Translation | English gold | Arabic |
|---|---|---|
| Cost | Cost | Cost |
| United Nation | United Nation | — |
| Peace Making | Russia | Peace Keeping |
| Human Rights | Human Rights | Human Rights |
| History | Geopolitical | Era |
| Jobs | Environment | Management |
| Paragraph | Resolution | Resolution |
| Letter | Committee | Committee |
| Law | Law | Law |
| International Conference | Women | Woman Issues |
| International Law | Financial Report | Economy |
| Mix | Services | Supply |
| Phrase Group | Phrase Group | Phrase Group |
| Date Mix | Space Program | Date Mix |

Shared discussions across all three setups include Cost, Human Rights, Law, Resolution/Committee, and Phrase Group (noise). Each setup also generates unique or distinct topic labels — Peace Making (translation) vs Russia (English gold) vs Peace Keeping (Arabic) — which are related but not directly equivalent. This indicates the translation successfully preserves thematic structure at a high level, while differing in the granularity of how sub-topics are separated.

### Named entity and number handling

Manual inspection revealed that translations struggle with:
- **Named entities:** inconsistent transliteration vs semantic translation; numbers in context sometimes lose their referential precision in translation
- **Context preservation:** despite named entity errors, the surrounding thematic context is typically intact — topics 0, 8936, and 9819 showed number translation issues but the topic assignment was still correct

This is consistent with the overall finding: translation noise affects surface form but not always semantic clustering.

---

## When to use this approach

**Use translate-then-cluster when:**
- The dataset is non-English and analyst-facing topic labels need to be in English
- Direct multilingual topic models produce less interpretable keyword representations
- The team lacks annotators fluent in the source language for topic labelling

**Expect a homogeneity cost** of approximately 5× relative to direct clustering or gold-standard translation (30% vs 6% heterogeneous topics in the Arabic/English experiment). Whether this cost is acceptable depends on the use case — for exploratory analysis or broad thematic breakdown, 70% topic homogeneity is often sufficient.

**Prefer direct multilingual clustering when:**
- Translation quality for the language pair is poor (see model selection benchmarks in `multilingual-mt/`)
- Topic labels do not need to be in English
- Compute budget is limited (translation is an additional step)

---

## Related resources

| Resource | Location |
|---|---|
| Translation model selection — Arabic | [`multilingual-mt/benchmarks/ar/`](https://github.com/ay94/multilingual-mt/tree/main/benchmarks/ar) |
| Translation model selection — Turkish | [`multilingual-mt/benchmarks/tr/`](https://github.com/ay94/multilingual-mt/tree/main/benchmarks/tr) |
| Translate-then-cluster workflow notebook | [`multilingual-topic-modeling/notebooks/workflow/04_translate_and_cluster.ipynb`](https://github.com/ay94/multilingual-topic-modeling/blob/main/notebooks/workflow/04_translate_and_cluster.ipynb) |
| Translator class | [`multilingual-topic-modeling/multilingual_topic/translation.py`](https://github.com/ay94/multilingual-topic-modeling/blob/main/multilingual_topic/translation.py) |
| Evaluation workflow | [`docs/evaluation/`](../evaluation/) |
| Annotation process | [`docs/annotation/`](../annotation/) |
