# Coding Problems: Java 8 Streams
Sample model: `record Employee(int id, String name, String dept, double salary, int age) {}`

```java
// 1. Names starting with 'A'
emps.stream().map(Employee::name).filter(n -> n.startsWith("A")).toList();

// 2. Highest paid employee
emps.stream().max(Comparator.comparingDouble(Employee::salary));

// 3. 2nd highest salary
emps.stream().map(Employee::salary).distinct().sorted(Comparator.reverseOrder()).skip(1).findFirst();

// 4. Count employees per department
emps.stream().collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));

// 5. Average salary per department
emps.stream().collect(Collectors.groupingBy(Employee::dept, Collectors.averagingDouble(Employee::salary)));

// 6. Highest paid employee in each department
emps.stream().collect(Collectors.groupingBy(Employee::dept,
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));

// 7. Names of employees per dept as comma string
emps.stream().collect(Collectors.groupingBy(Employee::dept,
        Collectors.mapping(Employee::name, Collectors.joining(", "))));

// 8. Partition: salary > 50000 or not
emps.stream().collect(Collectors.partitioningBy(e -> e.salary() > 50000));

// 9. Sum of salaries
emps.stream().mapToDouble(Employee::salary).sum();

// 10. Employee -> Map<id, name>
emps.stream().collect(Collectors.toMap(Employee::id, Employee::name));
//   duplicate keys throw IllegalStateException → give merge function (a, b) -> a

// 11. Top 3 highest paid
emps.stream().sorted(Comparator.comparingDouble(Employee::salary).reversed()).limit(3).toList();

// 12. Flatten list of lists
List<List<Integer>> nested = ...; nested.stream().flatMap(List::stream).toList();

// 13. Find duplicate elements in list
Set<Integer> seen = new HashSet<>();
list.stream().filter(n -> !seen.add(n)).collect(Collectors.toSet());

// 14. First non-repeating character
s.chars().mapToObj(c -> (char) c)
 .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))
 .entrySet().stream().filter(e -> e.getValue() == 1).map(Map.Entry::getKey).findFirst();

// 15. Even and odd numbers separately
Map<Boolean, List<Integer>> m = nums.stream().collect(Collectors.partitioningBy(n -> n % 2 == 0));

// 16. Squares of numbers > 50 (max/min/summary)
IntSummaryStatistics st = nums.stream().mapToInt(Integer::intValue).summaryStatistics();
// st.getMax(), getMin(), getAverage(), getSum(), getCount()

// 17. Reverse a string using streams
new StringBuilder(s).reverse().toString();   // simplest; stream version is unnatural

// 18. Sort strings by length then alphabetically
words.sort(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()));

// 19. Join with prefix/suffix
list.stream().collect(Collectors.joining(",", "[", "]"));

// 20. Any / all / none match
emps.stream().anyMatch(e -> e.age() > 60);
```
Traps: stream can be consumed **once**; `peek` is for debugging; `parallelStream` not for small/IO/ordered/shared-state work; `Collectors.toMap` NPE on null value; `Optional.get()` without check.
