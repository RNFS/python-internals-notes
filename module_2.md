# Module 2: Functions, classes, and metaprogramming


---

## Summary (in my own words)

- Closures: i can use closures to share state between functions.
- Decorators: i can use them to modify function calls, to loggin when the function is called, validate the return and validate the input for function calls, make rate limiters per function, etc..
- Generators: i can them to reduce memory usage for ex: using them to read big files in chuncks.
- Context managers: i can use them to handel erros aggresively, for ex closing db connection or an opend file.
- ABCs/Protocol: i can use them to create classes that share the same interface, for ex many classes all of them must have the scape interface to take a url and return scraped data, so enforcing spesfiec pattern.
- Properties/descriptors: i can use them to validate and take actions when a variable is called or set to a value.

---

## Key Concepts / Patterns

### Closures (cell-based captured state)
- Closures store captured variables in **cell objects**; the inner function keeps references to those cells via `func.__closure__`. [web:103][web:100]
- This is why closure state can outlive the outer function call and still be accessible later.

### Decorators (import-time rebinding)
- `@decorator(args)` expands at import time to `func = decorator(args)(func)`.
- Configurable decorators break if you use `@decorator` without parentheses when the decorator is written as a decorator factory (because the function can get bound into the wrong parameter).

### Rate limiting (per-function vs shared bucket)
- Per-function limiter: store `last_called` in a closure (each decorated function gets its own state).
- Shared limiter: store `last_call` on one shared instance; multiple wrapped functions share that state (bucket behavior).

### Generators (lazy iteration + streaming)
- Generators are useful to reduce memory usage by yielding items/batches instead of building a full list in memory. [web:246]
- Generators get exhausted because they resume from the last `yield` and eventually end (raising `StopIteration`), so they can’t be “replayed” without creating a new generator.

### Context managers (cleanup + exception protocol)
- `__exit__` suppresses exceptions only when it returns `True`. [web:216]
- Good practice: context managers should clean up resources (close DB/file) and usually **not suppress** exceptions unless the exception is expected and intentionally handled.

### ABCs / Protocols (interfaces for architecture boundaries)
- ABCs enforce a required interface for subclasses (your `Scraper` example).
- Architecturally: define “ports” (interfaces) in a boundary module and implementations in infrastructure to reduce coupling and avoid circular imports.

### Properties / descriptors (controlled attribute access)
- Properties are descriptors that allow validation and behavior on get/set. [web:157]
- Descriptor objects live on the class; the actual per-instance data usually lives in the instance `__dict__` under a private name.

---

## Examples / Scenarios (from your katas)

### Kata A: Decorators + closures + rate limiting
- `limiter(seconds=3)` uses `last_called` captured in a closure → per-function state.
- `RateLimiter(calls_per_second=2)` uses `self.last_call` and a `Lock` → shared bucket across multiple functions.

**Notes / fixes to remember**
- If using `calls_per_second`, compute `min_interval = 1 / calls_per_second` (interval vs rate confusion).

### Kata B: Generator batching for URL pipelines
- `iter_product_urls(...)` yields batches of URLs (chunking) rather than building all URLs.
- This maps well to “scraper yields → ingestion writes/batches”.

### Kata C: Context manager for DB connection
- `DBConnection.__enter__` connects; `__exit__` disconnects.
- `__exit__` should return `False` (or `None`) to propagate exceptions; raising inside `__exit__` is not the intended pattern. [web:216]

### Kata D: ABC boundary (Scraper)
- `Scraper` ABC with `.scrape(url) -> dict`.
- `MockScraper` and `RequestsScraper` as two implementations.

---

## Trade-offs & Gotchas

- Closure state is powerful, but it can also accidentally become hidden shared state if you capture mutables unintentionally.
- Decorators execute at import time (definition time), which can cause surprising behavior if they do heavy work.
- Rate limiting:
  - Per-function limiter vs shared bucket limiter is a design choice; shared state needs concurrency protection (locks) for threads.
- Generators:
  - Great for memory, but harder to debug and they are one-shot; you need to recreate them to iterate again.
- Context managers:
  - Suppressing exceptions with `__exit__ -> True` should be rare; it can hide real errors if overused. [web:216]
- ABC boundaries:
  - Avoid circular imports by keeping dependency direction one-way (ports in boundary module, implementations in infra).

