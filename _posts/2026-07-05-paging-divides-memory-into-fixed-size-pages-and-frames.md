---
layout: post
title: Paging Divides Memory into Fixed-Size Pages and Frames
date: 2026-07-05 07:55 +0000
type: mechanism
source: "OSTEP Ch. 18: Paging: Introduction"
tags:
  - virtualization
  - memory
  - paging
related:
  - "[[External Fragmentation Wastes Memory Between Variable-Sized Segments]]"
  - "[[Segmentation Generalizes Base-and-Bounds with Multiple Segments]]"
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
  - "[[The Page Table Stores Virtual-to-Physical Mappings]]"
  - "[[Virtual Addresses Split into VPN and Offset]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[External Fragmentation Wastes Memory Between Variable-Sized Segments]]"
  - "[[Segmentation Generalizes Base-and-Bounds with Multiple Segments]]"
categories: [books, OSTEP]
---

> Paging partitions both virtual and physical memory into fixed-size units called pages (virtual) and frames (physical). Any virtual page can map to any physical frame, eliminating external fragmentation because all allocation units are interchangeable. This flexibility is paging's fundamental advantage over variable-sized segmentation.
{: .prompt-tip }

## The Core Idea

Segmentation allocated variable-sized regions, leading to [[External Fragmentation Wastes Memory Between Variable-Sized Segments|external fragmentation]]—scattered holes that could not satisfy requests even when total free memory was sufficient. Paging takes the opposite approach: divide everything into uniform chunks.

**Virtual address space**: divided into fixed-size **pages** (e.g., 4 KB each)
**Physical memory**: divided into fixed-size **frames** (same size as pages)

When a process needs memory, the OS assigns any free frame to any virtual page. Since all frames are identical, there is no "hole too small" problem—any free frame works.

## Visualizing Paged Memory

Consider a tiny 64-byte address space with 16-byte pages, mapped to 128 bytes of physical memory:

```
Virtual Address Space        Physical Memory (8 frames)
+--------+ page 0           +--------+ frame 0
| code   |        ─────────>| (free) |
+--------+ page 1           +--------+ frame 1
| code   |        ─────┐    | code   | <- virtual page 0
+--------+ page 2      │    +--------+ frame 2
| heap   |        ─────│───>| (free) |
+--------+ page 3      │    +--------+ frame 3
| stack  |        ─────│─┐  | stack  | <- virtual page 3
+--------+             │ │  +--------+ frame 4
                       │ │  | heap   | <- virtual page 2
                       │ │  +--------+ frame 5
                       │ └─>| code   | <- virtual page 1
                       └───>+--------+ frame 6
                            | (free) |
                            +--------+ frame 7
                            | (free) |
                            +--------+
```

The virtual pages are contiguous (0, 1, 2, 3), but they map to non-contiguous physical frames (1, 5, 4, 3). This is perfectly fine—the hardware translates each page independently.

## No External Fragmentation

With segmentation, if physical memory had three 10 KB holes, a 25 KB segment could not be allocated. With paging, a 25 KB allocation simply needs seven 4 KB pages (28 KB, rounded up). Any seven free frames suffice—they need not be adjacent. This is the key insight: **fixed-size allocation eliminates external fragmentation**.

## The Trade-Off: Internal Fragmentation

Paging introduces a different kind of waste. If a process needs 25 KB but pages are 4 KB, it gets 7 pages (28 KB)—wasting 3 KB inside the last page. This is internal fragmentation, but it is bounded: the worst case is one page minus one byte per allocation. With typical 4 KB pages, this is negligible compared to the unbounded external fragmentation of segmentation.

## Why Fixed Size Works

The power of paging comes from uniformity. The [[The Page Table Stores Virtual-to-Physical Mappings|page table]] simply records which frame holds each page. The OS free list becomes trivial—any free frame is as good as any other. Allocation is O(1) (grab any frame), and there is no need for complex best-fit or coalescing algorithms.
