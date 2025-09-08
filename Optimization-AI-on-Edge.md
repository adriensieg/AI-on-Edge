# Optimization

```
Application code
   ↑
Runtime (e.g., JVM, Python VM, libc)
   ↑
Operating System (user-space + kernel)
   ↑
Kernel (process/memory/device management)
   ↑
CPU (with cores, executing ISA instructions)
   ↑
Hardware (memory, devices, etc.)
```

If hyper-threading is supported by the CPU, then there are 2 threads per core, otherwise, there is only 1 thread. 

- **Hardware** (cores, cache, DRAM, GPU/NPU, thermal)
- **Kernel** (scheduler, IRQ affinity, hugepages, swap, I/O schedulers)
- **Runtime** (Python/C runtime, allocators, threading libs, quantization)
- **Application** (batch size, pipelining, zero-copy IO)
- **Observability** (latency, faults, thermals)

## Hardwares
- **CPU cores**
   - Use `taskset`/`sched_setaffinity` to pin inference to specific cores.
   - Isolate “clean” cores by moving OS background daemons to other cores.
- **CPU frequency scaling**
   - Disable frequency scaling (use performance governor).
   - Avoid thermal throttling → add `heatsink`/`fan` if necessary.
- **Threads vs Hyperthreads**
   - Disable `SMT` (if hardware supports) if it adds contention.
   - Stick to physical cores for predictability. 
- **GPU / NPU / TPU**
   - ~~If your Pi has an attached Coral USB/PCIe TPU, Movidius NCS2, or built-in GPU, offload YOLO inference.~~
   - Quantized models (`int8`, `fp16`)
- **Memory (DRAM)**
   - Ensure sufficient RAM headroom to avoid swap.
   - Reduce memory latency: prefer single-bank allocations, lock important pagesinto RAM (`mlock`).
- **Cache (L1/L2/L3)**
   - Structure data to maximize cache locality (tensor layout matters). Tensor layout tuning (e.g., NHWC vs NCHW).
      - `N`: Number of data samples.
      - `C`: Image channels. A red-green-blue (RGB) image will have 3 channels.
      - `H`: Image height.
      - `W`: Image width.
   - Warm-cache core pinning.
   - Micro-level wins, not first-order.
   - Pin inference threads so they reuse warm caches.
- **Storage / I/O**
   - Preload weights into RAM; avoid runtime disk access.
   - Use `tmpfs/ramfs` for transient data.
 
 - **Data Ingestion**
   - Storage layout: Keep datasets/recordings on NVMe; avoid network filesystems for live inference. Increase file readahead for sequential reads: `sudo blockdev --setra 4096 /dev/nvme0n1`.
   - **Use hardware decode**: NVDEC via `GStreamer`/`FFmpeg` (e.g., `h264_cuvid`, `nvh264dec`) or `NVIDIA DeepStream`.
      - Avoid CPU decoding if you have an NVIDIA GPU.
   - **Zero/low-copy pipelines**:
      - Prefer `DMABUF` / `GPU surface handoff` (`GStreamer`) to keep frames on GPU.
      - In PyTorch, use pinned memory (`pin_memory=True`) and `non_blocking=True` on `.to(device)`.
   - **Prefetch & parallelize**: DataLoader with enough workers (`num_workers` ~= cores/4…cores/2), `prefetch_factor` tuned; or use NVIDIA DALI to move aug/decode to GPU.
     
- **Network interface (if streaming camera data remotely)**
   - Minimize packet copying (use kernel zero-copy: `sendfile`, `mmap`).
   - Reduce IRQ load by tuning NIC offloads and coalescing.
   - I/O-aware resizing: Pre-resize frames close to network size (e.g., `640×640`) with GPU ops (NPP/CV-CUDA) or hardware decoders to avoid CPU bottlenecks.
   - Reduce packet processing overhead: enable `GRO`/`LRO`, and tune `RPS/RFS`.
   - Busy-poll for low latency (carefully): `net.core.busy_poll`, `net.core.busy_read`.
   - Keep the NIC’s IRQs pinned off your hot app cores.

