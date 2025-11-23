Syllabus Indexer & Advisor Bot (n8n RAG Assignment)

A complete Retrieval-Augmented Generation system using n8n, Pinecone, OpenAI & Telegram.

📌 Overview

This assignment implements a production-style RAG pipeline inside n8n, consisting of two fully independent workflows:

WF-SYLLABUS-INDEXER — Converts a syllabus PDF into clean text, chunks it, embeds it, adds metadata, and stores vectors in Pinecone.

SYLLABUS-ADVISOR-BOT — A Telegram chatbot that performs semantic vector search over the syllabus and answers student questions using retrieved text.

This README explains:

Why each n8n node was selected

Why each RAG design choice (chunking, embeddings, metadata, Top-K, search type) was made

How the AI Agent prompt was engineered

Working Q&A output proving RAG correctness

Screenshots of both workflows

Loom narration script

The goal is not to "finish an assignment", but to simulate a real-world RAG deployment you can later apply to legal, property-management, finance, or enterprise clients.

🏗️ High-Level Architecture
flowchart LR
    A[Syllabus PDF<br/>Google Drive] --> B[WF-SYLLABUS-INDEXER]
    B -->|Chunks + Embeddings| C[(Pinecone Vector DB)]

    D[Student → Telegram] --> E[Telegram Trigger]
    E --> F[SYLLABUS-ADVISOR-BOT]
    F -->|Vector Query| C
    C -->|Top-K (15) Chunks| F
    F --> G[AI Agent<br/>Context-Grounded Answer]
    G --> H[Telegram Response]


Ingestion and serving are decoupled, just like production RAG systems.

📂 Workflow 1 — WF-SYLLABUS-INDEXER

This workflow ingests the syllabus and prepares the vector store.

📸 Workflow Screenshot

🔧 Node-by-Node Explanation
1. Manual Trigger

Used because ingestion runs only when syllabus is updated.

Prevents unintentional re-indexing.

2. Download File (Google Drive)

Fetches the syllabus PDF.

Using Drive ensures a single source of truth.

3. Function — File Structuring (JS)

Normalises the input binary structure.

Ensures the next node can properly convert it.

4. Move Binary → Text

Converts PDF binary into plain text.

RAG systems cannot index PDFs directly.

5. Function — Chunking + Metadata (JS)

Most important ingestion step.

Chunking logic:

chunk_size ≈ 300–500 characters

overlap ≈ 20–30%

Why?

Need	Reason
High search precision	Small chunks avoid mixing unrelated rows/sections
Maintain context	Overlap prevents information splitting across chunks
Good for LLM ingestion	Short chunks fit into context → stable answers

Metadata fields added:

{
  "filename": "syllabus.pdf",
  "subject": "Mathematics",
  "board": "CBSE",
  "year": "2025",
  "chunk_index": 12,
  "chunk_count": 58
}


Reason:

Scaling to multiple boards / classes later becomes trivial.

Future filtering possible (e.g., only CBSE class 10).

chunk_index helps debugging.

6. Embeddings (OpenAI)

Model: text-embedding-3-large

Why?

High semantic accuracy.

Best performance for tabular/semi-structured academic data.

Token-efficient.

7. Pinecone Vector Store — Upsert

Stores:

Embedding

Chunk text

Metadata

This completes ingestion.

📂 Workflow 2 — SYLLABUS-ADVISOR-BOT

This workflow exposes the indexed syllabus via Telegram.

📸 Workflow Screenshot

🔧 Node-by-Node Explanation
1. Telegram Trigger

Captures incoming student message.

Provides clean JSON structure.

2. Code — Extract Query

Simple JS:

return [{
  query: $json.message.text,
  chatId: $json.message.chat.id
}];


Why?

Abstracts Telegram structure away from the AI Agent.

Makes pipeline future-proof (WhatsApp, web-forms, etc.)

3. AI Agent Node

Central controller that:

Understands student queries

Calls Pinecone search tool

Reads retrieved chunks

Generates grounded answers

Follows strict non-hallucination rules

4. Chat Model (OpenAI)

Model used for reasoning (GPT-4.1 or GPT-3.5-turbo family).

Converts retrieved text → human language answers.

5. Pinecone — Syllabus Search (as Tool)

Critical RAG parameter settings:

Parameter	Value	Why
Top-K	15	Needed to answer global questions (“list all topics”). Lower values miss parts.
Search Type	Semantic	Students use varied phrasing; semantics > keywords
Filter	none	Single syllabus assignment
Return	Chunks + Metadata	Needed for contextual answers
6. Send Telegram Message

Builds message:

Chat ID: {{ $('Telegram Trigger').item.json.message.chat.id }}
Text:    {{ $json.output }}

🧠 System Prompt (Final Version)

This is the exact, optimised prompt used in production:

You are Syllabus_advisor_bot.

Your job is to answer ONLY using retrieved syllabus content from Pinecone. 
You must ALWAYS call the Syllabus Search tool before answering.

Rules:
- When user says “module”, “chapter”, treat it as Topic/NCERT chapter.
- Ignore filler messages like “ok”, “yes go ahead”.
- For LOCAL questions (hours, priority), give direct answers.
- For LIST questions (list topics, full syllabus), list ALL topics visible in retrieved chunks.
- Use: “Based on the syllabus sections I can see…” if uncertain.
- Only say “This information is not available in the syllabus.” when retrieval returns truly zero related content.
- Never hallucinate or mix outside knowledge.

🔬 RAG Parameters Explained
Component	Choice	Why
Chunking	300–500 chars, 20–30% overlap	Maintains context, avoids noise
Embeddings	text-embedding-3-large	High semantic quality
Search Method	Semantic only	Syllabus wording varies; semantic is superior
Top-K	15	Required for global completeness
Metadata	filename, subject, board, year	Scalable to multi-syllabus, multi-board
LLM Answer Style	Strict + grounded	Prevents hallucination

This is a Level-2 RAG (vector similarity + LLM reranking).

🧪 Real Telegram Q&A (Proof It Works)
✔ 1. Global Question

User: List all topics in the syllabus
Bot:

1. Real Numbers
2. Polynomials
3. Pair of Linear Equations in Two Variables
4. Quadratic Equations
5. Arithmetic Progressions
6. Triangles
7. Coordinate Geometry
8. Introduction to Trigonometry
9. Applications of Trigonometry
10. Circles
11. Constructions
12. Areas Related to Circles
13. Surface Areas and Volumes
14. Statistics
15. Probability

✔ 2. High Priority Only

User: What topics are marked HIGH priority?
Bot:

- Sets & Functions  
- Trigonometry  
- Permutations & Combinations  
- Sequences & Series  
- Limits & Derivatives  
- Probability

✔ 3. Local Question

User: How many practice hours are marked for Trigonometry?
Bot:

Trigonometry is allocated 8 practice hours.

✔ 4. Chapter Summary

User: Summarize NCERT Chapter 3
Bot:

NCERT Chapter 3 covers “Trigonometry”.
It is marked as high priority and allocated 8 practice hours.

🚀 How to Run the Project
1. Create Pinecone Index

Dimension must match embedding model (1536).

2. Import WF-SYLLABUS-INDEXER

Configure Drive credentials

Configure Pinecone credentials

Run once to populate the index

3. Import SYLLABUS-ADVISOR-BOT

Configure Telegram bot token

Connect AI Agent → Chat Model → Pinecone Search

Activate workflow

4. Test on Telegram
⚠️ Limitations

Single syllabus only

No hybrid search (but can be added)

No multi-document cross-knowledge RAG

No citations included in LLM answers