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

- **Hardware** (cores, cache, DRAM, GPU/NPU, thermal),
- **Kernel** (scheduler, IRQ affinity, hugepages, swap, I/O schedulers),
- **Runtime** (Python/C runtime, allocators, threading libs, quantization),
- **Application** (batch size, pipelining, zero-copy IO),
- **Observability** (latency, faults, thermals).
