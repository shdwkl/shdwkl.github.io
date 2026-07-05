---
layout: post
title: External Fragmentation Wastes Memory Between Variable-Sized Segments
date: 2026-07-05 07:47 +0000

type: mechanism
source: "OSTEP Ch. 16: Segmentation"
tags:
  - virtualization
  - memory
  - segmentation
  - fragmentation
related:
  - "[[Segmentation Generalizes Base-and-Bounds with Multiple Segments]]"
  - "[[Coarse-Grained vs Fine-Grained Segmentation Trade Flexibility for Simplicity]]"
  - "[[The OS Manages Memory at Process Creation and Context Switch]]"
  - "[[malloc and free Manage Heap Memory Dynamically]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Segmentation Generalizes Base-and-Bounds with Multiple Segments]]"
categories: [books, OSTEP]
---

# External Fragmentation Wastes Memory Between Variable-Sized Segments

> External fragmentation occurs when free physical memory is broken into small, non-contiguous holes between allocated segments. Although the total free space may be sufficient for a new segment, no single contiguous region is large enough. This is the fundamental problem of variable-sized allocation and the primary motivation for both free-space management algorithms and the eventual shift to fixed-size paging.
{: .prompt-tip }

## How Fragmentation Develops

Consider a system that allocates and frees segments over time:

```
Physical Memory (64 KB):
+--------+------+--------+------+--------+------+
|  Seg A |      |  Seg C |      |  Seg E |      |
| (8 KB) | 4 KB | (6 KB) | 2 KB | (4 KB) |10 KB |
+--------+------+--------+------+--------+------+
           free            free            free
```

Total free: 4 + 2 + 10 = 16 KB. But a new 12 KB segment cannot be allocated because no single contiguous hole is 12 KB. The memory is fragmented—free space exists but is unusable for large allocations.

## External vs. Internal Fragmentation

These are distinct problems:

- **Internal fragmentation** (seen with [Base-and-Bounds Provides Dynamic Relocation]({% post_url 2026-07-04-base-and-bounds-provides-dynamic-relocation %})): waste *inside* an allocated region—space allocated to a process but never used.
- **External fragmentation** (segmentation's problem): waste *between* allocated regions—free space that exists but cannot satisfy requests because it is scattered.

Segmentation eliminates internal waste by sizing each segment to fit its contents, but introduces external fragmentation because those variable-sized segments leave irregular gaps when freed.

## Approaches to External Fragmentation

The OS has several strategies to manage fragmentation:

**Compaction** moves all allocated segments to one end of physical memory, creating a single large free region. This is expensive—every segment must be copied, and all base registers must be updated. During compaction, the process cannot run. Compaction is the simplest conceptual solution but rarely used due to its cost.

**Free-space management algorithms** attempt to reduce fragmentation by choosing where to place new segments. Strategies like best-fit (smallest sufficient hole), worst-fit (largest hole), and first-fit (first sufficient hole) each produce different fragmentation patterns. These are explored in detail in the memory allocator context and the free-space management chapter.

**Fixed-size allocation** (paging) avoids external fragmentation entirely by dividing both physical and virtual memory into equal-sized units. Since every frame is the same size, any free frame can satisfy any page request—fragmentation is impossible. This insight drives the design of paging, which replaces segmentation in most modern systems.

## Why This Matters

External fragmentation is not just a theoretical concern. Long-running systems that allocate and free segments of varying sizes inevitably develop fragmented memory. The OS free list must track these scattered holes, and allocation requests may fail even when total free memory is sufficient. This unreliability is the core reason that paging—despite its own overheads (page tables, TLB misses)—became the dominant memory virtualization mechanism.
