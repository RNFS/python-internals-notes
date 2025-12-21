# Module 1 – Runtime, Objects, and Memory

**Date:** 2025-12-02  
**Time spent:** ~30 min

***

## Summary

- All Python “variables” are just pointers/references to objects in memory, not boxes that *contain* values. Each object is a CPython `PyObject` holding metadata like its type, refcount, and size.  
- CPython uses reference counting (plus a cyclic GC) to decide when to free objects; when an object’s refcount drops to 0, its memory can be reclaimed.  
- The big mental shifts for this module: understanding shallow vs deep copy, mutability vs immutability, and how shared mutable state (globals, defaults, class attributes) can create spooky bugs in real systems.  
- `__slots__` and `__dict__` control how attributes are stored; they directly affect memory usage, lookup cost, and how flexible your objects are.  
- Class attributes vs instance attributes and MRO explain *where* attributes actually come from when you do `obj.x`, which is crucial for avoiding hidden shared state.

***

## Key Concepts / Patterns

- **Variables and `PyObject`s**
  - Names are references to objects in memory (addresses), not the objects themselves.
  - Each object carries:
    - A pointer to its type (`int`, `str`, etc.).
    - A reference count (how many references point to it).
    - Size and other internal metadata.
  - When refcount hits 0, CPython can free that memory; cyclic GC handles cycles that refcount cannot see.

- **Mutability vs Immutability**
  - Mutable: `list`, `dict`, `set`, etc. – operations like append, assign, delete *modify the same object in place*.
  - Immutable: `int`, `float`, `str`, `tuple`, etc. – operations like `x += 1` create a *new* object and rebind the name.
  - Example mental model:  
    - `x: int = 1` then `x += 1` → read old int `1`, compute `2`, allocate a new int object at a different address, and rebind `x` to it.
  - Shared mutable state is dangerous: if multiple parts of the system share a reference to the same `dict`/`list`, any mutation is visible everywhere.

- **Identity vs Equality**
  - `id(obj)` and `is` talk about identity (same object in memory).
  - `==` talks about value equality.
  - `x is y` is equivalent to `id(x) == id(y)` in practice.

- **Shallow vs Deep Copy**
  - `copy.copy(obj)` copies the *outer* container but reuses references to inner objects.
    - Copying a `dict` via `copy.copy` gives a new dict, but the values (lists/other dicts) are the same underlying objects.
  - `copy.deepcopy(obj)` recursively copies all nested objects, building a new object graph.
  - Shallow copy + nested mutables ⇒ changes to inner lists/dicts in the copy can affect the original.

- **Function Defaults and Shared Mutable State**
  - When Python executes `def func(arg=[])`, it creates that list **once** at function definition time and stores it in the function’s defaults.
  - Every call that doesn’t pass `arg` uses the *same* list object.
  - Pattern to avoid bugs:
    ```python
    def func(arg=None):
        if arg is None:
            arg = []
        # use arg safely here
    ```
  - This avoids hidden shared lists/dicts between calls.

- **`__dict__` and `__slots__`**
  - By default, each instance has a `__dict__` that stores attributes; it’s flexible but adds per-instance overhead and dict lookup cost.
  - `__slots__ = ("name", "retries")` removes the per-instance `__dict__` and stores attributes in a fixed, array-like slot layout.
  - Benefits:
    - Lower memory per instance.
    - Potentially better cache locality for lots of small, uniform objects.
  - Trade-off:
    - You can only have attributes that appear in `__slots__` (unless you explicitly add `"__dict__"`).
    - You lose the ability to add arbitrary attributes on the fly.

- **Instance Attributes vs Class Attributes**
  - Class attributes live on the class (`MyClass.__dict__`); they are shared by all instances unless shadowed.
  - Instance attributes live in the instance’s namespace:
    - `obj.__dict__` for normal classes.
    - Slot storage for slotted classes.
  - Lookup order for `obj.attr`:
    1. Instance namespace (`__dict__` or slots).
    2. Class and bases (MRO).
  - Augmented assignment like `obj.x += 1`:
    - Reads `obj.x` via lookup.
    - Computes a new value.
    - Writes back `obj.x = <new>`:
      - Creates/shadows an instance attribute *only if* that attribute is allowed as an instance attribute (normal class or slotted name present in `__slots__`).

- **MRO (Method Resolution Order)**
  - Visible via `SomeClass.__mro__` or `obj.__class__.__mro__`.
  - Python searches for attributes starting from the instance’s class and moving through the MRO until it finds the first match.
  - The first class in the MRO with that attribute “wins.”

- **Cycles and `weakref`**
  - Reference cycles (e.g., `a.child = b` and `b.parent = a`) can delay memory reclamation until cyclic GC runs.
  - `weakref` can be used where you want relationships without owning references that keep objects alive (e.g., caches, graphs, registries).

***

## Examples / Scenarios

