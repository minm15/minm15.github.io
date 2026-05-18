# Memory Management

Linux memory management is responsible for translating virtual memory into physical memory while balancing performance, isolation, and reclamation.

## Core Areas

- Virtual address spaces are maintained per process.
- Page tables map virtual addresses to physical frames.
- The kernel tracks anonymous memory and file-backed memory differently.
- Reclaim decisions are influenced by memory pressure and access patterns.

## Important Mechanisms

### Demand Paging

Pages are usually populated on first access instead of process start. This keeps startup cheap and avoids allocating memory that may never be touched.

### Page Cache

File-backed pages can stay in memory after I/O so later reads avoid storage access. This is why filesystem and memory behavior are tightly connected.

### Reclaim

When free memory drops, the kernel scans reclaimable pages and may evict cache, write dirty pages back, or swap out anonymous pages.

## Practical Signals

- High major page fault counts usually indicate real I/O latency.
- Heavy reclaim activity can increase tail latency.
- Memory pressure can surface as stalled writes, slower allocations, or swap usage.
