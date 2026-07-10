---
layout: post
title: LRU Approximates Optimal by Evicting Least Recently Used Pages
date: 2026-07-13 06:19 +0300
type: algorithm
source: "OSTEP Ch. 22: Beyond Physical Memory: Policies"
tags:
  - virtualization
  - memory
  - paging
  - replacement-policy
related:
  - "[[Optimal Page Replacement Sets the Theoretical Lower Bound]]"
  - "[[FIFO Page Replacement Evicts the Oldest Page]]"
  - "[[The Clock Algorithm Approximates LRU with a Reference Bit]]"
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[Page Table Entries Contain PFN and Control Bits]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Optimal Page Replacement Sets the Theoretical Lower Bound]]"
  - "[[Page Faults Trigger the Swap Mechanism]]"
categories: [books, OSTEP]
---

# LRU Approximates Optimal by Evicting Least Recently Used Pages

> Least Recently Used (LRU) page replacement evicts the page that hasn't been accessed for the longest time. This policy uses past behavior to predict future access, leveraging temporal locality—recently used pages are likely to be used again soon. LRU performs close to optimal for many workloads but is expensive to implement perfectly.
{: .prompt-tip }

## The Key Insight: Past Predicts Future

[[Optimal Page Replacement Sets the Theoretical Lower Bound|OPT]] evicts the page used furthest in the future—but we can't know the future. LRU inverts this: evict the page used furthest in the past. This works because programs exhibit **temporal locality**: if a page was accessed recently, it's likely to be accessed again soon.

```
OPT:  Evict page with longest time to NEXT access (future)
LRU:  Evict page with longest time since LAST access (past)
```

## Worked Example

**Reference string**: 0, 1, 2, 3, 0, 1, 4, 0, 1, 2, 3, 4
**Physical frames**: 3

| Access | Frame 0 | Frame 1 | Frame 2 | Last Used | Fault? | Evicted |
|--------|---------|---------|---------|-----------|--------|---------|
| 0 | **0** | - | - | 0:t0 | ✓ | - |
| 1 | 0 | **1** | - | 0:t0, 1:t1 | ✓ | - |
| 2 | 0 | 1 | **2** | 0:t0, 1:t1, 2:t2 | ✓ | - |
| 3 | 0 | 1 | **3** | 0:t0, 1:t1, 3:t3 | ✓ | 2 (LRU at t2) |
| 0 | 0 | 1 | 3 | 0:t4, 1:t1, 3:t3 | - | - |
| 1 | 0 | 1 | 3 | 0:t4, 1:t5, 3:t3 | - | - |
| 4 | 0 | 1 | **4** | 0:t4, 1:t5, 4:t6 | ✓ | 3 (LRU at t3) |
| 0 | 0 | 1 | 4 | 0:t7, 1:t5, 4:t6 | - | - |
| 1 | 0 | 1 | 4 | 0:t7, 1:t8, 4:t6 | - | - |
| 2 | 0 | 1 | **2** | 0:t7, 1:t8, 2:t9 | ✓ | 4 (LRU at t6) |
| 3 | 0 | **3** | 2 | 0:t7, 3:t10, 2:t9 | ✓ | 1 (LRU at t8) |
| 4 | **4** | 3 | 2 | 4:t11, 3:t10, 2:t9 | ✓ | 0 (LRU at t7) |

**LRU: 8 faults** (vs. OPT's 7, FIFO's 9)

LRU keeps pages 0 and 1 around during their hot streak (accesses at t4-t8), only evicting them later.

## Implementation Challenges

Perfect LRU requires updating metadata on **every memory access**:

1. **Timestamp approach**: Record access time for each page. On eviction, find minimum timestamp. Problem: updating timestamp on every access is expensive.

2. **Stack approach**: Maintain a stack of pages; move page to top on access. On eviction, remove bottom. Problem: stack manipulation on every access.

Both approaches require O(n) work on eviction or constant overhead on every access—neither scales well.

## Hardware Support: Reference Bit

Rather than perfect timestamps, hardware provides a **reference bit** (accessed bit) in each Page Table Entries Contain PFN and Control Bits. The MMU sets this bit on any access. The OS can:

1. Periodically clear all reference bits
2. On eviction, prefer pages with reference bit = 0

This doesn't give exact LRU ordering, but approximates it well. The [The Clock Algorithm Approximates LRU with a Reference Bit - Clock algorithm]({% post_url 2026-07-10-the-clock-algorithm-approximates-lru-with-a-reference-bit %})  formalizes this approach.

## LRU Is a Stack Algorithm

LRU has a useful property: the set of pages in memory with n frames is always a subset of those with n+1 frames. This guarantees that more memory never increases faults (unlike [FIFO Page Replacement Evicts the Oldest Page - FIFO's anomaly]({% post_url 2026-07-10-fifo-page-replacement-evicts-the-oldest-page %})).

## Workloads Where LRU Struggles

LRU performs poorly on:
- **Scans**: Sequential access through a large file touches each page once. LRU keeps recently-scanned pages, which won't be accessed again.
- **Looping patterns larger than memory**: A loop accessing pages 0-10 with only 5 frames will fault constantly.

For these cases, other policies (like scan-resistant ARC) may help.
Adaptive Replacement Cache (e..g `OpenZFS`)