## Kernel & OS-Level
- **Scheduler**
   - Use `SCHED_FIFO` or `SCHED_RR` for inference threads.
      - `SCHED_FIFO` for inference thread.
      - `PREEMPT_RT` kernel if you need hard real-time deadlines.- 
   - Configure `cgroups` to isolate background services.
- **Kernel tick & preemption**
   - Consider low-latency kernel builds or `PREEMPT_RT` patches for predictability. 
- **Hugepages / Transparent Hugepages**
   - Lock model weights into hugepages to reduce TLB misses.
- **NUMA (if multi-socket board, not Pi)**
   - Pin memory allocations local to cores.
- **IRQ affinity**
   - Route camera/network interrupts away from inference cores.
- **I/O schedulers**
   - Use `noop`/`mq-deadline` instead of CFQ for predictable latency.
- **Networking stack**
   - Disable TCP features if not needed (e.g., window scaling).
   - For camera feeds, use V4L2 mmap with zero-copy.
- **Swap**
   - Disable swap or set `vm.swappiness=0`.
   - Use zram only as a last resort.
- **Transparent memory features**
   - Disable KSM (Kernel Samepage Merging) to avoid surprises.

## Runtime / Libraries
- **Python runtime**
   - Use PyPy or Cythonized extensions for hot paths.
   - Disable GIL overhead by pushing compute-heavy loops into C/C++ (via PyTorch/TFLite C API).
   - Avoid Python object churn (stick to NumPy arrays, memoryviews).
- **Allocator**
   - You already have jemalloc/tcmalloc.
   - Preallocate pools for tensors.
- **BLAS / OMP / threading libraries**
   - Tune `OMP_NUM_THREADS`, `MKL_NUM_THREADS`.
   - Bind `OpenMP` threads to cores explicitly.
- **Quantization / Compilation**
   - Use TensorRT, TFLite Edge TPU, or TVM.
   - Fuse layers at compile time (conv+BN+ReLU).
- **Garbage collection**
   - Already disabled in prod, good.

## Application & Data Flow
- **Frame handling**
   - Zero-copy capture (e.g., V4L2 mmap).
   - Resize/preprocess on GPU/NPU if available.
- **Batch size**
   - Stick to `batch=1` for real-time streaming.
- **Pipelining**
   - Run capture → preprocess → inference → postprocess in parallel stages.
   - Use lock-free queues or shared memory ring buffers.
- **Fallback**
   - Dynamic degrade path:
      - drop frame rate under load
      - switch to tiny model
      - reduce resolution dynamically
    
Use the latest YOLO family build you’ve standardized on (e.g., Ultralytics YOLOv8/YOLOv10). Export once to TensorRT:
yolo export model=yolov8n.pt format=engine half=True imgsz=640.
For INT8, run calibration on representative frames.

- NMS on GPU (built-in for TRT engines) to avoid CPU bottlenecks. If staying in PyTorch, use a GPU NMS kernel.
- Tile / stride: For high-res cameras, consider tiling with overlap to keep per-tile size near model size (if you need small objects at distance), and fuse results.
    
## Observability & Tuning
- Latency per frame (end-to-end, not just inference).
- Cache misses (via perf stat).
- IRQ distribution (via /proc/interrupts).
- Thermal throttling (via vcgencmd measure_temp).
- Page faults (major/minor).

## Image Processing & Pre-processing Optimization

**The goal is**: 
- minimize CPU bottlenecks,
- avoid copies,
- and match the model’s expected input efficiently.

#### 1. Image Sizing & Resizing
- Resize to network input size (e.g., `640×640`) as early as possible.
- Use GPU-accelerated resizing:
- NVIDIA CV-CUDA or NPP libraries.
- OpenCV with CUDA (cv2.cuda.resize).
- GStreamer elements (nvvidconv) in DeepStream pipelines.
- Avoid resizing twice (camera driver → CPU → GPU). Keep one resize in GPU space.

#### 2. Color Space Conversion
- Most YOLO models expect RGB:
- Convert from BGR (OpenCV default) on GPU if possible.
- In DeepStream: set video-convert or nvvidconv to output NV12 → RGBA → RGB.

#### 3. Normalization & Scaling
- Divide pixel values by 255.0 and optionally subtract mean/scale if the model requires.
- Do this on GPU tensors to avoid CPU-bound loops.

