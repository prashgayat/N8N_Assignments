# Syllabus Indexer & Advisor Bot — n8n RAG Assignment

## 1. Overview
This project implements a two‑workflow RAG pipeline inside n8n: WF‑SYLLABUS‑INDEXER for ingestion and SYLLABUS‑ADVISOR‑BOT for retrieval and answering.

## 2. High‑Level Architecture
```
flowchart LR
    A[Syllabus PDF] --> B[WF-SYLLABUS-INDEXER]
    B --> C[(Pinecone)]

    D[Student Telegram] --> E[Telegram Trigger]
    E --> F[SYLLABUS-ADVISOR-BOT]
    F --> C
    C --> F
    F --> G[LLM Answer]
    G --> H[Telegram Reply]
```

## 3. Architecture (Nodes & Wiring)

### Workflow A — WF‑SYLLABUS‑INDEXER
- **Manual Trigger** → runs indexing on demand.
- **Google Drive: Download File** → loads syllabus PDF.
- **File Structuring (Code)** → normalize binary.
- **Move Binary → Text** → extract plain text.
- **Chunking + Metadata (Code)** → chunk with overlap.
- **OpenAI Embeddings** → vectorize chunks.
- **Pinecone: Upsert** → store vectors.

### Workflow B — SYLLABUS‑ADVISOR‑BOT
- **Telegram Trigger** → receives student queries.
- **Extract Query (Code)** → parse chatId + query.
- **AI Agent** → orchestrates retrieval + reasoning.
- **Pinecone Search** → Top‑K=15 retrieval.
- **OpenAI Chat Model** → grounded answer.
- **Telegram: Send Message** → deliver result.

## 4. WF‑SYLLABUS‑INDEXER (Deep Dive)
Provides ingestion: PDF → text → chunk → embedding → Pinecone.

## 5. SYLLABUS‑ADVISOR‑BOT (Deep Dive)
Handles real-time RAG: Telegram → vector search → answer.

## 6. AI System Prompt
Strict grounding: no hallucinations, return all topics for global queries, etc.

## 7. Example Q&A
Includes topic listing, priority mapping, and chapter summaries.

## 8. How to Run
1. Configure Pinecone Index.
2. Import Indexer workflow and run once.
3. Import Advisor Bot workflow.
4. Test via Telegram.

## 9. Limitations
Single syllabus, no hybrid search, no multi-board routing.

## 10. Future Extensions
Metadata filters, multi-syllabus support, confidence scoring, hybrid search.
