# 🎓 STAR Assessment Parent Helper

A RAG-powered chatbot that helps **parents understand their child's STAR assessment results**. Ask a plain-English question about scaled scores, percentile ranks, growth, or what a report means — and get a clear answer grounded only in a trusted knowledge base, with an honest "I don't know" when the answer isn't there.

Built with **n8n (no-code)**, **Nebius Token Factory**, and **Pinecone**, following The Gen Academy's Mastering Agentic AI — Week 2 project.

> ⚠️ **Synthetic data notice.** All documents, students, and scores in this knowledge base are fictional, created for a demonstration. This tool is not medical, psychological, or educational advice. For a real child's results, always rely on the school's official Renaissance reports and the child's teacher.

---

## What it does

A parent types a question → it's embedded via Nebius → matched against the STAR knowledge base in Pinecone → a Nebius-hosted model answers using only the retrieved content. If the corpus doesn't contain the answer, the assistant says so and points the parent to their school.

## Architecture

```
INGESTION (run once)
Manual Trigger
  → Code node (5 docs)
  → Pinecone Vector Store [Insert]
        ├── Embeddings: Qwen/Qwen3-Embedding-8B (Nebius)
        └── Default Data Loader → text splitting (~1000 / 200)
  → 20 vectors in Pinecone (index: star-parent-rag1, 4096-dim, cosine)

QUERY (live)
Chat Trigger (hosted chat)
  → AI Agent (system prompt: answer only from retrieved content; else refuse)
        ├── Chat Model: Qwen/Qwen3-235B-A22B-Instruct-2507 (Nebius)
        └── Tool: Pinecone Vector Store [Retrieve as Tool]
                    └── Embeddings: Qwen/Qwen3-Embedding-8B (Nebius)
```

## Repository contents

| File / folder | What it is |
|---------------|-----------|
| `corpus/` | The 5 synthetic STAR knowledge-base documents (Markdown). |
| `n8n_code_node.js` | The Code node contents that load the corpus into the workflow. |
| `workflow.json` | The exported n8n workflow (export this from your n8n instance — see below). |
| `STAR_RAG_Writeup.md` | Full project write-up (overview, framework, iterations, learnings). |
| `Demo_Video_Script.md` | Script for the demo video. |

## Setup

**Prerequisites**
- An n8n instance (Cloud or self-hosted)
- A Nebius Token Factory API key
- A Pinecone account

**1. Create the Pinecone index**
- Name: `star-parent-rag1`
- Dimensions: **4096**
- Metric: **cosine**, type: **dense**, serverless, AWS `us-east-1`

**2. Create the Nebius credential in n8n**
- Use the **OpenAI** credential type (Nebius is OpenAI-compatible).
- API Key: your Nebius key.
- Base URL: `https://api.tokenfactory.nebius.com/v1/`

**3. Import the workflow**
- In n8n: `…` menu → Import from File → `workflow.json`.
- Reconnect the Pinecone and Nebius credentials.

**4. Ingest the corpus**
- Open the Code node, confirm it holds the 5 documents.
- Run the ingestion flow (Manual Trigger → Execute). Confirm ~20 records appear in Pinecone.

**5. Run the chat**
- Enable "Make Chat Publicly Available" on the Chat Trigger.
- Publish the workflow.
- Open the hosted chat URL and ask a STAR question.

## Models

| Role | Model (Nebius) | Notes |
|------|----------------|-------|
| Embeddings | `Qwen/Qwen3-Embedding-8B` | 4096-dim; same model for ingest and query. |
| Generation | `Qwen/Qwen3-235B-A22B-Instruct-2507` | Instruct model; "Use Responses API" must be **off**. |

## To export `workflow.json` from your n8n instance

In the n8n editor: open the `…` (overflow) menu near the top → **Download** / **Export** → save the JSON → add it to this repo as `workflow.json`. (Remove any embedded credentials before committing — n8n exports reference credentials by name, not secret, but double-check.)
