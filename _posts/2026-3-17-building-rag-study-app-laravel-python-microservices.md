---
layout: post
title: "Building a RAG-Powered Study App: Laravel + Python Microservices"
---

*How I combined Laravel, FastAPI, Celery, Qdrant, and OpenAI into an AI study platform: what worked, what didn't, and the chunking problem nobody warns you about.*

A few years ago I was grinding through certification study material , thick PDFs, documentation pages, whitepapers , and kept hitting the same wall: the tools that could help me learn efficiently were either too dumb (static flashcard decks you had to write yourself), too expensive, or didn't understand *my* material. What I wanted was something that could read my PDFs and generate questions for me, then schedule those questions based on how well I actually knew them.

So I built it. LongTermMemory is a SaaS study platform that uses Retrieval-Augmented Generation (RAG) to auto-generate question-answer pairs from uploaded materials and implements spaced repetition to move knowledge into long-term memory. This post is a technical walkthrough of the interesting engineering decisions, the mistakes I made, and specifically the one problem that took longer to solve than anything else: chunking.

---

## The Architecture Decision: Why Two Languages?

My first instinct was to build everything in Laravel. I've been writing PHP professionally for years, Laravel is excellent, and managing two runtimes, two Dockerfiles, and two test suites isn't thrilling.

The problem is that the AI/RAG ecosystem lives in Python. LlamaIndex, LangChain, the OpenAI Python client, all of the tooling for embeddings and vector operations , it's mature, well-documented, and under active development. The PHP equivalents are either nonexistent or years behind.

The compromise: **Laravel handles everything product-concern** , authentication, billing, user management, the REST API the frontend talks to, email notifications, database schema. **FastAPI + Celery handles everything AI-concern** , document ingestion, chunking, embedding generation, vector storage, Q&A generation. The two services communicate over an internal Docker network.

Here's the rough topology:

```
React (5173)
    │
    ▼
Nginx → PHP-FPM (Laravel 12)      ←→  MySQL
                │
                ▼
         FastAPI (8000)
                │
         Celery Worker  ←──────────── MinIO (raw documents)
                │        ←──────── Redis (broker + job state)
         ┌──────┴──────┐
         ▼             ▼
      Qdrant        OpenAI API
   (vectors)       (embeddings + LLM)
```

Documents live in MinIO (S3-compatible object storage). When a user uploads a PDF, Laravel stores it in MinIO and records the metadata in MySQL. When they trigger Q&A generation, Laravel POSTs a job request to the FastAPI service. Celery picks it up, retrieves the files from MinIO, processes them, and when done POSTs a callback to Laravel with the results.

---

## Async Processing and the Push Callback Model

Document processing is slow. A large PDF can take 30,120 seconds: extract text, chunk it semantically, generate embeddings for each chunk, store vectors in Qdrant, run the LLM to generate Q&A pairs. You can't hold an HTTP connection open for that long.

The flow is: Laravel calls `POST /api/generate-qa` → FastAPI immediately returns a `job_id` → Celery picks up the task → when done, Celery calls back to Laravel with the results.

I chose push callbacks over polling for the same reason webhooks are better than polling: the server-side work happens exactly once, at the right time, rather than on every tick of a polling loop.

```python
# When the Celery task finishes, it notifies Laravel directly
def _notify_laravel_job_finished(job_id, project_id, job_data, settings):
    payload = {
        "job_id": job_id,
        "project_id": project_id,
        "status": job_data.get("status"),
        "qa_pairs": job_data.get("qa_pairs", []),
        "error": job_data.get("error"),
    }
    url = f"{settings.laravel_app_url}/api/job-finished"
    with httpx.Client(timeout=10.0) as client:
        client.post(url, json=payload, headers={"X-API-Key": api_key})
```

Laravel receives this at a dedicated callback endpoint, saves the Q&A pairs, and fires an email notification to the user , all immediately when the job finishes.

