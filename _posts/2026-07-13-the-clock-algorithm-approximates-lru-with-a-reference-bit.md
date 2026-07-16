---
layout: post
title: The Clock Algorithm Approximates LRU with a Reference Bit
date: 2026-07-13 06:21 +0300
type: mechanism
source: "OSTEP Ch. 22: Beyond Physical Memory: Policies"
tags:
  - virtualization
  - memory
  - paging
  - replacement-policy
related:
  - "[[LRU Approximates Optimal by Evicting Least Recently Used Pages]]"
  - "[[Page Table Entries Contain PFN and Control Bits]]"
  - "[[FIFO Page Replacement Evicts the Oldest Page]]"
  - "[[Optimal Page Replacement Sets the Theoretical Lower Bound]]"
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[LRU Approximates Optimal by Evicting Least Recently Used Pages]]"
  - "[[Page Table Entries Contain PFN and Control Bits]]"
categories: [books, OSTEP]
---

> The Clock algorithm (also called second-chance) approximates LRU using only the hardware reference bit. Pages are arranged in a circular list with a clock hand pointer. On eviction, the hand sweeps through pages: if reference bit = 1, clear it and advance; if reference bit = 0, evict that page. This gives recently-used pages a "second chance" before eviction.
{: .prompt-tip }

## The Clock Mechanism

Imagine pages arranged in a circle with a clock hand:

```
        hand
          |
          v
    +---+---+---+---+
    | 0 | 1 | 2 | 3 |  <-- pages in circular buffer
    | 1 | 0 | 1 | 0 |  <-- reference bits
    +---+---+---+---+
```

When eviction is needed:

```c
while (pages[hand].reference_bit == 1) {
    pages[hand].reference_bit = 0;  // give second chance
    hand = (hand + 1) % num_pages;   // advance clockwise
}
victim = pages[hand];               // evict this page
hand = (hand + 1) % num_pages;      // advance past victim
```

## Worked Example

**Reference string**: 0, 1, 2, 3, 0, 1, 4, 0, 1, 2, 3, 4
**Physical frames**: 3

| Access | Pages (ref bits) | Hand    | Fault? | Action                       |
| ------ | ---------------- | ------- | ------ | ---------------------------- |
| 0      | [0:1, -, -]      | 0→1     | ✓      | Load 0                       |
| 1      | [0:1, 1:1, -]    | 1→2     | ✓      | Load 1                       |
| 2      | [0:1, 1:1, 2:1]  | 2→0     | ✓      | Load 2                       |
| 3      | [3:1, 1:0, 2:0]  | 0→1     | ✓      | Clear 0:1→0, evict 0, load 3 |
| 0      | [3:0, 0:1, 2:0]  | 1→2     | ✓      | Clear 1:1→0, evict 1, load 0 |
| 1      | [3:0, 0:1, 1:1]  | 2→0     | ✓      | Clear 2:1→0, evict 2, load 1 |
| 4      | [4:1, 0:1, 1:1]  | 0→1     | ✓      | Evict 3 (ref=0), load 4      |
| 0      | [4:1, 0:1, 1:1]  | 1       | -      | Hit, set ref=1               |
| 1      | [4:1, 0:1, 1:1]  | 1       | -      | Hit, set ref=1               |
| 2      | [4:0, 0:0, 2:1]  | 1→2→0→1 | ✓      | Clear 0,1, evict 4, load 2   |
| 3      | [3:1, 0:0, 2:1]  | 1→2     | ✓      | Evict 0 (ref=0), load 3      |
| 4      | [3:1, 4:1, 2:0]  | 2→0     | ✓      | Evict 2 (ref=0), load 4      |

**Clock: 9 faults** (same as FIFO for this trace, but often better)

## Why Clock Works

Clock combines FIFO's simplicity with LRU's recency awareness:

- **Like FIFO**: Pages ordered by load time (circular queue)
- **Like LRU**: Recently-accessed pages get second chances

The reference bit acts as a binary "recently used" indicator. When the hand sweeps, it clears bits—so a page must be accessed between sweeps to keep its protection.

## Enhanced Clock: Considering Dirty Bit

Some systems also consider the dirty bit when evicting:

| Reference | Dirty | Priority | Reason |
|-----------|-------|----------|--------|
| 0 | 0 | Best victim | Not used, no write needed |
| 0 | 1 | Second choice | Not used, but needs write |
| 1 | 0 | Third choice | Used, no write needed |
| 1 | 1 | Worst victim | Used and needs write |

This optimizes for both recency and I/O cost.

## Clock vs. LRU Performance

Clock doesn't maintain exact recency order—it only knows "accessed since last sweep" (binary). For most workloads, this coarse approximation of [LRU Approximates Optimal by Evicting Least Recently Used Pages - LRU]({% post_url 2026-07-13-lru-approximates-optimal-by-evicting-least-recently-used-pages %}) suffices:

- Near-LRU performance on typical workloads
- O(1) overhead per memory access (hardware sets bit)
- O(n) worst-case eviction (one full sweep)

The sweep ensures eventually finding a victim, and the second-chance mechanism prevents evicting hot pages.

## Practical Usage

Clock is the foundation of most real page replacement implementations:

- **Linux**: Uses a variant (two-list approximation of LRU)
- **BSD**: Uses clock with modifications
- **Windows**: Uses a working-set-based approach

Pure LRU is rarely implemented due to overhead; clock provides the best balance of performance and implementation cost.
