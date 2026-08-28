---
layout: post
title: Critical Sections Require Mutual Exclusion
date: 2026-08-28 14:35 +0300
type: concept
source: "OSTEP Ch. 26: Concurrency: An Introduction"
tags:
  - concurrency
  - critical-section
  - atomicity
  - mutual-exclusion
related:
  - "[[Threads - Chapter Overview]]"
  - "[[Race Conditions lead to Non-Deterministic Behavior]]"
  - "[[Locks - Chapter Overview]]"
prerequisites:
  - "[[Race Conditions lead to Non-Deterministic Behavior]]"
categories: [books, OSTEP]
---


To fix race conditions, we need to bring order to the chaos of concurrent execution. We do this by identifying the specific parts of the code that access shared resources and ensuring they execute safely.

## Key Definitions

*   **Critical Section**: A piece of code that accesses a shared resource (usually a variable or data structure). In the counter example, the line `cnt = cnt + 1` is the critical section.
*   **Race Condition**: The flaw that occurs when multiple threads enter a critical section at the same time and attempt to update the shared state.
*   **Indeterminate**: A program consisting of one or more race conditions; the output varies from run to run.
*   **Mutual Exclusion**: The property that guarantees that if one thread is executing within the critical section, the others will be prevented from doing so.

## Achieving Atomicity

The goal of mutual exclusion is to make the critical section execute **atomically**. The word atomic comes from "atom" (indivisible). We want the sequence of instructions (load, add, store) to appear as one single, indivisible unit to the rest of the system.

If we can enforce mutual exclusion, then when Thread A enters the critical section, Thread B is blocked from entering until Thread A finishes. The timeline would look like this:

1.  Thread A enters critical section.
2.  Thread A loads, updates, stores (uninterrupted or effectively so).
3.  Thread A exits critical section.
4.  Thread B enters critical section.
5.  Thread B loads, updates, stores.

Even if the OS interrupts Thread A in the middle, the mutual exclusion mechanism ensures that no other thread can peek at or modify the shared state until Thread A is done. This restores determinism to the program.

> A **critical section** is a segment of code that accesses shared resources. To prevent race conditions, we must enforce **mutual exclusion**, ensuring that only one thread can execute in the critical section at a time. This property provides **atomicity**, making a sequence of multiple instructions appear as a single, indivisible operation, thereby restoring deterministic behavior to concurrent programs.
{: .prompt-tip }