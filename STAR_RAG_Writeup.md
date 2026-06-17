# STAR Assessment Parent Helper — RAG Application Write-Up

**Project:** Week 2 — Build Your RAG Application
**Build track:** Track 1 (No-code with n8n)
**Model provider:** Nebius Token Factory (required)
**Vector store:** Pinecone

---

## 1. Project Overview

This project is a Retrieval-Augmented Generation (RAG) chatbot that helps **parents understand their child's STAR assessment results**. STAR (by Renaissance) is a widely used school assessment, and its score reports — scaled scores, percentile ranks, grade equivalents, growth percentiles — are notoriously confusing for non-educators. The app lets a parent ask a plain-English question ("What does my child's percentile rank mean?") and get a clear, grounded answer drawn only from a trusted knowledge base, with an honest "I don't know" response when the answer isn't in the corpus.

**One-line summary (the primer):**
> My RAG app helps **parents** understand **STAR assessment results** from a **knowledge base of STAR parent-guidance documents** in a **chatbot**, targeting **95% faithfulness** and **90% relevance**.

The app is fully working end to end and is accessible to an end user through a live hosted chat URL. A parent's question is embedded, matched against the knowledge base in Pinecone, and answered by a Nebius-hosted language model that is instructed to use only the retrieved content.

---

## 2. The RAG Framework (Part 1)

| Field | Detail |
|-------|--------|
| **Use case** | Parents ask what their child's STAR scores mean and how to support learning; surfaced through a chatbot. |
| **Corpus** | 5 documents (~16,000 characters) of parent-facing STAR guidance, authored as synthetic material grounded in how Renaissance actually defines the metrics. Owner of source-of-truth: the project author (synthetic). |
| **Ingestion + cleaning** | Documents are loaded into the workflow via a Code node (one item per document, with filename metadata). Content is clean Markdown, so minimal cleaning was required (no markup-stripping or boilerplate removal needed). |
| **Ingestion + freshness** | Ingestion is run manually via a Manual Trigger. STAR guidance changes roughly yearly, so the freshness requirement is low; re-running the ingestion flow re-indexes the corpus. |
| **Chunking + embedding** | Chunked using n8n's Default Data Loader text splitting (~1,000-character chunks with 200-character overlap), producing 20 chunks from 5 documents. Embedded with **Qwen/Qwen3-Embedding-8B** (4096 dimensions) via Nebius. |
| **Retrieve** | Pinecone serverless index (dense vectors, cosine metric, 4096 dimensions). Retrieval runs as a tool for an AI Agent, embedding the question with the same model and returning the most similar chunks as context. |

### Design decisions worth noting

- **Synthetic corpus for privacy.** Real STAR reports contain student education records governed by FERPA. Using authored synthetic documents (fictional students, invented scores, clearly labelled) gives a realistic demo with zero privacy exposure — and lets the corpus deliberately cover the cases parents find confusing (a high scorer, a "behind on percentile but growing" reader, an urgent-intervention case, a mid-range result).
- **Generic benchmark thresholds.** Benchmark cut scores (e.g. percentile rank 40) are presented as *typical examples* with repeated "ask your district" guidance, because real cut scores vary by district. This gives the bot honest material to retrieve and models the "I'm not certain, here's who to ask" behavior at the corpus level.
- **Faithfulness prioritized over relevance.** For a parent-facing tool dealing with a child's education, a confidently wrong answer is worse than an off-topic one — so the system is designed to refuse rather than guess.

---

## 3. Datasets Used

The corpus is five Markdown documents:

1. **STAR overview for parents** — what STAR is, the three tests (Early Literacy, Reading, Math), adaptive testing, and the framing that STAR is one screening data point, not a diagnosis.
2. **Understanding your scores** — definitions of Scaled Score, Percentile Rank, Grade Equivalent, Student Growth Percentile, benchmark categories, and domain scores, each with the common misreadings called out.
3. **Sample score reports** — four fictional students chosen to cover the profiles parents misread most often.
4. **Parent FAQ** — the questions parents actually ask ("What's a good score?", "The percentile dropped, should I worry?").
5. **Supporting your child at home** — actionable next steps by performance level.

All content is synthetic and labelled as such; metric definitions were grounded in publicly documented Renaissance STAR terminology.

---

## 4. Architecture

```
INGESTION (run once)
Manual Trigger
  → Code node (5 documents, one item each + filename metadata)
  → Pinecone Vector Store [Insert Documents]
        ├── Embeddings: Qwen/Qwen3-Embedding-8B (Nebius)
        └── Default Data Loader → text splitting (~1000 / 200)
  → 20 vectors stored in Pinecone (index: star-parent-rag1)

QUERY (live)
Chat Trigger ("When chat message received", hosted chat enabled)
  → AI Agent  [system prompt: answer only from retrieved content; refuse otherwise]
        ├── Chat Model: Qwen/Qwen3-235B-A22B-Instruct-2507 (Nebius)
        └── Tool: Pinecone Vector Store [Retrieve as Tool for AI Agent]
                    └── Embeddings: Qwen/Qwen3-Embedding-8B (Nebius)
  → Grounded answer returned to the parent
```

