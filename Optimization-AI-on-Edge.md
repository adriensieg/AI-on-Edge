# Optimization

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

#### Hardwares
- **CPU cores**
   - Use `taskset`/`sched_setaffinity` to pin inference to specific cores.
   - Isolate “clean” cores by moving OS background daemons to other cores.
- **CPU frequency scaling**
   - Disable frequency scaling (use performance governor).
   - Avoid thermal throttling → add heatsink/fan if necessary.
- **Threads vs Hyperthreads**
   - Disable SMT (if hardware supports) if it adds contention.
   - Stick to physical cores for predictability. 
- **GPU / NPU / TPU**
   - If your Pi has an attached Coral USB/PCIe TPU, Movidius NCS2, or built-in GPU, offload YOLO inference.
   - Quantized models (`int8`, `fp16`) are critical here.

- **Memory (DRAM)**
   - Ensure sufficient RAM headroom to avoid swap.
   - Reduce memory latency: prefer single-bank allocations, lock important pages into RAM (mlock).
- **Cache (L1/L2/L3)**
   - Structure data to maximize cache locality (tensor layout matters). Tensor layout tuning (e.g., NHWC vs NCHW).
   - Warm-cache core pinning.
   - Micro-level wins, not first-order.
   - Pin inference threads so they reuse warm caches.
- **Storage / I/O**
   - Preload weights into RAM; avoid runtime disk access.
   - Use `tmpfs/ramfs` for transient data.
     
- **Network interface (if streaming camera data remotely)**
   - Minimize packet copying (use kernel zero-copy: `sendfile`, `mmap`).
   - Reduce IRQ load by tuning NIC offloads and coalescing.
   - I/O-aware resizing: Pre-resize frames close to network size (e.g., 640×640) with GPU ops (NPP/CV-CUDA) or hardware decoders to avoid CPU bottlenecks.
 
   - **Use hardware decode**: NVDEC via `GStreamer`/`FFmpeg` (e.g., `h264_cuvid`, `nvh264dec`) or `NVIDIA DeepStream`.
      - Avoid CPU decoding if you have an NVIDIA GPU.
   - **Zero/low-copy pipelines**:
      - Prefer DMABUF / GPU surface handoff (GStreamer) to keep frames on GPU.
      - In PyTorch, use pinned memory (pin_memory=True) and non_blocking=True on .to(device).

Prefetch & parallelize: DataLoader with enough workers (num_workers ~= cores/4…cores/2), prefetch_factor tuned; or use NVIDIA DALI to move aug/decode to GPU.

Storage layout: Keep datasets/recordings on NVMe; avoid network filesystems for live inference. Increase file readahead for sequential reads:
sudo blockdev --setra 4096 /dev/nvme0n1.

#### Kernel & OS-Level
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

#### Runtime / Libraries
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

#### Application & Data Flow
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
    
#### Observability & Tuning

- Latency per frame (end-to-end, not just inference).
- Cache misses (via perf stat).
- IRQ distribution (via /proc/interrupts).
- Thermal throttling (via vcgencmd measure_temp).
- Page faults (major/minor).




















