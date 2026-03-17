# Blog Post Ideas

---

## 1. Passwordless Auth in Laravel 12: Implementing Magic Link Login with Sanctum

**Target keywords**: `magic link authentication laravel`, `passwordless login php`, `laravel sanctum magic link`

**Angle**: Walk through the full flow — `POST /api/auth/magic-link` → signed URL with authorization code → `POST /api/auth/exchange` → Sanctum token stored in `localStorage` as Bearer. Cover the `hasValidSignature()` check, why Sanctum `currentAccessToken()` matters (vs plain `actingAs()` in tests), the React `AuthCallback.tsx` double-execution issue with `StrictMode`, and timezone auto-detection on login. Real code from `AuthController` and `AuthTest.php`.

---

## 2. The SM-2 Algorithm in Practice: Building a Spaced Repetition System in Laravel

**Target keywords**: `spaced repetition algorithm implementation`, `SM-2 php`, `anki algorithm tutorial`, `flashcard scheduling algorithm`

**Angle**: Explain SM-2 (performance 1–5, ease factor, interval calculation) then show how it maps to a real MySQL schema: `scheduled_at NULL` = new item, `is_strict`, `difficulty_level easy/medium/hard` from the LLM, `session_id`, the `StudyPlan` factory `due()` and `future()` states. Show the 4-level self-assessment UI (`again/hard/good/easy` → 1min/10min/4days/8days) and the `POST /api/qa-item-evaluation` callback. Honest section on what's still TODO (full SM-2 interval math is planned post-study-session-UI).

---

## 3. SEO for React SPAs Without SSR: Puppeteer Prerendering in Production

**Target keywords**: `react spa seo prerendering`, `puppeteer prerender react`, `vite seo static html`, `react without nextjs seo`

**Angle**: The problem — SPAs are invisible to social crawlers and weak for Google. The decision against Next.js (would require a full rewrite). The two-variant route pattern: `LandingRoute` checks `localStorage.getItem('auth_token')` → `LandingPublic` for crawlers, `Landing` for authenticated users. The `scripts/prerender.mjs` Puppeteer script: clears localStorage, renders `LandingPublic` in headless Chrome, saves full HTML to `dist/index.html`. The `react-helmet-async` layer on top. What this solves (FAQ, pricing, hero text in raw HTML) and what it doesn't (dynamic routes).

---

## 4. Timezone-Aware Email Notifications in Laravel: Sending at 8 AM in Every User's Local Time

**Target keywords**: `laravel send email timezone`, `laravel scheduled notifications timezone`, `php 8am local time cron`, `laravel artisan command timezone`

**Angle**: The deceptively hard problem — hourly cron that only fires for users whose local time is 8 AM. The `Carbon::now($tz)->hour === 8` loop across all IANA timezones. The two-query pattern that avoids N+1 (EXISTS subquery for candidate users, single JOIN for due projects). The `insertOrIgnore` dedup on `notification_logs(user_id, type, sent_date)` to handle concurrent command runs. The 1-second incremental dispatch delay to stay within Resend's 2 req/s rate limit. The signed unsubscribe URL with `URL::signedRoute()` and `hasValidSignature()`. Auto-re-enable on next login.

---

## 5. Preventing Duplicate Background Jobs in Celery with Redis: A Production Pattern

**Target keywords**: `celery prevent duplicate tasks`, `celery redis job deduplication`, `python background job idempotency`, `celery 409 duplicate task`

**Angle**: The bug — double-click triggers two parallel Celery jobs writing to the same project, doubling OpenAI costs and producing duplicate Q&As. The fix: a Redis index key `project_job:{project_id}` → `job_id` with 24-hour TTL. The `set_project_active_job` / `get_project_active_job` / `clear_project_active_job` abstraction in `utils/job_storage.py`. The auto-cleanup: checking whether the referenced job is still active before returning the lock (handles stale keys after TTL expiry edge cases). The 409 response propagation from FastAPI → Laravel → React ("generation already in progress"). Comparison with Celery's built-in task result backend and why a custom Redis index is cleaner for cross-service job visibility.

---

## 6. Two-Stage Semantic Chunking for RAG in Python: Structural Splitting + Semantic Coherence

**Target keywords**: `semantic chunking python`, `llamaindex semantic splitter`, `rag chunking strategy`, `document chunking for embeddings`, `SemanticSplitterNodeParser`

**Angle**: Why fixed-size chunking breaks RAG quality (mid-sentence splits, context bleed across topics). The two-stage pipeline in `DocumentProcessor`: Stage 1 uses LlamaIndex's `SentenceSplitter` with `"\n\n"` as separator to respect paragraph and chapter boundaries; Stage 2 feeds those structural chunks into `SemanticSplitterNodeParser` (OpenAI embeddings) to merge or split further based on semantic similarity. Adaptive sizing: short content (< 10,000 tokens) uses `chunk_size=1024, buffer_size=1, breakpoint_percentile_threshold=95`; long content switches to `chunk_size=2048, buffer_size=3, threshold=97` to avoid over-splitting large documents. Chunk enrichment: `_extract_structural_heading()` parses Markdown-style headings from the chunk's surrounding text; if none found, `generate_chunk_title_with_llm()` calls GPT-3.5-turbo to generate a ≤10-word title. The resulting `TextChunk` objects (content, section_title, token_count, page_number, document metadata) go directly into the Qdrant embedding pipeline. Honest tradeoff: the semantic stage adds one OpenAI embeddings call per document at index time — cost and latency vs. retrieval quality.
