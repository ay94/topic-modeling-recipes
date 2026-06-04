# Evaluation Framework

> **Status:** Coming soon

## What this covers

How to evaluate the quality of thematic allocation — when to evaluate, what metrics to use, and how to handle the bias introduced by review-based annotation.

**Core ideas:**
- Two-phase evaluation — Phase 1: stratified sample per sub-theme (identifies underperforming sub-themes); Phase 2: random blind sample across all sub-themes (precision, recall, F1)
- Modified blind evaluation — divide the sample into a validation set (reviewer sees topic model output) and a test set (blind); if validation passes, proceed to blind test
- When to skip evaluation — exploratory projects may not require quantitative evaluation if findings are presented transparently as qualitative; quantitative projects should limit claims to evaluated levels only
- Multiple label tolerance — allows up to 3 label choices per message to capture inherent thematic overlap

**Metrics:**
- Precision: TP / (TP + FP)
- Recall: TP / (TP + FN)
- F1: 2 × (Precision × Recall) / (Precision + Recall)
