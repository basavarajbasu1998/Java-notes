# Coding Problems: Arrays

## B1. Two Sum ⭐⭐⭐ (HashMap)
**Idea:** for each number, look for `target - number` among numbers already seen.
```
nums=[2,7,11,15], target=9
i=0: need 7, map={} → store 2→0
i=1: need 2, map has 2 → answer [0,1]
```
```java
static int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        Integer j = seen.get(target - nums[i]);
        if (j != null) return new int[]{j, i};
        seen.put(nums[i], i);
    }
    throw new IllegalArgumentException("no solution");
}
```
O(n) time, O(n) space (brute force O(n²)).

## B2. Maximum subarray sum (Kadane) ⭐
**Idea:** at each element, either extend the previous subarray or start fresh.
```java
static int maxSubArray(int[] a) {
    int cur = a[0], best = a[0];
    for (int i = 1; i < a.length; i++) {
        cur = Math.max(a[i], cur + a[i]);
        best = Math.max(best, cur);
    }
    return best;
}
```
`[-2,1,-3,4,-1,2,1,-5,4]` → 6 (`4,-1,2,1`).

## B3. Move zeros to end (in place)
```java
static void moveZeros(int[] a) {
    int pos = 0;
    for (int x : a) if (x != 0) a[pos++] = x;
    while (pos < a.length) a[pos++] = 0;
}
```

## B4. Remove duplicates from sorted array (in place)
```java
static int removeDup(int[] a) {
    if (a.length == 0) return 0;
    int k = 1;
    for (int i = 1; i < a.length; i++) if (a[i] != a[i - 1]) a[k++] = a[i];
    return k;                        // new length
}
```

## B5. Rotate array right by k
Reverse whole array, then reverse first k, then reverse rest.
```java
static void rotate(int[] a, int k) {
    k %= a.length;
    reverse(a, 0, a.length - 1); reverse(a, 0, k - 1); reverse(a, k, a.length - 1);
}
static void reverse(int[] a, int i, int j) { while (i < j) { int t = a[i]; a[i++] = a[j]; a[j--] = t; } }
```

## B6. Merge two sorted arrays
```java
static int[] merge(int[] a, int[] b) {
    int[] r = new int[a.length + b.length];
    int i = 0, j = 0, k = 0;
    while (i < a.length && j < b.length) r[k++] = a[i] <= b[j] ? a[i++] : b[j++];
    while (i < a.length) r[k++] = a[i++];
    while (j < b.length) r[k++] = b[j++];
    return r;
}
```

## B7. Binary search ⭐
```java
static int binarySearch(int[] a, int t) {
    int lo = 0, hi = a.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;         // avoids int overflow of (lo+hi)/2
        if (a[mid] == t) return mid;
        if (a[mid] < t) lo = mid + 1; else hi = mid - 1;
    }
    return -1;
}
```
O(log n). Array must be sorted.

## B8. Find missing number in 1..n
`expected = n(n+1)/2` minus actual sum. (Or XOR all.)
```java
static int missing(int[] a, int n) { return n * (n + 1) / 2 - Arrays.stream(a).sum(); }
```

## B9. Find duplicate / second largest
```java
static int secondLargest(int[] a) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int x : a) {
        if (x > first) { second = first; first = x; }
        else if (x > second && x != first) second = x;
    }
    return second;
}
```

## B10. Merge intervals ⭐
Sort by start; if current start ≤ last end → extend.
```java
static int[][] mergeIntervals(int[][] in) {
    Arrays.sort(in, Comparator.comparingInt(x -> x[0]));
    List<int[]> out = new ArrayList<>();
    for (int[] cur : in) {
        if (out.isEmpty() || out.get(out.size() - 1)[1] < cur[0]) out.add(cur);
        else out.get(out.size() - 1)[1] = Math.max(out.get(out.size() - 1)[1], cur[1]);
    }
    return out.toArray(new int[0][]);
}
```
`[[1,3],[2,6],[8,10]]` → `[[1,6],[8,10]]`.

## B11. Fibonacci / Factorial / Prime
```java
static long fib(int n) {                         // iterative O(n); recursion is O(2^n)
    long a = 0, b = 1;
    for (int i = 0; i < n; i++) { long t = a + b; a = b; b = t; }
    return a;
}
static boolean isPrime(int n) {
    if (n < 2) return false;
    for (int i = 2; (long) i * i <= n; i++) if (n % i == 0) return false;
    return true;
}
```
