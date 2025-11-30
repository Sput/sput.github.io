---
layout: post
title: "From Weekly Updates to Instant Insight: A RAG-Powered Status App"
date: 2025-11-10 07:07:07 +0100
---

*For all code discussed here, visit the Github repository*  
https://github.com/Sput/weekly_status_RAG

Weekly updates are the heartbeat of most engineering teams—but too often, they’re trapped in scattered documents, chat threads, or inboxes. Context disappears, insights lag, and managers spend hours piecing together what actually happened across the organization.

This app solves that by turning weekly updates into a **retrieval-augmented knowledge surface**.

It combines a simple posting UI with an AI-powered chat that can answer questions like _“What did the mobile team ship last week?”_—grounded directly in the latest updates from your team.

The result: a **lightweight, explainable status hub** that transforms free-form updates into actionable, traceable insight.

---

### **TL;DR**

This app turns scattered weekly engineering updates into a searchable, AI-powered knowledge layer using retrieval-augmented generation (RAG). Each update is stored as a vector using embeddings so the system can answer questions like “What did the mobile team do last week?” with grounded, citation-rich responses. By pairing a clean UI with a FastAPI backend, pgvector search, and explainable LLM outputs, the app becomes a lightweight but powerful status hub that keeps teams aligned without meetings or manual digging.

---
## **Under the Hood: RAG & Embeddings**

### What is RAG?

**Retrieval-Augmented Generation (RAG)** is an AI technique that combines information retrieval with language generation to produce grounded, trustworthy answers.

Instead of relying solely on a model’s internal knowledge, RAG:

1. **Searches your structured knowledge** (here: weekly updates stored as vectors)
2. **Retrieves relevant snippets**
3. **Feeds them into the LLM** to generate grounded responses

This ensures accuracy, transparency, and adaptability. Users see exactly which sources were used, teams can update context instantly, and all responses stay anchored to real and recent data.

---

### **Embeddings 101**

Embeddings are numerical vectors representing a text’s **semantic meaning**.  
Similar sentences → vectors pointing in similar directions.

### **Vector Creation**

- Use the same embedding model for all text
- Light normalization only
- Long text → chunk + average
- Store vectors for reuse

---

### **Cosine Similarity**

![Cosine Similarity](/images/Screenshot%202025-11-12%20at%2012.02.40%20PM.png)

Cosine similarity measures how close two vectors are in direction.

If vectors are normalized (they are here), the dot product equals cosine similarity.

---

## **What the App Does**

Part **status feed**, part **AI query engine**, built for clarity and speed.

### **You can:**

- **Post updates** — single-field, low-friction input  
- **Browse the feed** — clean timeline with filters  
- **Ask questions** — RAG-driven chat interface  
- **See the sources** — every answer includes citations, timestamps, similarity scores

### **Why it matters**

- Eliminates “status hunting”  
- Turns updates into structured data  
- Enables high-signal summaries  
- Helps teams stay aligned asynchronously  

---

## **How RAG Works in This Application**

### 1. Core Steps

1. **Turn updates into vectors**  
   ![Status Update Vectorization](/images/Screenshot%202025-11-13%20at%207.06.31%20AM%201.png)

2. **Find similar updates** using pgvector `<=>` similarity  
   ![Similarity Calculation](/images/Screenshot%202025-11-13%20at%207.10.16%20AM.png)

3. **Feed retrieved context to the LLM**  
   ![RAG Augmentation](/images/Screenshot%202025-11-13%20at%207.13.55%20AM.png)

---

### **2. Data Flow Overview**

1. User posts an update → stored + embedded (1536-d vectors)
2. When someone asks a question, FastAPI:
   - Embeds the question
   - Uses pgvector similarity to retrieve top matches
   - Returns snippets + an LLM summary (GPT-4o-mini)
3. Frontend shows:
   - **Answer**
   - **Sources**
   - **Similarity scores**

This ensures every summary is grounded and auditable—**no hallucinations**.

---

## **Architecture Overview**

### **Frontend (Next.js 15 + React 19)**

Minimal UI using **shadcn/ui**.

- **Updates Page** (`/updates`)
  - Post new updates
  - Filter and browse feed
  - Embeddings auto-generated via Supabase triggers

- **Chat Page** (`/chat`)
  - Ask questions like “What did the infra team do this week?”
  - Shows both model answer + context snippets

- **API Proxy**  
  Ensures same-origin calls to the backend.

---

### **Backend (FastAPI)**

FastAPI handles all AI orchestration and retrieval.

**`api/main.py` handles:**

1. Embedding creation (OpenAI)
2. pgvector similarity search
3. Fallback logic (recency if embeddings unavailable)
4. LLM prompting (GPT-4o-mini)
5. Returning:
   - context[]
   - answer
   - debug metadata

The separation gives you secure LLM orchestration without exposing secrets to the browser.

---

## **Supabase Data Layer**

### **Tables**

- `users`
- `updates` (text + embedding + timestamps)
- `roles`

### **Views & Functions**

- `latest_updates_per_user`
- `match_latest_updates()`
- optional `create_embedding` trigger

### **Indexes**

- `ivfflat` over embeddings (cosine distance)

---

## **Why FastAPI + Next.js Instead of Only One**

### 1. **Security**
Secrets never touch the browser.

### 2. **Separation of concerns**
- Frontend = UX
- Backend = embeddings + retrieval + LLM logic

### 3. **Python ecosystem**
FastAPI integrates cleanly with vector tools, analytics, and async orchestration.

### 4. **One clean API contract**
- Single source of truth: The contract is defined once in FastAPI's Pydantic models, not duplicated across frontend/backend
- Frontend isolation: The browser never knows FastAPI exists—no CORS, no cross-origin complexity, just standard same; origin fetch calls
- Backend flexibility: Swap FastAPI for another service, add middleware (caching, rate limiting, auth), or change internal logic; the frontend contract stays the same
- Type safety: Pydantic validates requests and responses at the FastAPI boundary, catching contract violations before they reach your code