Both the embedding and generation calls run through **Nebius Token Factory**, satisfying the course requirement (one Nebius model call minimum — this build uses two).

---

## 5. Prompts and Iterations

Because this was a no-code build, "prompts" took two forms: (a) the **system prompt** that governs the agent's behavior, and (b) the sequence of **configuration iterations and fixes** required to get the OpenAI-compatible nodes talking to Nebius.

### The system prompt (grounding + refusal)

> You are a helpful assistant that answers parents' questions about the STAR assessment using ONLY the information retrieved from the knowledge base. If the retrieved documents do not contain the answer, say you don't have that information and suggest the parent contact their child's teacher or school. Never invent scores, policies, or facts. Keep answers clear and friendly for a non-expert parent.

### The tool description (so the agent reliably searches the corpus)

> Search the STAR assessment parent guide for information about score types (scaled score, percentile rank, grade equivalent, student growth percentile), benchmark categories, sample score reports, and how parents can support their child. Use this for any question about STAR results.

### Iteration log (what broke and how it was fixed)

1. **Base URL location.** The Nebius base URL (`https://api.tokenfactory.nebius.com/v1/`) belongs on the OpenAI **credential**, not the embeddings node's options — the credential is what both nodes read.
2. **Authorization failed.** Initial credential test failed; the fix was carefully re-pasting the exact Nebius API key (and confirming account credit).
3. **Embedding model 404s.** `BAAI/bge-en-icl` and bare `Qwen3-Embedding-8B` both returned `MODEL_NOT_FOUND`. The working ID was the exact dashboard form **`Qwen/Qwen3-Embedding-8B`** (org prefix required).
4. **Dimension mismatch.** Qwen3-Embedding-8B outputs 4096 dimensions, but the Pinecone index was created at 1536. Since the index was empty, it was deleted and recreated at **4096** (cosine, dense) — `star-parent-rag1`.
5. **Chat model connection error.** The OpenAI Chat Model node's **"Use Responses API"** toggle was on; Nebius doesn't implement that endpoint. Turning it **off** fixed it. The chat model used is `Qwen/Qwen3-235B-A22B-Instruct-2507` (an instruct model, chosen over reasoning/thinking variants to avoid stray reasoning text).
6. **Publishing blocked.** Publishing failed with "Missing required parameter: toolDescription." Adding the tool description (above) cleared it and also made the agent reliably call the retrieval tool.

### Behavior testing

- *"What does my child's percentile rank mean?"* → grounded answer pulling the corpus's PR explanation (the PR 60 / PR 40 examples, "not failing," read alongside Scaled Score and SGP). ✅
- *"What time does the school office open?"* → correctly refused: "I don't have information about the school office's opening hours. Please check with your school directly." ✅ (Demonstrates the refusal path and faithfulness.)

---

## 6. Evaluation Plan (faithfulness & relevance)

The one-liner targets 95% faithfulness and 90% relevance. These are measured, not configured — by running a fixed set of questions and scoring each answer. A starter 15-question evaluation set:

**Should answer from corpus (relevance + faithfulness):**
1. What does a scaled score measure?
2. What does my child's percentile rank of 60 mean?
3. Is a grade equivalent of 6.1 for a 4th grader a placement recommendation?
4. What is a Student Growth Percentile?
5. My 6th grader has a percentile of 31 but is growing — is that bad?
6. What does the "Urgent Intervention" label mean?
7. How often is the STAR test given?
8. What are the three STAR tests?
9. My child's percentile dropped from fall to winter — should I worry?
10. How can I help my child at home if they're "On Watch"?

**Should refuse / defer (faithfulness via honest "I don't know"):**
11. What time does the school office open?
12. What's the exact benchmark cut score my district uses?
13. Should this score get my child into the gifted program?
14. What will my child's score be next year?
15. Can you tell me my neighbor's child's score?

**Scoring:** for each, mark *faithful* (answer is supported by the corpus / correctly refuses) and *relevant* (answer addresses the question). Faithfulness % = faithful answers ÷ 15; relevance % = relevant answers ÷ 15.

---

## 7. Learnings and Observations

- **Embedding dimension must match the index exactly**, and it's set at index creation — pick the embedding model *first*, confirm its output dimension, then create the index.
- **OpenAI-compatible providers need three things right:** the base URL on the credential, the exact model ID (including org prefix), and the "Use Responses API" toggle **off**.
- **Question and documents must be embedded with the same model** — the vector space is model-specific.
- **Grounding emerged even before the system prompt**, but the explicit system prompt + tool description make faithful, on-topic behavior reliable rather than incidental.
- **Synthetic data was the right call** for a privacy-sensitive, child-focused domain, and it doubled as a way to deliberately design the demo's hardest cases.
- **No-code didn't mean no debugging** — most of the effort went into model IDs, dimensions, and provider quirks, exactly as the course framing predicted (RAG projects fail at ingestion/retrieval/config, not the model itself).
