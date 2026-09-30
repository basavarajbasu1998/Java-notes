# Coding Problems: Collections / HashMap

## C1. Find duplicates in a list
```java
Set<Integer> seen = new HashSet<>();
Set<Integer> dups = list.stream().filter(x -> !seen.add(x)).collect(Collectors.toSet());
```
(`add` returns false if already present.)

## C2. Group anagrams ⭐
Key = sorted letters.
```java
static List<List<String>> groupAnagrams(String[] words) {
    Map<String, List<String>> m = new HashMap<>();
    for (String w : words) {
        char[] c = w.toCharArray(); Arrays.sort(c);
        m.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(w);
    }
    return new ArrayList<>(m.values());
}
```

## C3. LRU Cache ⭐⭐⭐ (very common)
**Idea:** HashMap for O(1) lookup + doubly linked list order (most recent at head). `LinkedHashMap` with `accessOrder=true` does it for us:
```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LRUCache(int capacity) { super(capacity, 0.75f, true); this.capacity = capacity; }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) { return size() > capacity; }
}
```
Flow:
```
get(k)  → found? move node to "most recent" end
put(k)  → insert at most-recent; size > capacity? remove eldest (least recent)
```
Interviewer may ask to **build from scratch** (HashMap<Integer,Node> + doubly linked list with dummy head/tail):
```java
class LRU {
    class Node { int k, v; Node prev, next; Node(int k, int v){this.k=k;this.v=v;} }
    private final int cap; private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0,0), tail = new Node(0,0);
    LRU(int cap) { this.cap = cap; head.next = tail; tail.prev = head; }

    int get(int k) {
        Node n = map.get(k); if (n == null) return -1;
        remove(n); addFront(n); return n.v;
    }
    void put(int k, int v) {
        Node n = map.get(k);
        if (n != null) { n.v = v; remove(n); addFront(n); return; }
        if (map.size() == cap) { Node lru = tail.prev; remove(lru); map.remove(lru.k); }
        n = new Node(k, v); map.put(k, n); addFront(n);
    }
    private void remove(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }
    private void addFront(Node n) { n.next = head.next; n.prev = head; head.next.prev = n; head.next = n; }
}
```
Thread-safe? wrap with `Collections.synchronizedMap` or use lock; for production use Caffeine.

## C4. Implement your own simple HashMap (concept)
```
array of buckets → index = (n-1) & hash(key)
put: index → bucket empty? add node : walk list, equals(key)? replace : append
get: index → walk list comparing equals
resize when size > 0.75 × capacity (double + rehash)
```

## C5. Sort a Map by value / HashMap of employees
```java
map.entrySet().stream()
   .sorted(Map.Entry.<String,Integer>comparingByValue().reversed())
   .forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));
```

## C6. Top K frequent elements ⭐ (min-heap)
```java
static List<Integer> topK(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    PriorityQueue<Map.Entry<Integer, Integer>> pq = new PriorityQueue<>(Map.Entry.comparingByValue());
    for (var e : freq.entrySet()) { pq.offer(e); if (pq.size() > k) pq.poll(); }
    List<Integer> r = new ArrayList<>();
    while (!pq.isEmpty()) r.add(pq.poll().getKey());
    Collections.reverse(r);
    return r;
}
```
O(n log k).

## C7. Custom object as HashMap key (equals + hashCode)
```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
}
```

## C8. Sort employees by salary desc then name asc
```java
list.sort(Comparator.comparingDouble(Employee::getSalary).reversed()
                    .thenComparing(Employee::getName));
```

## C9. Remove elements while iterating (avoid ConcurrentModificationException)
```java
list.removeIf(x -> x % 2 == 0);                 // best
Iterator<Integer> it = list.iterator();
while (it.hasNext()) if (it.next() % 2 == 0) it.remove();
```