### Preventing Duplicate Jobs

One early bug: if a user clicked "Generate Study Plan" twice quickly, two Celery jobs would run in parallel, both writing Q&A pairs to the same project , duplicate questions and double API costs.

The fix is a Redis key per project: `project_job:{project_id}`. Before queuing a new task, the API checks if that key exists and the referenced job is still active. If so, it returns HTTP 409. Laravel propagates this to the frontend as "generation already in progress." The key is cleared when the job completes, fails, or is cancelled.

---

## The Hardest Problem: Chunking

This is the part nobody really prepares you for when you read RAG tutorials.

### Naive chunking is terrible

The obvious first approach is fixed-size chunking: split the document into 512-token windows with some overlap. Quick to implement, works on toy examples. In practice the Q&A quality was noticeably bad , questions would reference "the above equation" or "as mentioned in the previous section" with no context for either, because the split happened mid-concept.

### Semantic chunking with LlamaIndex

LlamaIndex's `SemanticSplitterNodeParser` uses embedding similarity between consecutive sentences to decide where to split. Instead of splitting every N tokens, it splits when the semantic distance between adjacent sentences exceeds a threshold , keeping conceptually related content together.

My implementation uses a two-stage approach: first `SentenceSplitter` for structural splits on paragraph breaks, then `SemanticSplitterNodeParser` for semantic coherence within those units. The result is chunks that read like coherent paragraphs rather than arbitrary text windows.

### The length problem

Here's the thing nobody tells you: **the parameters that work well for a 10-page article are completely wrong for a 300-page textbook.**

With the same settings on a long document you get hundreds of tiny chunks, many of them mid-sentence fragments. The LLM generates questions that are too narrow, testing individual sentences rather than concepts. Embedding costs scale linearly with chunk count , a 300-page book produces far more chunks than you'd want.

I discovered this when a user uploaded a comprehensive textbook and the generation took 8 minutes and produced 400+ Q&A pairs, most of them nearly identical questions about adjacent paragraphs.

The fix is dynamic parameter selection based on estimated content length:

```python
total_tokens = estimated_total_tokens if estimated_total_tokens else len(text) // 4

if total_tokens > 10_000:  # ~15 pages
    stage1_chunk_size = 2048
    stage2_buffer_size = 3
    stage2_breakpoint_threshold = 97  # only split at major topic shifts
else:
    stage1_chunk_size = 1024
    stage2_buffer_size = 1
    stage2_breakpoint_threshold = 95
```

For long content: larger chunk size, wider semantic buffers, higher breakpoint threshold. The result is ~75% fewer chunks for book-length content, with each chunk containing a full concept.

### The `breakpoint_percentile_threshold` confusion

This took me embarrassingly long to get right. The parameter name suggests a higher value means more splits, but it's the opposite. The threshold is a percentile of embedding distances across all sentence pairs. Setting it to the 97th percentile means "only split when the distance is in the top 3% of all distances" , only the most dramatic topic shifts trigger a split. **Higher = fewer splits = larger chunks.**

My initial instinct was to lower the threshold for long documents. That made things worse. For long documents, you *want* fewer, larger chunks , you're looking for major topic boundaries, not every paragraph break.

### Cost impact

Chunk count directly drives OpenAI API costs. Every chunk needs an embedding (input cost). Every chunk generates one Q&A pair (completion cost). If your 200-page textbook creates 800 chunks instead of 200, you're paying 4x. Adaptive chunking isn't just a quality improvement , it's a billing concern.

---

## Making Q&A Generation Actually Good

Once chunking is right, quality depends on how you use retrieved context and how you prompt the LLM.

### RAG retrieval for question generation

The naive approach: for each chunk, ask the LLM to generate a question. The problem is that a single chunk often lacks context , it references concepts defined elsewhere.

The better approach: before generating a question for a chunk, retrieve the 3 most semantically similar chunks from Qdrant. Include those as "related context" in the prompt. The LLM can now generate questions that test understanding across related concepts.