#### 4. Batch Preprocessing
- Preprocess in batches when possible — less kernel-launch overhead.
- Use frameworks like NVIDIA DALI or CV-CUDA for end-to-end GPU pipelines.

#### 5. Memory & Copy Efficiency
- Pinned memory for CPU→GPU transfers.
- Use .to(device, non_blocking=True) in PyTorch.
- For streaming: leverage zero-copy DMA (dmabuf) with GStreamer + DeepStream.

## YOLO Model Optimization (Inference Focus)

#### 1. Model Export & Runtime
- TensorRT Engine (preferred for NVIDIA GPUs):
- Export to TRT with FP16 or INT8.
- Example: yolo export model=yolov8n.pt format=engine imgsz=640 half=True
- ONNX Runtime (CUDA/TensorRT Execution Provider) if TRT is not viable.
- Torch-TensorRT for PyTorch integration.
- For CPU inference only: export to OpenVINO (Intel) or ONNX Runtime (CPU EP).

#### 2. Precision Optimization
- FP16 (half-precision) → ~2× speedup on supported GPUs.
- INT8 quantization (with calibration dataset) → ~3–4× speedup with minimal accuracy drop.

#### 3. Input Resolution
- Use the smallest image size that still meets detection accuracy:
  - 320×320 or 416×416 for fast, low-latency inference.
  - 640×640 default for balanced accuracy/speed.
  - Larger sizes (1280+) only if needed for small/distant objects.

#### 4. Model Variants
Choose the right YOLO variant:
- n (nano) or s (small) for edge devices.
- m, l, x for higher accuracy on GPUs.
Consider YOLOv8/YOLOv10 (better accuracy/speed trade-offs) or YOLO-NAS (Neural Architecture Search optimized).

#### 5. Batch Size
- For real-time single-stream: batch=1 (low latency).
- For offline processing / multi-stream: increase batch size until GPU ~95–99% utilized.

#### 6. NMS (Non-Max Suppression)
- Run NMS on GPU (available in TensorRT, ONNX, Ultralytics YOLO).
- Avoid Python loops for NMS → huge bottleneck.

#### 7. Graph & Kernel Optimizations
- CUDA Graphs (PyTorch 2.0+): remove Python overhead.
- Enable cudnn.benchmark = True for variable input sizes.
- Fuse layers where possible (TensorRT automatically does this).

#### 8. Post-processing
- Vectorized tensor ops instead of per-box loops.
- Keep outputs on GPU until the very last step.

| Step            | Baseline                     | Optimized Pipeline                 |
| --------------- | ---------------------------- | ---------------------------------- |
| Decode          | CPU (OpenCV)                 | GPU NVDEC (zero-copy)              |
| Resize/Preproc  | CPU (OpenCV)                 | GPU (CV-CUDA, NPP, DALI)           |
| Memory Transfer | Blocking copies              | Pinned mem, async, overlap         |
| Model Inference | PyTorch FP32 eager           | TensorRT FP16/INT8, CUDA Graphs    |
| NMS             | Python (CPU loop)            | GPU NMS                            |
| Throughput      | \~10–30 FPS (YOLOv8n, 640px) | 60–200+ FPS (same model, same GPU) |
| Latency         | High (pipeline bottlenecks)  | Low, real-time                     |


## Training Optimization Checklist (YOLO)
- Use mixed precision (AMP).
- Maximize batch size per GPU (with gradient accumulation if needed).
- Accelerate augmentation (DALI, CV-CUDA, Albumentations).
- Enable dataset caching / sharding.
- Train with DDP on multiple GPUs.
- Optimize optimizer (SGD/AdamW) + scheduler (cosine).
- Apply Mosaic/MixUp/Label smoothing for stronger models.


## Unrelated notes
- Multitasking & Processes
- Threads and Multithreading
- Context switch
- Thread Scheduler
- Mutual Exclusion
- Thread Contention
- Concurrency vs Parallelism
- MultiProcessor vs MultiCore vs Hyper-threading
- Threads and CPU cache
  
https://www.logicbig.com/quick-info/programming/multi-threading.html













