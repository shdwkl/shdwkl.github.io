---
layout: post
title: Swap Space Extends Memory Beyond Physical Limits
date: 2026-07-06 08:51 +0300
type: mechanism
source: "OSTEP Ch. 21: Beyond Physical Memory: Mechanisms"
tags:
  - virtualization
  - memory
  - paging
  - swapping
related:
  - "[[Paging Divides Memory into Fixed-Size Pages and Frames]]"
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
  - "[[Memory Virtualization Goals - Transparency Efficiency and Protection]]"
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[The Present Bit Distinguishes Memory-Resident from Swapped Pages]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Paging Divides Memory into Fixed-Size Pages and Frames]]"
  - "[[The Page Table Stores Virtual-to-Physical Mappings]]"
categories: [books, OSTEP]
---

> Swap space is a reserved region on disk that holds pages not currently in physical memory. By swapping pages between RAM and disk, the OS creates the illusion of an address space larger than physical memory. This enables running more processes simultaneously and supporting large applications, though at the cost of disk I/O when pages must be fetched.
{: .prompt-tip }

## The Memory Hierarchy Insight

Physical memory (RAM) is fast but limited and expensive. Disk storage is slow but large and cheap. The operating system exploits this hierarchy by keeping frequently-used pages in RAM and relegating rarely-used pages to disk. From the process's perspective, it still has access to its entire [[The Address Space Abstracts Physical Memory for Each Process|address space]]—the OS transparently manages which portions reside where.

## Swap Space Layout

The OS reserves a contiguous region of disk as swap space. This region is typically:

- A dedicated disk partition (common on Linux/Unix)
- A swap file within the file system (common on Windows, optional on Linux)

Pages in swap space are organized by **page-sized blocks**. The OS maintains a mapping from each swapped page to its disk location, either in the page table itself or in a separate data structure.

```
Physical Memory (RAM)         Swap Space (Disk)
+-------+-------+-------+    +-------+-------+-------+-------+
|  P0   |  P3   |  P7   |    |  P1   |  P2   |  P4   |  P5   |
+-------+-------+-------+    +-------+-------+-------+-------+
    ^                              ^
    |                              |
  Fast access (~100ns)        Slow access (~10ms)
```

## When Pages Move to Swap

Pages move from memory to swap (page-out) when:

1. **Memory pressure**: The system needs RAM for other pages
2. **Proactive swapping**: The OS preemptively swaps idle pages
3. **Process inactivity**: Background processes may have their pages swapped

The OS uses [[Page Table Entries Contain PFN and Control Bits|page table entries]] to track each page's location. A page in swap has its present bit cleared, signaling that accessing it requires disk I/O.

## Swap Size Considerations

Historically, swap space was sized at 2x physical RAM to handle worst-case scenarios where all processes were swapped out. Modern systems with abundant RAM use smaller swap ratios or none at all for desktop workloads. Servers often maintain substantial swap as a safety buffer.

The key trade-off: more swap allows more processes and larger working sets, but heavy swap usage ([[Thrashing Occurs When Working Set Exceeds Physical Memory|thrashing]]) dramatically degrades performance since disk is 100,000x slower than RAM.

## Swap vs. Memory-Mapped Files

Swap space differs from memory-mapped files: swap holds anonymous pages (heap, stack) with no file backing, while memory-mapped files have a natural disk location. When evicting a clean memory-mapped page, the OS simply discards it (the file is the backing store). Swap is specifically for pages that have no other home.
