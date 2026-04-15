# Video Understanding with Gemini

# The Problem Statement

### 1. The I/O Blocking Issue - Blocking vs. Non-blocking I/O.
- A "blocking" call stops all execution until it gets an answer.
- If we didn't use `asyncio.to_thread`, our entire FastAPI server would freeze. 
- **No other users could load the page**, and the SSE connection would likely time out because the Event Loop couldn't send "heartbeat" packets.

- When we use `asyncio.to_thread`, the OS has to **"save the state" of the Event Loop** and **"load the state" of the Worker Thread**. If you **spawn too many threads**, the CPU spends **more time switching between them than** actually **processing images**.

### 2. The Sequential Latency Stack - Serial Execution
- Tasks are executed one after another in a single line.
- Our `while` `True` loop reads a frame, waits for Gemini, then moves to the next.
- Even with `to_thread`, **we are only processing one frame at a time**.
- Gemini takes ~1.5s to respond. If we have **100 frames** and process them as **Subroutines**, our total time is $100 \times 1.5s = 150s$

### 3. The Decoding Overhead - CPU-Bound Redundancy
- `cap.read()` decodes every single frame. If your interval is `2.0s` and your video is `30fps`, you are decoding `59 frames` just to reach the `60th`.
- This **wastes CPU cycles** and **slows down the loop**, especially on **high-resolution** or **long videos**.
- Reading an entire book just to find the page number of the last chapter.

### 4. The Memory Pressure - Backpressure
- The ability of a system to signal the data producer to slow down because the **consumer is overwhelmed**.
- Currently, we don't have this because we process serially. But if we try to **go parallel without** a `Semaphore` or `Queue`, **we would spawn 100 threads at once**.
- Each thread would hold a JPEG in RAM. **100 high-res JPEGs** can easily **crash a small server** or **exceed the Gemini API rate limits instantly**.

### 5. The Stateless Connection - Persistence & Idempotency
- Ensuring that a dropped connection doesn't lose all progress.
- If the user refreshes the browser or the Wi-Fi blips, the `event_stream()` coroutine dies. When they reconnect, the video starts at Frame 0 again.
- We waste money and time re-analyzing frames you've already seen. It would be like a DVD player that restarts the movie from the beginning every time you pause it.

### 6. The Compute/Network Split (The "GIL Wall") - Parallelism vs. Concurrency
- Using **multiple cores** for math (**Parallelism**) vs. **managing multiple waits** (**Concurrency**).
- `frame_to_jpeg_bytes` is pure math (`OpenCV`/`Pillow`). This happens on the Event Loop Thread.
- While the CPU is busy resizing a frame, it cannot handle the incoming SSE heartbeats or other web requests. For a split second, the app is "deaf."

# The Solution

# Our Code

### Video metadata: `FPS`, `resolution`, `frame count`
- `cv2.VideoCapture` opens the file and reads a header — no pixels yet. From it you get:
- `FPS` — how many frames represent one second of real time. 30fps = 30 frames per second. Used to convert frame_index → timestamp via timestamp = frame_idx / fps.
- `Frame count` — total number of frames in the file. duration = total_frames / fps.
- `Width` × `height` — pixel dimensions of each frame.
  
- `frame_step` = `int(fps * interval_sec)` — if FPS=30 and interval=2s, **analyze every 60th frame**.

### `frame_to_jpeg_bytes` — inputs vs outputs
- **Input**: a raw **OpenCV frame** — **a 3D NumPy array** shaped [`height`, `width`, `3`]. Each cell is a `uint8` (0–255).
- The 3 channels are BGR (Blue, Green, Red) — OpenCV's unusual default, **not RGB**.

- **Output**: a **bytes object** — a flat, compressed binary blob.
- JPEG encoding discards some pixel data (lossy) to shrink the size.

The pipeline inside:
```
BGR frame (numpy)
  → resize if width > MAX_WIDTH      # fewer pixels = smaller payload
  → cvtColor BGR→RGB                 # fix channel order for PIL
  → PIL Image.fromarray()            # PIL works in RGB
  → pil_img.save(buf, format=JPEG)   # compress into BytesIO buffer
  → buf.getvalue()                   # extract raw bytes
```

- `BytesIO` is an in-memory file — it behaves like a file on disk but lives in RAM.
- `getvalue()` extracts the **bytes without writing anything to disk**.
- What a **bytes object** actually is: just **a sequence of integers 0–255**.
- A `1280×720` JPEG might be `~80 KB` = `80,000 bytes`.
- The **Gemini API** receives this **binary blob** + the `mime_type="image/jpeg"` hint so it knows how to decode it.

### SSE — Server-Sent Events
- StreamingResponse with media_type="text/event-stream" keeps an HTTP connection open.
- The server yields events one at a time; the browser receives them incrementally. Format is data: {json}\n\n. The client uses EventSource JS API to listen.
- This is why you see results frame-by-frame in the browser instead of waiting for the full video to process.


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
