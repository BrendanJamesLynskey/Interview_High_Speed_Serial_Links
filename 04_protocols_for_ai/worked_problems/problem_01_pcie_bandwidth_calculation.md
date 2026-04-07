# Worked Problem 01: PCIe Bandwidth Calculation

## Problem Statement

An AI inference server uses 4 GPUs, each connected to the host CPU via PCIe Gen5 x16. Each GPU processes inference requests that require: (a) loading a 70B parameter model (140 GB in FP16) from CPU memory to GPU HBM at startup, (b) processing batches of 32 requests, each with a 2048-token input (4 KB per request after tokenization), and (c) returning generated output tokens at 100 tokens/second per request. Calculate the PCIe bandwidth utilization for each phase and determine if PCIe Gen5 is sufficient.

---

## Worked Solution

### Step 1: Calculate PCIe Gen5 x16 bandwidth

```
Per-lane rate: 32 GT/s (NRZ)
Encoding overhead: 128b/130b = 1.54%
Effective per-lane BW: 32 * (128/130) = 31.508 Gbps = 3.938 GB/s
x16 unidirectional BW: 3.938 * 16 = 63.015 GB/s
x16 bidirectional BW: 126.03 GB/s (but each direction is 63 GB/s)
```

### Step 2: Phase A - Model loading

Model size: 140 GB (70B parameters in FP16, 2 bytes each)
Models distributed across 4 GPUs using tensor parallelism: 140 / 4 = 35 GB per GPU.

```
Transfer time per GPU = 35 GB / 63 GB/s = 0.556 seconds
```

All 4 GPUs load in parallel (each has its own PCIe link): total time = 0.556 seconds.

PCIe utilization during model loading: approximately 100% (saturating the link in one direction).

### Step 3: Phase B - Input data transfer (CPU to GPU)

Per-request input: 4 KB (2048 tokens, 2 bytes per token ID)
Batch size: 32 requests
Total input per batch: 32 * 4 KB = 128 KB

Assuming requests are distributed equally across 4 GPUs: 128 KB / 4 = 32 KB per GPU per batch.

```
Transfer time = 32 KB / 63 GB/s = 0.508 ns per batch per GPU
```

The input transfer is negligible compared to the batch processing time (typically 10-100 ms for a transformer inference batch).

PCIe utilization: less than 0.001% (trivially small).

### Step 4: Phase C - Output token generation

Tokens generated: 100 tokens/s per request * 32 requests = 3200 tokens/s total
Token size: approximately 4 bytes (token ID + probability)
Data rate: 3200 * 4 = 12.8 KB/s per GPU (for 8 requests per GPU)

```
PCIe utilization = 12.8 KB/s / 63 GB/s = 0.00002% (negligible)
```

### Step 5: Consider KV-cache transfers (if using disaggregated serving)

In disaggregated inference architectures, the KV-cache for each request may need to be transferred between GPUs or between CPU and GPU. For a 70B model with 80 layers, 128 heads, 128-dim per head:

```
KV-cache per token per layer: 2 * 128 * 128 * 2 bytes = 65.5 KB
KV-cache per token (all layers): 65.5 KB * 80 = 5.24 MB
KV-cache for 2048 tokens: 5.24 MB * 2048 = 10.74 GB
```

If KV-cache is transferred between GPUs via PCIe (through CPU memory):

```
Transfer time = 10.74 GB / 63 GB/s = 170 ms per request
PCIe utilization: potentially 100% during KV-cache transfer
```

This is significant and may bottleneck disaggregated inference architectures.

### Result

| Phase | Data per GPU | PCIe BW Utilization | Duration |
|-------|-------------|-------------------|----------|
| Model loading | 35 GB | ~100% | 0.56 s |
| Input transfer | 32 KB/batch | < 0.001% | ~0.5 ns |
| Token output | 12.8 KB/s | < 0.001% | Continuous |
| KV-cache transfer | 10.74 GB | ~100% | 170 ms |

PCIe Gen5 x16 is sufficient for standard inference workloads (model loading and token I/O). However, KV-cache transfers in disaggregated serving can saturate the PCIe link, making Gen6 (2x bandwidth) potentially necessary for advanced inference architectures.

### Key Takeaways

1. Model loading is a one-time cost that saturates PCIe but completes in under 1 second.
2. Steady-state inference I/O is negligible relative to PCIe bandwidth.
3. KV-cache transfers in disaggregated architectures are the primary bandwidth consumer.
4. PCIe Gen5 x16 provides 63 GB/s per direction, which is adequate for standard inference but may bottleneck advanced serving patterns.
5. For training workloads, NVLink (900 GB/s) is the primary interconnect; PCIe is secondary.
