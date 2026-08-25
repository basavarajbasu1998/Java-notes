# Java Collections — Interview Q&A (5–6 Years Experience)

---

## 1. ArrayList vs LinkedList

| Aspect | ArrayList | LinkedList |
|---|---|---|
| Internal structure | Dynamic array (Object[]) | Doubly linked list |
| get(index) | O(1) | O(n) |
| add/remove at end | O(1) amortized | O(1) |
| add/remove at start/middle | O(n) — shifting | O(1) once you have the node, O(n) to find it |
| Memory | Less overhead (contiguous array) | More overhead (each node stores prev/next pointers) |
| Implements | List, RandomAccess | List, Deque |

**Interview line:** "ArrayList is backed by a resizable array, so random access is O(1) but insertion/deletion in the middle is O(n) due to shifting. LinkedList is a doubly linked list — insertion/deletion is O(1) once you're at the node, but access is O(n) since you have to traverse. In practice, ArrayList wins in almost all real-world cases because of CPU cache locality; LinkedList is rarely used today except as a Deque."

---

## 2. How does ArrayList grow?

- Default initial capacity: **10** (only allocated on first `add()`, not at construction, since Java 7+).
- When full, it grows via `grow()`:
  ```
  newCapacity = oldCapacity + (oldCapacity >> 1)   // 1.5x growth
  ```
- Internally uses `Arrays.copyOf()` to copy elements into a new, larger array — this copy is **O(n)**.
- Because growth happens infrequently (geometric growth), the **amortized** cost of `add()` is O(1).

**Interview line:** "It grows by 1.5x, not doubling like some other languages. Resize is O(n) because of array copy, but since it happens exponentially less often as the list grows, the amortized cost per add stays O(1)."

---

## 3. Why is ArrayList get() O(1)?

Because the underlying array is contiguous in memory, and `get(index)` is just:
```java
return elementData[index];
```
It's a direct memory offset calculation (`base_address + index * element_size`) — no traversal needed. This is the definition of **random access**, which is why ArrayList implements the `RandomAccess` marker interface.

---

## 4. How does HashSet work internally?

- `HashSet` is backed by a `HashMap` internally.
- Every element you add becomes a **key** in that internal HashMap.
- The value is a dummy constant object: `private static final Object PRESENT = new Object();`
- So `set.add(x)` internally calls `map.put(x, PRESENT)`.
- Uniqueness, hashing, and null-handling all behave exactly like HashMap because it *is* a HashMap under the hood.

---

## 5. How does HashMap work internally?

- Backed by an **array of buckets** (`Node<K,V>[] table`).
- On `put(key, value)`:
  1. Compute `hash(key)` (see Q6/Q7).
  2. Find bucket index: `index = (n - 1) & hash` (n = table length, always a power of 2).
  3. If bucket is empty → insert new node.
  4. If bucket has entries → traverse chain, check `equals()` for duplicate key, else append.
- On `get(key)`: compute hash → find bucket → traverse chain/tree comparing `equals()` until match found.
- Default initial capacity: **16**, default load factor: **0.75**.
- Table size is always a **power of 2** so that `(n-1) & hash` behaves like a fast modulo.

---

## 6. What is a hash collision?

When **two different keys** produce the **same bucket index** (either same hashCode, or different hashCodes that map to the same index after masking). Since the array has finite buckets but potentially infinite keys, collisions are inevitable (pigeonhole principle). HashMap handles them by chaining entries in the same bucket (linked list or tree).

---

## 7. How does Java 8 HashMap handle collision?

- **Before Java 8:** collisions in a bucket were stored as a **linked list** → O(n) worst case for get/put if many collisions.
- **Java 8+ improvement:**
  - Still uses a linked list for small chains.
  - If a bucket's chain length exceeds **`TREEIFY_THRESHOLD = 8`** AND the table capacity is at least **`MIN_TREEIFY_CAPACITY = 64`**, the chain is converted into a **Red-Black Tree**.
  - This changes worst-case lookup from **O(n) → O(log n)**.
  - If the bucket shrinks below **`UNTREEIFY_THRESHOLD = 6`** (during resize), it's converted back to a linked list.
- Also, Java 8 changed the hash spreading function: `hash = (h = key.hashCode()) ^ (h >>> 16)` — XORs the high bits into the low bits to reduce collisions for poor hashCode implementations.

