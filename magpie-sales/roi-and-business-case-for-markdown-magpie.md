---
title: ROI and Business Case for Markdown Magpie
status: draft
---

# ROI and Business Case for Markdown Magpie

This document provides a framework for presenting a business case to a CFO or budget holder for purchasing Markdown Magpie. It covers the problems it solves, the quantifiable and qualitative benefits, and a sample ROI calculation based on typical team sizes and knowledge maintenance costs.

## The Problem Markdown Magpie Solves

Organisations maintain internal Markdown documentation — knowledge bases, runbooks, onboarding guides, product specs, API docs, and more. Over time, these repositories suffer from three common problems:

- **Won't rot:** Documentation becomes stale, incomplete, or contradictory as products and processes change.
- **Won't leak:** Sensitive or internal knowledge is scattered across files, making it hard to audit or control access.
- **Won't lie:** Answers to common questions are missing, forcing engineers and support staff to rediscover the same information repeatedly.

Traditional documentation tools either require heavy manual curation (wiki farms) or lock content into proprietary formats (Confluence, Notion). Markdown Magpie is an open-source, vendor-neutral knowledge maintenance system that addresses these problems through automated gap detection, proposal generation, and Git-backed workflows.

## Key Features Delivering Value

- **Automated Q&A with citations:** Users ask questions in natural language; Markdown Magpie retrieves relevant sections from the indexed Markdown knowledge base and returns cited answers. Low-confidence answers flag missing content.
- **Knowledge gap detection:** Every question is logged. When the system returns low confidence, it records a gap. Repeated gaps are clustered automatically, revealing exactly what documentation is missing or outdated.
- **Automated proposal generation:** From a gap cluster, Markdown Magpie drafts a Markdown proposal (a new page or edit) using an AI provider (e.g., GPT, Claude, DeepSeek). The proposal is submitted as a pull request for human review, merging the improvement into the live knowledge base.
- **Continuous patrol maintenance:** Instead of a single whole-KB maintenance pass, the system runs targeted patrols on rolling cursors: a fix-patrol checks correctness (verify, dedupe, split) and an improve-patrol grows fine-but-thin documents with source-backed additions. These patrols keep the knowledge base accurate and tidy without competing with gap work.
- **Seamless Git integration:** Changes are committed to branches, reviewed, and merged via standard Git workflows (GitHub, GitLab, Azure DevOps). No new tools or training required for engineers.
- **Multiple AI providers:** Supports OpenAI-compatible APIs, Azure OpenAI, local models, or CLI agents (Codex, Claude Code). The system can run completely on-premises or in the cloud.

## Quantified ROI Model

### 1. Reduction in Knowledge Re-discovery Time

**Assumption:** A team of 10 engineers each spends 30 minutes per week searching for or asking about undocumented information (Slack, ad hoc meetings, browsing stale wikis).

- Total weekly time lost: 10 engineers × 0.5 hours = **5 hours**.
- Annual cost (at blended burdened rate of $75/hour): 5 hours × 52 weeks × $75 = **$19,500**.

**With Markdown Magpie:** Engineers ask questions directly and get cited answers from the indexed knowledge base. Assume 60% reduction in re-discovery time.
- New weekly time: 10 × 0.2 hours = 2 hours.
- Annual cost: 2 × 52 × $75 = **$7,800**.
- **Annual savings: $11,700.**

### 2. Reduction in Onboarding Time for New Team Members

**Assumption:** A new hire requires 40 hours of ramp-up reading (tribal knowledge) before becoming productive. With a well-maintained knowledge base, this can be cut to 20 hours.
- For 5 new hires per year: 5 × 20 hours saved × $75/hour = **$7,500** saved.
- Also reduces mentor time: each new hire requires 5 hours of senior engineer mentoring that can now be reduced by half. Senior rate $120/hour: 5 hires × 2.5 hours × $120 = **$1,500** saved.
- Total onboarding savings: **$9,000** annually.

### 3. Reduction in Incidents Caused by Outdated Documentation

**Assumption:** One production incident per quarter is caused by engineers following outdated or missing runbooks. Average cost of incident (engineering time, rollback, customer impact): $10,000.
- Annual incident cost: 4 × $10,000 = **$40,000**.

