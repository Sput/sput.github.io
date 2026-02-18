# Building a Multi-Modal RAG Demo with Doc, JSON, and Database Sources

## Intro
I wanted to demonstrate how easy it is to perform Retrieval Augmented Generation (RAG) on multiple documents. I built a demo that answers audit and security questions by retrieving context from three different sources: a security program document (SSP.docx), a vulnerability scan JSON (grype-results.json), and a Supabase view of evidence requests. The point is not just a correct answer—it’s a transparent explanation of where that answer came from and what the system is doing at each step.

## The Problem
I had already created an application that does RAG on a single data source, I wanted to see how hard it would be to use multiple sources (answer: not much harder). Usually we need intelligence over multiple documents. Evidence lives in documents, in machine‑generated scan outputs, and in database tables. If a demo can’t show how it cross‑references those sources, the result looks like a black box and doesn’t build trust.

## The Solution
I built a multi‑modal RAG app that:
- Pre‑embeds and stores content from all three sources in a single vector table.
- Runs per‑source retrieval.
- Shows live pipeline steps and intermediate artifacts in the UI, so users can see the source of the insights.

It’s a clean, real RAG pipeline—no mock responses—with a user experience that makes the retrieval traceable.

## How It Works
### 1) Ingestion and Embedding
- **Docling-first parsing.** SSP.docx is sent to a Docling worker (`/parse`) to extract structured text (markdown when available). Docling is preferred because it preserves document structure and yields cleaner sections for retrieval.
- **Chunking strategy.** The extracted text is chunked into ~1200-character windows with overlap to preserve continuity across boundaries. A generator-based chunker avoids loading all chunks at once and guards against infinite loops when the remaining text is shorter than the overlap.
- **JSON normalization.** grype-results.json is normalized into per‑vulnerability entries, combining ID, severity, description, and artifact metadata into a single retrievable text block.
- **Database view formatting.** rows from an audit database are formatted into a human‑readable string that includes the evidence description plus control and audit context.
- **Embedding + storage.** All items are embedded with `text-embedding-3-small` and stored in a single `rag_chunks` table with `source_type`, `source_id`, and `chunk_index` so we can retrieve per source while keeping the schema simple.

### 2) Retrieval
At query time:
- The question is embedded once.
- It runs three filtered vector searches (doc/json/db) to keep results balanced.
- The top items from each source are merged into a structured context block.

### 3) Answering
The system calls `gpt-4o-mini` with the question and the merged context. The prompt asks the model to cite source types and IDs in the response.

### 4) UI Feedback
The UI is part of the product:
- Live log steps show what’s happening in real time.
- Intermediate artifacts display retrieved doc chunks, JSON matches, and evidence request rows.
- A chat panel shows the conversation and the model’s output.

### Diagram (ASCII)
```
[SSP.docx]      [grype-results.json]      [v_evidence_requests_with_context]
     |                    |                              |
     v                    v                              v
   chunk              normalize                         format
     |                    |                              |
     +--------- embed with text-embedding-3-small -------+
                       |      (source_type tagged)
                       v
                supabase.rag_chunks
                       |
          query -> embed -> filtered vector search (doc/json/db)
                       |
                merge context + call gpt-4o-mini
                       |
                     response + UI artifacts
```


## Example / Walkthrough
A user asks: “Which SSP sections mention access control, and how do they map to evidence requests?”
- The system embeds the question.
- It retrieves SSP chunks that mention access control, evidence request rows tied to relevant controls, and any related JSON matches.
- The UI shows the specific SSP snippets and the evidence request rows.
- The answer summarizes the connection and cites the relevant source IDs.