The 0.7 cosine similarity threshold matters: below it, the "related" chunks aren't actually related, they just share common words. Including irrelevant context actively hurts question quality.

### Prompt engineering

The system prompt is terse and specific , an expert educational content specialist designing for mastery learning. The user message template enforces constraints: the question must test conceptual understanding (not factual recall), be self-contained, and promote long-term retention.

Key insight: **"quality over quantity" as an explicit instruction in the prompt measurably improves output.** Without it, the LLM generates multiple surface-level questions ("What is X?") instead of one deeper one ("How does X relate to Y, and what are the implications for Z?").

The LLM returns structured JSON with `question`, `answer`, `key_concepts` (array), and `difficulty_level` (easy/medium/hard) , all stored in MySQL and exposed to the frontend for filtering.

---

## Spaced Repetition

Spaced repetition schedules reviews at increasing intervals based on recall performance. The SM-2 algorithm is the most widely used variant: performance is rated 1,5, and the next review interval is computed from the previous interval, the performance score, and an ease factor that adjusts over time.

The current schema stores Q&A pairs and their scheduling state together: `scheduled_at = NULL` means the item is new and has never been studied. Email reminders use a push model , an hourly artisan command finds users whose local time is 8 AM and sends a single consolidated email listing due items.

The study session UI , answering questions, rating recall quality, seeing the interval adjust , is the next major frontend feature to build.

---

## Production Gotcha: Celery Doesn't Auto-Reload

The Celery gotcha that everyone hits: **Celery workers do not auto-reload code changes.** FastAPI (via Uvicorn with `--reload`) picks up changes automatically. Celery doesn't. If you modify `celery_tasks.py` or any service module it imports and don't restart the worker, the old code keeps running.

```bash
docker compose restart celery-worker
```

The symptom is confusing: your FastAPI endpoints reflect the new code, but background processing behaves as if nothing changed. This is now in every CLAUDE.md and README for the project, and I still forget it regularly.

---

## What I'd Do Differently

**Start with semantic chunking from day one.** I started with fixed-size chunks as a "quick first pass" and spent more time undoing that than I would have spent implementing semantic chunking correctly from the start.

**Adaptive chunk sizing should be a first-class concern.** I didn't think about variable document lengths until users started uploading textbooks. PDFs range from a 2-page note to a 500-page manual and need fundamentally different treatment.

**Use a proper task result store earlier.** I started tracking Celery job state with ad-hoc Redis key patterns and built the abstraction layer later as things grew. Starting with a clean interface for job state (create, read, update, expire, index by project) would have saved refactoring time.

**The push callback model was the right call.** I've worked on systems that poll job status from a frontend timer. It always becomes a source of race conditions and extra load. The callback model is simpler to reason about and delivers results faster.

---

## Open Problems

- **Multi-modal documents**: PDFs with diagrams and mathematical notation are common in technical study material. Current text extraction ignores images entirely.
- **Self-hosted LLM**: Some users are uncomfortable uploading sensitive professional material to an OpenAI-backed system. LlamaIndex supports provider-swapping; the work is validating quality parity.
- **Chunk attribution**: Q&A pairs are stored with no reference back to the specific chunks they were generated from. Adding a `source_chunk_id` would enable "show me the source" functionality in the study interface.

---

The most interesting engineering happened at the intersection of the two services. The boundary between Laravel and FastAPI isn't just a language split , it forced clear thinking about which concerns belong where. Auth, billing, user data: PHP. Embeddings, vectors, async AI tasks: Python.

The chunking problem genuinely surprised me. Most RAG resources treat chunking as a detail , pick a size, move on. In practice it's where the most user-visible quality variation comes from, and adaptive sizing based on document length is not optional if your use case involves documents of wildly different lengths.

If you're building something similar, the project is at [longtermemory.com](https://longtermemory.com).
