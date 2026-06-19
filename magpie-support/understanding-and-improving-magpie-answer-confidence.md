---
title: Understanding and Improving Magpie Answer Confidence
status: draft
---

# Understanding and Improving Magpie Answer Confidence

This article explains why Magpie may give low‑confidence answers and provides actionable steps to improve answer quality.

## What Does Confidence Mean in Magpie?

Magpie assigns a confidence score to each generated answer. This score reflects how certain the system is that the answer is correct and well‑supported. Confidence is typically derived from:

- **Model probability**: The likelihood assigned by the underlying language model to the generated tokens.
- **Retrieval relevance**: When using Retrieval Augmented Generation (RAG), how closely the retrieved documents match the query.
- **Answer completeness**: Whether the answer directly addresses the question without ambiguity.

Low confidence does not necessarily mean the answer is wrong – it indicates that the system has less assurance due to missing or conflicting evidence.

## Common Causes of Low‑Confidence Answers

### 1. Insufficient or Poorly Formatted Context
If the knowledge base lacks relevant information or contains conflicting data, Magpie may produce an answer with low confidence.

### 2. Ambiguous or Vague Questions
Questions that are unclear or contain multiple possible interpretations reduce the system’s ability to pinpoint the correct answer.

### 3. Outdated or Incomplete Knowledge Base
When the source documents used for RAG are not up‑to‑date or missing key topics, the generated answer may rely on weak evidence.

### 4. Model Limitations
Smaller or less capable models may struggle with complex reasoning, leading to lower confidence even when context is sufficient.

## How to Improve Answer Quality

### 1. Refine Your Queries
- Be specific and avoid open‑ended phrasing.
- Include relevant keywords that match the structure of your knowledge base.

### 2. Enhance the Knowledge Base
- Regularly update documents to ensure accuracy.
- Remove duplicate or contradictory content.
- Add metadata (tags, categories) to improve retrieval relevance.

### 3. Adjust System Parameters
- **Temperature**: Lower values (e.g., 0.1–0.3) produce more deterministic answers; higher values increase creativity but may reduce confidence.
- **Top‑k / Top‑p**: Tighten these parameters to limit the model’s sampling space for more focused outputs.
- **Max tokens**: Ensure the answer length is sufficient to fully address the question.

### 4. Use Prompt Engineering
- Provide clear instructions in system prompts, specifying the desired format and level of certainty.
- Ask the model to explain its reasoning or cite sources when confident.

### 5. Evaluate and Iterate
- Review low‑confidence answers to identify patterns (e.g., specific topics or question types).
- Test changes to queries or knowledge base updates to measure improvement.

## Interpreting Confidence Scores

Confidence is a tool for developers and users, not an absolute measure of correctness. Use it as a guide to:
- Flag answers that require human review.
- Prioritise which knowledge base sections need enrichment.
- Compare model versions or prompt configurations.

If you consistently see low confidence across many answers, start by checking the underlying data quality and retrieval pipeline.

## Next Steps
- Review your Magpie configuration and ensure you are using the recommended model and settings.
- Refer to the [Magpie documentation on RAG tuning](https://github.com/AdamAwan/markdown-magpie) (once available) for advanced optimisation.

---

*This article was written based on general best practices for QA systems. No direct source material was available from the Magpie codebase at the time of writing.*