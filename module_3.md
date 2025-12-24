# Summary
- TaskGroup: if one task fails (non-cancellation exception), the other tasks will be cancelled and the error is raised as an exception group.   
- `asyncio.gather()` default behavior: if one awaitable raises, the first exception is propagated but the other awaitables **won’t be cancelled** and can keep running.   
- `asyncio.create_task()` needs a strong reference (or TaskGroup / a collection) because the event loop keeps only weak references and “fire-and-forget” tasks can disappear mid-execution.   
- Queue + worker pool is for bounded memory/backpressure (don’t spawn 100k tasks); semaphore is for bounding *in-flight* resource usage (e.g., max open pages / max concurrent HTTP calls).[1]
- Don’t swallow `CancelledError`; do cleanup in `finally`, then re-raise, because structured concurrency features (TaskGroup/timeout) rely on cancellation internally.   

# Key Concepts / Patterns
- **Structured concurrency (TaskGroup)**: “All tasks are awaited when the context manager exits,” and first non-cancellation failure cancels the remaining tasks and raises an `ExceptionGroup`/`BaseExceptionGroup`.   
- **Cancellation model**: cancellation injects `CancelledError` “at the next opportunity,” and catching it should generally be followed by propagating it after cleanup.   
- **Backpressure with `asyncio.Queue(maxsize=...)`**: `await put()` blocks when the queue reaches maxsize until an item is removed, so producers can’t outrun consumers.[1]
- **Completion tracking (`join()`/`task_done()`)**: unfinished-tasks count increments on `put()`, decrements on `task_done()`, and `join()` unblocks when it reaches zero.[1]
- **Semaphore as infra-owned limiter**: semaphore is a shared object with an internal counter; `acquire()` blocks when it hits zero until `release()` happens.   
- **`shield()` semantics (correction)**: `asyncio.shield(aw)` protects `aw` from being cancelled if the *caller coroutine* is cancelled; it does **not** protect `aw` from failing due to its own internal exception.   
- **Timeout semantics**: `asyncio.timeout()` cancels the current task and transforms the internal `CancelledError` into `TimeoutError`, and `TimeoutError` can only be caught *outside* the context manager.   

# Examples / Scenarios
- **Scraper orchestration**: app/orchestrator owns a bounded `Queue` (work backlog + backpressure), and infrastructure client owns a `Semaphore` (resource limits like “max open pages”).[1]
- **Why queue + semaphore together**: semaphore limits concurrent “active fetches,” while queue limits how much pending work is buffered so you don’t create 100k Task objects at once.[1]
- **Chunking alternative (your idea)**: batching 200 URLs at a time with `gather()` bounds tasks similarly to a queue, but a queue/worker pool keeps throughput steadier when durations vary (workers immediately pull next items instead of waiting for the slowest in a batch).[1]

# Trade-offs & Gotchas
- **Gotcha: 100k `create_task()` even with a semaphore**  
  - Even if only 5 tasks pass the semaphore at a time, you still allocate 100k Task/coroutine objects and keep them pending, which increases memory and scheduling overhead.   
- **Gotcha: `gather()` doesn’t cancel siblings on exception by default**  
  - If you rely on “fail-fast cancels the rest,” use TaskGroup or implement explicit cancellation logic.   
- **Gotcha: swallowing `CancelledError` breaks structured tools**  
  - TaskGroup/timeout use cancellation internally and can misbehave if a coroutine suppresses cancellation without using the required internal patterns.   
- **Gotcha: weakrefs for tasks**  
  - Fire-and-forget tasks can get garbage collected unless you keep a strong reference or use a structured scope.   
- **Correction (from Q5)**  
  - `shield()` does **not** protect from “DB connection fails”; it protects from the *caller being cancelled* (e.g., request cancelled/timeout), while internal failures still propagate through shield.   

# Questions / Follow-ups
- Where should “job state” live: what exact shape should a queue item have in your scrapers (`url`, `proxy`, `attempt`, `correlation_id`, etc.)?  
- In FastAPI: which background workloads must be “owned” by the app lifecycle (startup/shutdown), and which can be per-request?  
- When would you prefer batch chunking (`gather` per chunk) vs a continuous queue/worker pool in your real projects?  

