# AI-Powered Academic Paper Reviewer: Research & Architecture

**Date:** 2026-02-27
**Status:** Research / Early Exploration

---

## Motivation

Build an app that provides reviewer-like feedback on academic papers, tailored to particular disciplines. The core challenge: existing systems are trained on ML/AI conference data (OpenReview), but no equivalent review corpus exists for the social sciences. Our key insight is to substitute **outcome-based quality signals** (journal placement, citations, prizes) for the missing review text.

---

## Landscape of Existing Tools

### Research-Grade Systems

| System | Architecture | Performance | Limitation |
|--------|-------------|-------------|------------|
| [Stanford Agentic Reviewer](https://paperreview.ai/tech-overview) | Agentic pipeline: PDF→Markdown→web search→multi-dimensional scoring | 0.42 Spearman correlation with human reviewers (human-to-human: 0.41) | Trained on 150 ICLR papers; English only; works best where arXiv coverage is strong |
| [MARG](https://arxiv.org/abs/2401.04259) (Allen AI) | Multi-agent with leader/worker/expert agents | Reduces generic comments 60%→29%; 3.7 useful comments/paper (2.2x baseline) | Requires chunking papers across agents |
| [MAMORX](https://openreview.net/pdf/5d2e9f6043364b3d2a653c609ac1a1fe0ed7572e.pdf) | Multi-agent multi-modal; separate agents for novelty, figures, clarity | Multi-dimensional coverage | ML-focused |

### Commercial Products

| Product | Focus | Notes |
|---------|-------|-------|
| [Rigorous](https://news.ycombinator.com/item?id=44144280) | 24 specialized agents, open-source | Uses GPT-4.1-nano; ~8 min/paper; [code on GitHub](https://github.com/AI-4-Research/AI-4-Research.github.io) |
| [Paperpal](https://paperpal.com/) | Writing assistance (grammar, flow, structure) | Not deep review |
| [Enago Read](https://www.read.enago.com/ai-peer-review-workspace/) | AI peer review workspace | Human oversight model |

### Key Survey

[Large Language Models for Automated Scholarly Paper Review: A Survey](https://arxiv.org/html/2501.10326v1) (January 2025) — comprehensive review of architectures, datasets, and open problems.

---

## Three Architectural Approaches

### 1. Single-LLM + Structured Prompting (Simplest)

Feed the full paper to a long-context model (Claude, GPT-4) with a detailed review rubric prompt.

- **Pros:** Fast to build, cheap to run, works today
- **Cons:** Generic feedback, no novelty assessment, no related work grounding

### 2. Multi-Agent Specialized Review (State of the Art)

Multiple LLM instances, each with a specialized role:

- **Methodology agent** — evaluates research design, statistical approach
- **Novelty agent** — compares claims against related work
- **Writing agent** — clarity, structure, argumentation
- **Domain expert agent** — field-specific conventions and standards
- **Controller/editor agent** — synthesizes into coherent review

This is what MARG, MAMORX, Rigorous, and the Stanford system all use in various forms.

### 3. Fine-Tuned + RAG Pipeline (Most Ambitious)

Fine-tune a model on actual review-paper pairs, augmented with retrieval over related literature. Available datasets (all ML/CS):

| Dataset | Size | Source |
|---------|------|--------|
| ReviewMT | 26,841 papers / 92,017 reviews | Multi-turn dialogue simulation |
| Reviewer2 | 27k papers / 99k reviews | Feedback generation |
| LimGen | 4,068 papers | Limitation annotations |
| SchNovel | 15k paper pairs | Novelty assessment |
| NLPeer | 5k+ papers / 11k+ reports | Score prediction |
| OpenReview (ICLR) | 36,000+ threads (2017-2025) | CC BY 4.0 |

---

## The Social Science Gap

Almost everything built so far works in ML/AI because:

1. **Training data exists** — OpenReview has 36,000+ ICLR threads, all CC BY 4.0
2. **Related work is freely available** — arXiv
3. **Review norms are relatively standardized**

For social sciences, three gaps:

### Gap 1: No Review Text Corpus

Peer reviews in political science, sociology, and economics are confidential. No equivalent of OpenReview exists. Options: (a) partner with journals, (b) crowdsource from willing reviewers, or (c) use outcome-based quality signals instead of review text.

### Gap 2: Related Work Discovery Is Harder

Social science papers aren't all on arXiv. Need to search across JSTOR, SSRN, Google Scholar, discipline-specific repositories. The Stanford system's Tavily search works for arXiv-heavy fields; social sciences need broader search.

### Gap 3: Review Norms Vary Enormously

A sociology review evaluates theoretical framing and positionality very differently from how an economics review evaluates identification strategy. "Discipline-specific" isn't one thing — it's dozens of sub-field conventions.

---

## Using Outcome-Based Quality Signals

### The Core Idea

Instead of learning "what reviewers say," learn "what distinguishes papers that succeed from those that don't" — using observable outcomes as training signals.

### Signal 1: Journal Placement → Preference Pairs for DPO

**Strongest signal.** Construct contrastive pairs of papers on similar topics where one landed in a top-5 journal and the other in a lower-tier outlet. Use [Direct Preference Optimization (DPO)](https://arxiv.org/abs/2305.18290) to train a model to distinguish them.

**Building the dataset:**

- Use [OpenAlex](https://docs.openalex.org) to pull papers by topic/field, along with venue metadata
- Pair papers within narrow topic clusters (same OpenAlex concept, similar year) where one is in a top journal and one isn't
- Target ~5,000-10,000 pairs per discipline
- OpenAlex has [full-text PDFs](https://docs.openalex.org/download-all-data/full-text-pdfs) for ~60M papers; social science coverage is decent via open access mandates

**What this teaches the model:**

Not "this paper is good" in the abstract, but "given two papers on similar topics, what features distinguish the one that placed higher?" This implicitly captures: clearer identification strategy, better motivated theory, more robust checks, tighter writing.

**Why the confounder problem is manageable:**

Journal placement reflects reviewer/editor preferences, which embed biases. But for this use case, that's arguably a *feature* — you want to give feedback that helps authors place in good journals, which means learning what those journals reward.

### Signal 2: Citations → Calibration Signal (Use Carefully)

Citations are noisy due to the well-documented [Matthew effect](https://link.springer.com/article/10.1007/s11192-025-05455-3) — prestigious authors and institutions get cited more regardless of content.

**The approach:**

Train a citation prediction model on paper text features *only* (excluding author identity, institution, journal). [Research shows ~80% accuracy](https://arxiv.org/abs/2407.19942) at predicting top-20% cited papers from text alone.

**Concrete use:**

- Don't use raw citation counts
- Predict expected citations from text features
- Gap between predicted and actual → "this paper over/underperformed given its content"
- Use the *text features* that predict citations as dimensions for the review rubric
- Field-normalize everything (citations in sociology vs. economics have very different scales)

### Signal 3: Prizes → Few-Shot Exemplars

Prizes are too sparse for training (a few dozen per field per year) but perfect as gold-standard exemplars:

- Collect best paper award winners from ASA, APSA, AEA, discipline sections
- Use as few-shot examples in prompts: "Here is a paper that won the ASA's Distinguished Contribution to Scholarship Award. When reviewing the submitted paper, evaluate it along the dimensions that distinguish award-winning work."
- Lightweight but surprisingly effective for calibrating tone and depth

---

## Proposed Architecture

```
                    ┌─────────────────────────┐
                    │   Paper PDF uploaded     │
                    └────────┬────────────────┘
                             │
                             ▼
                    ┌────────────────────┐
                    │  PDF → Markdown    │
                    │  (Marker / docling) │
                    └────────┬───────────┘
                             │
                    ┌────────┴────────────────────────┐
                    │                                  │
                    ▼                                  ▼
        ┌───────────────────┐             ┌──────────────────────┐
        │  Related Work     │             │  Quality Prediction  │
        │  Retrieval        │             │  Model               │
        │  (Semantic Scholar │             │  (trained on journal │
        │   + OpenAlex)     │             │   placement pairs)   │
        └────────┬──────────┘             └──────────┬───────────┘
                 │                                   │
                 │    ┌──────────────────────┐       │
                 │    │  Discipline-Specific │       │
                 │    │  Review Rubric       │       │
                 │    │  (prompt template)   │       │
                 │    └──────────┬───────────┘       │
                 │               │                   │
                 ▼               ▼                   ▼
        ┌─────────────────────────────────────────────────┐
        │          Multi-Agent Review Generation          │
        │                                                 │
        │  Agent 1: Methodology  (calibrated by DPO       │
        │  Agent 2: Theory       model or reward model    │
        │  Agent 3: Novelty      that learned from        │
        │  Agent 4: Writing      journal placement pairs) │
        │  Agent 5: Statistics                            │
        │                                                 │
        │  + Few-shot exemplars from prize-winning papers  │
        └────────────────────┬────────────────────────────┘
                             │
                             ▼
                    ┌────────────────────┐
                    │  Controller Agent  │
                    │  Synthesizes into  │
                    │  structured review │
                    └────────────────────┘
```

---

## Two Implementation Paths

### Path A: Reward Model Approach (Heavier, More Powerful)

1. Collect ~10k paper pairs per discipline from OpenAlex (top journal vs. field journal, same topic)
2. Train a reward model (fine-tuned smaller model, e.g., Llama-3) that scores papers on quality dimensions
3. Use reward model to guide review generation — review agents generate candidate comments, reward model scores which comments best align with what distinguishes high-placement papers
4. Essentially RLHF where journal placement replaces human preference labels

### Path B: Feature Extraction + Prompting (Lighter, Faster to Build)

1. Use an LLM to extract structured features from a corpus of papers (identification strategy clarity, theoretical motivation, robustness, etc.)
2. Run a regression: which features predict journal tier / field-normalized citations?
3. Build the review rubric around the features with highest predictive power
4. Use these as scoring dimensions for multi-agent reviewer, with few-shot examples from prize-winning papers

**Recommendation:** Start with Path B. Move to Path A if building a real product.

---

## Confounders to Handle

| Confounder | Risk | Mitigation |
|-----------|------|------------|
| **Author prestige** (Matthew effect) | Model learns "sounds like a famous person" not "is good" | Use text-only features; exclude author/institution metadata from training |
| **Topic popularity** | Hot topics get cited more and placed better | Pair papers within narrow topic clusters, not across topics |
| **Methodological fashion** | RCTs/IV papers placed better in econ in 2010s may not reflect timeless quality | Include time period in pairing; or embrace it as "what journals reward now" |
| **Journal fit vs. quality** | A great qualitative paper rejected from a quant journal isn't "bad" | Train separate models per methodological tradition |
| **Open access citation advantage** | OA papers get cited more regardless of quality | Control for OA status in citation models |

---

## Data Infrastructure (Available Today)

| Resource | What It Provides | Access |
|----------|-----------------|--------|
| [OpenAlex](https://docs.openalex.org) | 240M+ works, venue metadata, citation counts, topic classifications, [full-text PDFs](https://docs.openalex.org/download-all-data/full-text-pdfs) | Free API; bulk download via S3 |
| [Semantic Scholar](https://api.semanticscholar.org/) | Related work retrieval; good social science coverage | Free API |
| SSRN | Working papers with eventual placement data | Terms may restrict scraping |
| OpenReview | 36k+ ML conference threads (for cross-domain transfer learning) | Free API; CC BY 4.0 |

---

## Known Limitations of Current Systems

From the [survey paper](https://arxiv.org/html/2501.10326v1) and [community feedback](https://news.ycombinator.com/item?id=44144280):

1. **Novelty assessment** — the hardest and most valuable part of peer review. LLMs can check if claims are well-supported but struggle to judge whether something is genuinely new.
2. **Deep domain expertise** — LLMs "lack actual understanding of domain-specific expertise and capacity to contextualize feedback in the nuanced framework of a particular field."
3. **Consistency** — same paper reviewed twice produces different scores and comments.
4. **Bias toward positivity** — models tend to rate papers higher and never give the lowest score.
5. **Shallow analysis** — models "offer short sentences and general words, suggesting evaluation only at the stage of imitatively evaluating text characteristics, not substantively evaluating core elements like originality and soundness."

**Realistic value proposition:** Not "replace Reviewer 2" — accelerating the feedback cycle so authors get structured, actionable feedback in minutes rather than months, improving papers before formal submission.

---

## Evaluation Strategy

How to know the reviewer is giving good feedback:

- **Holdout set:** Papers where journal outcome is known; check whether AI review correctly identified the dimensions that mattered
- **Expert validation:** Have disciplinary experts rate AI reviews for helpfulness, specificity, and accuracy (follow [MARG's evaluation protocol](https://arxiv.org/abs/2401.04259))
- **Correlation with outcomes:** For papers reviewed before submission, track whether following AI suggestions correlates with better placement

---

## Phased Roadmap

| Phase | Work | Timeline | Outcome |
|-------|------|----------|---------|
| **1** | Prompt-engineered multi-agent system with discipline-specific rubrics | Weeks | Working prototype; 70% of value |
| **2** | RAG for related work grounding (Semantic Scholar + OpenAlex) | 1-2 months | Novelty assessment capability |
| **3** | Calibration with real reviews (50-100 from willing reviewers) + prize-winning exemplars | 1-2 months | Better tone, depth, field alignment |
| **4** | DPO/reward model trained on journal placement pairs from OpenAlex | 3-6 months | Quality-calibrated feedback; product-grade system |

---

## References

- Chamoun, M. et al. (2025). Large language models for automated scholarly paper review: A survey. *arXiv:2501.10326*. https://arxiv.org/html/2501.10326v1
- D'Arcy, M. et al. (2024). MARG: Multi-agent review generation for scientific papers. *arXiv:2401.04259*. https://arxiv.org/abs/2401.04259
- Rafailov, R. et al. (2023). Direct preference optimization: Your language model is secretly a reward model. *arXiv:2305.18290*. https://arxiv.org/abs/2305.18290
- Kang, D. (2024). Predicting citation impact of research papers using GPT and other text embeddings. *arXiv:2407.19942*. https://arxiv.org/abs/2407.19942
- Stanford Agentic Reviewer. Tech overview. https://paperreview.ai/tech-overview
- Koltun, S. & Hafner, F. (2025). Mitigating consequences of prestige in citations of publications. *Scientometrics*. https://link.springer.com/article/10.1007/s11192-025-05455-3
- OpenAlex documentation. https://docs.openalex.org
