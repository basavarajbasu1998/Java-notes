# Deep Collections Internals (JDK 17 / 21) - Interview Prep for a 5-Year Java Developer

> Companion to `Collections.md` (Q&A basics) and `JavaFullstack.md` (one-liners). This file goes underneath them: source-level mechanics, traced examples, **real output from programs compiled and run on JDK 21.0.4**, production failures, 50+ graded interview questions and a cheat sheet.
>
> Conventions: "observed" = printed by a program I actually ran (code is in Section 3). "approximate" = derived from object layout on 64-bit HotSpot with compressed oops; it varies by JVM flags. Where a JDK detail is version-specific it is marked.

## Table of contents

1. 60-second mental model and analogy
2. Deep internals (HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap semantics, ArrayList, LinkedList, ArrayDeque, PriorityQueue, Set wrappers, specialised maps, COW, BlockingQueues, utilities, Comparators, Sequenced)
3. Runnable demos with real output
4. Production war stories
5. Interview questions (Easy / Medium / Hard) with follow-up chains and common wrong answers
6. Complexity and memory tables, decision tree, streams-vs-loops note
7. One-page cheat sheet

---

# 1. The 60-second mental model

**Everything in `java.util` is one of four shapes:**

| Shape | Members | Lookup by value | Lookup by position | Ordered? |
|---|---|---|---|---|
| Resizable array | `ArrayList`, `ArrayDeque`, `PriorityQueue` (heap in array) | O(n) | O(1) | insertion (List), heap order (PQ) |
| Linked nodes | `LinkedList`, `LinkedHashMap` (overlay) | O(n) | O(n) | insertion |
| Hash table | `HashMap`, `HashSet`, `LinkedHashMap`, `ConcurrentHashMap` | O(1) average | n/a | none (or overlay) |
| Balanced tree | `TreeMap`, `TreeSet`, treeified HashMap bins | O(log n) | n/a | sorted |

**Analogy - a hotel with numbered rooms (HashMap).**
The guest's surname is run through a scrambler (`hashCode` + spreading) and the result modulo the number of rooms picks a room (bucket). Two guests can be sent to the same room (collision): they queue in a corridor (linked list). If one corridor gets very long *and* the hotel already has at least 64 rooms, that corridor is re-organised into a sorted search tree; if the hotel is small the manager instead builds a bigger hotel (resize). When the hotel is 75% full the manager doubles the rooms and each guest either stays in their room number or moves exactly `oldRooms` doors down (that is the lo/hi split). If a guest changes surname after check-in (mutable key) the receptionist looks in the wrong room and says "nobody here" - yet the guest is still in the building.

**Analogy - ArrayList = a row of numbered lockers**: jump to locker 7 instantly, but inserting at locker 2 means shifting everybody right. **LinkedList = treasure hunt**: each clue points to the next; you must walk. **PriorityQueue = tournament bracket**: only the champion (root) is guaranteed; the rest is only "partially ordered".

**Five sentences that carry most interviews**

1. HashMap index = `(n-1) & (h ^ (h>>>16))`; `n` is a power of two so the mask is a cheap modulo, and the XOR lets high hash bits influence the low bits that the mask keeps.
2. Resize doubles the table; every bin splits into a *lo* list (same index) and a *hi* list (`index + oldCap`), decided by one bit `hash & oldCap`, order preserved.
3. A bin becomes a red-black tree only when it holds more than 8 nodes **and** the table has at least 64 slots; otherwise HashMap resizes instead.
4. Fail-fast iterators compare `modCount` with a snapshot; they are a bug detector, not a thread-safety feature, and can miss violations (second-to-last element quirk).
5. A collection that is "sorted by compareTo" (TreeMap/TreeSet/PriorityQueue ordering) uses `compare`, not `equals`; if they disagree, Sets and Maps silently drop elements.

---

# 2. Deep internals

## 2.1 HashMap (JDK 17/21 `java.util.HashMap`)

### 2.1.1 Data structures

```java
transient Node<K,V>[] table;      // lazily allocated on first put (length always a power of two)
transient int size;               // number of mappings
transient int modCount;           // structural modifications (for fail-fast)
int threshold;                    // next resize point = capacity * loadFactor  (before table exists: holds initial capacity)
final float loadFactor;           // default 0.75f

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;  final K key;  V value;  Node<K,V> next;
}
// treeified bin:
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {   // Entry extends Node, adds before/after
    TreeNode<K,V> parent, left, right, prev;  boolean red;
}
```

Constants: `DEFAULT_INITIAL_CAPACITY = 16`, `MAXIMUM_CAPACITY = 1<<30`, `TREEIFY_THRESHOLD = 8`, `UNTREEIFY_THRESHOLD = 6`, `MIN_TREEIFY_CAPACITY = 64`.

Points that are often missed:

- `hash` is **stored** in the node, so resize never calls `hashCode()` again and `get` can reject a candidate with an int comparison before calling `equals`.
- `key` and `hash` are `final`; only `value` and `next` change.
- A `TreeNode` still carries the `next` pointer: a tree bin is simultaneously a red-black tree (for lookup) and a doubly-linked list (for iteration and for the cheap untreeify). The root is moved to the front of the list. A tree node is roughly twice the size of a plain node (approximate: ~56-64 bytes vs 32).
- The table is **not allocated in the constructor**. `new HashMap<>(1000)` only stores `threshold = tableSizeFor(1000) = 1024`; the first `put` calls `resize()` which turns that into the real capacity (observed: `new HashMap: table=null`).

