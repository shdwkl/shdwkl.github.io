---
layout: post
title: Lock Design Goals - Mutual Exclusion Fairness and Performance
date: 2026-08-28 14:37 +0300

type: mechanism
source: "OSTEP Ch. 28: Locks"
tags:
  - concurrency
  - locks
  - performance
  - fairness
related:
  - "[[Locks - Chapter Overview]]"
  - "[[Critical Sections Require Mutual Exclusion]]"
prerequisites:
  - "[[Critical Sections Require Mutual Exclusion]]"
categories: [books, OSTEP]
---

> Evaluating a lock implementation requires assessing three key criteria: whether it strictly enforces mutual exclusion, whether it treats all waiting threads fairly without starvation, and how much overhead it introduces under both contended and uncontended workloads.
{: .prompt-tip }

# Lock Design Goals - Mutual Exclusion, Fairness, and Performance

When building or choosing a lock implementation, we evaluate it against three primary goals. These criteria help determine the suitability of a lock for a given workload or system.

## 1. Mutual Exclusion
The most fundamental requirement is that the lock actually works. It must prevent multiple threads from entering a critical section simultaneously. If a lock fails this test, it is useless, regardless of its other properties. Verification often involves reasoning about hardware atomic instructions and memory ordering.

## 2. Fairness
Fairness refers to whether each thread contending for the lock gets a fair chance to acquire it once it becomes free.
*   **Starvation:** Ideally, a lock should prevent starvation, where a thread waits indefinitely while others proceed.
*   **FIFO Ordering:** Strong fairness often implies a First-In-First-Out (FIFO) ordering, ensuring that the thread that has waited the longest is served next.
Ticket locks are a prime example of a fair locking mechanism.

## 3. Performance
Performance is the measure of time overhead added by the lock. We analyze this in three scenarios:
*   **No Contention:** A single thread acquiring and releasing the lock. The overhead should be minimal (just a few instructions).
*   **Contention on a Single CPU:** Multiple threads fighting for the lock on one core. Spinning here is wasteful because the holder cannot run while the waiter spins.
*   **Contention on Multiple CPUs:** Threads on different cores fighting for the lock. Here, spinning might be acceptable for short critical sections, but sleeping is better for long waits.

Balancing these goals often involves trade-offs. For example, spin locks are excellent for low-latency access on multicore systems (performance) but can lead to unfairness and high CPU waste if held too long.
