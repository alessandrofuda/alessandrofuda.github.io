---
layout: post
title: "Two-Stage Semantic Chunking for RAG in Python: Structural Splitting + Semantic Coherence"
---

*Fixed-size chunking splits text at arbitrary token boundaries, cutting mid-sentence and blending unrelated topics into the same chunk. Here's how to build a two-stage pipeline with LlamaIndex , structural splitting first, semantic coherence second , and why adaptive sizing matters for long documents.*

---

## The Problem With Fixed-Size Chunking

The simplest chunking strategy is a sliding window: split every N tokens with M tokens of overlap. It's easy to implement and works reasonably well on clean, uniform text. It breaks down in two common situations.

**Mid-sentence splits.** A chunk that ends at token 512 may cut a sentence in half. The embedding for that chunk represents a dangling thought , and when the retriever pulls it back, the LLM receives incomplete context. Overlap helps but doesn't eliminate the problem: two consecutive chunks now share a sentence fragment, both pulling each other slightly off-topic.

**Topic bleed.** A 1,024-token window over a textbook chapter will often straddle two sections , the end of "Cellular Respiration" and the start of "Photosynthesis." The embedding averages those topics, making the chunk a poor match for queries about either one.

The alternative is semantic chunking: let the content's own structure guide the split points.

LongTermMemory's `DocumentProcessor` uses a two-stage pipeline , structural splitting followed by semantic coherence , implemented in about 90 lines of Python using LlamaIndex.

---

## The Architecture at a Glance

```
Raw text
    │
    ▼
Stage 1: SentenceSplitter         ← structural: paragraph breaks, chapter boundaries
    │         (respects "\n\n")
    ▼
Stage 2: SemanticSplitterNodeParser  ← semantic: merge/split by embedding similarity
    │         (OpenAI embeddings)
    ▼
Structural heading extraction     ← heuristics on first line + node metadata
    │
    ▼
LLM title fallback                ← GPT-3.5-turbo when no heading found
    │
    ▼
TextChunk objects → Qdrant
```

---

## Stage 1: Structural Splitting With `SentenceSplitter`

The first stage uses LlamaIndex's `SentenceSplitter` to break the document into structurally coherent pieces. The key parameter is `separator="\n\n"` , the splitter preferentially splits on paragraph breaks before falling back to sentence boundaries:

```python
sentence_splitter = SentenceSplitter(
    chunk_size=stage1_chunk_size,
    chunk_overlap=stage1_chunk_overlap,
    separator="\n\n",  # Split on paragraph breaks
)
initial_nodes = sentence_splitter.get_nodes_from_documents([llama_doc])
```

With `chunk_size=1024`, each initial node is at most 1,024 tokens. But because `"\n\n"` is the preferred split point, a section that ends at token 900 followed by a paragraph break will produce a 900-token chunk , no mid-paragraph split , rather than running over into the next section to pad out to 1,024 tokens.

This stage doesn't require any API call. It's pure text processing, fast and free.

---

## Stage 2: Semantic Coherence With `SemanticSplitterNodeParser`

The second stage takes the structural chunks from Stage 1 and re-examines their boundaries using embedding similarity. Adjacent sentences are grouped by semantic similarity , if two consecutive sentences are closely related, they stay in the same chunk; if similarity drops below a threshold, a new split is inserted.

```python
semantic_splitter = SemanticSplitterNodeParser(
    buffer_size=stage2_buffer_size,
    breakpoint_percentile_threshold=stage2_breakpoint_threshold,
    embed_model=self.embed_model,
)
semantic_nodes = semantic_splitter.get_nodes_from_documents(
    [LlamaDocument(text=node.get_content(), metadata=node.metadata)
     for node in initial_nodes]
)
```

The Stage 1 nodes are re-wrapped as `LlamaDocument` objects before being passed to the semantic splitter. This is necessary because `SemanticSplitterNodeParser.get_nodes_from_documents` expects `Document` inputs, not `TextNode` inputs , passing `initial_nodes` directly would raise a type error.

**`buffer_size`** controls how many surrounding sentences are included when computing the embedding for a sentence. `buffer_size=1` means each sentence is embedded with one sentence of context on each side; `buffer_size=3` means three sentences of context. A larger buffer makes the embeddings smoother and more stable, reducing over-splitting on long content.

**`breakpoint_percentile_threshold`** sets how high the similarity drop must be before a split is inserted. At `95`, only the most semantically divergent sentence boundaries become chunk boundaries , the splitter produces fewer, larger chunks. At `97`, even fewer splits.

