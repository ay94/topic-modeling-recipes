# Evaluation Framework

How to assess the performance of thematic allocation — what modes to use, when each is appropriate, and a hybrid approach that balances bias control with analytical efficiency.

> **Cross-references:**
> - Core pipeline context: [`docs/workflow/`](../workflow/)
> - Annotation context: [`docs/annotation/`](../annotation/)

---

## Evaluation Workflow

```mermaid
flowchart TD
    A([Annotated data ready]) --> P1

    subgraph Phase1["Phase 1 — Subtheme validation"]
        P1["Stratified sample\n50 messages per subtheme"] --> P2{Precision ≥ 0.7\nper subtheme?}
        P2 -->|Yes — all subthemes| P3([Pass to Phase 2])
        P2 -->|No| P4{Remediation}
        P4 --> P4a["Remove subtheme\n— label topics irrelevant"]
        P4 --> P4b["Isolate underperforming\ntopic within subtheme"]
        P4 --> P4c["Merge with\nadjacent subtheme"]
        P4a --> P1
        P4b --> P1
        P4c --> P1
    end

    P3 --> B

    subgraph Phase2["Phase 2 — Full evaluation"]
        B{Balanced themes\n& sub-themes?}
        B -->|Yes| C["General sampling\n— proportional draw"]
        B -->|No| D["Stratified sampling\n— oversample small sub-themes"]
        C --> E
        D --> E

        E{Confidence in\ntopic theme map?}
        E -->|High| F["Small validation subset\n+ larger blind test subset"]
        E -->|Low / uncertain| G["Larger validation subset\n+ smaller blind test subset"]
        F --> H
        G --> H

        H["Validation subset\n— Review mode\nAnnotations visible"]
        H --> I{Validation\nperformance\nacceptable?}
        I -->|No| J["Revise thematic\nallocation"]
        J --> A
        I -->|Yes| K

        K["Test subset\n— Blind mode\nAnnotations hidden"]
        K --> L{Blind\nperformance\nacceptable?}
        L -->|No| M["Diagnose:\nreview test subset\nfor failure patterns"]
        M --> J
        L -->|Yes| N([Proceed to analysis])
    end

    classDef process fill:#ffffff,stroke:#1B1A18,stroke-width:1.5px,color:#1B1A18
    classDef decision fill:#C8A876,stroke:#1B1A18,stroke-width:1.5px,color:#1B1A18,font-weight:bold
    classDef terminal fill:#1B1A18,stroke:#1B1A18,color:#ffffff,stroke-width:1.5px
    classDef action fill:#F4EFE5,stroke:#1B1A18,stroke-width:1.5px,color:#1B1A18
    classDef remediation fill:#F4EFE5,stroke:#1B1A18,stroke-width:1px,color:#1B1A18

    class C,D,F,G,H,K,M,P1 process
    class B,E,I,L,P2,P4 decision
    class A,N,P3 terminal
    class J,P4a,P4b,P4c remediation
```

---

## Overview

Evaluation runs in two phases. Phase 1 is a subtheme-level precision gate that must be passed before Phase 2 begins. Phase 2 is the full evaluation — sampling strategy, mode, and complete metrics. The goal of the evaluation stage is to assess the performance of thematic allocation — confirming that messages are correctly assigned to their themes and sub-themes. Evaluation serves a dual purpose: it is both a performance measurement (precision and recall) and a review process that enables analysts to develop a comprehensive understanding of the data, identify stable thematic categories, and flag areas of noise in the annotations. This iterative process often surfaces labelling inconsistencies that require corrective action — a pattern observed consistently across projects.

---

---

## Phase 1 — Subtheme Validation

Before running the full evaluation, each subtheme is checked individually. A stratified sample of **50 messages per subtheme** is drawn and evaluated for precision only — the question is whether the messages assigned to a subtheme actually belong to it.

**Threshold: 0.7 precision.** Subthemes that meet or exceed 0.7 pass. Subthemes below 0.7 require remediation before Phase 2 begins.

### Remediation for underperforming subthemes

**Remove the subtheme** — if the subtheme cannot be made coherent, label all its constituent topics as irrelevant and remove it from the topic theme map. Apply this when the subtheme is too heterogeneous to be analytically useful.

**Isolate the underperforming topic** — if the subtheme contains multiple topics and one is dragging overall precision below threshold, identify it by reviewing which topic contributes the most irrelevant-labelled messages and remove that topic only. The remaining topics may then pass.

**Merge with an adjacent subtheme** — if many messages in the underperforming subtheme are being assigned to a neighbouring subtheme during evaluation, the two subthemes likely overlap too much to be meaningfully distinct. Merge them and re-evaluate.

> Phase 1 is a gate, not a full evaluation pass. Once all subthemes meet the threshold, proceed to Phase 2.

---

## Phase 2 — Full Evaluation

## Evaluation Modes

Evaluation data can be presented to analysts in two ways.

