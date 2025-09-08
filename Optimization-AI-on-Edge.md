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

### CPU cores
- Use `taskset`/`sched_setaffinity` to pin inference to specific cores.
- Isolate “clean” cores by moving OS background daemons to other cores.

### CPU frequency scaling
   - Disable frequency scaling (use performance governor).
   - Avoid thermal throttling → add heatsink/fan if necessary.
   
### GPU / NPU / TPU
   - If your Pi has an attached Coral USB/PCIe TPU, Movidius NCS2, or built-in GPU, offload YOLO inference.
   - Quantized models (`int8`, `fp16`) are critical here.

### Threads vs Hyperthreads
   - Disable SMT (if hardware supports) if it adds contention.
   - Stick to physical cores for predictability.

### Memory (DRAM)
- Ensure sufficient RAM headroom to avoid swap.
- Reduce memory latency: prefer single-bank allocations, lock important pages into RAM (mlock).

### Cache (L1/L2/L3)
- Structure data to maximize cache locality (tensor layout matters).
- Pin inference threads so they reuse warm caches.

### Storage / I/O
- Preload weights into RAM; avoid runtime disk access.
- Use tmpfs/ramfs for transient data.

### Network interface (if streaming camera data remotely)
- Minimize packet copying (use kernel zero-copy: `sendfile`, `mmap`).

Reduce IRQ load by tuning NIC offloads and coalescing.
