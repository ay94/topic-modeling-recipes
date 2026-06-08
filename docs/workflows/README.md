# Topic Modelling Workflows

This document describes the analytical workflows that can be proposed through topic modelling across various projects. It distinguishes between general exploration and more rigorous qualitative and quantitative analyses, detailing the methodologies, component variations, and architectural patterns involved. The aim is to provide a structured approach that can adapt based on project requirements.

> **Cross-references:**
> [`docs/workflow/`](../workflow/) · [`docs/annotation/`](../annotation/) · [`docs/evaluation/`](../evaluation/) · [`docs/guided/`](../guided/) · [`docs/layered/`](../layered/)

---

## Topic Modelling Architecture

Before discussing the various workflows, it is essential to understand the architecture of topic modelling. It can be dissected into several core components — some universal across all projects, others tailored to specific requirements.

### Generic Components

These components are present in every topic modelling project, though their internal implementation may vary:

1. **Data preprocessing** — cleaning and preparing the data for analysis. The extent and methods may vary depending on the data and project needs, but this step is always present.
2. **Embedding extraction** — transforming textual data into semantic representations using sentence transformers.
3. **Parameter tuning** — adjusting UMAP and HDBSCAN parameters to optimise clustering performance.
4. **Topic model creation** — feeding the prepared and embedded data into the topic modelling algorithm to identify underlying topics.
5. **Storage** — the final model and its outputs are stored for further analysis. What is stored may differ between projects, but storage is always required.

### Project-Specific Components

These components are selectively applied depending on the project's requirements and goals:

1. **Thematic mapping** — categorising discovered topics into a broader thematic breakdown. Requires domain expertise and is guided by the project's analytical objectives.
2. **Evaluation** — assessing the model's effectiveness and accuracy. See [`docs/evaluation/`](../evaluation/).
3. **Guided topic modelling** — for projects with specific analytical goals (such as uncovering particular narratives), embedding models can be fine-tuned using targeted techniques such as contrastive learning. This adjusts the model to focus more precisely on the project's areas of interest. See [`docs/guided/`](../guided/).
4. **Analysis and reporting** — the final stage where results are analysed to derive insights. The depth and style vary significantly: from exploratory summaries to detailed thematic analyses.

While the generic components form the backbone of any topic modelling project, the intricacies of their implementation differ based on each project's demands. Project-specific components are applied selectively based on requirements and goals.

---

## Adapting Components to Project Goals

Both generic and project-specific components may undergo internal modifications driven by the project's analytical goals and data characteristics.

### Project Proposal Variations

**General exploration workflow** — provides clients with data segmented by topic modelling, without further analysis. Often a pilot or minimally budgeted initiative. Variations may aim at enhancing client capabilities such as improving search functionality via keyword identification, or generating training examples for classifiers. These proposals leverage topic modelling for its clustering and data segmentation capabilities.

**Qualitative and quantitative projects** — focus on accurately analysing specific datasets or events over defined periods, providing precise quantifications that aid and direct qualitative analysis. Projects may be guided by predetermined themes or hypotheses. Topic modelling serves either as a direct classifier or as a tool for dividing data into a thematic breakdown that can then be refined via classifiers.

### Component Flexibility

The differentiation in project types requires flexibility within the topic modelling components. Whether generic or project-specific, components need to be adaptable to the unique demands of each project's scope, data nature, and analytical objectives.

---

## Generic Components and Their Internal Modifications

**Data preprocessing**
- *Standard processes:* typically includes removing links, emojis, etc. Specific adjustments may be necessary for multilingual datasets, where preprocessing is applied based on language differences.
- *Metadata integration:* incorporating metadata such as country or platform annotations requires a flexible schema that can accommodate different data types while maintaining a consistent output format.

**Embedding extraction**

Changes here relate to data source and language:
- *Multilingual data:* requires a decision between a language-specific model per language, a generic multilingual model, or translating all data into a unified language and selecting a model for that language.
- *Data source:* social media data tends to have restricted character length; articles are long-form and require models capable of handling longer context. Some models also require specific preprocessing — for example, `nomic-ai/nomic-embed-text-v1` requires a prefix added to each sentence before processing.

