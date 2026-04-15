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

### yield — generator function
`event_stream()` is an `async generator`. It produces values lazily — each yield sends one SSE event and suspends until the consumer (the HTTP response) asks for the next. The whole video never sits in memory at once.

### response_schema=ImageAnalysis — structured output
Without a schema, Gemini returns freeform text. 

With response_schema=ImageAnalysis (a Pydantic model), Gemini is constrained to output valid JSON matching that schema — guaranteed fields status, event, tray_id. This is called constrained decoding — the model's token sampling is steered to only produce valid structure.

```python
# Pydantic model — defines the exact JSON shape Gemini must return
class ImageAnalysis(BaseModel):
    status: str
    event: str
    tray_id: int

try:
    # to_thread — runs the blocking Gemini HTTP call in a thread pool
    # so "await" suspends THIS coroutine but keeps the event loop free for others
    response = await asyncio.to_thread(
        client.models.generate_content,
        model=MODEL,
        contents=[
            types.Part.from_bytes(data=img_bytes, mime_type="image/jpeg"),  # We're sending two things to Gemini at once: the raw JPEG bytes of the frame (the image), and the text prompt
            prompt,
        ],
        config=types.GenerateContentConfig(
            max_output_tokens=100,                   # not more than 100 tokens
            response_mime_type="application/json",   # tell Gemini to return JSON
            response_schema=ImageAnalysis,           # enforce the exact fields above
        )
    )
    text = response.text.strip()
```

- `max_output_tokens=100` — caps the response length; our `ImageAnalysis JSON` is tiny so 100 is plenty
- `response_mime_type="application/json"` — instructs Gemini to return **raw JSON**, not prose
- `response_schema=ImageAnalysis` — passes **our Pydantic model as a schema**, so Gemini is **forced to return exactly** `{"status": "...", "event": "...", "tray_id": ...}` — **nothing else**, no extra text

- `text = response.text.strip()` - Because we enforced the schema, response.text is guaranteed to be a valid JSON string like {"status": "ok", "event": "item_placed", "tray_id": 3}. The .strip() just removes any stray whitespace.













client.models.generate_content(...) is a plain blocking function — it makes an HTTP request to Gemini and just sits there waiting for the response, freezing whatever thread runs it. If you called it directly inside your async coroutine, it would freeze the entire event loop — no other frames could be processed, no other requests could be served, nothing.
What asyncio.to_thread does
It takes that blocking function and hands it off to a worker thread from Python's default thread pool. Your coroutine then does await — which means it suspends itself and hands control back to the event loop. The event loop is now free to run other coroutines (process other frames, serve other HTTP requests) while the Gemini call is sitting in that worker thread waiting for a network response.
When Gemini eventually replies, the thread finishes, and the event loop wakes your coroutine back up right at the response = line with the result.
event loop (main thread)          worker thread
─────────────────────────         ──────────────────────────
coroutine hits to_thread()   →    starts generate_content()
await suspends coroutine          ... waiting for Gemini HTTP ...
loop runs other coroutines        ... waiting ...
                             ←    Gemini responds
loop resumes this coroutine
response = <the result>
The arguments being passed
asyncio.to_thread takes the function first, then all the arguments to pass it. So this:
pythonawait asyncio.to_thread(
    client.models.generate_content,   # the function
    model=MODEL,                       # ─┐
    contents=[...],                    #  ├─ its keyword arguments
    config=...                         # ─┘
)
is equivalent to calling client.models.generate_content(model=MODEL, contents=[...], config=...) — just in a thread instead of on the event loop.

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


- yield / async generator
- GIL (Global Interpreter Lock)
- Future / Task
- Cooperative vs preemptive scheduling
- asyncio.run() / uvicorn


With `ThreadPoolExecutor`: All your threads — the main thread, worker 1, worker 2, worker 3, worker 4 — live inside one single process. 
That process runs on Core 0. The OS can physically shuffle it to Core 1 or Core 2, but it doesn't matter, because the GIL means only one thread runs Python at any moment anyway. 
Cores 1, 2, 3 are completely unused for your Python code. The threads are not "on different cores" — they are all taking turns on whatever single core the OS gives the process.

`ThreadPoolExecutor`: All threads live in one process, sharing the GIL. Threads that block on I/O release the GIL — so I/O-bound tasks get real concurrency. CPU-bound tasks fight for the GIL and run one-at-a-time. No true parallelism for Python code.

With `ProcessPoolExecutor`: Now each worker is a completely separate process. Core 0 runs your main process (event loop). Core 1 runs worker process 1 with its own GIL. Core 2 runs worker process 2 with its own GIL. Core 3 runs worker process 3 with its own GIL. They never share a GIL, they never compete, and they genuinely execute Python at the same clock tick. That's true parallelism.


# Force the output


















