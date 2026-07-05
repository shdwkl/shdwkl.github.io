---
layout: post
title: Segmentation Generalizes Base-and-Bounds with Multiple Segments
date: 2026-07-05 07:44 +0000
type: mechanism
source: "OSTEP Ch. 16: Segmentation"
tags:
  - virtualization
  - memory
  - segmentation
related:
  - "[[Base-and-Bounds Provides Dynamic Relocation]]"
  - "[[Address Translation Maps Virtual Addresses to Physical Addresses]]"
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
  - "[[Code Stack and Heap Each Get Their Own Segment]]"
  - "[[External Fragmentation Wastes Memory Between Variable-Sized Segments]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Base-and-Bounds Provides Dynamic Relocation]]"
categories: [books, OSTEP]
---

# Segmentation Generalizes Base-and-Bounds with Multiple Segments

> Segmentation extends dynamic relocation by giving each logical segment of the address space (code, heap, stack) its own base-bounds pair. Instead of allocating the entire address space contiguously—including unused gaps—the hardware maps only the segments that actually contain data. This eliminates the internal waste of simple base-and-bounds while preserving transparent address translation.
{: .prompt-tip }

## The Problem with a Single Base-Bounds

With [Base-and-Bounds Provides Dynamic Relocation]({% post_url 2026-07-04-base-and-bounds-provides-dynamic-relocation %}), the entire address space occupies a contiguous physical region. A process with a 16 KB virtual space that uses 2 KB of code, 1 KB of heap, and 1 KB of stack wastes 12 KB of physical memory on the unused gap between heap and stack. As address spaces grow (modern processes may request large virtual ranges), this internal waste becomes severe.

## The Segmentation Idea

Segmentation treats the address space not as one block but as a collection of variable-length **segments**, each representing a logically distinct region. The hardware maintains a separate base-bounds pair for each segment:

```
Segment   Base       Bounds   Grows Positive?
-------   --------   ------   ---------------
Code      32 KB      2 KB     Yes
Heap      34 KB      3 KB     Yes
Stack     28 KB      2 KB     No
```

Only the occupied portions of the address space consume physical memory. The gap between heap and stack simply does not exist in physical memory—no physical frames are wasted.

## Address Translation with Segments

The virtual address is split into two parts:

```
|  Segment Number  |     Offset     |
|    (top bits)    |  (remaining)   |
```

The hardware uses the segment number to index into the set of base-bounds pairs, then translates:

```
physical_address = segment_base + offset
if offset >= segment_bounds:
    raise segmentation fault
```

For example, with a 14-bit virtual address space and 2-bit segment selector:
- Virtual address `0x1A00` → segment `01` (heap), offset `0x200`
- Hardware looks up heap base (34 KB) → physical address = 34 KB + 512 = 35,328

The [[The Segment Table Maps Segment Numbers to Base-Bounds Pairs|segment table]] holds these mappings for each process, and the OS saves and restores them during [[Context Switch Saves and Restores Process State|context switches]].

## What Segmentation Achieves

Segmentation solves the internal waste problem: physical memory only backs segments that contain actual data. It also enables **sharing**—two processes can point their code segments at the same physical region if they run the same program, provided the segment is marked read-only.

However, because segments are variable-sized, physical memory develops [[External Fragmentation Wastes Memory Between Variable-Sized Segments|external fragmentation]]—small unusable holes between allocated segments. This limitation motivates fixed-size allocation via paging.
