# C# Collections: Complete Reference Guide

**Author:** Perplexity AI Notes  
**Date:** November 30, 2025  
**Topics Covered:** Core Collections + Concurrent Collections + Thread Safety Nuances

---

## Table of Contents

- [1. Core (Non-Concurrent) Collections](#1-core-non-concurrent-collections)
- [2. Concurrent Collections (Thread-Safe)](#2-concurrent-collections-thread-safe)
- [3. ConcurrentDictionary Deep Dive](#3-concurrentdictionary-deep-dive)
- [4. Quick Reference Cheat Sheets](#4-quick-reference-cheat-sheets)

---

## 1. Core (Non-Concurrent) Collections

**Namespace:** `System` + `System.Collections.Generic`  
**Thread Safety:** ❌ **NOT thread-safe** for concurrent writes

### 1.1 Quick Reference Table

| Collection                      | When to Use                    | Key Characteristics                  | Time Complexity                 |
| ------------------------------- | ------------------------------ | ------------------------------------ | ------------------------------- |
| `Array`                         | Fixed size, max performance    | Contiguous memory, O(1) index access | Read: O(1), Insert/Delete: N/A  |
| `List<T>`                       | **Default dynamic list**       | Resizable array, order preserved     | Read: O(1), Insert/Delete: O(n) |
| `Dictionary<TKey,TValue>`       | Fast key lookups               | Hash table, unique keys              | Lookup/Add/Remove: O(1) avg     |
| `HashSet<T>`                    | Unique items, membership tests | Hash table, no duplicates            | Contains/Add/Remove: O(1) avg   |
| `Queue<T>`                      | FIFO processing                | First-In-First-Out                   | Enqueue/Dequeue: O(1)           |
| `Stack<T>`                      | LIFO processing                | Last-In-First-Out                    | Push/Pop: O(1)                  |
| `LinkedList<T>`                 | Frequent middle inserts        | Doubly-linked nodes                  | Insert/Delete (with node): O(1) |
| `SortedList<TKey,TValue>`       | Sorted keys, small data        | Array-backed, binary search          | Lookup: O(log n), Insert: O(n)  |
| `SortedDictionary<TKey,TValue>` | Sorted keys, frequent updates  | Tree-backed                          | Lookup/Insert/Delete: O(log n)  |

### 1.2 Code Examples
