# Data Collection

Why data collection decisions are unusually consequential in topic modelling — and what to lock in before the model runs.

> **Cross-references:**
> - Core pipeline context: [`docs/workflow/`](../workflow/)
> - Sampling & extrapolation: [`docs/sampling/`](../sampling/)

---

## Overview

Data collection and topic modelling are often treated as sequential steps — collect first, model later. A more effective approach runs them in parallel: using early topic modelling to inform and validate the collection as it progresses. This surfaces data quality issues early, allows for adjustments before the collection is finalised, and avoids the situation where problems are only discovered once the full analysis is underway.

This is particularly valuable on projects with tight deadlines, where late-stage data quality issues leave little room for correction.

---

## Collection Methods

Two primary collection methods are used, each with distinct challenges and responsibilities.

### Seed list (actor-based)

Collection begins with a seed list of actors — accounts, sources, or entities identified as relevant to the project. The list may be expanded through network traversal or account crawling to broaden the scope.

**Challenges:**

- **Client-defined seed lists** — when the seed list is provided by the client, any bias in the selection is attributed to the client's choices. The analyst's responsibility is to communicate the implications of that bias clearly and to flag when the seed list produces a skewed data landscape.
- **Analyst-defined seed lists** — when the seed list is defined by the analyst, responsibility for bias shifts accordingly. This requires a more careful and deliberate selection process, with regular updates based on topic modelling insights to correct emerging biases.

**In both cases:** document who defined the seed list, when, and on what basis. This documentation is essential for interpreting topic modelling results and for communicating limitations to stakeholders.

### Keyword terms (term-based)

Collection is driven by search terms or keywords rather than a predefined set of actors. The keyword list determines what content is captured and is therefore a primary source of bias risk.

**Challenges:**

- **Ambiguity** — keywords that have multiple meanings across different contexts will pull in irrelevant content. Terms that are too specific will miss relevant content using different vocabulary.
- **Bias from keyword selection** — the framing of a keyword list reflects assumptions about how the topic is discussed. Those assumptions may not hold across all sources, languages, or time periods in the data.

**Mitigation:** Use preliminary topic modelling on an initial sample to evaluate keyword performance — whether the collected content reflects the intended scope, and whether relevant narratives are being missed. Iterate on the keyword list before committing to full-scale collection.

---

## Filtering Before Analysis

A filtering layer applied after collection but before analysis refines the dataset for relevance. This step uses keywords or classifiers to remove content that does not meet the analytical threshold, ensuring that topic modelling operates on pertinent data rather than the full raw collection.

**Key considerations:**

- **Relevance** — the goal is to remove tangential content without excluding data that may be relevant. Err toward inclusion at this stage; topics that turn out to be irrelevant can be deprioritised after modelling.
- **Keyword selection** — the same trade-off between specificity and inclusiveness applies here as in term-based collection. Over-specific filters narrow the dataset in ways that can constrain the conclusions of the analysis.
- **Iterative refinement** — the first pass of filtering will rarely be final. Initial topic modelling reveals what the filter let through and what it excluded, providing the basis for adjusting the filter criteria. Plan for at least one revision cycle.
- **Impact on analysis** — the filtering layer shapes what the topic model sees. It can restrict the conclusions that can be drawn just as much as the collection method itself. Document the filtering criteria and the reasoning behind them.
- **Stakeholder alignment** — define filtering criteria in collaboration with stakeholders where possible. Filters applied without agreement can lead to disputes later about why certain content was excluded.

---

## Running Collection and Topic Modelling in Parallel

Integrating early topic modelling into the collection process provides three concrete benefits:

**Early detection** — topic modelling on a sample of the collected data reveals data quality issues — noisy accounts, off-topic content, keyword drift — before they propagate into the full dataset. Issues found early are cheap to fix; issues found after full collection are expensive.

**Bias minimisation** — a preliminary topic model shows whether the seed list or keyword terms are producing a representative picture of the data landscape. Clusters that are unexpectedly large, small, or absent signal collection problems that can be corrected before the analysis phase.

**Efficiency** — identifying irrelevant or noisy data early avoids processing it through the full pipeline. This is particularly significant for large datasets where embedding and clustering are computationally expensive.

### Sprint-based collection structure

A sprint-based structure — collecting in batches with a topic modelling sanity check between sprints — is particularly effective for actor-based collection involving account expansion. Each expansion step introduces new accounts whose relevance is not guaranteed; a topic modelling check between sprints validates whether the expanded set is contributing relevant content or introducing noise.

---

## Collection vs. Analysis Topic Modelling

Topic modelling used during collection serves a different purpose than topic modelling used for analysis, and the two should not be confused.

**Collection topic modelling** — run on a sample or early batch of the data. Produces an abstract overview of the data landscape: broad narratives, potential quality issues, coverage gaps. Not intended for reporting. Results inform decisions about the collection, not conclusions about the subject matter.

**Analysis topic modelling** — run on the finalised, filtered dataset. Involves thorough parameter tuning, annotation, and validation. Results are used for quantitative and qualitative reporting. This is the model that gets reported.

Running collection topic modelling does not replace analysis topic modelling. The two models will differ, and the collection model's topics should not be used for reporting.

---

## Checklist

- [ ] Collection method defined: seed list (actor-based) / keyword terms / both
- [ ] Who defined the seed list or keyword terms documented (client / analyst)
- [ ] Preliminary topic modelling run on initial sample before full collection begins
- [ ] Keyword list or seed list reviewed and updated based on preliminary topic modelling
- [ ] Filtering layer defined and documented with reasoning
- [ ] Filtering criteria reviewed with stakeholders where relevant
- [ ] Collection finalised before analysis topic modelling begins — scope changes after this point require a full pipeline rerun
