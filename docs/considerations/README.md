# Methodological Considerations

This document summarises methodological considerations that have arisen across projects. It focuses on the topic modelling workflow and the translation component. It is a reflective document — the procedural how-to for each stage lives in the linked docs below; this document covers the why, the trade-offs, and the lessons learned.

**Terminology:** the *analyst* is the data science person who builds the topic model. The *annotator* is the person who characterises topics and interprets the output.

> **Cross-references:**
> [`docs/data-collection/`](../data-collection/) · [`docs/sampling/`](../sampling/) · [`docs/outliers/`](../outliers/) · [`docs/annotation/`](../annotation/) · [`docs/evaluation/`](../evaluation/) · [`docs/translation/`](../translation/) · [`docs/workflow/`](../workflow/)

---

## Data Collection

The data collection step precedes any discussion of topic modelling due to the inherent inflexibility of the approach. Topic models rely on data distribution and density, and any alterations to the data at any stage can necessitate model changes or the introduction of additional models, causing disruption to the analysis. It is crucial to ensure data collection is completed before initiating subsequent steps.

To illustrate potential disruptions: in the Gates project, the topic model pipeline was reapplied three times within a tight deadline due to the addition of accounts and an expanded time frame. In another project, a bug in one of the platforms resulted in missing and duplicated data, requiring a complete pipeline redo. While running a separate analysis to accommodate new data may seem logistically feasible, it poses significant analytical challenges.

### Analytical Challenges and the Impact on Recall

One of the primary challenges in the topic model approach is recall — we do not precisely know what we might be missing. Creating separate topic models and integrating their outputs can be problematic from a recall perspective. Although we rely on human annotations to create the thematic breakdown, those annotations are based on topics emerging from the density of the data. Comparing thematic breakdowns from different models means assessing the precision of each breakdown. However, applying one topic model across both datasets could yield a different perspective, potentially contradicting results obtained separately.

### Data Collection Validation Procedure

To mitigate disruptions caused by inaccurate data collection, the following steps are recommended:

1. **Confirmation of collection sources with the client**
   - Confirm data sources (account seed lists or specific keywords)
   - Verify the agreed time frame

2. **Volume analysis of collected data**
   - Plot and analyse the volume of data from each account to ensure no data is missing
   - Review data volume over time to detect gaps (missing data or collection failures)
   - Cross-check collected data against the agreed timeframe

3. **Handling ongoing or iterative data collection**
   - For projects with ongoing or iterative data additions, treat new data separately — existing models may not handle unseen data effectively (see [`docs/sampling/`](../sampling/))
   - Ensure the data collection phase is fully completed before proceeding

---

## Topic Modelling

This section covers considerations for each stage in the topic modelling workflow: preprocessing, topic model creation, annotation, evaluation, and analysis.

---

### Data Preprocessing

Understanding the nature of the data is an important consideration that requires methodological planning before any further steps. Two key factors arise: the size of the data and its source.

Data size directly influences the topic modelling strategy. The source — whether sentences, paragraphs, or scraped news articles — is another crucial factor. The distinction lies not in content but in sentence length, which matters particularly when selecting a sentence-transformer model. These aspects are discussed in the sections below.

#### The Size of the Data

The impact of data size on the topic modelling workflow is evident in projects where datasets were too large to be analysed by a single topic model. In those cases, the strategy involved sampling the data, training the topic model on the sample, creating a thematic breakdown, and applying it to the full dataset based on the sample-trained model.

However, this approach revealed a significant challenge — a bug emerged during the application of the sample-based topic model to the full dataset, resulting in a notable decrease in mapping accuracy (see [`docs/sampling/`](../sampling/)). In response, keyword-based filtering was introduced to the data before applying the topic model, but this required redoing the entire pipeline.

An alternative proposed in the same document is iterative transformation, which may eliminate the need for filtering. However, this solution requires validation.

Managing data size demands careful planning and clear communication with the client to prevent duplicated effort or the need to reapply the full pipeline.

#### The Source of the Data

Whether the data consists of sentences (common on Twitter and Facebook), paragraphs (potentially on Facebook and Telegram), or scraped articles (substantial text) introduces challenges in determining how to handle, represent, and communicate these decisions.

**Short-form social media:** For platforms where messages are relatively short, selecting an appropriate transformer model is crucial to meet sequence length constraints in the sentence transformer.