---

## Questions / Follow-ups

- When to choose ABC vs Protocol in real projects (type-checking vs runtime enforcement)?
- Where exactly should ports live in my real project layout to minimize circular imports?
- When is it architecturally correct to put behavior in a decorator vs in the service layer?

---

## Active Recall Questions

1. What does `func.__closure__` contain and why does it matter for keeping captured state alive? [web:103]
2. What does `@decorator(args)` desugar to, and when is it evaluated?
3. How do you decide between per-function rate limiting and shared-bucket rate limiting?
4. Why do generators reduce memory usage compared to returning lists, and why are they exhausted after one pass? [web:246]
5. In a context manager, what does returning `True` from `__exit__` mean, and why is raising inside `__exit__` usually wrong? [web:216]


## code 

```python
import functools
from symtable import Class
import threading
import time

# Kata A Decorators + closures + rate limiting
def limiter(seconds: int):
    def decorator(func):
        last_called = None
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print("Called wrapper", func.__name__)
            nonlocal last_called
            now = time.monotonic()
            if last_called is not None:
                elapsed = now - last_called
                if elapsed < seconds:
                    wait_time = seconds - elapsed
                    print(f"Rate limit exceeded. Sleeping for {wait_time:.4f} seconds.")                    
                    time.sleep(wait_time)
                    now = time.monotonic()
            last_called = now
            return func(*args, **kwargs)
        
        return wrapper
    return decorator




class RateLimiter:
    def __init__(self, calls_per_second=2):
        self.calls_per_second = calls_per_second
        self.last_call = None
        self._lock = threading.Lock()

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            with self._lock:
                now = time.monotonic()
                if self.last_call is not None:
                    elapsed = now - self.last_call

                    if elapsed < 1 / self.calls_per_second:
                        wait_time = 1 / self.calls_per_second - elapsed                   
                    else:
                        wait_time = 0

                    if wait_time > 0:
                        print(f"Rate limit exceeded. Sleeping for {wait_time:.4f} seconds.")
                        time.sleep(wait_time)
                        now = time.monotonic()
                self.last_call = now
            return func(*args, **kwargs)
        return wrapper



@limiter(seconds=3)
def api_call():
    print("API call made at:", time.strftime("%X"))

@limiter(seconds=3)
def database_query():
    print("Database query executed at:", time.strftime("%X"))

limiter_obj = RateLimiter(calls_per_second=2)

@limiter_obj
def another_api_call():
    print("Another API call made at:", time.strftime("%X"))

@limiter_obj
def last_api_call():
    print("Last API call made at:", time.strftime("%X"))







# Kata B (Generators)
def iter_product_urls(main_url: str, total_pages: int) -> list[str]:
    urls = []
    for page in range(1, total_pages + 1):
        for url in extract_page_urls():
            urls.append(url)
        if len(urls) >= 50:
            yield urls
            urls = []
    if urls:
        yield urls


def extract_page_urls() -> list[str]:
    return ["http://example.com/page1", "http://example.com/page2"]





# Kata C (Context manager)
class DBConnection:
    def __init__(self, conn_string: str):
        self.conn_string = conn_string
        self.connected = False

    def connect(self):
        if not self.connected:
            print(f"Connecting to database with {self.conn_string}")
            self.connected = True

    def disconnect(self):
        if self.connected:
            print("Disconnecting from database")
            self.connected = False


    def __enter__(self):
        self.connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
        return false  # Do not suppress exceptions

def test_db_connection():
    try:
        with DBConnection("db://localhost:5432/testdb") as db:
            print("Using the test database connection.")
            raise RuntimeError("Simulated error during database operation.")
    except RuntimeError as e:
        print(f"Caught an exception: {e}")


# Kata D (ABC boundary

from abc import ABC, abstractmethod

class Scraper(ABC):
    @abstractmethod
    def scrape(self, url: str) -> dict:
        pass
class MockScraper(Scraper):
    def scrape(self, url: str) -> dict:
        print(f"Scraping URL: {url}")
        return {"url": url, "data": "sample data"}


class RequestsScraper(Scraper):
    def scrape(self, url: str) -> dict:
        import requests
        response = requests.get(url)
        return {"url": url, "status_code": response.status_code, "content": response.text[:100]}

    
```