# How to Solve Any Coding Question
```
1. Repeat problem + ask about edge cases (null, empty, duplicates, negative, huge input)
2. Give brute-force idea + its complexity
3. Optimise (HashMap? two pointers? sort? heap? sliding window?)
4. Code cleanly (good names, small methods) and dry-run with a small example
5. State time/space complexity + how you'd test it
```

**Pattern cheat-sheet**
| Signal in question | Technique |
|---|---|
| "pair / complement / seen before" | HashMap / HashSet |
| sorted array, pair, palindrome | two pointers |
| longest/shortest substring/subarray | sliding window |
| matching brackets, undo, nested | stack |
| top K, kth largest | heap (PriorityQueue) |
| sorted / minimise-maximise search | binary search |
| overlapping ranges | sort + merge |
| count/group | HashMap + merge/computeIfAbsent |
| cache with eviction | LinkedHashMap / HashMap + doubly linked list |
| wait/signal between threads | BlockingQueue / Semaphore / wait-notify |

**Practice plan:** 2 problems/day – Mon strings, Tue arrays, Wed collections, Thu streams (write 20 above from memory), Fri threads (odd-even, producer-consumer, LRU), Sat design + SQL, Sun mock interview.
