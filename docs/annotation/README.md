# Granular Annotation Scheme

> **Status:** Coming soon

## What this covers

A replacement for the common fixed-sample (e.g. 10 messages per topic) annotation approach, which lacks methodological justification and performs poorly at granular levels.

**Core ideas:**
- Proportional sampling — 10% of messages per topic rather than a fixed count
- Message-level annotation — each message receives an individual description; most frequent description represents the topic
- Entropy-based stopping criteria — measures description diversity to determine when a topic is sufficiently characterised
- Annotation patience — predefined iterations before stopping, reducing annotator burden for large topics

**Why it matters:** Separates the tasks of assessing homogeneity, analytical acceptance and description into discrete steps — enabling downstream supervised classification and providing interpretable progress metrics.