**Interview line:** "Java 8 added treeification — if a single bucket gets more than 8 collisions and the table has at least 64 buckets, that bucket's linked list converts to a red-black tree, bringing worst-case lookup down from O(n) to O(log n). This protects against hash-flooding attacks and poor hashCode implementations."

---

## 8. What is load factor?

Load factor = threshold ratio that decides **when to resize** the HashMap.

```
threshold = capacity * loadFactor
```

Default is **0.75**, default capacity is **16**, so resize triggers at **12 entries**.

It's a trade-off knob:
- **High load factor** (e.g., 1.0) → less memory wasted, but more collisions → slower lookups.
- **Low load factor** (e.g., 0.5) → fewer collisions, faster lookups, but more memory used and more frequent resizing.

---

## 9. Why 0.75?

It's the sweet spot from the time-vs-space trade-off:
- Java's designers benchmarked it as the **best balance** between memory utilization and collision rate for typical (well-distributed) hashCodes.
- Using Poisson distribution math, at load factor 0.75, the expected chain length per bucket is very low (~0.5), keeping lookups close to O(1).
- Going higher (e.g., 0.9) saves memory but noticeably increases collision chains.
- Going lower (e.g., 0.5) wastes ~33% more memory for negligible performance gain.

**Interview line:** "0.75 is empirically the best trade-off Java engineers found — it keeps average bucket occupancy low enough for near-O(1) access while not wasting too much array space."

---

## 10. What happens during HashMap resize?

1. Triggered when `size > threshold` (capacity × loadFactor) after an insert.
2. Capacity **doubles** (new array of size `2 * oldCapacity`).
3. Every existing entry must be **rehashed and redistributed** into the new table.
4. **Java 8 optimization:** Instead of recomputing `hash % newCapacity` for every node, it uses the fact that capacity is a power of 2. Each old bucket splits into exactly **two** new buckets:
   - `lowIndex` = same index as before
   - `highIndex` = `oldIndex + oldCapacity`
   - Decided by checking one extra bit: `(hash & oldCapacity) == 0` → stays at low, else moves to high.
5. This avoids full rehashing and is done in-place per bucket — O(n) total but very efficient, and it **preserves relative order** within split chains.
6. Resize is expensive (O(n)) — this is why specifying an initial capacity upfront (if size is known) is a common optimization to avoid multiple resizes.

---

## 11. Why should equals() and hashCode() be overridden together?