- **Shared Mutable Default Trap**
  ```python
  def add_item(item, items=[]):
      items.append(item)
      return items
  ```
  - `items` is created once and reused for every call that doesn’t pass a list.
  - All callers accidentally share the same list, which is usually not what you want.

- **Shallow Copy of Nested Dict**
  ```python
  import copy

  base = {"headers": {"User-Agent": "UA1"}, "meta": {"retries": 0}}
  cfg1 = copy.copy(base)
  cfg2 = copy.copy(base)

  cfg1["meta"]["retries"] += 1
  cfg2["headers"]["User-Agent"] = "UA2"
  ```
  - `cfg1` and `cfg2` are different dicts, but they share the same inner `meta` and `headers` dicts unless deep-copied.

- **`JobConfig` with `__slots__`**
  ```python
  class JobConfig:
      count = 0
      __slots__ = ("job_id", "job_name")

      def __init__(self, job_id, job_name):
          self.job_id = job_id
          self.job_name = job_name
  ```
  - `count` is a shared class attribute (int, safe and immutable).
  - `job_id` and `job_name` are per-instance attributes stored in slots (compact, no `__dict__`).

- **Class Attribute Turning into Instance Attribute (non-slotted)**
  ```python
  class ScraperJob:
      retries = 0

  a = ScraperJob()
  b = ScraperJob()

  a.retries += 1  # turns into a.retries = a.retries + 1
  b.retries += 1
  ```
  - Both `a` and `b` end up with their own `retries` in `.__dict__`, while `ScraperJob.retries` stays `0`.

***

## Trade-offs & Gotchas

- **Shared Mutable State**
  - Using mutable defaults, mutable class attributes, or module-level mutable globals can create invisible coupling between calls, requests, or workers.
  - Best practice: avoid mutables in defaults and global state; keep shared state explicit and controlled.

- **Shallow vs Deep Copy**
  - Shallow copy is faster and cheaper but “leaks” mutations through shared inner objects.
  - Deep copy is safer for independent graphs but can be expensive for large or complex structures.

- **`__slots__` vs `__dict__`**
  - `__slots__`:
    - Pros: less memory, potentially faster attribute access, explicit layout.
    - Cons: less dynamic; cannot add arbitrary attributes, and must maintain the slot list.
  - `__dict__`:
    - Pros: highly flexible; easy to add or remove attributes.
    - Cons: more per-instance overhead and dictionary lookup work.

- **Class vs Instance Attributes**
  - Class attributes that are mutable (e.g., `jobs = []`) become *shared containers* across all instances, which is often a bug.
  - Immutable class attributes (constants, config defaults) are generally safe and useful.
  - Misunderstanding augmented assignment (`obj.x += 1`) can lead you to think you changed a shared class attribute when you actually created a per-instance one (or got an `AttributeError` with `__slots__`).

- **Cyclic References**
  - Cycles don’t immediately leak memory, but they can:
    - Delay cleanup until GC runs.
    - Interact badly with objects that define `__del__`.
  - `weakref` helps when you want references that don’t keep objects alive.

***

## Questions / Follow-ups

- Need a concrete example with cyclic object references and how `weakref` breaks the cycle in practice.
- Want a clearer, step-by-step picture of how `weakref` works and when to use it in real systems (e.g., caches, object graphs).
- Still want a deeper, mechanical picture of how `__slots__` is implemented under the hood vs `__dict__` (layout, lookup).
- Need to explore:
  - Privacy and module API:
    - How name mangling (`__attr`) actually works.
    - Leading underscore conventions in modules and classes.
    - How to use `__all__` to define a module’s public API and control `from module import *`.
- How to connect module-level singleton behavior and shared state (e.g., settings, logging config) with these memory/model concepts in large apps.

***

## Active Recall Questions

1. Explain in your own words the difference between a *variable*, an *object*, and a *reference* in CPython. How does this relate to `id()` and `is`?
2. Given a nested dict with inner lists and dicts, what does `copy.copy` do vs `copy.deepcopy`? Describe a concrete bug that could appear in a scraper or backend if you used the wrong one.
3. Why is `def func(arg=[])` dangerous? Walk through what happens in memory from function definition time through multiple calls.
4. In a class with `__slots__ = ("name", "retries")`, what happens when you try `obj.some_new_attr = 1`? How is this different from a normal class with a `__dict__`?
5. What is the difference between a class attribute and an instance attribute in terms of *where they are stored* and *how Python finds them* when you access `obj.attr`? How can this create hidden shared state if you are not careful?


## Advanced: GC, Weakrefs, and Module Privacy

### Reference Cycles & Garbage Collection
- **The Trap:** Two objects referencing each other (e.g., `parent.child` and `child.parent`) create a **reference cycle**.
- **Refcount behavior:** Even if you `del parent` and `del child` in your main code, their refcounts never drop to 0 because they point to each other.
- **GC's role:** CPython's cyclic GC will eventually find and free them, but:
  - It takes time (latency spikes).
  - If `__del__` is defined, it can be tricky/unpredictable.