The `embed_model` is `OpenAIEmbedding(model="text-embedding-3-small")`, initialized once in `DocumentProcessor.__init__` and reused across all documents in the job.

---

## Adaptive Sizing: Short vs. Long Content

The parameters above are not fixed , they switch based on estimated document length:

```python
total_tokens = estimated_total_tokens if estimated_total_tokens is not None else len(text) // 4

if total_tokens > settings.long_content_threshold:   # default: 10,000 tokens
    stage1_chunk_size = settings.long_chunk_size              # 2048
    stage1_chunk_overlap = settings.long_chunk_overlap        # 200
    stage2_buffer_size = settings.long_buffer_size            # 3
    stage2_breakpoint_threshold = settings.long_breakpoint_threshold  # 97
else:
    stage1_chunk_size = 1024
    stage1_chunk_overlap = 200
    stage2_buffer_size = 1
    stage2_breakpoint_threshold = 95
```

The token estimate is `len(text) // 4` , one token per four characters, the standard approximation. At 10,000 tokens the threshold is around 40,000 characters, or roughly 15,20 pages of dense text.

**Why larger chunks for long content?** Each call to `SemanticSplitterNodeParser` embeds every sentence in every Stage 1 node. A 100-page textbook at standard settings (chunk_size=1024) produces ~40 Stage 1 nodes, each of which the semantic splitter processes sentence-by-sentence , potentially hundreds of embedding API calls. At long-content settings (chunk_size=2048, buffer_size=3, threshold=97), the Stage 1 pass produces fewer, larger nodes, the semantic pass is less aggressive about splitting, and the total embedding count drops substantially.

The tradeoff is retrieval granularity: larger chunks are coarser, but for long documents the alternative is prohibitive API cost and latency.

All five parameters are configurable via environment variables, so the thresholds can be tuned without a code change.

---

## The Fallback: Structural Splitting Only

If no OpenAI API key is provided , or if embedding model initialization fails , the semantic stage is skipped:

```python
if self.embed_model is None:
    logger.warning("No embedding model provided, skipping semantic splitting")
    # Fallback to sentence splitter results
    semantic_nodes = initial_nodes
else:
    semantic_splitter = SemanticSplitterNodeParser(...)
    semantic_nodes = semantic_splitter.get_nodes_from_documents(...)
```

Stage 1 output is used as-is. The chunks are structurally clean (paragraph-respecting, size-bounded) but not semantically optimized. For development, testing, or cost-sensitive environments where embedding costs matter more than retrieval quality, this is a usable fallback.

---

## Chunk Enrichment: Section Titles

After chunking, each `TextChunk` gets a `section_title` , a short label that tells the Q&A generator what the chunk is about. This improves Q&A quality: a chunk labeled "The Krebs Cycle" produces more focused questions than unlabeled prose.

Title assignment happens in `append_section_title_to_chunks`, with two priority levels:

**Priority 1 , structural heading extraction.** `_extract_structural_heading` scans each node's metadata and content for heading signals:

```python
# 1. Check node metadata
if 'header' in node.metadata:
    return node.metadata['header']
if 'section' in node.metadata:
    return node.metadata['section']

# 2. Heuristics on first line
first_line = lines[0].strip()
if len(first_line) < 100 and len(first_line) > 3:
    # Numbered section: "3.1 The Krebs Cycle", "Chapter 5"
    if re.match(r'^(\d+\.)*\d+\s+', first_line) or \
       re.match(r'^(Chapter|Section|Part)\s+\d+', first_line, re.IGNORECASE):
        return first_line
    # Standard academic keywords
    if re.match(r'^(Introduction|Conclusion|Abstract|Methods?|Results?|Discussion|...)\s*$',
                first_line, re.IGNORECASE):
        return first_line.strip()
    # Title case, ≤10 words
    if first_line.istitle() and len(first_line.split()) <= 10:
        return first_line
    # All caps, 3,10 words
    if first_line.isupper() and 3 <= len(first_line.split()) <= 10:
        return first_line
```

The `title` metadata key from LlamaIndex is intentionally filtered: if it equals `"document"` (the placeholder passed during wrapping), or if it looks like a filename or file path, it's discarded.

**Priority 2 , LLM-generated title.** When no structural heading is found, `generate_chunk_title_with_llm` calls GPT-3.5-turbo with the first 500 characters of the chunk:

