# Worked Problem 03: Multi-GPU Interconnect Topology

## Problem Statement

Design the interconnect topology for an 8-GPU AI training node. Each GPU has 18 NVLink 4.0 ports (each port provides 25 GB/s bidirectional bandwidth). The system requires: all-to-all communication for data-parallel gradient all-reduce, maximum bisection bandwidth, and minimum all-reduce latency for a 1 GB gradient tensor.

Compare three topologies: (a) ring, (b) fully-connected via NVSwitch, and (c) 2D torus. Calculate the all-reduce time for each.

---

## Worked Solution

### Step 1: Define the parameters

- 8 GPUs, each with 18 NVLink 4.0 ports
- Each port: 25 GB/s bidirectional (12.5 GB/s each direction)
- Gradient tensor: 1 GB per GPU
- NVSwitch latency: 5 ns per hop (in addition to SerDes latency of ~20 ns per hop)

### Step 2: Topology A - Ring

In a ring, each GPU connects to its two neighbors. With 18 ports, we can allocate 9 ports per direction:

```
BW per direction in ring: 9 * 12.5 GB/s = 112.5 GB/s
```

Ring all-reduce time (using ring algorithm):
```
T_ring = 2 * (N-1)/N * data_size / BW_ring
T_ring = 2 * 7/8 * 1 GB / 112.5 GB/s
T_ring = 1.75 * 1/112.5
T_ring = 15.6 ms
```

Wait, that seems too high. Let me reconsider. The ring all-reduce sends data_size * (N-1)/N in each phase, with two phases (scatter-reduce and allgather):

```
T_ring = 2 * (N-1)/N * data_size / BW_per_link
= 2 * (7/8) * 1 GB / 112.5 GB/s
= 15.56 us (microseconds, not milliseconds)
```

Actually: 1 GB / 112.5 GB/s = 8.89 ms. No:

```
1 GB = 1e9 bytes
112.5 GB/s = 112.5e9 bytes/s
1e9 / 112.5e9 = 8.89e-3 s = 8.89 ms
T_ring = 2 * (7/8) * 8.89 ms = 15.56 ms
```

This is correct but very slow. The issue is that 1 GB at 112.5 GB/s takes 8.89 ms. Let me recheck: 1 GB / 112.5 GB/s = 0.00889 seconds = 8.89 ms. Yes, that is correct.

Bisection bandwidth (ring): BW per link in each direction = 112.5 GB/s. The bisection cuts 2 links (ring has 2 links crossing any bisection), so bisection BW = 2 * 112.5 = 225 GB/s.

### Step 3: Topology B - Fully-connected via NVSwitch

With 4 NVSwitch ASICs, each GPU allocates approximately 18/7 = 2-3 ports per remote GPU (18 ports connecting to 7 other GPUs). For equal distribution:

```
Ports per remote GPU: 18/7 ≈ 2.57
BW per GPU pair: 2.57 * 25 GB/s = 64.3 GB/s bidirectional
```

Actually, with NVSwitch, the allocation is: each GPU connects all 18 ports to the NVSwitch fabric, and the NVSwitch provides any-to-any switching. The per-GPU bandwidth to the switch is:

```
Per-GPU total BW: 18 * 25 GB/s = 450 GB/s bidirectional (225 GB/s each direction)
```

For all-reduce with a fully-connected topology (reduce-scatter + allgather):
```
T_fc = 2 * (N-1)/N * data_size / per_GPU_BW
T_fc = 2 * (7/8) * 1 GB / 225 GB/s
T_fc = 1.75 / 225 = 7.78 ms
```

But actually, the fully-connected topology can do better with a direct all-reduce:
```
T_fc_direct = data_size / per_GPU_BW + latency_overhead
= 1 GB / 225 GB/s + 7 * 25 ns (per hop latency per remote GPU)
= 4.44 ms + 0.000175 ms
= 4.44 ms
```

No, the correct formula for ring-based all-reduce on a fully connected topology is the same ring algorithm but with higher per-link bandwidth. Alternatively, the recursive halving-doubling algorithm:

```
T_rhd = log2(N) * (alpha + data_size / (N * BW_per_pair))
= 3 * (25 ns + 1 GB / (8 * 64.3 GB/s))
= 3 * (25 ns + 1.94 ms)
= 3 * 1.94 ms = 5.83 ms
```

For a properly optimized all-reduce on fully-connected:
```
T_fc = 2 * (N-1)/N * data_size / per_GPU_BW_egress
= 2 * 7/8 * 1e9 / (225e9)
= 7.78 ms
```

Bisection BW: 4 * 225 = 900 GB/s (4 GPUs on each side, each with 225 GB/s to the switch).

### Step 4: Topology C - 2D Torus (4x2)

In a 4x2 torus, each GPU has 4 neighbors (2 in each dimension). Allocate ports: 18/4 = 4.5 ports per neighbor = 4 ports per neighbor (16 used, 2 spare).

```
BW per neighbor: 4 * 25 = 100 GB/s bidirectional (50 GB/s each direction)
```

All-reduce on 2D torus uses dimension-based reduction:
```
T_2d = sum over each dimension of: 2 * (N_dim - 1)/N_dim * data_size / BW_dim
```

For 4x2 torus:
- Dimension 1 (size 4): T_1 = 2 * 3/4 * 1 GB / 50 GB/s = 30 ms? 

Let me recalculate: 1 GB / 50 GB/s = 20 ms.
T_1 = 2 * (3/4) * 20 ms = 30 ms

This is very slow. But note that after dimension 1, the data size is reduced by the dimension 1 factor. Actually, for the reduce-scatter phase in dimension 1, each GPU sends/receives 1/4 of the data:

```
T_1_scatter = (N1-1)/N1 * data_size / BW_dim1 = 3/4 * 1 GB / 50 GB/s = 15 ms
T_1_gather = same = 15 ms
```

Hmm, this is the right calculation but 30 ms is indeed slow.

Bisection BW: the bisection of a 4x2 torus cuts 2*2 = 4 links in dimension 1 plus 2*4 = 8 links in dimension 2. Total bisection BW = min(4*100, 8*100) = 400 GB/s.

### Step 5: Compare topologies

| Metric | Ring | Fully-Connected | 2D Torus |
|--------|------|----------------|----------|
| Links per GPU | 2 (9 ports each) | 18 (to switch) | 4 (4 ports each) |
| BW per link (bidir.) | 225 GB/s | 450 GB/s total | 100 GB/s |
| Bisection BW | 225 GB/s | 900 GB/s | 400 GB/s |
| All-reduce time (1 GB) | 15.6 ms | 7.8 ms | ~30 ms |
| Extra hardware | None | 4 NVSwitch | None |
| Hop count (worst case) | 4 | 1 (via switch) | 3 |

### Step 6: Practical considerations

The fully-connected topology with NVSwitch provides the best all-reduce performance (7.8 ms for 1 GB) at the cost of 4 additional NVSwitch ASICs. The ring topology is simpler but 2x slower. The 2D torus is the slowest due to the lower per-link bandwidth (spreading 18 ports across 4 neighbors instead of concentrating them).

In practice, NVIDIA uses the fully-connected topology for DGX systems precisely because the all-reduce performance advantage justifies the NVSwitch cost and power.

### Result

The fully-connected topology via NVSwitch is optimal for an 8-GPU training node, providing 2x better all-reduce performance than a ring and 4x better than a 2D torus. The NVSwitch overhead (4 ASICs at ~150W each = 600W) is justified by the training throughput improvement.

### Key Takeaways

1. Bisection bandwidth is the key metric: fully-connected provides 900 GB/s vs 225 GB/s for ring.
2. All-reduce time scales inversely with bisection bandwidth for large data sizes.
3. NVSwitch's cost (power, silicon area) is justified by the training efficiency improvement.
4. The 2D torus is bandwidth-inefficient for 8 GPUs because ports are spread across 4 neighbors.
5. For larger systems (1000+ GPUs), hierarchical topologies combining NVSwitch intra-node with network inter-node are necessary.