- **The Fix (Weakref):** Use `weakref.ref` (or `weakref.proxy`) for the "back-pointer" (child -> parent).
  - A weak reference **does not increment the refcount**.
  - If the only thing pointing to the parent is the child's weak ref, the parent is destroyed.
  - When parent dies, it releases the child, breaking the cycle.

### Module Privacy & Singletons
- **Module Singleton:** A module is loaded once per process. `import settings` in 5 different files returns the **same module object**.
  - Global variables in a module are effectively **process-global singletons**.
  - Mutable module globals (e.g., `list` or `dict`) are shared state traps.

- **Public vs Private API:**
  - `_variable` (leading underscore): Convention for "internal/private".
  - `__all__ = ["API_KEY"]`: Controls what is imported by `from module import *`.
  - **Note:** Python does not enforce privacy. `from settings import _BASE_URL` still works, but it violates the intended contract.




## Cycles, `__del__`, and `weakref`

- In the naive `Node` graph, you have:  
  - `root` (a name) → `Node("root")` → `.children[0]` → `child` object.  
  - `child` object → `.parent` → `root` object.  
  That is a strong reference cycle: each node keeps the other’s refcount > 0. Your description of the cycle itself is correct.  

- CPython’s reference counting alone cannot reclaim such a cycle, but the **cyclic GC** periodically scans for groups of objects that only reference each other and have no external referrers, and then frees them.  This does add GC work and pauses, which matters in long‑running, latency‑sensitive processes.[3]

- `__del__` complicates this: if objects in a cycle define finalizers, the GC may not be able to safely determine destruction order. Historically such cycles could be left uncollected or moved to `gc.garbage`; the practical rule is: **avoid cycles involving `__del__` or break them explicitly**.[3]

- In `SafeNode`, `_parent_ref` holds a **weakref object**, not the parent itself. A weakref:
  - Does *not* increase the target’s refcount.  
  - Returns the target when you call it (`_parent_ref()`), or `None` once the target is collected.[2]
  So when the only strong references to the parent disappear, its refcount can drop to 0 and it is freed as usual; the weakref just “goes dead” and stops returning a live object. There is no strong edge from child to parent anymore, so the graph is no longer a strong cycle.

## When to use `weakref` in real systems

- Use a weak reference when:
  - You want a *back‑pointer* or registry/observer list, but you don’t want it to *own* the objects (e.g., child → parent, subscribers in an event bus, cache entries keyed by objects).  
  - You are fine with “this reference might suddenly become `None` because the real object was freed,” and you’ll handle that case.

- Avoid `weakref` when:
  - The relationship is truly ownership (e.g., a repository owning its connection pool).  
  - You need guaranteed lifetimes; weak references are about “I’d like to see you if you happen to be alive, but I won’t keep you alive.”

## Module API, `__all__`, and singletons

- Import semantics: each module (`settings`, `logging_config`) is imported **once per process**; subsequent imports reuse the same module object from `sys.modules`. So there is exactly one `settings` and one `logging_config` module object per interpreter process.[1][3]

- `from settings import *`:
  - If `__all__ = ["API_KEY"]` exists, only `API_KEY` is imported into `worker`’s globals; `_BASE_URL` stays in the `settings` module but is not pulled into `worker`.  
  - If `__all__` is *missing*, the default rule is “import all names not starting with `_`,” so again `API_KEY` is imported and `_BASE_URL` is skipped because of the leading underscore.[3]

- The leading underscore on `_BASE_URL` is a **convention** meaning “internal to this module; not part of the public API.” Combined with:
  - `__all__` listing only public names, and  
  - Call sites using explicit imports like `from settings import API_KEY`,  
  you get a clean separation between *public settings* and *internal implementation details*.

## Global logger singleton: pros and cons

- `logger = logging.getLogger("shopify")` in `logging_config.py` plus `configure_logging()` makes that `logger` effectively a **process‑wide singleton**: all modules that import it share the same underlying logger object and configuration.[1]
- Pros:
  - One place to configure handlers, formatters, log level, JSON output, etc.  
  - All parts of the system can log in a consistent, centralized way.

- Cons:
  - Hidden coupling: changing logging config in one module affects everything.  
  - If a library re‑configures logging, it can unexpectedly break or duplicate logs.  
  - In tests, you often need to reset or isolate logging config to avoid cross‑test bleed.

The pattern you want in real backends/scrapers is usually: **one module that configures logging once at process startup**, exposes loggers via `logging.getLogger(__name__)` or a few shared loggers, and never reconfigures them deep inside libraries or workers.


## Refrences
[Garbage Collector Interface](https://docs.python.org/3/library/gc.html#module-gc)