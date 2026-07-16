---
layout: post
title: FIFO Page Replacement Evicts the Oldest Page
date: 2026-07-10 06:25 +0300
type: algorithm
source: "OSTEP Ch. 22: Beyond Physical Memory: Policies"
tags:
  - virtualization
  - memory
  - paging
  - replacement-policy
related:
  - "[[Optimal Page Replacement Sets the Theoretical Lower Bound]]"
  - "[[LRU Approximates Optimal by Evicting Least Recently Used Pages]]"
  - "[[The Clock Algorithm Approximates LRU with a Reference Bit]]"
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[FIFO Scheduling is Simple but Suffers from Convoy Effect]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[Optimal Page Replacement Sets the Theoretical Lower Bound]]"
categories: [books, OSTEP]
---

# FIFO Page Replacement Evicts the Oldest Page

> First-In, First-Out (FIFO) page replacement evicts the page that has been in memory the longest. It's simple to implement using a queue, but ignores usage patterns entirely. A frequently-used page loaded early will still be evicted first, making FIFO perform poorly compared to algorithms that consider recency of access.
{: .prompt-tip }

## The FIFO Strategy

FIFO maintains a queue of pages in load order. When eviction is needed, the page at the front (oldest) is removed:

```c
// Page load: add to back
queue_enqueue(page);

// Page fault with full memory: evict from front
victim = queue_dequeue();
evict(victim);
queue_enqueue(new_page);
```

This simplicity is FIFO's main advantage—no tracking of accesses required.

## Worked Example

**Reference string**: 0, 1, 2, 3, 0, 1, 4, 0, 1, 2, 3, 4
**Physical frames**: 3

| Access | Frame 0 | Frame 1 | Frame 2 | Queue | Fault? | Evicted |
|--------|---------|---------|---------|-------|--------|---------|
| 0 | **0** | - | - | [0] | ✓ | - |
| 1 | 0 | **1** | - | [0,1] | ✓ | - |
| 2 | 0 | 1 | **2** | [0,1,2] | ✓ | - |
| 3 | **3** | 1 | 2 | [1,2,3] | ✓ | 0 |
| 0 | 3 | **0** | 2 | [2,3,0] | ✓ | 1 |
| 1 | 3 | 0 | **1** | [3,0,1] | ✓ | 2 |
| 4 | **4** | 0 | 1 | [0,1,4] | ✓ | 3 |
| 0 | 4 | 0 | 1 | [0,1,4] | - | - |
| 1 | 4 | 0 | 1 | [0,1,4] | - | - |
| 2 | 4 | **2** | 1 | [1,4,2] | ✓ | 0 |
| 3 | 4 | 2 | **3** | [4,2,3] | ✓ | 1 |
| 4 | 4 | 2 | 3 | [4,2,3] | - | - |

**FIFO: 9 faults** (vs. OPT's 7 faults)

Notice how pages 0 and 1 are evicted despite being accessed repeatedly—FIFO doesn't consider usage patterns.

## Bélády's Anomaly

FIFO exhibits a counterintuitive behavior: **adding more frames can increase faults**. With the reference string 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5:

- 3 frames: 9 faults
- 4 frames: 10 faults

This anomaly occurs because FIFO's eviction decisions don't correlate with optimal choices. LRU and other "stack algorithms" are immune to this anomaly—more memory always helps or stays the same.

## Comparison with Scheduling FIFO

Like [[FIFO Scheduling is Simple but Suffers from Convoy Effect|FIFO scheduling]], FIFO page replacement is simple but ignores important characteristics. In scheduling, FIFO ignores job length (causing convoy effect). In page replacement, FIFO ignores access patterns (evicting heavily-used pages).

## Why FIFO Fails

FIFO assumes old pages are unused—but this isn't true:

- **Code pages**: Loaded early, used throughout execution
- **Stack base**: Loaded at process start, always relevant
- **Frequently-accessed data**: Age doesn't correlate with importance

The fundamental problem: **time in memory ≠ likelihood of future use**. Better algorithms track actual access patterns.

## When FIFO Might Be Acceptable

FIFO can work reasonably when:
- Access patterns are roughly uniform
- Memory is plentiful (few evictions needed)
- Implementation simplicity outweighs performance

However, LRU approximations (like [The Clock Algorithm Approximates LRU with a Reference Bit - Clock]({% post_url 2026-07-13-the-clock-algorithm-approximates-lru-with-a-reference-bit %})) add minimal complexity while performing much better.
