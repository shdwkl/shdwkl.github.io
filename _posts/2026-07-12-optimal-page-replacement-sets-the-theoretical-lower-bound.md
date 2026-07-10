---
layout: post
title: Optimal Page Replacement Sets the Theoretical Lower Bound
date: 2026-07-12 06:21 +0300
type: algorithm
source: "OSTEP Ch. 22: Beyond Physical Memory: Policies"
tags:
  - virtualization
  - memory
  - paging
  - replacement-policy
related:
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[FIFO Page Replacement Evicts the Oldest Page]]"
  - "[[LRU Approximates Optimal by Evicting Least Recently Used Pages]]"
  - "[[The Clock Algorithm Approximates LRU with a Reference Bit]]"
  - "[[Swap Space Extends Memory Beyond Physical Limits]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[The Present Bit Distinguishes Memory-Resident from Swapped Pages]]"
categories: [books, OSTEP]
---

# Optimal Page Replacement Sets the Theoretical Lower Bound

> The optimal (OPT or MIN) page replacement algorithm evicts the page that will be accessed furthest in the future. This minimizes page faults for any given reference string, but requires knowledge of future accesses—making it impossible to implement in practice. OPT serves as a theoretical benchmark to evaluate how well real algorithms perform.
{: .prompt-tip }

## The Optimal Strategy

When a page fault occurs and memory is full, OPT examines all pages currently in memory and asks: "When will each page next be used?" It evicts the page whose next access is furthest away. If a page will never be accessed again, it's the ideal eviction candidate.

This strategy is provably optimal—no other algorithm can achieve fewer page faults for a given reference string and memory size.

## Worked Example

**Reference string**: 0, 1, 2, 3, 0, 1, 4, 0, 1, 2, 3, 4
**Physical frames**: 3

| Access | Frame 0 | Frame 1 | Frame 2 | Fault? | Evicted | Reason |
|--------|---------|---------|---------|--------|---------|--------|
| 0 | **0** | - | - | ✓ | - | Cold miss |
| 1 | 0 | **1** | - | ✓ | - | Cold miss |
| 2 | 0 | 1 | **2** | ✓ | - | Cold miss |
| 3 | 0 | 1 | **3** | ✓ | 2 | 2 used at t=9, 3 at t=10 |
| 0 | 0 | 1 | 3 | - | - | Hit |
| 1 | 0 | 1 | 3 | - | - | Hit |
| 4 | 0 | 1 | **4** | ✓ | 3 | 3 used at t=10, others sooner |
| 0 | 0 | 1 | 4 | - | - | Hit |
| 1 | 0 | 1 | 4 | - | - | Hit |
| 2 | **2** | 1 | 4 | ✓ | 0 | 0 never used again |
| 3 | 2 | **3** | 4 | ✓ | 1 | 1 never used again |
| 4 | 2 | 3 | 4 | - | - | Hit |

**OPT: 7 faults** (3 cold misses + 4 replacement faults)

## Why OPT Is Impossible to Implement

OPT requires knowledge of future memory accesses, which the OS doesn't have. The reference string unfolds as the program runs—we can't look ahead. This makes OPT a theoretical construct, not a practical algorithm.

However, OPT remains valuable:

1. **Benchmarking**: Compare real algorithms against OPT to measure quality
2. **Upper bound on performance**: Any algorithm achieving OPT-like results is doing well
3. **Insight**: Understanding what's optimal guides approximation strategies

## Historical Note: Belady's Algorithm

OPT is sometimes called **Belady's algorithm** after László Bélády, who formalized it in 1966. Belady also discovered the anomaly named after him: with FIFO replacement, adding more frames can paradoxically *increase* faults (see [FIFO Page Replacement Evicts the Oldest Page]({% post_url 2026-07-10-fifo-page-replacement-evicts-the-oldest-page %})).

## Practical Approximations

Since we can't know the future, practical algorithms use the **past** as a predictor:

- [LRU Approximates Optimal by Evicting Least Recently Used Pages \| LRU]({% post_url 2026-07-10-lru-approximates-optimal-by-evicting-least-recently-used-pages %}): Evict least recently used (past predicts future)
- [The Clock Algorithm Approximates LRU with a Reference Bit \| Clock]({% post_url 2026-07-10-the-clock-algorithm-approximates-lru-with-a-reference-bit %}): Approximate LRU with hardware support


The key insight: programs exhibit **temporal locality**—recently used pages are likely to be used again soon. This makes past usage a reasonable proxy for future usage.