**Parameter tuning**

While tuning of most parameters remains consistent across projects, the choice of clustering algorithm may vary depending on specific project needs. See [`docs/considerations/`](../considerations/) for details.

**Topic model creation**

The process generally remains the same. However, thematic allocation may require generating sample data for annotation. In projects focused solely on topic segmentation without thematic analysis, this step may not apply. Regardless of project type, annotating data with topic information is a standard requirement.

**Storage**

Essential elements — embeddings, the topic model, and annotated data — are stored universally. Sample storage is project-dependent, primarily required when thematic allocation is involved.

---

## Project-Specific Components and Their Internal Modifications

**Thematic mapping**
- *Basic sampling:* for some projects, selecting a small number of messages per topic (e.g. 10) for preliminary analysis may be sufficient, though this generally requires several rounds of refinement.
- *In-depth analysis:* more detailed projects may require examining a larger proportion of each topic's content (e.g. 10%) to gain a deeper understanding. This demands more time and analytical effort but is more flexible — making use of metrics such as entropy and silhouette scores to identify when analysis has reached a point of diminishing returns. See [`docs/annotation/`](../annotation/).

**Evaluation**

The evaluation shifts from assessing basic topic relevance in simpler projects to conducting detailed evaluations of thematic and sub-thematic layers in more complex analyses. The aspects being evaluated vary, but the underlying evaluation criteria remain constant. See [`docs/evaluation/`](../evaluation/).

**Guided topic modelling**

Only applicable when the project is provided with a predefined set of narratives or themes to identify. The main variation is whether SetFit contrastive learning is used or an alternative guided approach. See [`docs/guided/`](../guided/).

**Analysis and reporting**

A constant across all workflow types, varying significantly in scope and depth:
- *Methodology focus:* in some projects, analysis and reporting focus solely on the methodology, with topic modelling as one phase among many rather than the ultimate objective.
- *Thematic analysis:* other workflows require a more in-depth approach, incorporating both methodology and thematic analysis — a blend of qualitative and quantitative examination.

---

## Architecture Flexibility

The following sections describe how the component set changes depending on the type of project and its objectives. Each workflow type corresponds to a different architectural pattern.

### General Exploration Workflows

#### Data Segmentation

This workflow primarily uses generic components to provide clients with data segmented by topic modelling, without further analysis. It typically integrates into broader projects rather than existing as a standalone initiative.

*Components:* preprocessing → embedding → parameter tuning → topic model creation → storage

<!-- Figure 2: Data Segmentation Architecture -->

#### Active Learning

Topic modelling here serves a dual purpose: segmenting data for better understanding and identifying training examples to enhance filtering algorithms. This could involve identifying relevant keywords within topics, or extracting sentences valuable for training classifiers. The architecture incorporates active learning strategies and may optionally integrate guided topic modelling and evaluation.

*Components:* preprocessing → embedding → parameter tuning → topic model creation → storage → *(optional)* guided topic modelling → *(optional)* evaluation

<!-- Figure 3: Active Learning Architecture -->

---

### Qualitative vs Quantitative Workflows

These workflows balance qualitative insights with quantitative analysis.

- **Qualitative emphasis** — necessitates thematic mapping, with sampling strategies reflecting the depth of qualitative analysis desired. Layered topic modelling or zero-shot approaches may be used to dissect heterogeneous topics further. See [`docs/layered/`](../layered/).
- **Quantitative emphasis** — prioritises the evaluation component, dedicating substantial effort to validating the model's effectiveness. Topic modelling serves as a classifier; evaluation is essential.

In some cases, keyword-based classifiers are used to refine the content of heterogeneous topics. The choice of classifier varies depending on topic complexity — keywords, zero-shot, or layered topic modelling are all options depending on how homogeneous the resulting topics need to be.

*Components:* preprocessing → embedding → parameter tuning → topic model creation → thematic mapping → *(optional)* guided topic modelling → evaluation → analysis and reporting

<!-- Figure 4: Qual vs Quant Architecture -->