**Longer-form content:** In one project, the chosen model's small sequence length raised concerns about whether all message content was captured, impacting topic modelling and overall analysis. A small investigation into sentence length distribution was needed to inform the model choice.

**Articles:** Whether to feed articles unmodified to the transformer model (assuming the most valuable content is at the beginning), divide them into chunks, or aggregate paragraphs before topic modelling — each approach has trade-offs that need to align with project goals and client expectations.

In one project, article data was divided into sentences and analysis was conducted at the article level — for example, if an article contained 5 positive sentences and 3 negative, the article was classified as positive.

#### Preprocessing Validation Procedure

1. **Verification of cleaning steps** — confirm cleaning runs before removing empty strings
2. **Duplication removal** — execute within the topic model notebook to keep the process contained
3. **Sentence transformer model assessment**
   - Check sequence length against data characteristics
   - Plot sentence length distribution to identify outliers (long Facebook posts, full articles)
4. **Article processing decision** — split into sentences, chunk by sequence length, or process as a whole unit
5. **Representation of processed articles** — if split, decide whether to aggregate units by averaging or analyse individually
6. **Post-tokenisation analysis** — plot sentence length distribution before and after tokenisation to assess data loss from the model's `max_seq_len`

---

### Topic Model Creation

The topic modelling stage encompasses methodological considerations around the development environment, embedding model choice, reproducibility, and hyperparameter decisions.

#### Development Environment

The underlying reason these issues arise is that in Colab, you are relying on the environment of the instance you are working on — and you have no control over that environment. This creates a two-directional compatibility risk: Colab can update in a way that is not compatible with the version of BERTopic installed, or BERTopic can release an update that is not compatible with the current Colab environment. Either direction can silently break the ability to reload a previously saved model.

- **Colab usage:** Colab is frequently used for building topic models. Periodic system updates may enforce Python version changes, causing disruptions.
- **Model reload issues:** If a system update occurs between creating the model and applying it to the full dataset, it may prevent reloading the topic model due to dependency changes in BERTopic.
- **Library updates:** BERTopic is actively developed and releases can introduce compatibility issues.
- **Mitigation:** Document BERTopic and Python versions at topic model creation time. Begin annotation immediately after model creation — do not wait for evaluation — to maintain independence from the model environment.

#### Choice of Sentence Transformers

Evaluate sentence length distribution by applying the tokenizer of the selected sentence transformer to the data. This produces the tokenised sentence length distribution, which reveals sensitivity to the chosen `max_seq_len` (see Preprocessing Validation Procedure above).

Be cautious: sentence transformers are wrappers around transformer models, and the wrapper may specify a different `max_seq_len` than the original model on HuggingFace. For example, `all-mpnet-base-v2` specifies 128 tokens on HuggingFace but 384 on the Sentence Transformers page.

Also verify that the model was trained on a language applicable to the data being analysed.

#### Hyperparameters

There are two aspects to topic modelling hyperparameters: tuning (trade-offs between topic count and outlier volume) and reproducibility.

**Tuning**

One of the main challenges in topic modelling is reducing the number of topics while ensuring the analysis is granular enough and the volume of messages classified as -1 (outliers) remains manageable. The main hyperparameters to tweak are those associated with HDBSCAN:

1. **`min_cluster_size`** — adjusting this parameter plays a crucial role in controlling the number of topics generated. Increasing it can reduce the number of topics, but at the cost of potentially impacting the homogeneity and specificity of the resulting topics.

2. **`min_samples`** — this parameter influences the number of messages classified as -1. By adjusting it, the model becomes more or less lenient in determining whether a message is an outlier. Reducing `min_samples` makes the model more forgiving (fewer -1 messages); increasing it does the opposite.

Two additional approaches were explored specifically to address the -1 problem: soft labelling and k-means reclustering. See [`docs/outliers/`](../outliers/) for full documentation of both approaches.

**Reproducibility and Analysis**

Reproducibility in topic modelling involves not only the environment but also a careful consideration of hyperparameters. Understanding the inner workings of BERTopic is crucial for ensuring reproducibility and meaningful analysis.

BERTopic relies on two essential underlying models: UMAP and HDBSCAN.

*UMAP random state*

UMAP is a stochastic model. Setting `random_state` is essential for ensuring reproducibility when generating the topic model. This becomes particularly relevant when reapplying the topic model is necessary — for example, due to a system update in Colab.

