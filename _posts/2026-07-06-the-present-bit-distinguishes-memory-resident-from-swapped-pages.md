---
layout: post
title: The Present Bit Distinguishes Memory-Resident from Swapped Pages
date: 2026-07-06 08:51 +0300
type: mechanism
source: "OSTEP Ch. 21: Beyond Physical Memory: Mechanisms"
tags:
  - virtualization
  - memory
  - paging
  - swapping
  - hardware
related:
  - "[[Page Table Entries Contain PFN and Control Bits]]"
  - "[[Page Faults Trigger the Swap Mechanism]]"
  - "[[Swap Space Extends Memory Beyond Physical Limits]]"
  - "[[The Page Table Stores Virtual-to-Physical Mappings]]"
  - "[[Memory Virtualization Goals - Transparency Efficiency and Protection]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Page Table Entries Contain PFN and Control Bits]]"
  - "[[Swap Space Extends Memory Beyond Physical Limits]]"
categories: [books, OSTEP]
---

# The Present Bit Distinguishes Memory-Resident from Swapped Pages

> The present bit in each page table entry indicates whether a page currently resides in physical memory. When present = 1, the PFN field contains a valid frame number. When present = 0, the page exists but has been swapped to disk—the remaining PTE bits can store the disk location. This single bit is the hardware hook that makes demand paging possible.
{: .prompt-tip }

## Present vs. Valid: Two Different Questions

The page table entry contains both a **valid bit** and a **present bit**, which answer different questions:

| Bit | Question | If 0 |
|-----|----------|------|
| Valid | Does this virtual page exist at all? | Segmentation fault (invalid address) |
| Present | Is this valid page in physical memory? | Page fault (fetch from disk) |

A page can be valid but not present (swapped out). A page cannot be present but invalid—that would be nonsensical.

## Hardware Interpretation

On every memory access, the MMU checks these bits in sequence:

```
Check valid bit
  → If 0: raise segmentation fault (process error)
  → If 1: continue

Check present bit
  → If 0: raise page fault (OS handles swap-in)
  → If 1: extract PFN, form physical address
```

The hardware needs only one bit to trigger the page fault exception. Everything else—finding the page on disk, loading it, updating the PTE—is the OS's responsibility.

## Repurposing PTE Bits When Not Present

When present = 0, the PFN field is meaningless (there's no physical frame). The OS can repurpose these bits to store the **swap location**:

```
Present = 1 (in memory):
+---------------------------+---+-+
|          PFN              |   |1|
+---------------------------+---+-+

Present = 0 (swapped out):
+---------------------------+---+-+
|    Swap disk location     |   |0|
+---------------------------+---+-+
```

This is an elegant design: the same data structure serves both states without additional overhead.

## Example: Linux Swap Entry

On Linux, when a page is swapped out, the PTE encodes:

- Bit 0 = 0 (not present)
- Bits 1-7: Swap type (which swap device)
- Bits 8-57: Swap offset (location within swap)

When a page fault occurs, the handler extracts this information and knows exactly where to find the page.

## The Present Bit and Demand Paging

Demand paging relies entirely on this mechanism:

1. Process starts with all pages marked not-present
2. First access to each page triggers a fault
3. OS loads page, sets present = 1
4. Subsequent accesses proceed without faults

This lazy loading means processes start faster and use only the memory they actually touch—a significant efficiency gain over loading everything upfront.

## Dirty Bit Interaction

When evicting a page, the OS checks the dirty bit:

- **Clean page (dirty = 0)**: Can be discarded if backed by a file, or if already in swap
- **Dirty page (dirty = 1)**: Must be written to swap before eviction

The OS then clears the present bit, repurposes the PFN field for swap location, and the page is now "swapped out."