The **hashCode-equals contract** (from `Object` class docs):
1. If two objects are `equal()`, they **must** have the same `hashCode()`.
2. If two objects have the same `hashCode()`, they are **not required** to be equal (collision is allowed).
3. Equal objects must consistently return equal hashCodes across multiple invocations (as long as object state doesn't change).

**If you override only `equals()`:**
- Two "equal" objects may land in **different buckets** (different default hashCode from `Object`, based on memory address).
- HashMap/HashSet won't recognize them as duplicates → you get **duplicate entries** in a Set, or `map.get(key)` returns `null` even though an "equal" key exists.

**If you override only `hashCode()`:**
- Objects may land in the same bucket, but `equals()` (default = reference equality) will say they're different → wasted collision but no correctness issue directly, though behavior is still broken for logical equality use-cases.

**Interview line:** "Break the contract and hash-based collections silently misbehave — you can insert 'duplicate' objects into a HashSet, or fail to retrieve a value with a logically-equal key. Always override both, and keep them consistent with the same fields."

---

## 12. What happens if a HashMap key is mutable?

If you mutate a key **after** inserting it (changing a field that's part of `hashCode()`/`equals()`):
- The key's hashCode changes.
- But the entry is still sitting in the **old bucket** (computed from the old hashCode) — HashMap doesn't rehash on mutation.
- Result: `map.get(mutatedKey)` returns `null` because it now computes a different bucket index, even though the object is technically still "in" the map.
- The entry becomes **unreachable/orphaned** — a memory leak, and `containsKey()` also fails.

**Interview line:** "This is exactly why HashMap keys should be immutable — String, Integer, or custom immutable classes. If you must use a mutable object as a key, never mutate the fields used in hashCode/equals after insertion, or you effectively lose that entry."

---

## 13. HashMap vs Hashtable

| Aspect | HashMap | Hashtable |
|---|---|---|
| Thread safety | Not synchronized | Synchronized (every method) |
| Performance | Faster (no locking overhead) | Slower (legacy, coarse locking) |
| Null keys/values | 1 null key, multiple null values allowed | No nulls allowed (throws NPE) |
| Iteration | Fail-fast iterator | Uses **Enumerator** (not fail-fast) |
| Introduced | Java 1.2 (Collections Framework) | Java 1.0 (legacy) |
| Recommended alternative | — | Use `ConcurrentHashMap` for thread safety |

**Interview line:** "Hashtable is legacy and synchronizes every single method call, making it slow and mostly obsolete. If you need thread safety today, use ConcurrentHashMap, not Hashtable."

---

## 14. HashMap vs ConcurrentHashMap

| Aspect | HashMap | ConcurrentHashMap |
|---|---|---|
| Thread safety | None | Thread-safe |
| Locking mechanism | N/A | Java 7: segment-based locking (16 segments). Java 8+: bucket-level locking via CAS + synchronized on bucket head node |
| Null keys/values | Allowed | **Not allowed** (avoids ambiguity in concurrent reads) |
| Iterator | Fail-fast (throws ConcurrentModificationException) | **Weakly consistent** — doesn't throw CME, may or may not reflect concurrent updates |
| Performance under concurrency | Needs external sync (`Collections.synchronizedMap`) → full lock, poor scalability | High concurrency — only locks the specific bucket being modified |

**Interview line:** "ConcurrentHashMap achieves thread safety without locking the entire map. In Java 8, it moved from segment locking to finer-grained per-bucket locking using synchronized blocks on the first node, combined with CAS operations for lock-free reads — reads generally don't block at all."

---

## 15. HashMap vs LinkedHashMap vs TreeMap

| Aspect | HashMap | LinkedHashMap | TreeMap |
|---|---|---|---|
| Ordering | No guaranteed order | **Insertion order** (or access order if configured) | **Sorted order** (natural or via Comparator) |
| Underlying structure | Array + linked list/tree buckets | HashMap + doubly linked list running through entries | Red-Black Tree |
| get/put complexity | O(1) avg | O(1) avg | O(log n) |
| Null keys | 1 allowed | 1 allowed | Not allowed (NPE — needs comparison) |
| Use case | Fast lookup, order doesn't matter | LRU cache (access-order mode), predictable iteration | Sorted data, range queries (`firstKey`, `subMap`, etc.) |

**Interview line:** "Use HashMap by default. Use LinkedHashMap when you need predictable iteration order or want to build an LRU cache — it has a built-in access-order constructor that pairs perfectly with `removeEldestEntry()`. Use TreeMap when you need sorted keys or range operations."

---

## 16. HashSet vs LinkedHashSet vs TreeSet

Same relationship as Q15, but for Sets:

| Aspect | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Backed by | HashMap | LinkedHashMap | TreeMap (NavigableMap) |
| Order | None | Insertion order | Sorted order |
| Performance | O(1) avg | O(1) avg | O(log n) |
| Use case | Fast uniqueness check | Predictable iteration + uniqueness | Sorted unique elements, range queries |

---

## 17. Comparable vs Comparator

| Aspect | Comparable | Comparator |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `int compareTo(T o)` | `int compare(T o1, T o2)` |
| Where defined | Inside the class itself (single natural ordering) | External class/lambda (multiple orderings possible) |
| Modifies original class? | Yes, class implements it | No, doesn't touch the original class |
| Use case | Default/natural sort order (e.g., Integer, String) | Custom sort logic, multiple sort strategies, sorting classes you can't modify |

```java
// Comparable — natural order, defined once in the class
class Employee implements Comparable<Employee> {
    public int compareTo(Employee o) { return this.id - o.id; }
}

// Comparator — external, flexible, composable (Java 8+)
Comparator<Employee> bySalary = Comparator.comparing(Employee::getSalary);
Comparator<Employee> bySalaryThenName = bySalary.thenComparing(Employee::getName);
```

**Interview line:** "Comparable defines one natural ordering baked into the class. Comparator lets you define as many external, swappable orderings as you want — and with Java 8's `Comparator.comparing().thenComparing().reversed()` chains, it's the more flexible, commonly-used approach in real code."

---

## 18. Iterator vs ListIterator

| Aspect | Iterator | ListIterator |
|---|---|---|
| Direction | Forward only | **Bidirectional** (forward and backward) |
| Applicable to | All Collections (List, Set, Map's keySet, etc.) | **Only List** implementations |
| Modify during iteration | `remove()` only | `remove()`, `add()`, and `set()` (replace element) |
| Index access | No | Yes — `nextIndex()`, `previousIndex()` |

```java
ListIterator<String> it = list.listIterator();
while (it.hasNext()) {
    String val = it.next();
    if (val.equals("x")) it.set("y"); // modify in place — not possible with Iterator
}
```

---

## 19. Fail-fast vs concurrent collection iteration

**Fail-fast** (ArrayList, HashMap, HashSet, etc.):
- Uses an internal `modCount` (modification counter).
- Iterator checks `modCount` on each `next()` call; if it changed since iteration started (i.e., someone did structural modification outside the iterator), it throws `ConcurrentModificationException`.
- Fails **fast**, not guaranteed — it's a best-effort detection mechanism, not a strict guarantee (per Javadoc).
- Happens even in single-threaded code: e.g., calling `list.remove(x)` directly inside a for-each loop.

**Fail-safe / weakly consistent** (ConcurrentHashMap, CopyOnWriteArrayList):
- Doesn't throw CME.
- Either iterates over a **snapshot/copy** of the data (CopyOnWriteArrayList) or is **weakly consistent** — reflects some but not necessarily all concurrent modifications (ConcurrentHashMap).
- Safe for concurrent read/write, but you might not see the very latest updates during iteration.

**Interview line:** "Fail-fast collections detect structural modification via a modCount check and throw CME — it's a debugging aid, not a concurrency guarantee. Fail-safe collections like CopyOnWriteArrayList iterate over a snapshot taken at iterator-creation time, so they never throw CME but may show stale data."

---

## 20. ArrayList vs CopyOnWriteArrayList

| Aspect | ArrayList | CopyOnWriteArrayList |
|---|---|---|
| Thread safety | Not thread-safe | Thread-safe |
| Write mechanism | Direct mutation of array | **Copies the entire underlying array** on every write (`add`, `remove`, `set`) |
| Read performance | Fast, no locking | Fast, **no locking at all** (reads happen on the current immutable snapshot) |
| Write performance | Fast | **Expensive** — O(n) copy per write |
| Iterator behavior | Fail-fast (throws CME) | Iterates over a snapshot — **never throws CME**, doesn't reflect concurrent changes |
| Best use case | General purpose, single-threaded or externally synchronized | **Read-heavy, write-rare** scenarios in concurrent environments (e.g., listener lists, config caches) |

**Interview line:** "CopyOnWriteArrayList trades write performance for lock-free, blazing-fast reads by copying the whole backing array on every mutation. It's ideal for something like an event-listener list — added to rarely, iterated over constantly, and you never want a ConcurrentModificationException while notifying listeners."

---

## Quick Revision Cheat-Sheet

- **ArrayList** → array, O(1) get, O(n) insert/delete middle, grows 1.5x.
- **LinkedList** → doubly linked, O(1) insert/delete at known node, O(n) get.
- **HashMap** → array of buckets, hash + `(n-1)&hash` for index, treeifies at 8 collisions/64 capacity, resizes at 0.75 load factor by doubling.
- **HashSet** → HashMap in disguise with a dummy value.
- **equals+hashCode contract** → must override together or hash collections break.
- **Mutable keys** → never mutate hash-affecting fields post-insertion → orphaned entries.
- **Hashtable** → legacy, fully synchronized, no nulls.
- **ConcurrentHashMap** → bucket-level locking (Java 8+), no nulls, weakly consistent iterator.
- **LinkedHashMap** → HashMap + insertion/access order, great for LRU cache.
- **TreeMap/TreeSet** → Red-Black tree, sorted, O(log n).
- **Comparable** → one natural order inside class. **Comparator** → many external orders.
- **Iterator** → forward-only, remove() only. **ListIterator** → bidirectional, add/set/remove.
- **Fail-fast** → modCount check, throws CME. **CopyOnWriteArrayList** → snapshot iteration, no CME, expensive writes.
