---
layout: post
title: Page Faults Trigger the Swap Mechanism
date: 2026-07-06 08:51 +0300
type: mechanism
source: "OSTEP Ch. 21: Beyond Physical Memory: Mechanisms"
tags:
  - virtualization
  - memory
  - paging
  - swapping
  - exceptions
related:
  - "[[Swap Space Extends Memory Beyond Physical Limits]]"
  - "[[The Present Bit Distinguishes Memory-Resident from Swapped Pages]]"
  - "[[Page Table Entries Contain PFN and Control Bits]]"
  - "[[System Calls Transfer Control via Trap and Return-from-Trap]]"
  - "[[Context Switch Saves and Restores Process State]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Swap Space Extends Memory Beyond Physical Limits]]"
  - "[[Page Table Entries Contain PFN and Control Bits]]"
categories: [books, OSTEP]
---

# Page Faults Trigger the Swap Mechanism

> When a process accesses a page that exists but is not in physical memory (present bit = 0), the hardware raises a page fault. The OS page fault handler locates the page on disk, finds a free frame (possibly evicting another page), reads the page into memory, updates the page table, and restarts the faulting instruction. This mechanism enables transparent demand paging.
{: .prompt-tip }

## The Page Fault Flow

A page fault is not an error—it's the normal mechanism for bringing pages into memory on demand. The complete flow:


1. Process accesses virtual address
2. TLB miss → consult page table
3. PTE has present bit = 0 → hardware raises page fault
4. Trap to OS page fault handler
5. Handler checks: Is this a valid access?
   - Invalid (truly unmapped) → kill process (segfault)
   - Valid (just swapped out) → proceed
6. Find physical frame (may require eviction)
7. Issue disk I/O to read page from swap
8. Block process; schedule another
9. Disk interrupt signals I/O complete
10. Update PTE: set PFN, set present bit = 1
11. Restart faulting instruction


## Hardware vs. OS Responsibilities

The hardware's role is minimal but crucial:

- Detect present bit = 0
- Save faulting address to a special register (CR2 on x86)
- Raise page fault exception
- After OS fixes the fault, re-execute the instruction

The OS handles everything else: locating the page in [[Swap Space Extends Memory Beyond Physical Limits|swap space]], managing disk I/O, updating page tables, and deciding which page to evict if memory is full.

## Why Instructions Must Restart

Consider a `mov` instruction that faults mid-execution. The hardware must be able to restart from the beginning after the page is loaded. This requires:

1. **Precise exceptions**: The processor state at fault time must exactly reflect the state before the instruction
2. **Idempotent restart**: Re-executing the instruction produces correct results

Complex instructions (like x86 string operations) complicate this—they may have partially completed. Hardware architects must ensure clean restart semantics.

## Blocking and Scheduling

Disk I/O takes millions of CPU cycles (~10ms versus ~1ns per cycle). During this time, the faulting process is blocked and the OS schedules another process. This interleaving is why swapping "works"—we pay the disk cost but overlap it with useful computation.

```text
Process A:  [run]----[fault]----------------[continue]
Process B:       [run]--------[run]
                       ^--- disk I/O for A completes
```

## Page Fault Types

- **Major fault**: Page must be read from disk (swap or file)
- **Minor fault**: Page exists in memory but PTE wasn't set (e.g., copy-on-write, shared library already loaded)

Minor faults are much cheaper—no disk I/O required. The OS exploits this through techniques like sharing pages between processes and copy-on-write fork.

## Connection to Demand Paging

Page faults enable **demand paging**: pages are only loaded when first accessed, not when a process starts. This speeds up process creation and reduces memory usage for pages that are never touched. The trade-off is occasional page fault overhead during execution.
