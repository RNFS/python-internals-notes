# Python Internals & Runtime Architecture Notes

> "Python is easy to learn, but one of the hardest languages to truly master."

This repository contains my structured learning notes on Python's internal behavior, runtime architecture, and design patterns. These notes focus on what happens **under the hood**—moving beyond syntax to understand how CPython actually executes code.

The goal is to bridge the gap between "writing scripts that work" and "architecting systems that are reliable, performant, and bug-free."

## Curriculum Structure

These notes are organized into modules, each focusing on a specific layer of the Python runtime or application architecture.

### [Module 1: Runtime, Objects, & Memory](./module_1.md)
*Focus: The CPython Object Model*
- Names vs. Objects, Reference Counting, and Garbage Collection.
- Mutability, Aliasing, and "Spooky Action at a Distance."
- Identity vs. Equality (`is` vs `==`).
- Shallow vs. Deep Copy trade-offs.
- The mechanics of `__slots__` vs. `__dict__` and memory optimization.
- Class vs. Instance attributes and the MRO lookup chain.
- Handling Reference Cycles with `weakref`.

### [Module 2: Functions & Metaprogramming](./module_2.md)
*Focus: Scopes, State, and Control Flow*
- Closures, Cells, and captured state.
- Decorators: from simple wrappers to configurable factories.
- Generators, Coroutines, and lazy evaluation pipelines.
- Context Managers (`with` statement internals and error suppression).
- Descriptors and Properties: Customizing attribute access.
- ABCs and Protocols: Defining interfaces for robust architecture.

### [Module 3: Async & Concurrency](./module_3.md) 
*Focus: The Event Loop and Scalable Backends*
- The `async/await` runtime model vs. blocking code.
- Managing the Event Loop: Tasks, Futures, and Scheduling.
- Concurrency Patterns: Fan-out, Bounded Concurrency, and TaskGroups.
- Handling Timeouts and Cancellation correctly.
- Structured Concurrency principles in modern Python.

## Philosophy

These notes are written with a **"runtime-first"** mindset:
1.  **Mechanics over Magic:** We don't just memorize syntax; we ask "What is the interpreter doing with this object?"
2.  **Trade-offs over Rules:** Every feature (e.g., `__slots__`, `weakref`, `asyncio`) has a cost and a benefit. We explore when to use them—and when to avoid them.
3.  **Architectural Impact:** Small runtime details (like mutable defaults or circular references) create massive architectural bugs in distributed systems. We focus on these high-impact interactions.

## How to Use These Notes

- **Read sequentially:** The modules build on each other.
- **Run the code:** The snippets are designed to be run and inspected (use `id()`, `dir()`, and `sys.getrefcount()` freely).
- **Test your mental model:** Each module ends with "Active Recall Questions" to test if you can explain the *why* behind the behavior.

## License

MIT License. Feel free to use these notes for your own learning or teaching.
