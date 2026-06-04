# Sampling & Extrapolation

> **Status:** Coming soon

## What this covers

How to handle datasets too large to model in full, and the critical problem that arises when you try to extrapolate a topic model trained on a sample to the full corpus.

**Core ideas:**
- Account-stratified sampling — capping messages per source to maintain representativeness
- The extrapolation problem — UMAP learns a latent space for training data and handles out-of-distribution data poorly, causing topic definitions to shift dramatically when applied to a larger dataset
- Two solutions: keyword filtering to reduce the large dataset first; incremental transformation by dividing the unseen data into batches matching training size

**Why it matters:** Training on a sample of a very large dataset without addressing extrapolation risk is one of the most common silent failures in applied topic modeling.
