---
layout: post
title: "Preventing Duplicate Background Jobs in Celery with Redis: A Production Pattern"
---

*A user double-clicks "Generate Study Plan". Two parallel Celery workers start processing the same project simultaneously, doubling OpenAI costs and writing duplicate Q&A pairs to the database. Here's how to fix it with a Redis index key , and why TTL alone isn't enough.*

---

## The Bug

[LongTermMemory](https://longtermemory.com) has a Q&A generation pipeline: users upload documents, and a FastAPI service queues a Celery task that runs a RAG pipeline , chunking documents, generating embeddings with OpenAI, producing Q&A flashcard pairs, and calling back to the Laravel backend with the results.

The pipeline is expensive. A moderate document set can cost several cents in OpenAI tokens and take a minute to complete. A double-click on "Generate Study Plan" would trigger two `POST /api/generate-qa` requests in quick succession, each passing the duplicate check (there was none), each creating its own Celery task, both running in parallel on the same project data.

The result: doubled costs, duplicate Q&A pairs in the database, and a callback race where both tasks notify Laravel they're "done" , potentially with partial results overwriting each other.

The fix is a per-project active job index in Redis.

---

## Two Redis Keys, Two Responsibilities

The `JobStorage` class uses two distinct key namespaces:

- `job:{job_id}` , stores the full job metadata as a JSON blob (status, progress counters, Q&A pairs, errors). One key per job, 24-hour TTL.
- `project_job:{project_id}` , stores the currently active `job_id` for a project. One key per project, 24-hour TTL.

The second key is the deduplication index. Its only purpose is to answer one question at request time: *does this project already have a running job?*

```python
def _job_key(self, job_id: str) -> str:
    return f"job:{job_id}"

def _project_job_key(self, project_id: int) -> str:
    return f"project_job:{project_id}"
```

The TTL is set to 86400 seconds (24 hours) on both key types. This is a safety net , if a task crashes without hitting any of its cleanup paths, the lock releases automatically the next day rather than blocking the project forever.

---

## The Index: Set, Check, Clear

Three methods manage the index:

**`set_project_active_job`** , called immediately after the job is created in Redis, before the Celery task is queued:

```python
def set_project_active_job(self, project_id: int, job_id: str) -> None:
    key = self._project_job_key(project_id)
    self.redis_client.setex(key, self.job_ttl, job_id)
```

`setex` sets the key with an atomic TTL in one call. No separate `expire` needed.

**`get_project_active_job`** , called at the start of every `POST /api/generate-qa` request:

```python
def get_project_active_job(self, project_id: int) -> Optional[str]:
    key = self._project_job_key(project_id)
    job_id = self.redis_client.get(key)

    if job_id is None:
        return None

    # Verify the job still exists and is in an active state
    job_data = self.get_job(job_id)
    if job_data is None or job_data.get("status") not in ("queued", "processing"):
        # Job finished or expired , clean up stale index
        self.redis_client.delete(key)
        return None

    return job_id
```

The key detail: the function doesn't just check whether the index key exists , it also checks the referenced job's status. If the job has `status = "completed"` or `status = "failed"`, or if the `job:{job_id}` key has expired, the index is stale and gets deleted. The function returns `None`, allowing a new job to proceed.

This handles the edge case where `clear_project_active_job` was never called , a task that timed out or was killed by the OS before reaching its exception handlers. Without this check, the 24-hour TTL would be the only safety valve. With it, a new request automatically heals the stale state.

**`clear_project_active_job`** , called in the Celery task at every terminal state:

```python
def clear_project_active_job(self, project_id: int) -> None:
    key = self._project_job_key(project_id)
    self.redis_client.delete(key)
```

---

## The FastAPI Endpoint: Check Before Queue

The `POST /api/generate-qa` endpoint in `routers/qa.py` does the duplicate check before creating anything:

```python
@router.post("/generate-qa", response_model=GenerateQAResponse)
async def generate_qa(request: GenerateQARequest, settings: Settings = Depends(get_settings)):
    # Check if there's already an active job for this project
    active_job_id = job_storage.get_project_active_job(request.project_id)
    if active_job_id:
        active_job = job_storage.get_job(active_job_id)
        active_status = active_job["status"] if active_job else "unknown"
        raise HTTPException(
            status_code=409,
            detail=f"A study plan generation is already in progress for this project "
                   f"(status: {active_status}). Please wait for it to complete..."
        )

    # Create job in Redis
    job_id = str(uuid.uuid4())
    job_data = {"id": job_id, "project_id": request.project_id, "status": "queued", ...}
    job_storage.create_job(job_id, job_data)
    job_storage.set_project_active_job(request.project_id, job_id)

    # Queue Celery task , same UUID used as both job_id and Celery task_id
    task = process_content_task.apply_async(
        args=[job_id, request.project_id, ...],
        task_id=job_id,
        queue="rag_processing"
    )

    return GenerateQAResponse(job_id=job_id, status="queued", ...)
```

The `job_id` and the Celery `task_id` are the same UUID. This simplifies status polling: `GET /api/generate-qa/{job_id}` can look up both `job:{job_id}` in Redis and `AsyncResult(job_id)` in Celery using a single identifier.

If `get_project_active_job` returns a non-null value, the endpoint raises 409 immediately , before allocating a job ID, before writing to Redis, before touching the Celery queue. The duplicate request is rejected at the earliest possible point.

---

## The 409 Propagation: FastAPI → Laravel → React

The FastAPI service runs as a private Python microservice, not directly accessible from the browser. Requests flow through Laravel, which proxies them to FastAPI. The 409 is intercepted and re-thrown in `StudyPlansController::callPythonRagApi()`:

```php
private function callPythonRagApi($request, Collection $documents, Collection $weblinks, $userNotes): array
{
    try {
        $response = Http::withHeaders([
            'X-API-Key' => config('services.rag-service.api_key'),
            'Accept'    => 'application/json',
        ])->post(config('services.rag-service.url') . '/api/generate-qa', [
            'project_id' => $request->project_id,
            'user_id'    => $request->user()->id,
            // ...
        ]);

        if ($response->status() === 409) {
            $detail = $response->json('detail') ?? 'A study plan generation is already in progress for this project.';
            throw new HttpException(409, $detail);
        }

        if ($response->failed()) {
            throw new Exception("Python RAG service error: {$response->body()}");
        }

    } catch (HttpException $e) {
        throw $e; // re-throw to preserve HTTP status code
    } catch (Exception $e) {
        throw new Exception("RAG service error: {$e->getMessage()}");
    }

    return $response->json();
}
```

The `catch(HttpException $e){ throw $e; }` re-throw is load-bearing. Without it, the outer `catch(Exception $e)` would catch the `HttpException` (which extends `Exception`) and wrap it in a plain `Exception("RAG service error: ...")`, destroying the 409 status code. The explicit re-throw preserves the `HttpException(409, ...)` so it reaches Laravel's exception handler, which serializes it into a JSON response with the original `detail` message. The React frontend receives a 409 with the error text and displays it inline: "A study plan generation is already in progress for this project."

The message intentionally includes the current job status (`queued` or `processing`) so the user knows whether the first request is still waiting for a worker or actively running.

---

## Cleanup in the Celery Task

`clear_project_active_job` is called at every terminal exit point in `process_content_task`:

```python
# Success
job_storage.update_job(job_id, job_data)
job_storage.clear_project_active_job(project_id)
_notify_laravel_job_finished(job_id, project_id, job_data, settings)

# OpenAI errors (EmbeddingError, LLMError)
except (EmbeddingError, LLMError) as e:
    error_info = _categorize_openai_error(str(e))
    job_storage.set_job_error(job_id, error_info["user_message"], error_info)
    job_storage.clear_project_active_job(project_id)
    _notify_laravel_job_finished(...)

# Any other exception
except Exception as e:
    job_storage.set_job_error(job_id, f"Unexpected error: {str(e)}", {...})
    job_storage.clear_project_active_job(project_id)
    _notify_laravel_job_finished(...)
```

Three branches, three cleanup calls. This covers every path the task can exit through. The index key is deleted before the Laravel callback is sent , so if Laravel immediately triggers a new generation in response to the failure notification, the check at the top of `generate_qa` will find no active job.

The 24-hour TTL is the last line of defense for situations the code can't handle: a worker process killed by OOM, a Docker container restarted mid-task, a Redis connection error in the cleanup call itself.

---

## Why Not Celery's Built-In Task Result Backend?

Celery has a native result backend (Redis, database, or others) that stores task state , `PENDING`, `STARTED`, `SUCCESS`, `FAILURE`. It's tempting to use this directly for deduplication: store the last task ID per project, check `AsyncResult(task_id).state`.

The issue is visibility boundaries. The Celery result backend tracks task state from Celery's perspective. The custom `job:{job_id}` Redis key tracks job state from the application's perspective , including progress counters, Q&A pair counts, error details, and the multi-stage pipeline status that Celery has no concept of. The two states can diverge: a task that's `STARTED` in Celery may be on step 2 of 6 in the pipeline, and the job key reflects that granularity.

The `project_job:{project_id}` index is a thin layer on top of the existing job tracking system. It adds one key per project, costs one Redis read per incoming request, and doesn't require polling Celery at all. The check in `get_project_active_job` calls `get_job()` (a Redis GET on `job:{job_id}`) rather than `AsyncResult(job_id).state` , staying within the same storage layer.

---

## What I'd Do Differently

**Use `SET NX` for atomic lock acquisition.** The current implementation calls `create_job` then `set_project_active_job` as two separate Redis writes. In theory, two simultaneous requests could both pass the `get_project_active_job` check before either has written the index key. Using `SET project_job:{project_id} {job_id} NX EX 86400` (set if not exists, with TTL) would make the lock acquisition atomic , only one of the two requests would succeed, and the other would get the key's existing value on its next read. For the current traffic volume this race window is negligible, but it's the correct approach at scale.

**Expose a job cancellation hook to the frontend.** The `POST /api/generate-qa/{job_id}/cancel` endpoint exists on the FastAPI side and calls `celery_app.control.revoke(job_id, terminate=True)` followed by `clear_project_active_job`. But the React frontend has no cancel button , users who trigger a generation and want to abort it have no way to do so short of waiting it out. Surfacing this as a UI action would also naturally resolve the UX problem that prompted the deduplication fix in the first place.

**Log the duplicate attempt for cost attribution.** When a 409 fires, the only record is a `logger.warning()` line in the FastAPI service. Persisting a lightweight audit record (project ID, timestamp, rejected job details) would make it easy to track which projects hit the duplicate guard most often , useful data if per-project generation quotas become relevant.

---

The core pattern is simple: one Redis key per project, pointing to the active job ID. The complexity is in the edge cases , stale keys after unexpected termination, the two-key read in `get_project_active_job`, the three-branch cleanup in the Celery task. Getting those right is what separates a deduplication scheme that works in testing from one that holds up in production.

The full implementation is part of [LongTermMemory](https://longtermemory.com) , an AI study platform built on FastAPI, Celery, Redis, and Laravel 12.
