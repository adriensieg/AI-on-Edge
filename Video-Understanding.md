# Video Understanding with Gemini

# The Problem Statement

### 1. The I/O Blocking Issue

If we didn't `use asyncio.to_thread`, our entire FastAPI server would freeze. 
No other users could load the page, and the SSE connection would likely time out because the Event Loop couldn't send "heartbeat" packets.

A "blocking" call stops all execution until it gets an answer.

# The Solution

## Which should you use for Gemini?

| Goal                     | Use This                                                                 | Why?                                                                 |
|--------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------------|
| Waiting for API          | `asyncio` / Threads / `ThreadPoolExecutor`                             | Gemini does the "thinking." Your CPU is just waiting.                |
| Resizing/Encoding Video  | `multiprocessing` / `ProcessPoolExecutor`                              | This is heavy math. Using multiple CPUs will make it much faster.    |
| Real-time Streaming      | `asyncio` + SSE                                                        | Keeps the connection open without blocking the rest of the app.      |


# Dictionary of concepts
- The "Single-Threaded" Constraint
- The Execution Units (The "Who")
    - Subroutine
    - Coroutine
    - Thread
- The Management System (The "How")
    - Event Loop
    - Task
    - Future
- The Coordination Tools
    - Semaphore
    - IPC (Inter-Process Communication) = Queues or Pipes
- Concurrency (One CPU, Many Tasks) - `asyncio` (Coroutines), threading
- Parallelism (Multiple CPUs, Multiple Tasks) - `multiprocessing`

With `ThreadPoolExecutor`: All your threads — the main thread, worker 1, worker 2, worker 3, worker 4 — live inside one single process. 
That process runs on Core 0. The OS can physically shuffle it to Core 1 or Core 2, but it doesn't matter, because the GIL means only one thread runs Python at any moment anyway. 
Cores 1, 2, 3 are completely unused for your Python code. The threads are not "on different cores" — they are all taking turns on whatever single core the OS gives the process.

`ThreadPoolExecutor`: All threads live in one process, sharing the GIL. Threads that block on I/O release the GIL — so I/O-bound tasks get real concurrency. CPU-bound tasks fight for the GIL and run one-at-a-time. No true parallelism for Python code.

With `ProcessPoolExecutor`: Now each worker is a completely separate process. Core 0 runs your main process (event loop). Core 1 runs worker process 1 with its own GIL. Core 2 runs worker process 2 with its own GIL. Core 3 runs worker process 3 with its own GIL. They never share a GIL, they never compete, and they genuinely execute Python at the same clock tick. That's true parallelism.