# Active Recall Questions
- If one task inside `TaskGroup` raises `ValueError`, what happens to sibling tasks, and what exception type is raised to the caller?   
- Why can `create_task()` “disappear” without a strong reference, and how do you prevent it?   
- What does `Queue(maxsize=N)` give you that a `Semaphore(N)` does not?[1]
- Why is catching `CancelledError` and returning silently usually a bug in asyncio code?   
- What does `asyncio.shield()` actually protect against, and what does it not protect against?




  
**production-ready template** that incorporates every Module 3 concept: structured concurrency (`TaskGroup`), backpressure (`Queue`), infrastructure layering (`Semaphore`), and resilience (`retries` decorator + `timeout`).

It simulates a robust scraper pipeline.

### The Architecture
1.  **Infrastructure Layer (`AsyncHttpClient`)**: Owns the `Semaphore` and connection logic. It uses a **Context Manager** to manage lifecycle (open/close sessions) and a **Decorator** for retries.
2.  **Application Layer (`Worker` / `Producer`)**: Decoupled via a **Queue**.
3.  **Orchestrator (`main`)**: Wires everything together using a `TaskGroup` and handles clean shutdown.

### `robust_scraper_template.py`

```python
import asyncio
import logging
import random
from dataclasses import dataclass
from typing import Optional, List
from contextlib import asynccontextmanager
import functools

# --- Configuration ---
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%H:%M:%S"
)
logger = logging.getLogger("ScraperSystem")

# --- 1. Infrastructure Layer (Decorators & Client) ---

def with_retries(max_attempts: int = 3, delay: float = 1.0):
    """Decorator: Application-agnostic retry logic."""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    if isinstance(e, asyncio.CancelledError):
                        raise  # Never swallow cancellation!
                    last_exc = e
                    logger.warning(f"Attempt {attempt} failed: {e}. Retrying...")
                    await asyncio.sleep(delay * attempt)  # Linear backoff
            logger.error(f"All {max_attempts} attempts failed.")
            raise last_exc
        return wrapper
    return decorator

@dataclass
class ScrapeResult:
    url: str
    status: int
    data_len: int

class AsyncHttpClient:
    """
    Infrastructure wrapper. 
    Owns the concurrency limit (Semaphore) and session lifecycle.
    """
    def __init__(self, concurrency_limit: int = 5):
        self._sem = asyncio.Semaphore(concurrency_limit)
        self._session_open = False

    async def __aenter__(self):
        # Simulating opening a persistent HTTP session (e.g., aiohttp.ClientSession)
        self._session_open = True
        logger.info("--- HTTP Session Opened ---")
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        # Simulating cleanup
        self._session_open = False
        logger.info("--- HTTP Session Closed ---")

    @with_retries(max_attempts=3)
    async def fetch(self, url: str) -> ScrapeResult:
        """
        Bounded by Semaphore. Includes retries and timeouts.
        """
        if not self._session_open:
            raise RuntimeError("Session is closed")

        # 1. Acquire Semaphore (Infrastructure constraint)
        async with self._sem:
            # 2. Set strict Timeout (SLA constraint)
            async with asyncio.timeout(2.0):
                logger.debug(f"Fetching {url}...")
                
                # Simulate network I/O and random flakiness
                await asyncio.sleep(random.uniform(0.1, 0.5))
                if random.random() < 0.1:
                    raise ConnectionError("Simulated network blip")
                
                return ScrapeResult(url=url, status=200, data_len=random.randint(100, 5000))

# --- 2. Application Layer (Workers & Logic) ---

async def worker(worker_id: int, queue: asyncio.Queue, client: AsyncHttpClient):
    """
    Consumer: Reads from queue, uses client to fetch, processes result.
    Ends when CancelledError is raised.
    """
    logger.info(f"Worker {worker_id} started")
    try:
        while True:
            # Wait for a job
            url = await queue.get()
            try:
                # Process the job
                result = await client.fetch(url)
                logger.info(f"✅ Worker {worker_id}: {result.url} (Size: {result.data_len})")
                
                # Simulate CPU-heavy parsing (blocking!) -> Offload to thread
                # await asyncio.to_thread(parse_heavy_html, result.data) 
                
            except TimeoutError:
                logger.error(f"❌ Worker {worker_id}: {url} Timed Out")
            except Exception as e:
                logger.error(f"❌ Worker {worker_id}: {url} Failed: {e}")
            finally:
                # Critical: Mark item as done regardless of success/failure
                queue.task_done()
    except asyncio.CancelledError:
        logger.info(f"Worker {worker_id} stopping...")
        raise  # Propagate cancellation to TaskGroup

async def producer(queue: asyncio.Queue, target_count: int):
    """
    Producer: Feeds the queue. Pauses (Backpressure) if queue is full.
    """
    logger.info("Producer started")
    for i in range(target_count):
        url = f"https://example.com/product/{i}"
        
        # This BLOCKS if queue is full, preventing memory explosion
        await queue.put(url) 
        logger.debug(f"Queued: {url}")
        
    logger.info("Producer finished generating URLs")

# --- 3. Orchestration ---

async def main():
    # Setup dependencies
    queue = asyncio.Queue(maxsize=10) # Backpressure Buffer
    
    # Context Manager for Client Lifecycle
    async with AsyncHttpClient(concurrency_limit=5) as client:
        
        # Structured Concurrency Scope
        try:
            async with asyncio.TaskGroup() as tg:
                # 1. Spawn Workers
                workers = [
                    tg.create_task(worker(i, queue, client)) 
                    for i in range(3)
                ]
                
                # 2. Run Producer (as a task, or await directly if it was a single function)
                producer_task = tg.create_task(producer(queue, target_count=20))
                
                # 3. Wait for Producer to finish
                # Note: We can't simple `await` the task inside TG without blocking exit,
                # but we need to know when *production* is done to start shutdown.
                # In this pattern, we let the producer task finish on its own.
                pass 
                
                # 4. Wait for Queue to Drain (all items processed)
                # We need to wait for the producer to finish putting items FIRST.
                await producer_task
                await queue.join()
                
                # 5. Signal Shutdown
                logger.info("All work done. Cancelling workers...")
                for w in workers:
                    w.cancel()
                    
        except ExceptionGroup as eg:
            logger.error(f"Crashed with errors: {eg}")
        except asyncio.CancelledError:
            logger.info("Main cancelled (Signal?)")

if __name__ == "__main__":
    try:
        # asyncio.run handles the loop and cleanup
        asyncio.run(main())
    except KeyboardInterrupt:
        # Because we use TaskGroup and proper cancellation, 
        # Ctrl+C usually exits cleanly, but this catches the final exit.
        pass
```

### Key Takeaways in This Code
1.  **`maxsize=10`**: This creates **backpressure**. If the workers are slow (network lag), the `Producer` will eventually block at `queue.put()`. This prevents you from loading 1M URLs into RAM.
2.  **`AsyncHttpClient`**: This is your **Infrastructure boundary**.
    *   It owns the `Semaphore`, so no matter how many workers you spawn (3 or 300), you never exceed 5 concurrent requests.
    *   It wraps `timeout` and `retries`, keeping the `worker` logic clean.
3.  **`queue.task_done()`**: Essential. It tells `queue.join()` that one unit of work is fully complete. If you forget this, `await queue.join()` hangs forever.
4.  **Graceful Shutdown**:
    *   We wait for the producer to finish (`await producer_task`).
    *   We wait for the queue to empty (`await queue.join()`).
    *   Then we explicitly cancel workers (`w.cancel()`). The `TaskGroup` catches these cancellations and exits cleanly.

### How to Reuse This
*   **Replace `producer`**: With a file reader or an API pager.
*   **Replace `fetch`**: With Playwright logic (`async with async_playwright() ...`).
*   **Replace `worker`**: Add database saving logic after the fetch.

This template is "Module 3 Certified"—it leaks no memory, respects limits, and shuts down cleanly.