**With Markdown Magpie:** Automated patrols (verify + dedupe + split) ensure runbooks are current and accurate. Assume 50% reduction in documentation-related incidents.
- New annual incident cost: 2 × $10,000 = **$20,000**.
- **Annual savings: $20,000.**

### 4. Efficiency Gains from Automated Proposal Generation

**Assumption:** A knowledge base maintainer (or team) spends 4 hours per week reviewing and updating documentation without automation.
- Annual cost: 4 hours × 52 weeks × $75 = **$15,600**.

**With Markdown Magpie:** The system drafts proposals; the maintainer reviews and merges drafts, cutting time by 70%.
- New weekly time: 1.2 hours × 52 × $75 = **$4,680**.
- **Annual savings: $10,920.**

### Total Annual Hard Savings

| Category | Savings |
|----------|---------|
| Knowledge re-discovery | $11,700 |
| Onboarding | $9,000 |
| Incident reduction | $20,000 |
| Documentation maintenance | $10,920 |
| **Total annual hard savings** | **$51,620** |

### Additional Qualitative Benefits

- **Improved accuracy:** Cited answers reduce reliance on memory and speculation.
- **Faster product development:** Less time spent on documentation issues means more time for feature work.
- **Better compliance and auditability:** Git history provides a full audit trail of documentation changes.
- **Vendor lock-in avoidance:** Markdown is open and portable; no proprietary tools are required.
- **Scalable knowledge management:** The system works with many repositories and scales to hundreds of documents without proportional human effort.

## Cost of Markdown Magpie

The product is open source (MIT license) with no licensing fees. Costs are limited to:
- **Infrastructure:** A Postgres instance (e.g., $20–100/month for small teams), optional embedding API costs (~$5–50/month depending on volume).
- **AI provider usage:** If using a paid API (e.g., OpenAI), estimated $50–200/month for a team of 10.
- **Setup and maintenance:** Roughly 1–2 days of initial setup by an engineer (one-time), then minimal ongoing overhead (~1 hour per week).

**Estimated first-year cost:** $1,500–3,000 (infrastructure + AI usage + setup labour).

## ROI Summary

- **First-year net savings:** $51,620 – $3,000 = **$48,620**.
- **Payback period:** Less than 1 month (setup labour of ~$1,200 pays for itself within a few weeks of operational savings).
- **Ongoing annual savings:** $51,620 (infrastructure costs constant).
- **ROI percentage:** 1,700%+ (annual savings / annual cost).

## Implementation Roadmap (Phased Approach)

1. **Phase 1 – Pilot (Month 1):** Deploy Markdown Magpie for one knowledge base (e.g., internal engineering runbooks). Index existing Markdown, test Q&A with 3–5 engineers, monitor gaps and feedback.
2. **Phase 2 – Expand (Month 2–3):** Add more knowledge bases, enable automated proposal generation, train the team on review workflows.
3. **Phase 3 – Full adoption (Month 4+):** Turn on patrol maintenance (fix-patrol, improve-patrol), integrate with CI/CD pipelines, and roll out to all teams.

## Risk Mitigation

- **Risk:** AI provider outage or cost overrun. *Mitigation:* The system supports multiple providers; fallback to a local model or mock is possible. Usage caps can be set.
- **Risk:** Low adoption by engineers. *Mitigation:* Integration with existing tools (Slack, MCP clients like Claude Code) makes interaction frictionless. The pitch deck emphasises "won't lie · won't leak · won't rot" — clear value proposition.
- **Risk:** Generated proposals are too low quality. *Mitigation:* Human review is required before merging; the mock provider can be used for deterministic baseline quality.

## Conclusion

Markdown Magpie delivers a clear financial return by reducing time wasted on knowledge re-discovery, accelerating onboarding, preventing incidents from stale documentation, and cutting manual maintenance effort. With a payback period of less than a month and ongoing ROI exceeding 1,700%, it is a low-risk, high-reward investment for any team that relies on Markdown documentation.

---

*For more technical details, see the [Architecture](../docs/architecture.md) and [AI Jobs](../docs/ai-jobs.md) documentation.*