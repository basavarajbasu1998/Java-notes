# Coding Problems: Strings

## A1. Reverse a string (without `reverse()`)
**Idea:** swap characters from both ends moving inward.
```java
static String reverse(String s) {
    char[] c = s.toCharArray();
    for (int i = 0, j = c.length - 1; i < j; i++, j--) {
        char t = c[i]; c[i] = c[j]; c[j] = t;
    }
    return new String(c);
}
```
O(n) time, O(n) space. `"java"` → swap j↔a → `"avaj"`.

## A2. Palindrome check
```java
static boolean isPalindrome(String s) {
    int i = 0, j = s.length() - 1;
    while (i < j) {
        while (i < j && !Character.isLetterOrDigit(s.charAt(i))) i++;
        while (i < j && !Character.isLetterOrDigit(s.charAt(j))) j--;
        if (Character.toLowerCase(s.charAt(i++)) != Character.toLowerCase(s.charAt(j--))) return false;
    }
    return true;
}
```
"A man, a plan, a canal: Panama" → true. Two pointers, O(n), O(1) space.

## A3. Anagram check
**Idea:** same letters, same counts.
```java
static boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] count = new int[26];
    for (int i = 0; i < a.length(); i++) {
        count[a.charAt(i) - 'a']++;
        count[b.charAt(i) - 'a']--;
    }
    for (int x : count) if (x != 0) return false;
    return true;
}
```
O(n), O(1). Alternative: sort both, compare (O(n log n)).

## A4. First non-repeating character
```java
static char firstUnique(String s) {
    Map<Character, Integer> m = new LinkedHashMap<>();     // keeps insertion order
    for (char c : s.toCharArray()) m.merge(c, 1, Integer::sum);
    for (var e : m.entrySet()) if (e.getValue() == 1) return e.getKey();
    throw new NoSuchElementException();
}
```
`"swiss"` → w. Why LinkedHashMap? To scan in original order.

## A5. Count character frequency / duplicate chars
```java
Map<Character, Long> freq = s.chars().mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()));
```

## A6. Longest substring without repeating characters (sliding window) ⭐
**Flow:**
```
window [left..right] always has unique chars
move right:
   char seen inside window? → move left past its previous index
   update best = max(best, right-left+1)
```
```java
static int longestUnique(String s) {
    Map<Character, Integer> last = new HashMap<>();
    int best = 0, left = 0;
    for (int r = 0; r < s.length(); r++) {
        char c = s.charAt(r);
        if (last.containsKey(c) && last.get(c) >= left) left = last.get(c) + 1;
        last.put(c, r);
        best = Math.max(best, r - left + 1);
    }
    return best;
}
```
`"abcabcbb"` → 3 (abc). O(n).

## A7. Valid parentheses (stack) ⭐
```java
static boolean valid(String s) {
    Deque<Character> st = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '{' || c == '[') st.push(c);
        else {
            if (st.isEmpty()) return false;
            char o = st.pop();
            if ((c == ')' && o != '(') || (c == '}' && o != '{') || (c == ']' && o != '[')) return false;
        }
    }
    return st.isEmpty();
}
```
Rule: last opened must be first closed → stack. `"{[()]}"` true, `"([)]"` false.

## A8. Compress string `aaabbc` → `a3b2c1`
```java
static String compress(String s) {
    StringBuilder sb = new StringBuilder();
    int count = 1;
    for (int i = 1; i <= s.length(); i++) {
        if (i < s.length() && s.charAt(i) == s.charAt(i - 1)) count++;
        else { sb.append(s.charAt(i - 1)).append(count); count = 1; }
    }
    return sb.toString();
}
```

## A9. Make a custom **immutable** String-like / check two strings are rotations
`s2` is a rotation of `s1` if `(s1 + s1).contains(s2)` and lengths equal.