```python
response = self.openai_client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a concise summarizer. Generate only the title, nothing else."},
        {"role": "user",   "content": f"Generate a concise title (maximum 10 words)...\n\nText: {truncated_content}"}
    ],
    max_tokens=30,
    temperature=0.3,
    timeout=10.0
)
```

`max_tokens=30` bounds the response. `temperature=0.3` keeps the title deterministic. The first 500 characters are enough to capture the chunk's topic without sending the full chunk , which would be wasteful for long chunks and isn't needed for a title.

If the LLM returns a title longer than 15 words (the model occasionally ignores the 10-word instruction), it's truncated to 10 words. If the LLM call fails after two retries, `section_title` is set to `None` and a `logger.error` is emitted.

---

## The `TextChunk` Data Model

Every chunk coming out of the pipeline is a Pydantic model:

```python
class TextChunk(BaseModel):
    content: str
    chunk_index: int
    page_number: Optional[int] = None
    section_title: Optional[str] = None
    token_count: Optional[int] = None
    document_id: int = 0
    document_path: str = ""
    filename: str = ""
    description: str = ""
```

`token_count` is the `len(content) / 4` estimate computed in `convert_chunks_to_text_chunks`. `document_id`, `document_path`, `filename`, and `description` are populated by the Celery task after the chunk is returned from `chunk_document` , the processor itself doesn't know about the database record, only the content.

---

## The Complete Pipeline: `chunk_document`

The public entry point is `chunk_document`, which orchestrates the full sequence:

```python
def chunk_document(self, document_path: str, original_filename: str) -> list[TextChunk]:
    # 1. Download from MinIO
    content, content_type = self.download_document(document_path)

    # 2. Extract text (PDF / DOCX / XLSX)
    file_ext = original_filename.lower().split('.')[-1]
    if file_ext == 'pdf':
        text, page_count = self.extract_text_from_pdf(content)
    elif file_ext == 'docx':
        text = self.extract_text_from_docx(content)
    elif file_ext == 'xlsx':
        text = self.extract_text_from_xlsx(content)

    # 3. Two-stage semantic chunking
    semantic_chunks, structural_headings = self.semantic_chunk_text(text, document_title=original_filename)

    # 4. Wrap in TextChunk objects (adds token_count)
    chunks = self.convert_chunks_to_text_chunks(semantic_chunks)

    # 5. Enrich with section titles (structural heading → LLM fallback)
    self.append_section_title_to_chunks(chunks, structural_headings)

    return chunks
```

`page_count` from the PDF extractor is not currently propagated into `TextChunk.page_number` , that field is populated separately when the Celery task has per-page data available. For weblinks, `semantic_chunk_text` is called directly (bypassing `chunk_document`) with a pre-computed token estimate passed as `estimated_total_tokens` to avoid a redundant `len(text) // 4` computation.

---

## What I'd Do Differently

**Cache the structural heading extraction result from Stage 1 into Stage 2.** The current pipeline runs `_extract_structural_heading` on Stage 2 output , nodes that the semantic splitter may have merged or split relative to Stage 1 nodes. Headings that appeared at the start of a Stage 1 node may no longer appear at the start of the corresponding Stage 2 node. Passing heading metadata through the node pipeline (rather than re-extracting from content) would be more reliable.

**Use a token counter instead of `len(text) // 4`.** The character-to-token ratio varies significantly across languages and content types , code, Chinese text, and LaTeX all have different ratios. `tiktoken` with the `cl100k_base` encoding would give exact counts for GPT and embedding models at negligible cost.

**Batch the LLM title calls.** `append_section_title_to_chunks` calls `generate_chunk_title_with_llm` one chunk at a time in a loop. For a document with 40 chunks needing LLM titles, that's 40 sequential API calls. A single prompt with all chunk previews, or a batch of parallel async calls, would reduce wall-clock time substantially.

**Propagate `page_number` from the PDF extractor.** PyMuPDF's block-based extraction processes the document page by page. The page number is available during extraction but not carried into `TextChunk`. For Q&A generation, knowing the source page is useful for generating citations and for debugging retrieval quality.

---

The two-stage approach costs one embedding API call per document at index time , the semantic stage processes every sentence in every Stage 1 node. For a 50-page document on short-content settings, that's on the order of a few hundred embedding vectors. The payoff is chunks that respect both document structure and semantic boundaries, which translates directly to fewer garbage retrievals when a user's flashcard session asks the RAG pipeline for context.

The full implementation is part of [LongTermMemory](https://longtermemory.com) , an AI study platform built on FastAPI, LlamaIndex, Qdrant, and Laravel 12.