### 2.1.2 Hash spreading and index

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
index = (n - 1) & hash;      // n = table.length
```

Why spread? With `n = 16` the mask `0b1111` keeps only the lowest 4 bits. Hash codes that differ only in upper bits (for example `Float` values, or keys like `i * 65536`) would all land in bucket 0. XOR-ing the top 16 bits into the bottom 16 lets them participate. It is a single cheap operation and is a trade-off, not a cure for a bad `hashCode`.

Observed (`D1_Buckets`, section A):
```
h=1           spread=1           idx@16=1  idx@32=1
h=17          spread=17          idx@16=1  idx@32=17
h=33          spread=33          idx@16=1  idx@32=1
h=65537       spread=65536       idx@16=0  idx@32=0      <- 65537 ^ (65537>>>16) = 65537 ^ 1 = 65536
h=2147418113  spread=2147450878  idx@16=14 idx@32=30
```

`null` key: hash is defined as 0, so it lives in bucket 0 (observed: `get(null)=1`). `HashMap` therefore allows one null key and null values. Because `get(k)==null` is ambiguous (missing vs mapped to null), use `containsKey` when nulls are possible (observed: `containsKey(x)=true get(x)=null`). This ambiguity is exactly why `ConcurrentHashMap` bans nulls.

### 2.1.3 Why capacity is a power of two

1. `(n-1) & hash` replaces `hash % n`: one AND instruction instead of a division, and it never returns a negative number (`%` on a negative hash would).
2. With `n` = 2^k, `n-1` is all ones in the low k bits, so every bucket is reachable and every bit of the (spread) hash counts equally.
3. **Resize becomes a one-bit test.** Doubling adds exactly one more mask bit (`oldCap`), so a node either keeps its index or moves to `index + oldCap`.
4. Cost: a non-power-of-two capacity request is rounded **up** (`tableSizeFor`), so `new HashMap<>(1000)` gives 1024 slots and `new HashMap<>(17)` gives 32.

```java
static final int tableSizeFor(int cap) {           // smallest power of two >= cap, min 1, max 1<<30
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
// tableSizeFor(17): cap-1 = 16 = 0b10000 -> nlz=27 -> -1>>>27 = 0b11111 = 31 -> 32
// tableSizeFor(16): cap-1 = 15 = 0b01111 -> nlz=28 -> -1>>>28 = 15 -> 16
// tableSizeFor(1000) -> 1024
```

### 2.1.4 `putVal` step by step

```
put(key, value) -> putVal(hash(key), key, value, onlyIfAbsent=false, evict=true)

 1. table == null or length == 0 ?          -> resize()   (allocates: default 16, or the stored initial capacity)
 2. i = (n-1) & hash;  p = tab[i]
 3. p == null ?                              -> tab[i] = newNode(hash,key,value,null)                 [done: goto 6]
 4. else look for an existing key:
      a. p.hash == hash && (p.key == key || key != null && key.equals(p.key))   -> e = p   (first node matches)
      b. p is a TreeNode                     -> e = putTreeVal(...)   (tree descent; may create node)
      c. else walk the chain: binCount = 0, 1, 2, ...
            - if a node matches               -> e = that node; stop
            - if reached tail (next == null)  -> tail.next = newNode(...)     (TAIL insertion; JDK 7 inserted at HEAD)
                                                 if binCount >= TREEIFY_THRESHOLD - 1 (=7): treeifyBin(tab, hash)
                                                 stop
 5. e != null (key existed)?  -> oldValue = e.value; if (!onlyIfAbsent || oldValue == null) e.value = value;
                                 afterNodeAccess(e);           // LinkedHashMap hook (access order)
                                 return oldValue;               // NOTE: no ++modCount, no size change
 6. ++modCount;
    if (++size > threshold) resize();      // strictly GREATER: 12th entry of a 16-table does not resize, the 13th does
    afterNodeInsertion(evict);             // LinkedHashMap hook (removeEldestEntry)
    return null;
```

Details worth quoting:

- Equality test order is `hash == hash` (int), then `==`, then `equals`. `equals` is only called when the full 32-bit spread hash already matches.
- The chain walk with `binCount >= 7` means: the new node was appended as the **9th** node (7 = index of the 8th existing node's iteration). So "treeify at 8" really means "when a bin that already has 8 nodes receives a 9th". Observed below: chain length 8 stays a list, the 9th insertion triggers `treeifyBin`.
- `treeifyBin` first checks `tab.length < MIN_TREEIFY_CAPACITY (64)`; if true it just calls `resize()` and returns. Otherwise it converts nodes to `TreeNode`s and builds the red-black tree (`treeify`).
- Overwriting an existing key does not touch `modCount`; so `map.put(existingKey, v)` inside a for-each over `entrySet()` does **not** throw CME. `map.put(newKey, v)` does.
- After `treeifyBin` resizes, the trailing `++size > threshold` may resize again on the same `put`.

### 2.1.5 Fully traced example (small capacity, real dump)

Keys are `Integer`, so `hash == value` for small ints (`h>>>16 == 0`). `new HashMap<>(4)` -> `tableSizeFor(4)=4`, threshold = 4 stored (table not yet allocated).

```
Step 0  new HashMap<>(4)                 table = null, threshold field = 4 (initial capacity)

Step 1  put(5)   first put -> resize(): oldThr=4>0 so newCap=4, newThr=4*0.75=3
                 idx = 5 & 3 = 1          -> bucket empty, insert
                 size=1
        [0] .   [1] 5   [2] .   [3] .

Step 2  put(9)   idx = 9 & 3 = 1          -> collision with 5, walk chain, no equal, append (tail)
                 size=2
        [1] 5 -> 9

Step 3  put(1)   idx = 1 & 3 = 1          -> chain now 5 -> 9 -> 1
                 size=3       (3 > 3 ? no, no resize yet)

Step 4  put(2)   idx = 2 & 3 = 2          -> insert; size=4 > threshold 3 -> resize() to 8, threshold 6
                 SPLIT of old bucket 1 (chain 5 -> 9 -> 1), oldCap = 4, test (hash & 4):
                      5 & 4 = 4 != 0  -> hi list
                      9 & 4 = 0       -> lo list
                      1 & 4 = 0       -> lo list
                 lo list (9 -> 1) stays at index 1;  hi list (5) goes to 1 + 4 = 5    (relative order preserved)
        [1] 9 -> 1     [2] 2     [5] 5

Step 5  put(13)  idx = 13 & 7 = 5         -> collides with 5:  [5] 5 -> 13     size=5
Step 6  put(9)   already present         -> value overwritten, returns old value, size stays 5, modCount unchanged
Step 7  put(6)   idx = 6 & 7 = 6          -> size=6 (6 > 6 ? no)
Step 8  put(17)  idx = 17 & 7 = 1         -> chain 9 -> 1 -> 17, size=7 > 6 -> resize to 16, threshold 12
                 split of bucket 1, oldCap=8:  9&8=8 -> hi;  1&8=0 -> lo;  17&8=0 -> lo
                 the same resize also splits bucket 5 (5 -> 13): 5&8=0 -> lo stays at 5;  13&8=8 -> hi moves to 5+8 = 13
        [1] 1 -> 17    [5] 5    [9] 9    [13] 13
```

Real dump produced by `D8_Dump` (reflection used only to print the table):

```
put(2) idx@4=2   size=4 threshold=6 table.length=8
   [1] 9 -> 1
   [2] 2
   [5] 5
put(9) again (update, no structural change)   size=5 threshold=6 table.length=8
   [1] 9 -> 1     [2] 2     [5] 5 -> 13
put(17)   size=7 threshold=12 table.length=16
   [1] 1 -> 17    [2] 2     [5] 5    [6] 6    [9] 9    [13] 13
```

(The dump above is abbreviated to non-empty buckets.)

**Iteration-order consequence (observed, `D1_Buckets` section B).** Keys `33,17,1,2,18,3..9` in a default map (cap 16). Bucket 1 holds `33 -> 17 -> 1`.
```
after 12 puts (threshold 12): table.length=16   iteration: [33, 17, 1, 2, 18, 3, 4, 5, 6, 7, 8, 9]
after 13th put -> resize:      table.length=32   iteration: [33, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 17, 18]
```
33 and 1 both have bit `16` clear -> stay at index 1 (lo, order kept: 33 then 1); 17 and 18 have bit 16 set -> move to index 17 and 18. The iteration order changed just because a resize happened. Never depend on it.

### 2.1.6 Treeify, and treeify's precondition (observed)

Keys `i*1024` (i = 1..12) all map to index 0 for capacities up to 1024.

```
put #1..#8  table.length=16  maxListChain grows 1..8        <- a chain of 8 is still a list
put #9      table.length=32  treeBins=0  maxListChain=9     <- 9th node: treeifyBin() -> table < 64 -> resize instead
put #10     table.length=64  treeBins=0  maxListChain=10    <- still a list, but table just reached 64
put #11     table.length=64  treeBins=1  maxListChain=0     <- now treeifyBin() really treeifies
put #12     table.length=64  treeBins=1
after removing keys down to 3 nodes: treeBins=0 maxListChain=3     <- untreeified
```

Untreeify rules:
- **On resize (`TreeNode.split`)**: each of the lo/hi halves with `<= 6` nodes becomes a plain list again; a half that stays large and whose sibling half is non-empty is re-treeified.
- **On remove (`removeTreeNode`)**: the tree is converted back to a list when the tree is structurally tiny (root null, or root.right null, or root.left null, or root.left.left null) - a shape heuristic, not exactly "6". That is why the observed untreeify happened somewhere between 12 and 3 nodes, not at a clean 6.
- The hysteresis (treeify at 9th node, untreeify at 6) prevents a bin from flapping when a key is repeatedly added and removed around the threshold.

### 2.1.7 Tree bins and `Comparable` (why a bad `hashCode` hurts unequal keys)

Inside a tree bin nodes are ordered by: (1) `hash`; (2) if the hashes are equal and the keys' class implements `Comparable<ThatClass>` (checked via `comparableClassFor`), by `compareTo`; (3) otherwise a tie-break by class name then `System.identityHashCode` - which orders nodes consistently for insertion but is **useless for lookup**.

`find()` in a tree: if hashes differ, go left/right by hash. If the hash equals: try `equals`; else if keys are comparable go by `compareTo`; **else recurse into both subtrees** (`right.find(...)`, then continue left). With all hashes identical and non-comparable keys that degrades to a linear scan, O(n) - even though the bin is "a tree".

Observed (`D2_BadHash`, all keys `hashCode() == 42`; put n keys then get them all):
```
n=2000   Good=   0 ms   BadComparable=  12 ms   BadNotComparable=  153 ms
n=5000   Good=   2 ms   BadComparable=   5 ms   BadNotComparable=  246 ms
n=10000  Good=   0 ms   BadComparable=   4 ms   BadNotComparable= 1196 ms
(2nd pass) n=10000 Good= 0 ms   BadComparable= 6 ms   BadNotComparable= 835 ms
```
Bad-but-Comparable keys degrade only to O(log n); bad-and-not-Comparable keys degrade to O(n) per operation (quadratic overall). This also motivates why `String` and `Integer` (Comparable) keys are resilient to hash-flooding attacks after JDK 8. (Timings are single runs on a laptop: trust the ratio, not the absolute numbers.)

### 2.1.8 `get` and `remove`

```java
final Node<K,V> getNode(Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & (hash = hash(key))]) != null) {
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;                                 // fast path: first node
        if ((e = first.next) != null) {
            if (first instanceof TreeNode) return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do { if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
`remove` locates the node the same way, then unlinks it (predecessor.next = node.next, or bucket head replaced, or `removeTreeNode`), then `++modCount; --size; afterNodeRemoval(node)`. **The table never shrinks**: after removing 99% of a million entries, the table still has 2^21 slots and iteration still costs O(capacity). (`clear()` nulls the slots but keeps the length.)

### 2.1.9 Load factor and the Poisson reasoning

The JDK source comment states: under random hash codes, bin sizes follow a Poisson distribution with parameter about **0.5** for the default 0.75 resize threshold (the average load between just-after-resize 0.375 and just-before-resize 0.75 is around 0.5, with big variance). Probability of a bin having exactly k nodes:

```
k=0  0.60653066     k=4  0.00157952     k=8  0.00000006
k=1  0.30326533     k=5  0.00015795     more: < 1 in ten million
k=2  0.07581633     k=6  0.00001316
k=3  0.01263606     k=7  0.00000094
```
So a chain of 8 with a decent hashCode is a "one in 17 million bins" event: the tree is a **defence against bad hashCodes and attacks**, not something normal maps use.

Trade-off: load factor `a` -> expected node per bin ~a. Lower `a` (0.5): more memory, fewer collisions, more resizes. Higher (1.0): about 25% less table memory, longer chains and more misses. 0.75 is a tuned compromise; changing it is rarely justified. Constructor `HashMap(int, float)` rejects non-positive/NaN loadFactor; loadFactor > 1 is allowed (chains become the norm).

### 2.1.10 Resize cost and pre-sizing

`resize()` allocates `2 * oldCap` (or the initial capacity on first call), moves all nodes (no rehash, no `hashCode()` call - uses stored `hash`), and processes each old bin with the lo/hi split:

```java
// per old bin, plain list case
do {
    next = e.next;
    if ((e.hash & oldCap) == 0) { append e to lo list }     // index unchanged
    else                        { append e to hi list }     // index + oldCap
} while ((e = next) != null);
newTab[j] = loHead;  newTab[j + oldCap] = hiHead;           // order inside each list preserved
```
Cost: O(n) work plus a new array allocation of `2n` references; in a big map that array is a large object (with 2^22 slots and compressed oops, 16 MB) - expect a GC-visible spike and cache misses. Growth is geometric, so amortised O(1) per put, but a single put can take milliseconds.

Observed (`D7_Resizes`, 1,000,000 Integer puts into a default map):
```
default: 16@1 32@13 64@25 128@49 256@97 512@193 1024@385 2048@769 4096@1537 8192@3073 16384@6145
         32768@12289 65536@24577 131072@49153 262144@98305 524288@196609 1048576@393217 2097152@786433
   resizes after first allocation=17   nodes moved during resizes ~ 1,572,852 for N=1,000,000
newHashMap(N): 2097152@1                (one allocation, zero resizes)
```
Note the entries in the format `length@sizeWhenAllocated`. The default map resized 17 times and moved ~1.57 N nodes; the pre-sized one none.

**Pre-sizing formula.** To hold `n` entries without a resize you need `capacity * 0.75 >= n`, i.e. `capacity >= n / 0.75`. Then HashMap rounds up to a power of two.
- Classic: `new HashMap<>((int) (expected / 0.75f) + 1)` (the `+1` covers truncation; e.g. 768 needs capacity 1024 and would not resize, but this formula gives 1025 -> 2048 and wastes memory: use `ceil`).
- **JDK 19+**: `HashMap.newHashMap(int numMappings)` (also `LinkedHashMap.newLinkedHashMap`, `HashSet.newHashSet`, `LinkedHashSet.newLinkedHashSet`, `WeakHashMap.newWeakHashMap`) does `ceil(n / 0.75)` for you. It fixes the common bug `new HashMap<>(n)`, which does *not* mean "room for n entries".

Observed:
```
new HashMap<>(1000): 1000 keys inserted -> table 2048   (1024 slots, threshold 768, so the 769th put resized)
new HashMap<>(750),  750 keys           -> table 1024   (threshold 768 >= 750: no resize)
new HashMap<>(768),  768 keys           -> table 1024
new HashMap<>(769),  769 keys           -> table 2048   (tableSizeFor(769)=1024, threshold 768 < 769 -> resize)
HashMap.newHashMap(1000)                -> table 2048   (ceil(1000/0.75)=1334 -> 2048, allocated up front)
```
Trade-off: pre-sizing for 1,000,000 entries allocates a 2^21 table (about 8 MB of references) immediately - right when you know you will fill it, wasteful if you may not.

The copy constructor `new HashMap<>(otherMap)` and `putAll` already pre-size using `s / loadFactor + 1`.

### 2.1.11 hashCode/equals contract, mutable keys, iteration

Contract: (1) consistent while the fields used do not change; (2) `a.equals(b)` implies `a.hashCode() == b.hashCode()`; (3) unequal objects *may* share hash codes. Breaking (2) makes a logically-equal key hash to a different bucket: `get` misses. Breaking equals reflexivity/symmetry/transitivity makes `contains` non-deterministic. Also `equals(Object)` must accept any Object (not overload `equals(MyType)`).

Standard hash codes worth knowing: `Integer` = value; `Long` = `(int)(v ^ (v>>>32))`; `String` = `s[0]*31^(n-1) + ... + s[n-1]` (cached in the `hash` field; `"Aa"` and `"BB"` both give 2112, observed); `Boolean` 1231/1237; `Objects.hash(a,b,c)` = `Arrays.hashCode` with 31-multiplier starting at 1 (allocates a varargs array); `List.hashCode` is defined by the List contract; arrays use identity hash (never use arrays as keys); `Double` uses the bits, so `0.0` and `-0.0` are different keys and `NaN` equals `NaN`; records derive `hashCode/equals` from all components (the exact combination is unspecified).

Mutable key, observed (`D4_Maps` section B):
```
after k.id=2: get(k)=null containsKey=false size=1 iteration still sees it: {K2=value}
after restoring id=1: get(k)=value
new K1 key put: size=1     (a new K1 key equal to the restored one would have collided with the entry -> replaced)
```
The entry sits in the bucket computed from the *old* hash and its cached `hash` field is the old one; lookups compute a new index and compare `e.hash == hash` first, so it is unreachable. Not removable either (`remove` also misses). It is a leak until you restore the state or `clear()`.

Iteration: table order, bucket by bucket, chain order within a bucket; cost O(capacity + size). Not specified, can change between JDK versions, between resizes (observed above) and, for objects with identity `hashCode`, between runs. Note `Set.of` / `Map.of` (immutable, JDK 9+) deliberately randomise iteration order per JVM instance so nobody relies on it.

Multi-value: `computeIfAbsent(k, x -> new ArrayList<>()).add(v)`; the mapping function must not modify the map (JDK 9+ HashMap throws `ConcurrentModificationException` if it detects that).

---

## 2.2 LinkedHashMap

`LinkedHashMap<K,V> extends HashMap<K,V>`; its `Entry extends HashMap.Node` adds `before`/`after` and the map keeps `head`, `tail`, and a boolean `accessOrder`. Buckets still work exactly as in HashMap; a doubly-linked list is threaded through all entries in the order you chose.

```
table:  [0] . [1] E1 -> E5 [2] . [3] E3 ...            (hash structure, as HashMap)
list :  head -> E3 <-> E1 <-> E5 <-> ... <- tail       (before/after pointers = iteration order)
```
HashMap defines empty hook methods called from `putVal/get/remove` which LinkedHashMap overrides:
- `newNode` -> creates the entry and links it at the tail.
- `afterNodeAccess(e)` -> if `accessOrder`, unlink `e` and re-link at tail; `++modCount`.
- `afterNodeInsertion(evict)` -> if `evict` and the list is non-empty and `removeEldestEntry(head)` returns true, remove the head.
- `afterNodeRemoval(e)` -> unlink.

Consequences:
- **Insertion order**: re-putting an existing key does *not* reorder (observed `{z=3, y=2}`).
- **Access order** (`new LinkedHashMap<>(16, 0.75f, true)`): `get`, `getOrDefault`, `put`-on-existing, `putIfAbsent`-on-existing, `compute*`, `merge` move the entry to the tail. Observed: `getOrDefault(c)` and `putIfAbsent(d)` both reordered. So `get` is a *structural* modification: iterating an access-ordered map while calling `get` throws CME.
- Iteration is O(size) not O(capacity) (walks the linked list) - one reason to prefer it when the map is sparse after deletions.
- LRU = access order + `removeEldestEntry` returning `size() > max`; it is called after every insertion (not after replacing an existing key's value).
- Memory: entry has 2 extra references (about 40 B vs 32 B per entry, approximate).
- Not thread-safe; LRU under concurrency needs `Collections.synchronizedMap(...)` (whole map lock; even `get` mutates) or a purpose-built cache (Caffeine).
- `Sequenced` in Java 21: `firstEntry/lastEntry/pollFirstEntry/putFirst/putLast/reversed/sequencedKeySet` (see 2.13).

---

## 2.3 TreeMap / TreeSet (red-black tree)

```java
static final class Entry<K,V> { K key; V value; Entry<K,V> left, right, parent; boolean color = BLACK; }
private final Comparator<? super K> comparator;    // null => natural ordering (Comparable)
private transient Entry<K,V> root;
```
**Red-black properties** (they guarantee height <= 2 * log2(n+1)):
1. Every node is red or black.  2. The root is black.  3. Null leaves count as black.
4. A red node has only black children (no two reds in a row).
5. Every path from a node to its descendant null leaves has the same number of black nodes ("black height").

```
          20(B)
         /     \
      10(R)     30(B)         Black-height from the root to any null = 2. Longest path <= 2 * shortest path.
     /    \        \
  5(B)   15(B)     40(R)
```
Operations (all O(log n)): `put` does an ordinary BST insert of a **red** node then `fixAfterInsertion` walks upward: if the uncle is red -> recolour parent/uncle/grandparent and continue upward; if the uncle is black -> 1 or 2 rotations and stop. Delete has more cases (at most 3 rotations). Insertion needs at most 2 rotations in total, which is why RB trees have cheaper writes than AVL trees (AVL is more strictly balanced -> faster reads).

Rotation (conceptual - it preserves in-order sequence, changes only shape/height):
```
 rotateLeft(x):            x                 y
                          / \      ==>      / \
                         a   y             x   c
                            / \           / \
                           b   c         a   b
```
Rules and gotchas:
- **Ordering uses `compare`/`compareTo` only.** `equals`/`hashCode` are never called. So if `compare(a,b)==0`, TreeMap treats them as the same key: observed `TreeSet(String.CASE_INSENSITIVE_ORDER)` of `Java, JAVA, java` -> `[Java]`; `BigDecimal("1.0")` vs `("1.00")`: `equals=false`, `compareTo=0`, `HashSet size=2`, `TreeSet size=1`. The `Map`/`Set` contract is defined via `equals`; a comparator inconsistent with equals makes the collection work "as designed" yet violate the interface docs. Always ask "is my comparator consistent with equals?".
- `put(null, v)` -> NPE with natural ordering (observed). The first `put` on an empty map still calls `compare(key, key)` as a type/null check, so a non-Comparable key fails with `ClassCastException` on the **first** insert.
- `firstEntry()/floorEntry()/...` return immutable snapshots (`setValue` throws UOE); `subMap/headMap/tailMap/descendingMap` are **live views**.
- `TreeSet` = `TreeMap` with a dummy value; `TreeSet.first()` = `firstKey()`.
- Cannot use `TreeMap` for keys whose order can change while stored (same mutable-key problem, different symptom: corrupt search path).

`NavigableMap` cheat table (observed, keys 10,20,30,40,50):

| Call | Meaning | Result |
|---|---|---|
| `floorKey(25)` | greatest key <= 25 | 20 |
| `ceilingKey(25)` | least key >= 25 | 30 |
| `lowerKey(20)` | greatest key < 20 (strict) | 10 |
| `higherKey(20)` | least key > 20 (strict) | 30 |
| `floorKey(5)` | none exists | `null` |
| `subMap(20, 40)` | `[20, 40)` half-open | `{20, 30}` |
| `subMap(20,true,40,true)` | closed range | `{20, 30, 40}` |
| `headMap(30)` / `tailMap(30)` | `< 30` / `>= 30` | `{10,20}` / `{30,40,50}` |
| `descendingMap()` | reverse view | `{50..10}` |
| `pollLastEntry()` | remove and return max | `50=v50` |

Views are live: after `sm = t.subMap(20,40); t.put(25,..)`, `sm` shows 25 (observed); `sm.put(99, ..)` -> `IllegalArgumentException: key out of range`.

Use cases: range queries ("all orders between T1 and T2"), "nearest lower/higher", leaderboards, time-window indexes, interval scheduling (floorKey on start times), order books.

---

## 2.4 ConcurrentHashMap - API semantics vs HashMap (internals covered elsewhere)

| Aspect | HashMap | Hashtable | `Collections.synchronizedMap` | ConcurrentHashMap |
|---|---|---|---|---|
| Null key / value | 1 null key / null values | NPE | delegates to wrapped map | **NPE both** (observed) |
| Individual ops thread-safe | no | yes (`synchronized` methods) | yes (one mutex) | yes |
| Compound ops (`get` then `put`) | racy | racy | racy | racy - unless you use its atomic methods |
| `putIfAbsent/compute/computeIfAbsent/computeIfPresent/merge/replace(k,old,new)/remove(k,v)` | not atomic under threads | atomic (each is `synchronized`, whole-table lock) | atomic (wrapper holds the mutex) | **atomic per key**, only that bin is locked |
| Iterator | fail-fast (CME) | `Enumeration` not fail-fast; collection views fail-fast | **must lock the map manually** while iterating | weakly consistent, never CME |
| `size()` | exact | exact | exact | estimate under concurrent updates; `mappingCount()` returns `long` |
| Extra API | | | | `forEach/search/reduce(parallelismThreshold, ...)`, `newKeySet()`, `keySet(defaultValue)` |

Semantics to state precisely:
- `compute*`/`merge` run the lambda while holding the bin's lock: keep them **short, side-effect free, and never touch the same map** inside (deadlock or `IllegalStateException: Recursive update` in JDK 9+). Return `null` from `compute/merge` to remove the key.
- Weakly consistent iterator: traverses elements that existed when it was created, may or may not reflect later changes, **never** throws CME and never returns an element twice. Observed: adding while iterating the keySet did not throw (number of iterations varies, do not depend on it).
- `size()` may be stale the moment it returns - do not use it as a synchronisation condition. `isEmpty()` likewise.
- Individually atomic methods do not make **your** multi-step logic atomic: observed with 8 threads x 20,000 increments (expected 160000):

```
synchronizedMap(getOrDefault + put) = 33794
ConcurrentHashMap(getOrDefault + put) = 121071
ConcurrentHashMap.merge(k, 1, Integer::sum) = 160000
```
Both "thread-safe" maps lost updates; only the atomic `merge` was right. (Numbers vary per run except the last.)
- Counters: `ConcurrentHashMap<K, LongAdder>` with `computeIfAbsent(k, x -> new LongAdder()).increment()` under heavy contention.
- Common wrong choice: `Hashtable` (whole-object lock) or `synchronizedMap` when a single hot map is read by many threads.

---

## 2.5 ArrayList

```java
transient Object[] elementData;           // non-private for nested-class access
private int size;
private static final int DEFAULT_CAPACITY = 10;
private static final Object[] EMPTY_ELEMENTDATA = {};                   // new ArrayList(0)
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};   // new ArrayList()  (distinct so we know to use 10)
protected transient int modCount = 0;     // inherited from AbstractList
```

**Lazy allocation**: `new ArrayList<>()` points at the shared empty array (observed `cap=0`); the first `add` allocates 10.

**Growth**: `grow(minCapacity)` computes
```java
newCapacity = ArraysSupport.newLength(oldCapacity,
                                      minCapacity - oldCapacity,   // minimum growth
                                      oldCapacity >> 1);           // preferred growth (50%)
// newLength = oldLength + max(minGrowth, prefGrowth), guarded against overflow (SOFT_MAX_ARRAY_LENGTH = Integer.MAX_VALUE - 8)
```
Observed sequence of capacities: **10 -> 15 -> 22 -> 33 -> 49 -> 73 -> 109** (growth triggered at sizes 1, 11, 16, 23, 34, 50, 74). Integer division makes it "about 1.5x" (15 + 7 = 22, not 22.5). Corner cases (observed): `new ArrayList<>(0)` then one `add` -> cap 1 (0 + max(1, 0)); `new ArrayList<>(List.of(1,2,3))` -> cap exactly 3 (no slack). If `minCapacity` is larger than 1.5x (e.g. `addAll` of a large collection) it grows straight to what is needed. Once past the soft maximum the request is "hugeLength": OOM-style failure ("Required array length ... is too large").

Amortised O(1) `add`: geometric growth => total copying <= about 3n for n adds (sum of a geometric series with ratio 1.5). Pre-size with `new ArrayList<>(n)` or `ensureCapacity(n)` when known; `trimToSize()` releases slack (rarely needed).

Key operations:
- `get(i)` -> `Objects.checkIndex` + array read.
- `add(e)` -> `modCount++`, write at `size++`, grow if full.
- `add(i, e)` / `remove(i)` -> `System.arraycopy` shift (a fast native memmove), O(n - i). `remove` nulls the vacated tail slot so it can be GC'd.
- `set(i, e)` does **not** increment `modCount` (non-structural).
- `contains/indexOf` -> linear scan with `equals`.
- `removeIf(pred)` -> single pass that marks survivors then compacts: O(n). A hand-written loop calling `remove(i)` in each iteration is O(n^2).
- Not thread-safe: concurrent `add` can lose elements or throw `ArrayIndexOutOfBoundsException`.

### 2.5.1 modCount and fail-fast mechanics (traced)

```java
private class Itr implements Iterator<E> {
    int cursor;                       // index of next element to return
    int lastRet = -1;                 // index of last returned, -1 if none
    int expectedModCount = modCount;  // snapshot at iterator creation

    public boolean hasNext() { return cursor != size; }          // NOTE: != , not <
    public E next() {
        checkForComodification();                                // modCount != expectedModCount -> throw CME
        int i = cursor;
        if (i >= size) throw new NoSuchElementException();
        ...  cursor = i + 1;  return (E) elementData[lastRet = i];
    }
    public void remove() { ...; ArrayList.this.remove(lastRet); cursor = lastRet; lastRet = -1;
                           expectedModCount = modCount; }        // iterator re-syncs its own expected value
}
```
The for-each loop is `Iterator it = list.iterator(); while (it.hasNext()) { x = it.next(); ... }`. `hasNext()` does **not** check `modCount`; only `next()` does. Combined with `cursor != size`, this produces the famous quirk.

Trace on `[a, b, c, d]`, removing inside the for-each via `list.remove(obj)`:

```
Case 1: remove "a" (first)
  next()->a  cursor=1 ; list.remove("a") -> size=3, modCount++ (modCount != expected)
  hasNext: cursor(1) != size(3) -> true ; next() -> checkForComodification -> CME                 THROWS

Case 2: remove "c" (second-to-last)
  next()->a (cursor=1), next()->b (2), next()->c (cursor=3) ; list.remove("c") -> size=3
  hasNext: cursor(3) != size(3) -> FALSE -> loop ends. next() never called -> NO exception.       SILENT, and "d" was never visited!

Case 3: remove "d" (last)
  ... next()->d (cursor=4) ; remove -> size=3
  hasNext: cursor(4) != size(3) -> TRUE ; next() -> CME                                           THROWS

Case 4: [a,b,c] remove "b" (again second-to-last)
  next()->b (cursor=2); remove -> size=2 ; hasNext: 2 != 2 false -> exits silently                 SILENT
```
Real output (`D3_Lists` section B):
```
a -> CME! list=[b, c, d]
a b c -> finished normally, list=[a, b, d]       <- second-to-last: no CME, 'd' skipped
a b c d -> CME! list=[a, b, c]
a b -> finished normally, list=[a, c]
```
Lessons: (1) "It throws CME when you remove during for-each" is false in general; (2) the guarantee is best effort - never write code that relies on catching CME; (3) correct removal: `Iterator.remove()`, `removeIf`, iterate backwards by index, or collect-then-`removeAll`. `HashMap` iterators check in `nextNode()` (and `hasNext()` is `next != null`), so the quirk is specific to the ArrayList `cursor != size` test (LinkedList and others have their own similar edge cases).

### 2.5.2 Traps

- **`remove(int)` vs `remove(Object)`** on `List<Integer>`: `list.remove(1)` removes the element at **index 1**; `list.remove(Integer.valueOf(1))` removes the value 1. Observed on `[10,20,30,1,2]`: `remove(1)` -> `[10,30,1,2]`, then `remove(Integer.valueOf(1))` -> `[10,30,2]`.
- **`subList(from, to)`** is a **view** (offset + parent reference) - `set` writes through, `clear()` deletes that range from the parent (`list.subList(a,b).clear()` is the idiomatic range-delete, O(n) not O(k*n)). Any *structural* modification of the parent after creating the sublist makes later sublist use throw CME (observed). Do not keep sublists around; copy with `new ArrayList<>(list.subList(a,b))`. A tiny sublist retains the whole parent.
- **Which "unmodifiable list" is it?** (observed, `D3_Lists` section E)

| Factory | Mutability | Nulls | Relation to source | Notes |
|---|---|---|---|---|
| `Arrays.asList(arr)` | fixed-size: `set` ok, `add/remove` -> `UnsupportedOperationException` | allowed | **write-through** to the array (`asl.set(0,"X")` -> `arr[0]=="X"`) | primitive array becomes `List<int[]>` (size 1) |
| `List.of(...)` (9+) | immutable | **NPE** on null element, and `contains(null)` **also throws NPE** | independent | compact special classes for 0-2 elements; keeps element order |
| `List.copyOf(coll)` (10+) | immutable | NPE on null | **independent snapshot** (may return the same instance if the source is already an immutable list) | |
| `Collections.unmodifiableList(l)` | read-only **view** | allowed | **live**: later changes to `l` are visible (observed `[p, q, r]`) | wrapper only blocks writes through the view |
| `stream.toList()` (16+) | unmodifiable | allowed | snapshot | `Collectors.toList()` gives an unspecified mutable list |
| `new ArrayList<>(coll)` | mutable | allowed | independent copy | shallow: elements shared |

- `ArrayList.contains(null)` is fine (observed `false`), unlike `List.of`.
- `Collections.emptyList()`, `singletonList`, `nCopies` are immutable specials.
- `ArrayList<int>` does not exist; boxed `Integer` costs ~16 bytes per element plus the 4-byte slot - for big numeric data use primitive arrays or a primitive-collections library.
- Sorting: `list.sort(cmp)` on ArrayList sorts the backing array directly (`Arrays.sort`), and bumps `modCount`.

---

## 2.6 LinkedList and ArrayDeque

### LinkedList
```java
private static class Node<E> { E item; Node<E> next; Node<E> prev; }   // 24 bytes (approximate, compressed oops)
transient Node<E> first, last;  transient int size;
```
`LinkedList implements List<E>, Deque<E>`, allows `null`, `get(i)` walks from the nearer end (`i < size>>1` from head else tail) so it is O(n) but at most n/2 hops.

```
 first -> [prev=null|A|next] <-> [B] <-> [C] <-> [D|next=null] <- last
```
Why it is usually slower than ArrayList - even where big-O favours it:
1. **Finding** the position is O(n) pointer chasing with cache misses; the O(1) splice is the cheap part. Observed (`D5_Misc` K, 2,000 inserts at the middle *by index*):
```
n=20000  : ArrayList=2658 us   LinkedList=30415 us
n=100000 : ArrayList=7818 us   LinkedList=175930 us      (second pass: 8305 vs 139758 us)
```
`ArrayList` wins because `System.arraycopy` moves contiguous memory at memory-bandwidth speed and the CPU prefetcher loves it.
2. Memory: ~24 B node + the element per item vs 4 B slot per element (approximate).
3. Poor GC behaviour: millions of small objects.

When LinkedList *can* win: many insertions/removals **at a `ListIterator` position** in the middle of a huge list (no re-search), or `removeFirst` on huge queues - but `ArrayDeque` is better for the second. In practice: "never" unless you need null elements in a Deque or splice-heavy iterator work.

### ArrayDeque
Resizable circular array with `head` and `tail` indices: `addFirst` decrements head (wrapping), `addLast` writes at tail and increments (wrapping). No shifting. The slot at `tail` is always empty (that is how full vs empty is told apart); growth allocates a larger array and copies the two segments (about doubling when small, 1.5x when large - conceptually, JDK 9+ rewrite).
```
capacity 8, head=6, tail=3 (wrapped):   idx: 0 1 2 3 4 5 6 7
                                            [c d e . . . a b]   logical order: a b c d e
                                                     ^tail(next free)  ^head
```
- **No nulls** (null is the "empty slot" marker and `poll()` returns null for empty): `add(null)` -> NPE (observed). `LinkedList` allows null.
- Preferred `Stack` replacement (`push/pop/peek` at the head) and queue (`offer/poll` at tail/head): faster and denser than `LinkedList`, no synchronisation unlike `Stack`/`Vector` (legacy, synchronised, index-based `Stack` misuse).
- Observed: `push(1),push(2),push(3); pop()->3; peek()->2; deque=[2, 1]`; `offer 1,2,3; poll()->1`. `pop()` on empty throws `NoSuchElementException`, `poll()` returns `null`. Method families: throwing (`add/remove/element/addFirst/removeFirst/getFirst`) vs special-value (`offer/poll/peek`).
- Not thread-safe; iterator is fail-fast (best effort). For concurrency: `ConcurrentLinkedDeque`, `LinkedBlockingDeque`.

---

## 2.7 PriorityQueue (binary min-heap in an array)

```java
transient Object[] queue;   // default initial capacity 11; heap stored level by level
private final Comparator<? super E> comparator;   // null => natural ordering
```
For node at index `k`: parent = `(k-1) >>> 1`, left child = `2k+1`, right child = `2k+2`. Heap property: `queue[parent] <= queue[child]` (min-heap; for a max-heap pass `Comparator.reverseOrder()`). It is a **complete binary tree**, so the array has no holes.

```
array [1, 3, 2, 5, 9, 8]            1
                                  /   \
                                 3     2
                                / \   /
                               5   9 8
```
Operations:
- **offer(e)**: put at index `size`, `siftUp`: while `e < parent` swap upward. O(log n).
- **poll()**: take `queue[0]`; move the **last** element to the root; `siftDown`: swap with the smaller child until heap-ordered. O(log n).
- **peek()** O(1). **remove(Object)**/`contains` O(n) (linear search) + O(log n) re-heapify. **size** O(1).
- `new PriorityQueue<>(collection)` runs **heapify** (siftDown from `n/2-1` to 0): O(n), cheaper than n offers.
- Growth: doubles while small (`+2` when `< 64`), then 50% (`oldCap >> 1`). No nulls. Elements must be mutually comparable else `ClassCastException`. **Not stable**: equal-priority items come out in arbitrary order (add a sequence number tie-breaker if FIFO matters). Not thread-safe (`PriorityBlockingQueue`).
- **Iteration and `toString` show the array, not sorted order.** Mutating an element's priority after insertion breaks the heap invariant silently (same class of bug as a mutable key).

Traced siftUp/siftDown (observed output, `D5_Misc` A). Offer `5, 3, 8, 1, 9, 2`:
```
offer 5 -> [5]
offer 3 -> [3, 5]              3 < parent 5 : swap
offer 8 -> [3, 5, 8]
offer 1 -> [1, 3, 8, 5]        1 goes to idx3, parent idx1 (5): 1<5 swap -> [3,1,8,5]; parent idx0 (3): 1<3 swap -> root
offer 9 -> [1, 3, 8, 5, 9]     parent idx1 (3) <= 9, stay
offer 2 -> [1, 3, 2, 5, 9, 8]  idx5 parent idx2 (8): 2<8 swap; parent idx0 (1) <= 2 stop
```
Then `poll()` repeatedly (siftDown of the moved-last element):
```
poll -> 1 ; last=8 moves to root, children 3 and 2, smaller=2 -> 2 goes up, 8 sinks: [2, 3, 8, 5, 9]
poll -> 2 ; last=9 to root; children 3,8 -> 3 up; then children of idx1 are 5 (idx3) and none: 9 < 5? no -> 5 up: [3, 5, 8, 9]
poll -> 3 : [5, 9, 8]     poll -> 5 : [8, 9]     poll -> 8 : [9]     poll -> 9 : []
polled sequence: 1 2 3 5 8 9
```
Heapify of `[9,8,7,6,5,4,3,2,1]` gives `[1, 2, 3, 6, 5, 4, 7, 8, 9]` (observed) - a valid heap, not sorted.

Patterns (both run in the demos):
- **Top-K largest** in a stream of n items: keep a **min-heap of size K**; offer each, and if size > K `poll()` (evicts the smallest). O(n log K) time, O(K) memory. Observed for `{7,2,9,4,11,5,8,1,10}`, K=3 -> `[9, 10, 11]`, heap root 9 is the K-th largest.
- **Merge K sorted lists**: heap of (`value`, `listIndex`); poll smallest, push that list's next element. O(N log K). Observed merged `[1..9]` from `[1,4,7],[2,5,8],[3,6,9]`.
- Others: Dijkstra (with lazy deletion instead of decrease-key), scheduling by deadline, running median (two heaps).

---

## 2.8 Sets are Map wrappers

| Set | Backing | Notes |
|---|---|---|
| `HashSet<E>` | `HashMap<E,Object>` and `static final Object PRESENT` | `add` = `map.put(e, PRESENT) == null`; one null allowed; `HashSet(Collection c)` sizes `max(c.size()/.75f + 1, 16)` |
| `LinkedHashSet<E>` | `LinkedHashMap` | insertion order; JDK 21 is a `SequencedSet` |
| `TreeSet<E>` | `NavigableMap` (a `TreeMap`) | `first/last/floor/ceiling/subSet/headSet/tailSet/descendingSet`; comparator decides duplicates |
| `EnumSet<E>` | bit vector | see 2.9 |

Set memory per element is the same as the map entry (about 40 B in HashSet incl. table slot, approximate) - the dummy value costs nothing per entry since all share `PRESENT`.

---

## 2.9 EnumMap, EnumSet, IdentityHashMap, WeakHashMap

- **EnumMap**: `Object[] vals` indexed by `key.ordinal()`; no hashing, no collisions, iteration in enum declaration order (observed `{MON=m, WED=w, FRI=f}`), null keys rejected, null values allowed (internally masked). Fastest map for enum keys: prefer over `HashMap<EnumType,...>`.
- **EnumSet**: abstract; `RegularEnumSet` (one `long` bitmask, enums up to 64 constants) or `JumboEnumSet` (`long[]`). `of/range/allOf/noneOf/complementOf`; operations are bit ops. Observed `range(TUE,THU)=[TUE, WED, THU]`, `complementOf({MON})=[TUE, WED, THU, FRI]`. Use as a compact flag set.
- **IdentityHashMap**: compares keys with `==` and hashes with `System.identityHashCode`; **deliberately violates the `Map` contract** (equal Strings are distinct keys; observed size 2 vs HashMap size 1). Open addressing (linear probing) in one interleaved key/value array. Uses: object-graph traversal (serialisation, deep copy, cycle detection), proxies/`instanceof`-style registries.
- **WeakHashMap**: keys are held through `WeakReference` (entries subclass `WeakReference` registered with a `ReferenceQueue`); when a key becomes weakly reachable GC clears it and the entry is *expunged lazily* on the next map operation. Values are **strong**: if a value references its own key the entry never goes away. Do not use `String` literals/boxed small `Integer`s as keys (never collected). Observed after `System.gc()`: size 1 (the key with a live strong ref survived, the other vanished) - GC timing is not guaranteed. It is an ephemeral, metadata-attachment structure, **not** a cache with size/time policy (no LRU, no expiry, GC decides).

---

## 2.10 CopyOnWriteArrayList / CopyOnWriteArraySet

Holds `volatile Object[] array`. Every mutator takes a lock, copies the whole array (`Arrays.copyOf`), modifies the copy, then publishes it. Readers just read the volatile reference: lock-free. `iterator()` captures the current array: it is a **snapshot** and never throws CME, but also never sees later writes, and `Iterator.remove/set/add` throw `UnsupportedOperationException`.

Observed: iterating `[1,2,3]` and calling `cow.add(4)` at element 1 printed `1 2 3` (snapshot) while the list became `[1, 2, 3, 4]`; `it.remove()` -> UOE. Write cost is O(n) per write: 50,000 appends took **777 ms** vs **2 ms** for `ArrayList` (quadratic total). Use for **read-mostly, small, rarely-changing** collections: listener/observer lists, routing tables, feature-flag lists. Never for write-heavy queues or big collections. `CopyOnWriteArraySet` is the same idea backed by COWAL (O(n) `add`/`contains`). Batch changes with `addAll`, or build a new list and swap a volatile reference.

---

## 2.11 BlockingQueue family (quick contrast)

Four method styles per operation: **throws** (`add/remove/element`), **special value** (`offer/poll/peek`), **blocks** (`put/take`), **timed** (`offer(e,t,u)/poll(t,u)`). Observed on `ArrayBlockingQueue(2)`: `offer` returns `false` when full, `add` throws `IllegalStateException: Queue full`, `offer(e, 50ms)` waits then returns `false`.

| Class | Bound | Structure / locking | Notes |
|---|---|---|---|
| `ArrayBlockingQueue` | fixed | array, one lock (both ends), optional fairness | predictable memory |
| `LinkedBlockingQueue` | optional; default `Integer.MAX_VALUE` (observed) | linked nodes, two locks (put/take) | higher throughput; **unbounded by default => OOM risk** (used by `Executors.newFixedThreadPool`) |
| `PriorityBlockingQueue` | unbounded | heap, one lock | `put` never blocks |
| `DelayQueue` | unbounded | heap of `Delayed` | element becomes available after its delay |
| `SynchronousQueue` | 0 | direct hand-off | `newCachedThreadPool` |
| `LinkedTransferQueue` | unbounded | lock-free linked | `transfer()` waits for consumer |
| `LinkedBlockingDeque` | optional | double-ended, one lock | work stealing |

Backpressure rule: prefer bounded queues and decide the overflow policy (block, reject, drop oldest).

---

## 2.12 `Collections` utilities and Comparator pitfalls

- **Sort stability**: `List.sort`, `Collections.sort`, `Arrays.sort(Object[], cmp)` use **TimSort** (a stable, adaptive merge sort: finds natural runs, merges with galloping; O(n log n) worst, O(n) for already-sorted input; needs up to n/2 temp space). Stable means equal elements keep their relative order, so **multi-key sorting by successive passes works** (observed `[a, d, f, bb, cc, ee]` sorting by length). `Arrays.sort(int[])` (primitives) uses dual-pivot quicksort - not stable, irrelevant for primitives. `Arrays.parallelSort` for big arrays.
- `unmodifiableXxx` are **views** (observed live change), `List.copyOf`/`Set.copyOf`/`Map.copyOf` are snapshots.
- `Collections.synchronizedList(...)`: single methods are synchronised on a mutex, but **iteration and compound actions need `synchronized (list) { for (x : list) ... }`** - it does not stop CME (observed: for-each plus `add` on a synchronizedList throws CME even single-threaded).
- `Collections.binarySearch` needs a sorted list (result undefined otherwise); O(log n) on `RandomAccess` lists, O(n) traversal on LinkedList. Negative return = `-(insertionPoint)-1`.
- `Collections.emptyList()/singletonList/nCopies` are cheap shared immutables. `Collections.shuffle(list, new Random(seed))` reproducible.
- `Collections.max/min` iterate once. `Collections.frequency`, `disjoint`, `swap`, `rotate`, `reverse` all O(n).

**Comparator pitfalls (observed, `D5_Misc` D and `D6_Extra` A)**
1. **Subtraction overflow**: `(x, y) -> x - y` with `Integer.MAX_VALUE` and `-5`: `MAX - (-5) = -2147483644` (negative -> says MAX < -5). `Arrays.sort` returned `[2147483647, -5, 3]` (wrong, no exception). Use `Integer.compare(x, y)` / `Comparator.comparingInt(...)`.
2. **Inconsistent comparator**: TimSort may detect a violation of the sort contract and throw `IllegalArgumentException: Comparison method violates its general contract!`. Observed with `(x, y) -> x < y ? -1 : 1` (never returns 0 for equal elements: `compare(a,a)` = 1 and `compare(a,b)` = `compare(b,a)` = 1) on 5,000 elements: 5 of 5 trials threw; a random comparator `r.nextInt(3)-1` threw as well. **Small lists (< 32 elements use binary insertion sort) or lucky data do not throw**: the bug is latent and hits in production with larger data. Contract: `sgn(compare(a,b)) == -sgn(compare(b,a))`, transitivity, and `compare(a,b)==0` implies same sign vs any third element.
3. **Double/float**: `Double.compare` handles `NaN`, `-0.0`; `a - b` cast to int loses information.
4. **Null handling**: `Comparator.nullsFirst/nullsLast(...)`.
5. **Reversed via negation** `-cmp.compare(a,b)` breaks on `Integer.MIN_VALUE`; use `.reversed()`.
6. Comparator lambdas should be **consistent with equals** if used in TreeSet/TreeMap, otherwise you delete "duplicates" (2.3).
7. `thenComparing` chains compose stable multi-field orders: `Comparator.comparing(Emp::dept).thenComparing(Emp::salary, Comparator.reverseOrder())`.

---

## 2.13 Java 21 SequencedCollection

New interfaces: `SequencedCollection<E>` (`reversed()`, `addFirst/addLast`, `getFirst/getLast`, `removeFirst/removeLast`), `SequencedSet<E>`, `SequencedMap<K,V>` (`firstEntry/lastEntry/pollFirstEntry/pollLastEntry/putFirst/putLast/reversed/sequencedKeySet/sequencedValues/sequencedEntrySet`).

Retrofit: `List` and `Deque` are `SequencedCollection`; `LinkedHashSet` and `SortedSet` (so `TreeSet`) are `SequencedSet`; `LinkedHashMap` and `SortedMap` (so `TreeMap`) are `SequencedMap`. **`HashSet` and `HashMap` are not sequenced.**

Observed:
```
[1,2,3]: getFirst=1 getLast=3 reversed=[3, 2, 1]
after addFirst(0) and reversed().addFirst(99): [0, 1, 2, 3, 99]     <- reversed() is a VIEW; its "first" is the original's last
LinkedHashMap {a=1,b=2,c=3}: firstEntry=a=1 lastEntry=c=3 reversed={c=3, b=2, a=1}
putFirst(z,0) -> {z=0, a=1, b=2, c=3}     sequencedKeySet().reversed() = [c, b, a, z]
new ArrayList<Integer>().getFirst() -> NoSuchElementException
```
Old idioms it replaces: `list.get(list.size()-1)`, `deque.getLast`, `new ArrayList<>(list)` + `Collections.reverse` for reverse iteration, `linkedHashSet.iterator().next()`, `stream().reduce((a,b)->b)`. Gotchas: `List.of(...).addFirst` -> UOE (immutable); `TreeSet.addFirst` -> UOE (position determined by comparator); `getFirst` on an empty list throws (unlike `Deque.peekFirst` returning null); JDK 21 only.

---

# 3. Runnable demos (compiled and run with `javac`/`java` 21.0.4)

Run everything with plain `javac Demo.java && java Demo`. Demos marked **[reflection]** peek at `HashMap.table`; they need `java --add-opens java.base/java.util=ALL-UNNAMED Demo` (JDK 17+ blocks reflective access to JDK internals by default). Nothing else uses reflection. All outputs below were copied from real runs; timing numbers vary by machine.

## Demo 1 - Same hashCode, different keys (no reflection)

```java
import java.util.*;
public class Collide {
    public static void main(String[] a) {
        Map<String,Integer> m = new HashMap<>();
        m.put("Aa", 1); m.put("BB", 2); m.put("AaAa", 3); m.put("BBBB", 4); m.put("AaBB", 5); m.put("BBAa", 6);
        System.out.println("Aa.hash==BB.hash? " + ("Aa".hashCode() == "BB".hashCode())
            + " get(BB)=" + m.get("BB") + " size=" + m.size());
        System.out.println("AaAa=" + "AaAa".hashCode() + " BBBB=" + "BBBB".hashCode()
            + " AaBB=" + "AaBB".hashCode() + " BBAa=" + "BBAa".hashCode());
    }
}
```
Real output:
```
Aa.hash==BB.hash? true get(BB)=2 size=6
AaAa=2031744 BBBB=2031744 AaBB=2031744 BBAa=2031744
```
Reading: `"Aa"` = `65*31+97 = 2112`, `"BB"` = `66*31+66 = 2112`. Every concatenation of these blocks collides, so an attacker can generate 2^k strings with one hash code (the HashDoS attack). `equals` still keeps the six keys distinct and `get("BB")` returns 2. Interesting side effect: `Set.of("Aa".hashCode(), "BB".hashCode())` throws `IllegalArgumentException: duplicate element: 2112` - `Set.of` rejects duplicates at creation.

## Demo 2 - Bad `hashCode` timing: Comparable vs not (no reflection)

```java
static final class Bad {              // hashCode() == 42, NOT Comparable
    final int id; Bad(int id) { this.id = id; }
    @Override public int hashCode() { return 42; }
    @Override public boolean equals(Object o) { return o instanceof Bad b && b.id == id; }
}
static final class BadComparable implements Comparable<BadComparable> {   // same, but Comparable
    final int id; BadComparable(int id) { this.id = id; }
    @Override public int hashCode() { return 42; }
    @Override public boolean equals(Object o) { return o instanceof BadComparable b && b.id == id; }
    @Override public int compareTo(BadComparable o) { return Integer.compare(id, o.id); }
}
static final class Good { /* hashCode = Integer.hashCode(id) */ }

static long time(IntFunction<Object> mk, int n) {
    Map<Object,Integer> m = new HashMap<>();
    Object[] ks = new Object[n]; for (int i = 0; i < n; i++) ks[i] = mk.apply(i);
    long t0 = System.nanoTime();
    for (int i = 0; i < n; i++) m.put(ks[i], i);
    long s = 0; for (int i = 0; i < n; i++) s += m.get(ks[i]);
    return (System.nanoTime() - t0) / 1_000_000;          // ms
}
```
Real output (put n then get n):
```
n=2000   Good=   0 ms   BadComparable=  12 ms   BadNotComparable=  153 ms
n=5000   Good=   2 ms   BadComparable=   5 ms   BadNotComparable=  246 ms
n=10000  Good=   0 ms   BadComparable=   4 ms   BadNotComparable= 1196 ms
(2nd pass) n=2000   Good= 0 ms   BadComparable= 0 ms   BadNotComparable=  30 ms
(2nd pass) n=5000   Good= 0 ms   BadComparable= 2 ms   BadNotComparable= 284 ms
(2nd pass) n=10000  Good= 0 ms   BadComparable= 6 ms   BadNotComparable= 835 ms
```
Reading: 5x the keys costs roughly 25x the time for the non-Comparable case (quadratic); `Comparable` keeps the treeified bin logarithmic. (An earlier attempt at n=20,000 took 6.2 s and n=60,000 did not finish in reasonable time in the harness - quadratic, as predicted.)

## Demo 3 - Bucket dump and resize trace **[reflection]**

```java
HashMap<Integer,String> m = new HashMap<>(4);
dump("new HashMap<>(4)", m);
for (int k : new int[]{5, 9, 1, 2, 13}) { m.put(k, "v"); dump("put(" + k + ") idx@4=" + (k & 3), m); }
m.put(9, "again"); dump("put(9) again", m);
m.put(6, "v");  dump("put(6)", m);
m.put(17, "v"); dump("put(17)", m);
// dump(): reflectively read HashMap.table, HashMap$Node.key/next; print "[i] k1 -> k2"
```
Real output (empty buckets removed for brevity, headers verbatim):
```
new HashMap<>(4)   size=0 threshold=4 table.length=0
put(5) idx@4=1   size=1 threshold=3 table.length=4       [1] 5
put(9) idx@4=1   size=2 threshold=3 table.length=4       [1] 5 -> 9
put(1) idx@4=1   size=3 threshold=3 table.length=4       [1] 5 -> 9 -> 1
put(2) idx@4=2   size=4 threshold=6 table.length=8       [1] 9 -> 1   [2] 2   [5] 5
put(13) idx@4=1  size=5 threshold=6 table.length=8       [1] 9 -> 1   [2] 2   [5] 5 -> 13
put(9) again     size=5 threshold=6 table.length=8       (unchanged)
put(6)           size=6 threshold=6 table.length=8       [6] 6 added
put(17)          size=7 threshold=12 table.length=16     [1] 1 -> 17  [2] 2  [5] 5  [6] 6  [9] 9  [13] 13
```
Reading: the table is `null` until the first put; the stored `threshold=4` is the initial capacity. The split at the 4th put shows lo list `9 -> 1` staying at index 1 and hi list `5` moving to `1 + oldCap = 5`; order inside each list is preserved. The 17th key lands on bucket 1 at capacity 8 (`17 & 7 = 1`) and after doubling `17 & 15 = 1` again (`17 & 8 = 0`, lo).

## Demo 4 - Treeify only at table >= 64, untreeify on remove **[reflection]**

```java
HashMap<Integer,Integer> t = new HashMap<>();
for (int i = 1; i <= 12; i++) { t.put(i * 1024, i); System.out.println("put #" + i + " -> " + peek(t)); }
for (int i = 12; i >= 4; i--) t.remove(i * 1024);
System.out.println("after removing down to 3 keys: " + peek(t));
// peek(): table.length, count of bins whose head class is HashMap$TreeNode, longest plain chain
```
Real output:
```
put #1 -> table.length=16 treeBins=0 maxListChain=1
...                                   (chain grows one per put)
put #8 -> table.length=16 treeBins=0 maxListChain=8
put #9 -> table.length=32 treeBins=0 maxListChain=9      <- 9th node: chain>=8 but table<64: RESIZE, no tree
put #10 -> table.length=64 treeBins=0 maxListChain=10     <- resize again (each put re-triggers treeifyBin)
put #11 -> table.length=64 treeBins=1 maxListChain=0      <- table is 64: now it treeifies
put #12 -> table.length=64 treeBins=1 maxListChain=0
after removing down to 3 keys: table.length=64 treeBins=0 maxListChain=3
```
Interview-worthy: a chain of 8 is still a list. Two extra "resize instead of treeify" steps were needed before the tree appeared.

## Demo 5 - Pre-sizing and `newHashMap` **[reflection]** (plus resize counts)

```java
HashMap.newHashMap(1000)                 -> table.length=2048        // ceil(1000/0.75)=1334 -> 2048, no resize while filling
new HashMap<>(1000) + 751 puts           -> table.length=1024 (threshold 768 not yet crossed)
new HashMap<>(1000) + 1000 puts          -> table.length=2048 (resized once at the 769th put)
new HashMap<>(768)  + 768 puts           -> table.length=1024
new HashMap<>(769)  + 769 puts           -> table.length=2048
```
Filling 1,000,000 Integer keys (real output of `D7_Resizes`):
```
default: 16@1 32@13 64@25 ... 1048576@393217 2097152@786433
   resizes after first allocation=17  nodes moved during resizes ~ 1572852 for N=1000000
newHashMap(N): 2097152@1
   resizes after first allocation=0   nodes moved during resizes ~ 0 for N=1000000
```
Honest caveat: I also tried to measure the *worst single `put` latency* (default vs `newHashMap`) over 4,000,000 puts. Results were dominated by GC/JIT noise (worst put 40-80 ms in both variants, at positions unrelated to resize points), so I do **not** claim a timing win from that run; the reliable evidence is the resize *count* and nodes moved above. Measure your own workload with JFR/JMH.

## Demo 6 - ArrayList growth, CME quirk, safe removal, `remove` traps **[reflection only for capacity]**

```java
static void tryIter(List<String> l, String removeVal) {
    try {
        for (String s : l) { System.out.print(s + " "); if (s.equals(removeVal)) l.remove(s); }
        System.out.println("-> finished normally, list=" + l);
    } catch (ConcurrentModificationException e) { System.out.println("-> CME! list=" + l); }
}
tryIter(new ArrayList<>(List.of("a","b","c","d")), "a");   // first
tryIter(new ArrayList<>(List.of("a","b","c","d")), "c");   // second-to-last
tryIter(new ArrayList<>(List.of("a","b","c","d")), "d");   // last
tryIter(new ArrayList<>(List.of("a","b","c")),     "b");   // second-to-last again
```
Real output:
```
new ArrayList(): cap=0
size=1->cap 10 | size=11->cap 15 | size=16->cap 22 | size=23->cap 33 | size=34->cap 49 | size=50->cap 73 | size=74->cap 109 |
new ArrayList(0)+1 add: cap=1
copy-constructor of 3: cap=3

a -> CME! list=[b, c, d]
a b c -> finished normally, list=[a, b, d]
a b c d -> CME! list=[a, b, c]
a b -> finished normally, list=[a, c]

removeIf even -> [1, 3, 5]
iterator.remove 3 -> [1, 5]
remove(1) as index -> [10, 30, 1, 2]
remove(Integer.valueOf(1)) -> [10, 30, 2]
asList.set writes through: arr[0]=X
asList.add -> UnsupportedOperationException
unmodifiableList sees change: [p, q, r]   List.copyOf does not: [p, q]
List.of(null elem) -> NPE
List.of(..).contains(null) -> NPE
ArrayList.contains(null) -> false
sub.set(0,99) writes through: [0, 1, 99, 3, 4, 5, 6, 7]
sub.clear() removes range from base: [0, 1, 5, 6, 7]
sub used after parent structural change -> CME
stack push 1,2,3; pop -> 3, peek -> 2, deque=[2, 1]
queue offer 1,2,3; poll -> 1, deque=[2, 3]
ArrayDeque.add(null) -> NPE
LinkedList allows null: [null]
empty pop() -> NoSuchElementException; poll() -> null
```
(The traced explanation of the four CME lines is in 2.5.1: the `hasNext()` test is `cursor != size`.)

## Demo 7 - LRU cache with `LinkedHashMap`

```java
static class LRU<K,V> extends LinkedHashMap<K,V> {
    private final int max;
    LRU(int max) { super(16, 0.75f, true); this.max = max; }        // true = access order
    @Override protected boolean removeEldestEntry(Map.Entry<K,V> eldest) { return size() > max; }
}
LRU<String,Integer> c = new LRU<>(3);
c.put("a",1); c.put("b",2); c.put("c",3);      print(c.keySet());
c.get("a");                                    print(c.keySet());
c.put("d",4);                                  print(c.keySet());
c.getOrDefault("c", 0); c.putIfAbsent("d", 9); print(c.keySet());
```
Real output:
```
after a,b,c: [a, b, c]
after get(a): [b, c, a]  (a moved to tail = most recent)
after put(d) (evicts eldest=b): [c, a, d]
after getOrDefault(c), putIfAbsent(d): [a, c, d]
insertion-order map, re-put z does not reorder: {z=3, y=2}
```
Production notes: (1) not thread-safe: wrap with `Collections.synchronizedMap` (every `get` mutates); (2) size-based only - for TTL/weight eviction use Caffeine; (3) `removeEldestEntry` runs on the writer thread inside `put` - never do slow work (I/O) there; (4) values that are large objects mean "max entries" is not a memory bound.

## Demo 8 - Mutable key lost, TreeMap navigation, comparator/equals mismatch

```java
MutKey k = new MutKey(1);  m.put(k, "value");  k.id = 2;
System.out.println(m.get(k) + " " + m.containsKey(k) + " " + m.size() + " " + m);
k.id = 1;  System.out.println(m.get(k));
```
Real output:
```
after k.id=2: get(k)=null containsKey=false size=1 iteration still sees it: {K2=value}
after restoring id=1: get(k)=value
new K1 key put: size=1
```
TreeMap / NavigableMap:
```
floorKey(25)=20 ceilingKey(25)=30 lowerKey(20)=10 higherKey(20)=30 floorKey(5)=null
subMap(20,40)={20=v20, 30=v30}  subMap(20,true,40,true)={20=v20, 30=v30, 40=v40}
headMap(30)={10=v10, 20=v20} tailMap(30)={30=v30, 40=v40, 50=v50} descendingMap={50=v50, 40=v40, 30=v30, 20=v20, 10=v10}
firstEntry=10=v10 pollLastEntry=50=v50
view sees later put(25): {20=v20, 25=v25, 30=v30}
put outside subMap range -> IllegalArgumentException
TreeMap null key -> NPE
TreeSet(CASE_INSENSITIVE) of Java,JAVA,java -> [Java] (compare==0 means duplicate, equals ignored)
BigDecimal 1.0 vs 1.00: equals=false compareTo=0  HashSet size=2  TreeSet size=1
```

## Demo 9 - Concurrent maps: null bans, atomic methods, compound race

```java
Map<String,Integer> sync = Collections.synchronizedMap(new HashMap<>());
ConcurrentHashMap<String,Integer> cm1 = new ConcurrentHashMap<>(), cm2 = new ConcurrentHashMap<>();
// 8 threads x 20,000 iterations each:
sync.put("n", sync.getOrDefault("n", 0) + 1);      // check-then-act on a synchronizedMap
cm1.put("n", cm1.getOrDefault("n", 0) + 1);        // check-then-act on a ConcurrentHashMap
cm2.merge("n", 1, Integer::sum);                   // atomic
```
Real output:
```
CHM null value -> NPE
CHM null key -> NPE
putIfAbsent first=null second=1  merge=11 compute=22 computeIfAbsent(n)=7
CHM modify during iteration: no CME, iterated=7 final size=14           (count varies run to run)
mappingCount=14 size=14
expected=160000  synchronizedMap(get+put)=33794  CHM(get+put)=121071  CHM.merge=160000
synchronizedList for-each + add -> CME (single thread even); iteration needs synchronized(list){...}
```
(`HashMap.get(null)` returns `null` while `ConcurrentHashMap`/`Hashtable` `get(null)` throw NPE - the second half I state from the API/source, not from a run.)

## Demo 10 - PriorityQueue array order, top-K, merge-K, Comparator pitfalls

Real output (sections A-D of `D5_Misc`, `D6_Extra`):
```
offer 5 -> array [5]
offer 3 -> array [3, 5]
offer 8 -> array [3, 5, 8]
offer 1 -> array [1, 3, 8, 5]
offer 9 -> array [1, 3, 8, 5, 9]
offer 2 -> array [1, 3, 2, 5, 9, 8]
iteration (toString) is heap order, not sorted: [1, 3, 2, 5, 9, 8]
poll -> array now [2, 3, 8, 5, 9]
poll -> array now [3, 5, 8, 9]
poll -> array now [5, 9, 8]
poll -> array now [8, 9]
poll -> array now [9]
poll -> array now []
polled sequence: 1 2 3 5 8 9
PriorityQueue(collection [9..1]) heapify -> [1, 2, 3, 6, 5, 4, 7, 8, 9]
max-heap peek=8
top3 (heap order) = [9, 10, 11] smallest of them=9
merged = [1, 2, 3, 4, 5, 6, 7, 8, 9]
subtraction sort -> [2147483647, -5, 3]  (WRONG)
Integer.compare sort -> [-5, 3, 2147483647]
MAX - (-5) = -2147483644 (overflow, negative => says MAX < -5)
stable by length: [a, d, f, bb, cc, ee]
```
Comparator contract violation (`(x, y) -> x < y ? -1 : 1` on 5,000 random ints in 0..29):
```
trial 0..4: Comparison method violates its general contract!      (5 of 5)
random comparator (r.nextInt(3)-1) on 5,000 ints: Comparison method violates its general contract!
```
On a 200-element list with a random comparator I saw **no exception** ("bug is latent") - a good reminder that small test data will not catch this.

## Demo 11 - Java 21 sequenced, EnumMap, IdentityHashMap, WeakHashMap, COW, BlockingQueue

```
getFirst=1 getLast=3 reversed=[3, 2, 1]
after addFirst(0) and reversed().addFirst(99): [0, 1, 2, 3, 99]
firstEntry=a=1 lastEntry=c=3 reversed={c=3, b=2, a=1}
putFirst(z) -> {z=0, a=1, b=2, c=3} sequencedKeySet reversed=[c, b, a, z]
empty getFirst -> NoSuchElementException
EnumMap order = ordinal order: {MON=m, WED=w, FRI=f}  EnumSet.range(TUE,THU)=[TUE, WED, THU] complementOf(MON)=[TUE, WED, THU, FRI]
two equal Strings: IdentityHashMap size=2 HashMap size=1
after GC size=1 values=[kept] (strongKey still referenced: true)
1 2 3 -> loop saw snapshot only; list now [1, 2, 3, 4]
COW iterator.remove -> UnsupportedOperationException
50k appends: ArrayList=2 ms, CopyOnWriteArrayList=777 ms
offer1=true offer2=true offer3(full)=false offer timeout=false
add on full -> IllegalStateException: Queue full
LinkedBlockingQueue default remainingCapacity=2147483647
```

---

# 4. Production war stories

> These are composites of well-known incident patterns, written as the story you would tell in an interview; the mechanics are verified by the demos above, the specific numbers (30 M entries, 300-800 ms) are illustrative, not measurements.

### 4.1 The customer whose orders "disappeared" (mutable key)
- **Symptom**: `Map<Order, Status>` where `Order` is a JPA entity whose `hashCode()` included the mutable `status` and a database-generated `id` (null before persist). Orders "vanished" from the cache after a state transition; `containsKey` false, but a heap dump and iteration showed them.
- **Root cause**: key mutated after insertion; entry stuck in the bucket of the old hash (and cached `hash` in the node). Same symptom for `HashSet<Entity>` with the id assigned after `add`.
- **Fix**: key by an immutable natural id or business key (record, `UUID`, `String`); if you must key by an entity, use `id`-only hash *after* assignment or key the map by the id. Prevention: `final` fields, records, code review rule "never put a mutable object in a hash key".
- **Test**: unit test that mutates and asserts `get` finds it (it will not - that proves the design is wrong); static analysis (Error Prone `MutableKey`-style checks).

### 4.2 The p99 spike every N minutes (huge HashMap resize)
- **Symptom**: a service holding a 30 M-entry in-heap `HashMap` (loaded lazily by requests) shows periodic 300-800 ms request stalls that get rarer and longer as the map grows.
- **Root cause**: the table doubles (a 2^26-slot array is 256 MB of references) - an O(n) move by the thread that triggered it, while other threads wait if the map is guarded by a lock; plus the humongous array allocation under G1 (humongous region allocation and a full-ish evacuation), plus cache misses walking the old table.
- **Fix**: pre-size (`HashMap.newHashMap(expected)`), or warm up before taking traffic; partition into N smaller maps (shard by hash) so each resize is 1/N; or move to `ConcurrentHashMap` (cooperative incremental resize) / an off-heap or external cache. Observe with GC logs (`-Xlog:gc*`) and JFR allocation events (large `Object[]`).
- Evidence here: a default map filled to 1 M keys performs 17 resizes and moves about 1.57 M nodes (Demo 5). My own micro-measure of the worst single `put` was too noisy to quote - measure in your environment.

### 4.3 The "cache" that ate the heap (unbounded map as cache)
- **Symptom**: `OutOfMemoryError: Java heap space` after days; heap dump: a `static final Map<String, Result>` with millions of entries, dominator of the heap.
- **Root cause**: no eviction: keys like `userId + ":" + queryString` are effectively unbounded. `WeakHashMap` was tried and did nothing (keys were strongly referenced Strings; values referenced keys). `HashMap` never shrinks its table either.
- **Fix**: bounded cache with policy: `LinkedHashMap` LRU (synchronised) for small needs, or Caffeine/Guava with `maximumSize`/`expireAfterWrite`/`weigher`; add metrics (size, hit rate, evictions). Also check unbounded `LinkedBlockingQueue` in executors (same class of bug: `Executors.newFixedThreadPool` uses an unbounded queue).

### 4.4 CME under load (the second-to-last bug and its opposite)
- **Symptom**: `ConcurrentModificationException` in a scheduled task only in production, around 1 per 10,000 runs; never in tests. In another module the same pattern *silently skipped* an element.
- **Root cause A**: one thread iterates a shared `ArrayList` while another adds to it. The CME is a best-effort detection of an unsynchronised data race, and it is only thrown if the iterator sees a changed `modCount`; with a visibility race (no happens-before) it may not throw at all and instead return stale/duplicated data.
- **Root cause B**: `for (x : list) if (cond) list.remove(x)` on a single thread - throws for most positions, silently skips the element after a removed second-to-last (Demo 6).
- **Fix**: `removeIf`, `Iterator.remove`, iterate a copy (`new ArrayList<>(list)`) taken under a lock, or use `CopyOnWriteArrayList` (rare writes) / `ConcurrentLinkedQueue`; for `synchronizedList` hold `synchronized (list)` during the entire traversal. Never "fix" by catching CME.

### 4.5 Parallel `HashMap` corruption (unsafe publication and concurrent writes)
- **Symptom (JDK 7 era)**: 100% CPU on several threads, thread dumps stuck in `HashMap.get` / `transfer`. **JDK 8+**: lost entries, wrong `size()`, `null` from `get` for present keys, occasional `ClassCastException`/`NullPointerException` inside `TreeNode` methods, rare infinite loops in tree bins.
- **Root cause**: `HashMap` is not thread-safe; concurrent `resize()` corrupts the bucket pointers. JDK 7 inserted at the head and reversed chains during transfer, so two threads could create a **cycle** in a bucket chain, and `get` looped forever. JDK 8 keeps order (tail insertion, lo/hi lists) so cycles from resize are gone, but lost updates and inconsistent `size` remain.
- **Fix**: `ConcurrentHashMap` (use `merge/compute`), or confine the map to one thread, or build it fully then publish safely (`volatile`, `final` field, `Map.copyOf` immutable). Reads of a fully built, never-mutated `HashMap` from many threads are safe *if safely published*.
- The demo (Demo 9) shows even "thread-safe" maps lose updates when you compose `get` then `put` - the bug is your algorithm, not the map.

### 4.6 More one-liners from real incidents
- `Arrays.asList(intArray)` returned a `List<int[]>` of size 1 - the loop "processed" one element.
- `List.of(...)`-returned list passed into legacy code that called `sort()` -> UOE in production.
- `subList` stored in a long-lived field kept a 2 GB parent list alive.
- `Comparator` using `a.getSize() - b.getSize()` (ints near `MAX_VALUE`) mis-ordered a priority queue -> starving tasks.
- `TreeSet<Employee>` with a comparator on `salary` only: employees with equal salary "disappeared".
- `stream.collect(Collectors.toMap(...))` threw `IllegalStateException: Duplicate key` at the first duplicate in production data.
- `HashMap<Double,...>` with computed keys: `0.1+0.2` != `0.3`, lookups missed; use `BigDecimal` with `stripTrailingZeros` or integer cents.

---

# 5. Interview questions (56) - graded, with follow-up chains and common wrong answers

Format: **Q** -> model answer -> `F:` follow-up chain -> `Wrong:` typical incorrect answers.

## 5.1 Easy (1-12)

**E1. HashMap vs Hashtable vs ConcurrentHashMap?**
HashMap: unsynchronised, 1 null key + null values, fail-fast iterators. Hashtable: legacy JDK 1.0, every method `synchronized` on the whole table, no nulls. ConcurrentHashMap: thread-safe with fine-grained locking/CAS, no nulls, weakly consistent iterators, atomic `compute/merge`.
`F:` Why no nulls in CHM? -> `get(k)==null` would be ambiguous (absent vs null) and you cannot resolve it with a `containsKey` afterwards because another thread may change the map in between. `F:` Is `Collections.synchronizedMap` the same as Hashtable? -> Similar coarse lock, but wraps any Map (nulls per the wrapped map) and iteration needs manual `synchronized`.
`Wrong:` "ConcurrentHashMap locks the whole map like Hashtable but faster"; "CHM allows one null key".

**E2. ArrayList vs LinkedList - which do you default to?**
ArrayList: array, O(1) `get`, amortised O(1) append, O(n) middle insert but a fast `arraycopy`. LinkedList: doubly linked, O(1) splice at a known node but O(n) to find it. Default to ArrayList; observed insert-at-middle-by-index: ArrayList 8 ms vs LinkedList 140-176 ms (n=100k, 2000 inserts).
`F:` When would LinkedList win? -> many ListIterator insert/remove in the middle of a very large list; realistically almost never; ArrayDeque for queue/stack. `F:` Memory? -> a node is about 24 B plus the element vs 4 B slot (approximate).
`Wrong:` "LinkedList is faster for insertion" (without the finding-cost caveat).

**E3. Default capacity and load factor of HashMap; when does it resize?**
16 and 0.75; threshold 12; resize when `++size > threshold`, i.e. on the 13th insertion (observed: table 16 -> 32 on the 13th put).
`F:` Is the table allocated in the constructor? -> No, lazily on first put (observed `table=null`).
`Wrong:` "resizes when 75% of buckets are occupied" (it counts entries, not occupied buckets).

**E4. How does HashSet work?**
Wraps a `HashMap<E,Object>`, `add(e)` = `map.put(e, PRESENT) == null`. Same hashing/equality rules.
`F:` What do LinkedHashSet and TreeSet wrap? -> LinkedHashMap, TreeMap.

**E5. Does HashMap allow null?**
One null key (hash 0, bucket 0) and any number of null values; `get` returning null is ambiguous - use `containsKey` or `getOrDefault`. TreeMap: no null keys (NPE) with natural ordering. Hashtable/CHM/`Map.of`/`List.of`: none.

**E6. State the equals/hashCode contract.**
Equal objects must have equal hash codes; hash codes must be stable while the used fields do not change; unequal objects may collide. Override both, using the same fields.
`F:` What if only equals is overridden? -> equal keys hash to different buckets (identity hash), `get` misses, Set holds duplicates. `F:` Only hashCode? -> same bucket, but `equals` identity says different -> duplicates/misses.

**E7. What is a fail-fast iterator?**
It snapshots `modCount` at creation and throws `ConcurrentModificationException` from `next()` if the collection was structurally modified in a way the iterator did not do. Best effort, not a concurrency guarantee.
`F:` Does removing an element inside a for-each always throw? -> No (second-to-last quirk, section 2.5.1).

**E8. What does `Arrays.asList` return?**
A fixed-size `List` view of the array (write-through `set`, `add/remove` -> UOE). With a primitive array you get a one-element `List<int[]>`.

**E9. Comparable vs Comparator.**
Comparable = natural order inside the class (`compareTo`), one per class. Comparator = external, many orders, composable (`comparing().thenComparing().reversed()`). Use `Integer.compare`, never subtraction.

**E10. `List.of` vs `Collections.unmodifiableList` vs `List.copyOf`?**
`List.of`: truly immutable, no nulls (even `contains(null)` NPEs). `unmodifiableList`: read-only *view* - underlying changes show through (observed). `List.copyOf`: immutable snapshot (no nulls).
`Wrong:` "unmodifiableList makes the data immutable".

**E11. Why prefer ArrayDeque over Stack?**
`Stack` extends `Vector` (synchronised, legacy, exposes index methods). `ArrayDeque` is a faster circular array with `push/pop/peek`; but no nulls.

**E12. `list.remove(1)` on a `List<Integer>` - what happens?**
Calls `remove(int index)` (exact-match overload wins over boxing) -> removes the element at index 1. To remove the value: `list.remove(Integer.valueOf(1))`.

## 5.2 Medium (13-38)

**M13. Walk through `HashMap.put`.**
Hash = `h ^ (h>>>16)`; allocate table if null; `i=(n-1)&hash`; empty bin -> new node; else compare first node (`hash`, `==`, `equals`), tree bin -> `putTreeVal`, list -> walk and append at tail (treeifyBin if the chain was already 8 long); existing key -> replace value, return old (no modCount/size change); else `++modCount`, `++size > threshold` -> resize.
`F:` Head or tail insertion? -> tail since JDK 8 (head in 7). `F:` Which comparison happens first? -> hash int, then `==`, then `equals`. `F:` Does `put` on an existing key change modCount? -> No.

**M14. Why must the capacity be a power of two?**
Fast index (`(n-1)&hash` instead of `%`), all bits used evenly, and resize splits a bin with a single-bit test (`hash & oldCap`).
`F:` What if I pass 1000? -> rounded up to 1024 by `tableSizeFor`. `F:` And 1025? -> 2048.

**M15. Why `h ^ (h >>> 16)`?**
Only the low `log2(n)` bits pick the bucket; mixing the high 16 bits into the low 16 lets high-bit differences (floats, multiples of 65536) influence the index. Cheap compromise: it is not a real avalanche function.
`F:` Do you need it for `Integer` keys 0..15? -> No effect for small ints (`h>>>16 == 0`).

**M16. When exactly does a bin become a tree?**
When a bin already holds 8 nodes and a 9th is added, and the table length is at least 64. If the table is smaller, `treeifyBin` calls `resize()` instead (observed: 2 resizes 16 -> 32 -> 64 before the first tree at put #11).
`F:` When back to list? -> on resize split when a half has <= 6 nodes; on remove when the tree shape is tiny. `F:` Why 8 and 6, not both 8? -> hysteresis so a bin does not flap.
`Wrong:` "treeifies at 8 collisions regardless of table size".

**M17. Explain the resize split.**
Capacity doubles; for each old bin at index j, nodes with `(hash & oldCap) == 0` go to j (lo list), the others to `j + oldCap` (hi list). Order within each list is preserved; no rehash. Trace: cap 4 -> 8: `5->9->1` becomes `[1] 9->1`, `[5] 5`.
`F:` Why is one bit enough? -> new mask = old mask + one more low-order bit, that bit is `oldCap`. `F:` Concurrency implication? -> still unsafe under concurrent `put`.

**M18. Pre-size a HashMap for 1000 entries.**
Need `capacity >= 1000/0.75 = 1334` -> table 2048. JDK 19+: `HashMap.newHashMap(1000)`. Before: `new HashMap<>((int) Math.ceil(1000 / 0.75))`. `new HashMap<>(1000)` gives 1024 (threshold 768) and resizes at the 769th put (observed).
`F:` Trade-off? -> allocates the big table immediately; for 1 M entries it avoids 17 resizes (observed). `F:` And for a `HashSet`? -> `HashSet.newHashSet(n)`.
`Wrong:` "new HashMap<>(n) holds n entries without resizing".

**M19. What happens when a key is mutated after insertion?**
Entry stays in the old bucket with the old cached hash; `get/containsKey/remove` miss (observed), iteration still finds it; restoring the state makes it reachable again. Leak until removed. Use immutable keys.
`F:` Does it matter for TreeMap? -> yes: the tree order is stale, so searches follow the wrong path. `F:` For `HashSet<Entity>` with a DB-generated id? -> same bug when the hash uses the id assigned after `add`.

**M20. Implement an LRU cache.**
`LinkedHashMap(16, .75f, true)` + `removeEldestEntry(e) -> size() > max` (Demo 7). O(1) get/put. Not thread-safe -> `Collections.synchronizedMap`.
`F:` Does `getOrDefault`/`putIfAbsent` refresh recency? -> yes (observed). `F:` Iterate while calling get? -> CME, `get` is structural in access order. `F:` Without LinkedHashMap? -> `HashMap` + hand-made doubly linked list. `F:` Better for prod? -> Caffeine (TinyLFU, TTL, async).

**M21. When TreeMap over HashMap?**
Sorted iteration, range queries (`subMap/headMap/tailMap`), nearest key (`floor/ceiling`), min/max, at O(log n) per op and no hashing.
`F:` Null key? -> NPE. `F:` TreeMap with an inconsistent comparator? -> see M22.

**M22. What does "compare consistent with equals" mean and what breaks?**
`compare(a,b)==0 <=> a.equals(b)`. TreeMap/TreeSet decide key identity with `compare`. Observed: `TreeSet(CASE_INSENSITIVE_ORDER)` of `Java, JAVA, java` -> 1 element; `BigDecimal 1.0 / 1.00`: `HashSet` size 2, `TreeSet` size 1.
`F:` How does that bite in a PriorityQueue? -> ties are fine (PQ keeps both), but `remove(Object)`/`contains` use `equals`. `F:` Fix? -> add tie-break fields to the comparator.

**M23. Explain `floorKey`, `ceilingKey`, `lowerKey`, `higherKey`, `subMap`.**
`floor` <=, `ceiling` >=, `lower` <, `higher` >; return null if none. `subMap(a,b)` is `[a,b)` and a live view; `subMap(a,true,b,true)` closed. Values in section 2.3 (keys 10..50: `floorKey(25)=20`, `ceilingKey(25)=30`).
`F:` Complexity of `subMap(...).size()`? -> O(k) (walks the range), not O(1). `F:` Put outside the view range? -> `IllegalArgumentException`.

**M24. How does ArrayList grow?**
Lazily allocated (cap 0 -> 10); then `old + max(minGrowth, old>>1)` via `ArraysSupport.newLength`: 10, 15, 22, 33, 49, 73, 109 (observed). Copy with `Arrays.copyOf`; amortised O(1).
`F:` `new ArrayList<>(0)` then add? -> capacity 1. `F:` Copy constructor of 3 elements? -> capacity 3. `F:` Max size? -> about `Integer.MAX_VALUE - 8` soft limit.
`Wrong:` "doubles", "allocates 10 in the constructor" (since JDK 8 lazily).

**M25. Why does removing the second-to-last element in a for-each not throw CME?**
`hasNext()` is `cursor != size`, and only `next()` checks `modCount`. After removing the second-to-last element, `cursor == size` so the loop exits before `next()` runs: no exception and the last element is skipped. Removing the last element leaves `cursor = size+1 != size` so `next()` throws (traces in 2.5.1).
`F:` How many elements are skipped? -> one (the last). `F:` Is this specified behaviour? -> no, fail-fast is best effort.

**M26. How do you remove elements safely while iterating?**
`Iterator.remove()`, `Collection.removeIf` (O(n) for ArrayList), reverse index loop, collect and `removeAll`, or stream-filter to a new list. `ListIterator` for replace/insert.
`F:` Complexity of `removeIf` vs a loop of `remove(i)`? -> O(n) vs O(n^2).

**M27. What is `subList`? Pitfalls?**
A view over `[from,to)` of the parent (offset + parent modCount check). Writes go through (`sub.clear()` deletes the range from the parent in one `arraycopy`); structural change to the parent invalidates the view -> CME on use (observed); holds the parent alive.

**M28. Is PriorityQueue iteration sorted?**
No: iteration/`toString` follow the array of the binary heap (observed `[1, 3, 2, 5, 9, 8]`); only `poll()` order is sorted. To get sorted output either poll repeatedly or copy and sort.
`F:` Stable for equal priorities? -> no; add a sequence number.

**M29. PriorityQueue complexities.**
`offer/poll` O(log n); `peek` O(1); `remove(Object)`/`contains` O(n); construct from a collection O(n) (heapify).
`F:` How to do "decrease-key" for Dijkstra? -> insert a new entry and skip stale ones (lazy deletion); `remove` is O(n).

**M30. Top-K largest of a large stream.**
Min-heap capped at K: offer, and if size > K then poll the smallest. O(n log K), O(K) memory; heap root = K-th largest (observed `[9,10,11]`). For top-K frequent: count with HashMap then same heap by frequency.
`F:` Why not sort? -> O(n log n) and needs all data in memory. `F:` QuickSelect? -> O(n) average, mutating an array.

**M31. ArrayDeque vs LinkedList as a queue.**
ArrayDeque: circular array, no per-element node, better cache locality, faster; no nulls. LinkedList: nodes, allows nulls, also a List. Both implement Deque. Choose ArrayDeque unless you need nulls.
`F:` Why no nulls in ArrayDeque? -> null is the empty-slot sentinel and `poll()` returns null for empty.

**M32. When use CopyOnWriteArrayList?**
Read-mostly small lists needing lock-free iteration without CME (listeners). Every write copies the whole array: 50k appends took 777 ms vs 2 ms (observed). Snapshot iterators do not reflect later writes and do not support `remove`.
`F:` Alternatives for write-heavy? -> `ConcurrentLinkedQueue`, `Collections.synchronizedList` with copy under lock, or a lock-protected ArrayList.

**M33. Why can't ConcurrentHashMap hold nulls?**
Ambiguity between "absent" and "mapped to null" cannot be resolved atomically (`containsKey` then `get` is a race); Doug Lea chose to forbid it. Hashtable/`Map.of` also forbid nulls.

**M34. Is `synchronizedMap.put(k, get(k)+1)` thread-safe?**
Each call is atomic but the compound action is not: observed 33,794 of 160,000 increments survived. Use `ConcurrentHashMap.merge(k, 1, Integer::sum)` (atomic, 160,000) or `LongAdder` values.
`F:` Iteration over `synchronizedMap.keySet()`? -> must hold `synchronized (map)`.

**M35. Why is `(a, b) -> a - b` a bug?**
Integer overflow: `Integer.MAX_VALUE - (-5)` = `-2147483644`, i.e. wrong sign (observed a mis-sorted result and no exception). Use `Integer.compare` / `Comparator.comparingInt`.

**M36. Is HashMap iteration order guaranteed?**
No. It is bucket order: can change on resize (observed `33,17,1` order changed to `33,1 ... 17` after growth), across JDK versions, and for identity-hash keys between runs. `Set.of/Map.of` even randomise per JVM run. Use LinkedHashMap/TreeMap when order matters.

**M37. Is `Collections.sort` stable?**
Yes for objects (TimSort, a merge-sort hybrid; O(n log n), O(n) on sorted input); primitives use dual-pivot quicksort (stability moot). Enables multi-pass sorting (sort by secondary key first) - or use `thenComparing`.

**M38. Why use EnumMap / EnumSet?**
Array-indexed by ordinal / bitmask: no hashing or collisions, ordered by declaration, compact, fast; null keys rejected.

## 5.3 Hard (39-56)

**H39. All keys have the same hashCode: complexity in JDK 7 vs 17?**
JDK 7: one linked list, O(n) per op. JDK 8+: after the bin exceeds 8 nodes and table >= 64, a red-black tree ordered by hash then `compareTo` (if `Comparable<K>`): O(log n). If keys are **not** Comparable, ties are broken by `identityHashCode` for insertion, but lookups must search **both subtrees**: O(n) (observed 1.2 s vs 4 ms at 10k keys).
`F:` So does treeification save me from a bad hash? -> only for Comparable keys. `F:` HashDoS on Strings? -> String is Comparable, so the attack degrades to O(log n) per op.

**H40. Why is MIN_TREEIFY_CAPACITY 64 and why resize first?**
A long chain in a small table is more likely caused by table crowding than by bad hashing; doubling is cheaper than building a tree, and treeified nodes cost roughly twice the memory. So resize until 64, then treeify (observed 16 -> 32 -> 64 -> tree). I state the rationale from the implementation comments; the number 64 is empirical.

**H41. Justify 0.75 and the 8/6 thresholds.**
JDK comment: bin sizes ~ Poisson with parameter ~0.5 for load 0.75; probabilities 0.6065, 0.3033, 0.0758, 0.0126, 0.00158, 0.00016, 0.000013, 0.00000094, 0.00000006 for 0..8; so 8 is a "should never happen with a good hash" threshold. 6 < 8 gives hysteresis. 0.75 balances space vs collision cost; 1.0 would save ~25% table memory with longer chains.

**H42. Explain the JDK 7 infinite loop.**
`transfer()` re-inserted nodes at the head of the new bucket, reversing chain order. Two threads resizing the same map could each reverse parts of a chain and link A.next=B and B.next=A - a cycle; `get` on that bucket spins forever (100% CPU). JDK 8 preserves order with lo/hi lists and appends at the tail, removing that specific bug - but the map is still not thread-safe (lost updates, wrong size).
`F:` So can I use HashMap concurrently on JDK 17 if writes are rare? -> no; use CHM or safe publication of an immutable map.

**H43. Design a thread-safe LRU cache.**
Options: (a) `Collections.synchronizedMap(new LRU<>(max))` - correct but one lock (even `get` mutates); (b) segmented LRUs by `hash % N`, each synchronised; (c) `ConcurrentHashMap` + a concurrent recency structure with buffered reads (what Caffeine does: read buffers + write buffers replayed by one maintenance thread, TinyLFU admission); (d) `ReentrantLock` around a `LinkedHashMap` for compound `get-or-load`. Address stampede (`computeIfAbsent` per key, do not do I/O under the map lock for CHM), TTL, weights, metrics.
`F:` Why is `ConcurrentHashMap.computeIfAbsent` with slow loader risky? -> it blocks other writers to the same bin and can deadlock on recursion.

**H44. Capacity arithmetic: how many resizes for 1,000,000 puts into `new HashMap<>()`? into `new HashMap<>(1_000_000)`?**
Default: sizes 16 -> 2^21 = 2,097,152: 17 doublings (observed) with about 1.57 M nodes moved. `new HashMap<>(1_000_000)`: `tableSizeFor` = 2^20 = 1,048,576, threshold 786,432 -> one resize at 786,433 to 2^21. `HashMap.newHashMap(1_000_000)`: 2^21 immediately, no resize. (Default and `newHashMap` cases observed in Demo 5; the middle case follows the same rule as the observed `new HashMap<>(1000)` -> 1024 then 2048.)
`F:` Why not `new HashMap<>(1_500_000)`? -> that rounds to 2^21 also.

**H45. How are TreeNodes ordered when hash equal and keys not comparable?**
`tieBreakOrder`: compare `getClass().getName()`, then `System.identityHashCode`. This gives a total order for insertion only; lookup cannot use it (the searching key's identity is unrelated), so `find` explores both children.
`F:` Which classes count as comparable? -> exactly `class C implements Comparable<C>` (checked reflectively by `comparableClassFor`); a subclass of a class that implements `Comparable<Parent>` does not qualify.

**H46. Red-black vs AVL; what invariants does TreeMap keep?**
RB: node colours, root black, no red-red, equal black height -> height <= 2 log2(n+1); at most 2 rotations on insert, 3 on delete, lower write cost. AVL: balance factor within 1 -> shallower (<= 1.44 log2 n) so faster reads, more rotations. TreeMap is RB; iteration is in-order successor traversal (amortised O(1) per step, O(log n) for the first).
`F:` Why not a hash map for range queries? -> no order.

**H47. Why is building a heap from n items O(n) and not O(n log n)?**
Heapify runs siftDown from the last parent (n/2 - 1) to the root. Nodes at height h cost O(h) and there are about n/2^(h+1) of them: sum h * n / 2^(h+1) converges to O(n). Observed: `new PriorityQueue<>(List.of(9..1))` -> `[1,2,3,6,5,4,7,8,9]`. n separate `offer`s are O(n log n) in the worst case (each siftUp can climb log n levels).
`F:` `addAll` on an existing PQ? -> n offers.

**H48. When does TimSort throw "Comparison method violates its general contract!"?**
When merging runs it detects an inconsistency (e.g. `compare(a,b) > 0` and `compare(b,a) > 0`, or non-transitivity). Only detected with >= 32 elements (below that: binary insertion sort, no check) and only for some data. Observed: `(x,y) -> x<y ? -1 : 1` threw in 5/5 trials on 5,000 elements, random comparator threw; 200 random elements did not throw. Fix: correct comparator (return 0 for equal, use `Integer.compare`), never randomise inside `compare` (use `Collections.shuffle`).
`F:` Does it apply to `TreeMap`? -> no exception; silent malfunction.

**H49. What are the rules for `ConcurrentHashMap.compute/merge/computeIfAbsent` functions?**
They are atomic per key but execute under the bin lock: keep them short, no blocking I/O (other writers to that bin wait), never modify the same map (can deadlock; JDK 9+ may detect `IllegalStateException: Recursive update`). Return `null` from `compute/merge/computeIfPresent` removes the mapping. `merge` requires non-null value.
`F:` Difference between `putIfAbsent(k, expensive())` and `computeIfAbsent(k, f)`? -> the first evaluates the value eagerly even when present.

**H50. What is weakly consistent iteration, exactly?**
The iterator reflects the state at some point at or after creation, may or may not see concurrent updates, never throws CME, never yields an element twice, and never skips elements that were there for the whole traversal. (Contrast: snapshot = COW; fail-fast = ArrayList/HashMap.) `size()` and `isEmpty()` on CHM are estimates.

**H51. WeakHashMap "cache" does not free memory - why?**
Values are strong; if a value references its key the entry is never weakly reachable. Keys that are interned literals or cached boxed values are never collected. Cleanup is lazy (on map access) and timing is GC-dependent; there is no size limit. Use it only to attach metadata to objects whose lifetime you do not control; for caches use Caffeine with weak/soft references deliberately.

**H52. What did Java 21 change for collections?**
Sequenced interfaces (`SequencedCollection/Set/Map`): uniform `getFirst/getLast/addFirst/addLast/removeFirst/removeLast/reversed`; `List`, `Deque`, `LinkedHashSet`, `SortedSet`, `LinkedHashMap`, `SortedMap` retrofitted; `reversed()` returns a view (observed `reversed().addFirst(99)` appended at the end of the original). `HashSet/HashMap` are not sequenced. Gotchas: immutable lists and `TreeSet.addFirst` -> UOE; `getFirst()` on empty -> `NoSuchElementException`.

**H53. Estimate the memory of `HashMap<Integer,Integer>` with 10 M entries.**
Approximate (64-bit, compressed oops): Node 32 B + key `Integer` 16 B + value `Integer` 16 B + table slot ~4 B/(load 0.4-0.75) ~ 6-10 B: about 70 B per entry -> roughly 0.7 GB (2^24-slot table = 64 MB of it). `Integer` cache only covers -128..127. Alternatives: `long[]`/primitive maps (fastutil, Eclipse Collections, Trove) at 8-16 B per entry, or a sorted parallel-array structure.
`F:` What about TreeMap? -> entry 40 B + boxed key/value: ~ 72-80 B/entry, slower.

**H54. Design: "top 10 most frequent words in a 100 GB log".**
Stream words through a `HashMap<String, Integer>` with `merge(w, 1, Integer::sum)` if the vocabulary fits memory; then a size-10 min-heap over entries (O(V log 10)). If the vocabulary does not fit: partition by hash into files (each word entirely in one partition), count per partition, keep a global top-10 heap; or count-min sketch + heavy-hitter heap for approximate answers. Discuss `TreeMap` alternative (sorted words) and why HashMap is right.

**H55. What is the actual difference between `HashMap.computeIfAbsent` and `ConcurrentHashMap.computeIfAbsent` regarding recursion and concurrency?**
HashMap (JDK 9+): if the mapping function modifies the map, it throws `ConcurrentModificationException` (detected via `modCount`); before that the behaviour was undefined (could corrupt or loop; the classic recursive Fibonacci memo bug). ConcurrentHashMap: function must not update the map (may deadlock or throw `IllegalStateException`), and it is atomic and blocks other updates to that bin while running. Neither is safe for an expensive/recursive loader.

**H56. A senior asks: "Would you ever iterate a HashMap in a hot loop?" (capacity vs size)**
Iteration is O(capacity + size), and the table never shrinks: a map that once held 10 M entries and now holds 100 still scans 16 M+ slots per iteration. Options: rebuild (`new HashMap<>(old)`), use LinkedHashMap (iteration O(size)), or restructure to avoid scanning.

## 5.4 Common wrong answers - quick list

| Claim | Reality |
|---|---|
| "HashMap treeifies when a chain reaches 8" | needs 9th node **and** table >= 64; else resize |
| "ArrayList doubles" | 1.5x: 10, 15, 22, 33, ... |
| "ArrayList allocates 10 up front" | lazily on first add (JDK 8+) |
| "Removing in for-each always throws CME" | not for the second-to-last element (skips the last) |
| "Fail-fast guarantees thread safety" | best-effort bug detector |
| "ConcurrentHashMap size() is exact" | estimate under concurrency; use `mappingCount()` |
| "ConcurrentHashMap makes `get`-then-`put` safe" | only atomic methods (`merge`, `compute`) are |
| "unmodifiableList = immutable" | read-only view of a mutable list |
| "`new HashMap<>(n)` holds n entries without resize" | capacity n, threshold 0.75n |
| "PriorityQueue iterates in order" | heap array order |
| "LinkedList insert is O(1)" | O(1) only at a node you already hold |
| "TreeSet uses equals" | uses compare/compareTo |
| "TimSort is unstable / quicksort" | objects: stable TimSort; primitives: dual-pivot quicksort |
| "HashMap resize rehashes every key" | reuses stored hash, splits by one bit |
| "HashMap shrinks after removals" | never shrinks the table |
| "Arrays.asList is immutable" | fixed-size, `set` allowed, write-through |

---

# 6. Reference tables

## 6.1 Complexity (n = size; "amort." = amortised)

| Structure | get / contains | add / put | remove | Iteration | Notes |
|---|---|---|---|---|---|
| `ArrayList` | `get(i)` O(1); `contains` O(n) | end O(1) amort.; middle O(n) | by index O(n-i); by value O(n) | O(n) | array copy `arraycopy` |
| `LinkedList` | `get(i)` O(n); `contains` O(n) | ends O(1); middle O(n) to find + O(1) | ends O(1); middle O(n) | O(n), poor locality | Deque + List |
| `ArrayDeque` | `contains` O(n) | ends O(1) amort. | ends O(1) | O(n) | no nulls |
| `PriorityQueue` | `peek` O(1); `contains` O(n) | `offer` O(log n) | `poll` O(log n); `remove(o)` O(n) | O(n), heap order | heapify ctor O(n) |
| `HashMap/HashSet` | O(1) avg; O(log n) worst (Comparable keys, treeified); O(n) worst (non-Comparable, same hash) | O(1) avg; resize O(n) | O(1) avg | O(capacity + n) | table never shrinks |
| `LinkedHashMap/Set` | O(1) avg | O(1) avg | O(1) avg | **O(n)** | access order optional |
| `TreeMap/TreeSet` | O(log n) | O(log n) | O(log n) | O(n) in order | `floor/ceiling/subMap` O(log n) to locate |
| `EnumMap/EnumSet` | O(1) | O(1) | O(1) | O(#constants) | array / bitmask |
| `IdentityHashMap` | O(1) avg | O(1) avg | O(1) avg | O(capacity) | `==` semantics |
| `ConcurrentHashMap` | O(1) avg, lock-free reads | O(1) avg, per-bin lock/CAS | O(1) avg | O(n) weakly consistent | no nulls, `size()` estimate |
| `CopyOnWriteArrayList` | `get` O(1); `contains` O(n) | **O(n)** | **O(n)** | O(n) snapshot | read-mostly |
| `ArrayBlockingQueue` / `LinkedBlockingQueue` | | `put/offer` O(1) | `take/poll` O(1) | | locks |
| `Collections.sort` / `List.sort` | | | | O(n log n), O(n) if presorted | TimSort, stable, up to n/2 temp |

## 6.2 Memory per entry (approximate: 64-bit HotSpot, compressed oops, 12-byte object header; ignoring key/value objects unless stated)

| Structure | Per-entry cost (approximate) | Derivation |
|---|---|---|
| `ArrayList` | 4 B slot (x up to 1.5 for slack) | one reference per element |
| `LinkedList` | ~24 B node | header 12 + item 4 + next 4 + prev 4 |
| `ArrayDeque` / `PriorityQueue` | 4 B slot (+ slack) | array of refs |
| `HashMap` | ~32 B node + ~6-10 B table share = ~40 B | header 12 + hash 4 + key 4 + value 4 + next 4 = 28 -> 32 aligned; table 4 B per slot at load 0.375-0.75 |
| `HashSet` | ~40 B | same as HashMap (shared `PRESENT`) |
| `LinkedHashMap` | ~40 B node + table share = ~48 B | two more references (before/after) |
| `TreeMap` | ~40 B node | header 12 + key/value/left/right/parent 20 + colour 1 -> 40 aligned |
| Tree bin node (`TreeNode`) | ~56-64 B | Node + Entry (before/after) + parent/left/right/prev + red flag |
| `ConcurrentHashMap` | ~32 B node + table share | similar to HashMap node (`val` and `next` volatile) |
| Boxed `Integer` / `Long` / `Double` | 16 B each | header 12 + 4/8 payload (padded); small Integers -128..127 cached |
| `String` key | 24 B String + array (16 B header/length + bytes, padded) | Latin-1 compact strings JDK 9+ |

Example: `HashMap<Integer,Integer>` of 10 M entries is roughly 40 B + 2 x 16 B = ~72 B per entry, about 0.7 GB (approximate). A primitive `long[]` pair structure or a primitive-collections library uses 8-16 B per entry.

## 6.3 Choosing the right collection (decision tree)

```
Need key -> value lookup?
|-- yes
|   |-- Multi-threaded shared, concurrent updates?          -> ConcurrentHashMap (merge/compute for counters and check-then-act)
|   |-- Need sorted keys / range / nearest key?             -> TreeMap (ConcurrentSkipListMap if concurrent)
|   |-- Need iteration in insertion order or LRU eviction?  -> LinkedHashMap (access order + removeEldestEntry)
|   |-- Keys are an enum?                                   -> EnumMap
|   |-- Identity (==) semantics or graph traversal?         -> IdentityHashMap
|   |-- Metadata attached to objects with GC lifetime?      -> WeakHashMap (not a cache)
|   |-- Read-only lookup table?                             -> Map.of / Map.copyOf (immutable, unordered, no nulls)
|   `-- otherwise                                           -> HashMap (pre-size with HashMap.newHashMap(n) if n known)
|-- no: a collection of elements
    |-- Unique elements?
    |     |-- sorted / range / floor-ceiling                -> TreeSet
    |     |-- insertion order                               -> LinkedHashSet
    |     |-- enum                                          -> EnumSet
    |     |-- concurrent, read-mostly                       -> CopyOnWriteArraySet / ConcurrentHashMap.newKeySet()
    |     `-- otherwise                                     -> HashSet
    |-- Access by position, mostly reads / appends          -> ArrayList
    |-- Stack or queue (single thread)                      -> ArrayDeque (null needed -> LinkedList)
    |-- Repeatedly take the smallest/largest                -> PriorityQueue (top-K, merge-K, scheduling)
    |-- Producer/consumer between threads                   -> BlockingQueue (bounded ArrayBlockingQueue; LinkedBlockingQueue with explicit capacity)
    |-- Listener list, iterate often, mutate rarely         -> CopyOnWriteArrayList
    `-- Immutable snapshot for callers                      -> List.copyOf / Set.copyOf / stream.toList()
```
Defaults: `ArrayList`, `HashMap`, `HashSet`, `ArrayDeque`; move off them only for order, sorting, concurrency, or memory reasons you can name.

## 6.4 Streams vs loops on collections (quick note)

- Streams are lazy pipelines; the source is not modified. Modifying the source during a terminal operation is a bug (CME or undefined). `Collectors.toList()` -> unspecified mutable list; `stream.toList()` (16+) -> unmodifiable (allows nulls); `toMap` throws `IllegalStateException: Duplicate key` without a merge function; `groupingBy` builds `HashMap`s (choose `TreeMap::new` or `LinkedHashMap::new` via the map-factory overload for ordered output); `sorted()` is stable on ordered streams.
- `parallelStream()` splits well on `ArrayList`, arrays, `IntStream.range`; badly on `LinkedList`, `HashSet`, `Stream.iterate`; it uses the common ForkJoinPool - shared with everything else in the JVM; avoid for I/O or tiny workloads; the reduction functions must be associative and stateless.
- Boxed streams (`Stream<Integer>`) allocate; prefer `IntStream`/`mapToInt(...)` for numbers.
- Performance rule of thumb: for small collections in hot code paths, a plain loop is usually the same speed or faster and easier to profile; for clarity of filter/map/group logic streams win. Do not choose on speed without measuring (JMH).
- Idioms: counting `Collectors.groupingBy(f, Collectors.counting())`; index a list `Collectors.toMap(Item::id, Function.identity())`; removal -> `removeIf` (in place) vs `filter` (new list).

---

# 7. One-page cheat sheet

**HashMap**
- `hash = h ^ (h>>>16)`; `idx = (n-1) & hash`; `n` = power of 2; default n=16, load 0.75, threshold = n*0.75; `resize` when `++size > threshold` (13th put for n=16).
- Table lazily allocated; never shrinks; iteration O(capacity + size); tail insertion (JDK 8+).
- `putVal`: empty bin -> insert; else compare `hash`, `==`, `equals`; tree bin -> tree insert; chain -> append, if chain had 8 nodes then treeifyBin; existing key -> replace, no modCount++.
- Treeify: bin has 8 and a 9th arrives **and** table >= 64; else `resize()`. Untreeify: split half <= 6, or tiny tree on remove.
- Resize: double; node stays at `j` if `(hash & oldCap)==0` else moves to `j+oldCap`; order preserved; no rehash.
- Tree bin order: hash, then `compareTo` (only if `Comparable<same class>`), then class name + identityHashCode. Bad hash + non-Comparable => O(n).
- Pre-size: `HashMap.newHashMap(n)` (JDK 19+) = capacity `ceil(n/0.75)`; `new HashMap<>(n)` is **not** "room for n".
- Null key -> bucket 0; `get==null` is ambiguous; mutable keys get lost (entry reachable only by iteration).
- Poisson (lambda ~0.5): chain 8 ~ 6e-8 => treeify is a defence, not the norm.

**Fail-fast**: `modCount` snapshot; only `next()` checks; ArrayList `hasNext = cursor != size` => second-to-last removal silently ends the loop (skips last). Fix: `Iterator.remove`, `removeIf`.

**ArrayList**: lazy 10; new = old + max(needed, old>>1) => 10,15,22,33,49,73,109; `remove(int)` vs `remove(Object)`; `subList` is a view (CME after parent structural change); `Arrays.asList` fixed-size write-through; `List.of` no nulls (even `contains(null)`); `unmodifiableList` is a live view; `List.copyOf` a snapshot.

**LinkedList**: 24 B node, O(n) get; usually slower than ArrayList even for middle inserts; use `ArrayDeque` for stack/queue (no nulls).

**PriorityQueue**: binary heap in array; parent `(k-1)>>>1`, children `2k+1, 2k+2`; `offer/poll` O(log n), `peek` O(1), `remove/contains` O(n), heapify O(n); iteration = array order; unstable; top-K = min-heap of size K.

**LinkedHashMap**: HashMap + doubly linked list; access order + `removeEldestEntry` = LRU; `get` is structural in access order; iteration O(size).

**TreeMap**: red-black (height <= 2 log2(n+1)); uses `compare` only (make it consistent with equals!); `floor <=`, `ceiling >=`, `lower <`, `higher >`; `subMap(a,b)` = `[a,b)`, live views; no null keys.

**Concurrent**: CHM no nulls, atomic `merge/compute/putIfAbsent`, weakly consistent iterators, `size()` estimate, compound `get`+`put` still racy (observed 121071/160000); `synchronizedMap`/`synchronizedList` need manual sync for iteration; COWAL snapshot iterators, O(n) writes (777 ms vs 2 ms for 50k adds); `LinkedBlockingQueue` default unbounded.

**Comparators**: never subtract (`Integer.compare`); consistent + transitive or TimSort throws "Comparison method violates its general contract!" (only with >= 32 elements); TimSort stable (objects).

**Java 21**: `SequencedCollection/Set/Map`: `getFirst/getLast/addFirst/addLast/reversed`; List, Deque, LinkedHashSet, SortedSet, LinkedHashMap, SortedMap; not HashSet/HashMap.

**Pick**: default ArrayList / HashMap / HashSet / ArrayDeque; TreeMap for order/range; LinkedHashMap for order/LRU; EnumMap for enums; CHM for concurrency; PriorityQueue for "next best"; CopyOnWriteArrayList for listeners.

---

## Appendix: how these demos were produced

Sources are simple single-file programs (`D1_Buckets`, `D2_BadHash`, `D3_Lists`, `D4_Maps`, `D5_Misc`, `D6_Extra`, `D7_Resizes`, `D8_Dump`) compiled with `javac` and run with `java` on JDK 21.0.4 (HotSpot 64-bit, Windows). Programs that peek at `HashMap.table` or `ArrayList.elementData` used `--add-opens java.base/java.util=ALL-UNNAMED`. Timing numbers are single-run, laptop, JIT-warmup affected - compare ratios only. Not verified by run: `Hashtable.get(null)` NPE, JDK-7 infinite-loop mechanics (described from the JDK 7 `transfer()` algorithm), G1 humongous-allocation detail in war story 4.2, exact TreeNode/`ArrayDeque` growth constants (described conceptually).