*BERTopic model manipulation*

While BERTopic is trained based on the contextualised representation of the data, any actions related to model manipulation or visualisation are performed on the c-TF-IDF representation — a matrix representing word overlap across topics, not semantic meaning.

1. **Manipulation impact on reproducibility** — functionalities such as reducing the number of topics or specifying the number of topics involve manipulating the c-TF-IDF matrix. This manipulation can be stochastic, making it challenging to reproduce the same topic model outputs.

2. **Visualisation consistency** — visualisations produced by BERTopic are based on the same c-TF-IDF matrix. Visual insights derived from the matrix may not align with insights from the contextual embeddings, potentially leading to misleading interpretations.

Understanding these dynamics is crucial for maintaining reproducibility and interpreting visualisations accurately.

---

### Annotation Scheme

See [`docs/workflow/`](../workflow/) for an overview of the thematic annotation process and its role in the topic modelling workflow. This section covers methodological considerations specific to annotation scheme design.

Two factors most significantly influence the design and strictness of an annotation scheme:

1. **Nature of the project** — exploratory/qualitative projects can afford a more flexible annotation scheme. Projects focused on quantifying specific behaviours or narratives require more rigour.
2. **Granularity requirements** — projects requiring granular insights demand cautious decision-making and explicit agreement with the client on what level of granularity is achievable. See [`docs/annotation/`](../annotation/) for details.

The following examples from past projects illustrate how annotation schemes were adapted and what challenges emerged.

#### Example: A Disinformation Monitoring Project

**Approach:** Annotators examined a large sample (200 messages from the top 50 topics), later reduced to 5 messages per topic. A topic was classified as relevant if it contained at least one disinformation message.

**Challenges:**
- Focusing on the top 50 topics risked overlooking nuanced narratives not prevalent in the data
- Justifying a sample of 5 messages per topic was methodologically difficult
- The classification criterion (one disinformation message = relevant) appeared arbitrary

**Key lesson:** Exploratory annotation schemes need explicit, justifiable criteria for sample size and thematic assignment — even in qualitative projects.

#### Example: Gates Project

**Approach:** 10 messages per topic were prepared for the annotator to examine. A topic was assigned to a theme if more than 5 of its messages were relevant. Zero-shot outputs were also annotated for qualitative insight.

**Challenges:**
- Thematic assignment was difficult for heterogeneous topics
- Sample size justification was potentially weak, though it did not become a significant issue
- Occasional misalignment between analyst and annotator on whether findings were to be quantified or presented qualitatively, particularly in the zero-shot setting
- Foreign language content required external translators and intensive coordination; Google Translate was used as a fallback but was time-consuming and impractical

#### Annotation Scheme Methodological Considerations

1. **Analysis nature consideration** — align the annotation scheme with the project's objectives and the nature of the analysis.

2. **Flexibility in annotation criteria** — develop an annotation scheme that can adapt to the diverse nature of datasets. This includes flexibility in the number of messages sampled per topic and the criteria for thematic categorisation. A one-size-fits-all approach may not be effective given the variability in data across different projects.

3. **Clarification of annotation objectives** — ensure clear communication of the annotation objectives to all team members, especially between analysts and annotators.

4. **Sample size justification** — develop a rationale for the chosen sample size that is both methodologically sound and practical. This should take into account the project's scope, the nature of the data, and the resources available.

5. **Balancing qualitative and quantitative analysis** — establish a clear methodology for balancing qualitative and quantitative aspects in the annotation process. This involves deciding when to focus on the quantity of relevant messages versus the qualitative depth of the narratives they convey.

6. **Handling of heterogeneous topics** — create guidelines for dealing with heterogeneous topics that may not neatly fit into predefined themes. This could include strategies for theme assignment and handling ambiguous or overlapping narratives.

7. **Documentation of annotation decisions** — maintain comprehensive documentation of all annotation decisions and their justifications.

#### Proposed Annotation Scheme

A revised annotation scheme with the following properties has been proposed:

1. **Enhanced sampling strategy** — more structured sampling to ensure representative analysis and reduce the risk of unjustifiable decisions
2. **Clearer topic categorisation criteria** — a stricter set of criteria for thematic assignment
3. **Balanced qualitative and quantitative approach** — extracts granular insights while maintaining annotation quality
4. **Methodological justifiability** — the scheme is easy to explain and defend