### Review mode
Analysts are shown the topic model annotations alongside each message. They agree or disagree with each assignment. Because they can see the model's output, they immediately understand where the thematic breakdown is working and where it is noisy, and can identify what corrective action is needed. This was the standard approach across the majority of projects.

### Blind mode
Annotations are hidden. Analysts read the message and independently assign a theme and sub-theme. Their assignments are then compared against the model's. This eliminates anchoring bias — analysts cannot defer to the model's output.

The cost: if performance is low, analysts must go through the same sample two or three times — once to evaluate, then again to identify the specific misallocations, then again to address them. In review mode, problematic annotations are identified and understood in a single pass.

**The bias concern:** In practice, showing analysts the topic model annotations can influence their judgements — they are more likely to agree with the model's assignment than to flag it, inflating apparent precision. This is a known risk of review mode that should be weighed against its efficiency advantages.

---

## Sampling Strategies

Evaluation can be run at two hierarchical levels — **theme** and **sub-theme** — and with two sampling strategies.

### General sampling
Draws a representative sample proportional to the theme and sub-theme distribution in the full annotated dataset. Appropriate when themes are balanced — when no single theme dominates to the point that a proportional sample would underrepresent others. Produces a precision and recall estimate reflecting overall performance.

### Stratified sampling
Oversamples from specific themes or sub-themes — typically the smaller or less-represented ones. Appropriate when the analytical focus is on those underrepresented sub-themes, or when a balanced evaluation is needed for reporting purposes despite a skewed underlying distribution. Answers the question of whether the model performs consistently across the full thematic breakdown, not just well on the largest themes.

> **Note:** The choice of sampling strategy should be agreed before evaluation begins. Changing it after initial results have been produced makes it difficult to compare across evaluation rounds.

---

## The Tradeoff

| | Review mode | Blind mode |
|---|---|---|
| Bias risk | Higher — analyst may anchor to model output | Lower — independent judgement |
| Efficiency | Higher — issues identified in a single pass | Lower — may require multiple passes |
| Analytical insight | Higher — analyst understands failure modes immediately | Lower — requires additional review pass to diagnose |
| Best for | Projects with uncertainty about annotation robustness | Projects with high confidence in the topic theme map |

---

## Recommended Approach: Hybrid Evaluation

The proposed resolution combines the strengths of both modes by dividing the evaluation sample into two subsets.

**Step 1 — Validation subset (review mode)**
A portion of the sample is shown to analysts with annotations visible. This is the review pass — analysts check for labelling inconsistencies, identify noise, and build their understanding of the topic structure.

**Step 2 — Gate check**
If performance on the validation subset is satisfactory, this indicates the thematic breakdown is robust enough to proceed. If performance is low, corrective action is taken before continuing.

**Step 3 — Test subset (blind mode)**
The remaining portion is evaluated without annotations. This produces an unbiased precision and recall estimate at theme or sub-theme level.

This approach is more efficient than a fully blind evaluation because corrective actions are identified during the validation pass, concurrently with the evaluation — rather than requiring an additional pass after a low blind result.

**Key decision variable:** Analyst confidence in the topic theme map. If the map is clear and topics are well-characterised, the test subset can be proportionally larger. If there is uncertainty about annotation robustness, increase the validation subset before moving to blind.

---

## Metrics

| Metric | Formula |
|---|---|
| Precision | TP / (TP + FP) |
| Recall | TP / (TP + FN) |
| F1 | 2 × (Precision × Recall) / (Precision + Recall) |

Evaluation is conducted at the agreed hierarchical level — theme or sub-theme. Quantitative analysis claims should be limited to evaluated levels only: do not report sub-theme precision if evaluation was run at theme level.

> **Multiple label tolerance:** In some projects, analysts are permitted up to 3 label choices per message to capture inherent thematic overlap. Where this is used, it should be agreed and documented before evaluation begins, as it affects how precision and recall are calculated.

---

## When to Skip Evaluation

Exploratory projects focused on qualitative narrative discovery may not require quantitative evaluation if:
- Findings are presented transparently as qualitative observations
- No quantitative claims about theme volumes or model precision are made in the output

If quantitative claims appear in the output, evaluation is required regardless of project type.

---

## Checklist

**Phase 1**
- [ ] 50-message stratified sample drawn per subtheme
- [ ] Precision calculated per subtheme
- [ ] All subthemes ≥ 0.7 — or remediation applied (remove / isolate / merge)
- [ ] Topic theme map updated to reflect any removals or merges

**Phase 2**
- [ ] Evaluation mode agreed: review, blind, or hybrid
- [ ] Sampling strategy agreed: general or stratified
- [ ] Evaluation level agreed: theme or sub-theme
- [ ] Sample size sufficient for the chosen strategy
- [ ] Multiple label tolerance policy agreed if applicable
- [ ] Analysis claims limited to evaluated levels only