See [`docs/annotation/`](../annotation/) for the full scheme. Note that its effectiveness still requires testing and validation.

---

### Evaluation

See [`docs/evaluation/`](../evaluation/) for a description of the current evaluation procedure and proposed approaches.

The evaluation strategy divides into two dimensions: procedure/mode and approach.

**Evaluation necessity based on project nature:**
- Exploratory/qualitative projects may not require formal evaluation, but this requires transparent communication with the client
- Quantitative projects that break data into measurable segments require evaluation to validate accuracy

**Evaluation modes:**
- General evaluation, stratified evaluation, theme/relevancy evaluation, sub-theme evaluation
- For balanced themes, a general sample mirroring theme presence in the overall dataset is often sufficient
- For imbalanced themes or specific sub-themes, stratified sampling is recommended

**Evaluation approaches:**
- *Review-based:* annotators see the topic model annotations and agree or disagree
- *Blind-based:* annotators evaluate without access to topic model annotations, reducing potential bias
- *Combined:* divides the evaluation sample into a validation set (review-based) and a test set (blind-based) — untested but proposed

---

### Analysis

The primary factor in determining the analysis approach is the nature of the project and the specific agreement with the client.

**Qualitative projects:**
- Avoid indicating quantities unless thoroughly tested and validated
- Any quantification should be approached with caution and be inferable from the adopted approach
- For topics requiring quantification, sampling can be used to divide narratives within a topic, but this requires careful interpretation, statistical validation of representativeness, and avoidance of broad generalisations across the full dataset
- Zero-shot analysis can be employed for qualitative exploration by formulating questions derived from the topic theme map. This approach shows high precision in retrieving relevant information when message content aligns closely with question vocabulary, but has limitations in semantic understanding

**Quantitative projects:**
- Quantification is limited to aspects that have undergone evaluation — generalisations beyond evaluated elements are not methodologically justified
- When data is evaluated at the theme level, quantification should be limited to that level; good theme-level performance does not imply accurate sub-theme mapping
- Insights must be directly related to the level and type of evaluation conducted

---

## Translation

See [`docs/translation/`](../translation/) for the translate-then-cluster methodology and [`multilingual-mt`](https://github.com/ay94/multilingual-mt) for translation model evaluation and benchmarking.

The translation process involves translating the source text, applying BERTopic on the translated text, and mapping the identified topics back to the source data.

### Evaluation

Translation evaluation divides into two types:

- **Intrinsic evaluation:** uses a parallel corpus to assess model performance on a given language — useful for establishing whether the model achieves reasonable baseline performance
- **Extrinsic evaluation:** evaluates translation quality on actual project data (no parallel corpus) — reveals how the model adapts to domain shift and project-specific contexts

### Language Performance Variability

Performance varies meaningfully across languages with different scripts. For example, Turkish-to-English translations tend to outperform Farsi-to-English on METEOR and BERTScore. Understanding these variations is crucial for anticipating challenges and setting appropriate expectations.

### Metrics for Evaluation

- **METEOR:** chosen for its flexibility in handling exact and partial matches, which aligns well with topic modelling's need to capture general meaning
- **BERTScore:** evaluates semantic similarity at the embedding level, mirroring the BERTopic process of clustering semantically similar content. Note that BERTScore can be less sensitive to changes — using a fine-tuned model such as `facebook/bart-large-mnli` is recommended over the default
- **XLMScore:** cross-lingual evaluation that does not require a parallel corpus — useful when gold standard translations are unavailable

See [`multilingual-mt`](https://github.com/ay94/multilingual-mt) for full metric comparisons and benchmarking results across Arabic and Turkish.

### Error Analysis

Thorough error analysis is essential, especially when working with project-specific data that lacks parallel translations. Key nuances include METEOR's sensitivity to word usage variation and word order.

### Information Loss Considerations

- **Tokenisation impact:** tokenisers tend to generate more tokens for non-Latin scripts, which can exceed `max_seq_len`, causing translation to be generated from partial input and losing context
- **Named entity handling:** person names and locations are often handled inconsistently across languages; inaccuracies here can significantly affect topic modelling quality
- **Paraphrasing:** models sometimes paraphrase in ways that lose the intended meaning — this is distinct from outright mistranslation and requires its own attention in error analysis
