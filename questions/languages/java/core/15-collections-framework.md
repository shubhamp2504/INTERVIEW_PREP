# ☕ Core Java — Collections Framework (Q1–Q22)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Collection interface, List, ArrayList, Vector, LinkedList, Iterator, ListIterator, Set, HashSet, TreeSet, Map, HashMap, SortedMap, Hashtable

---

<a id="q1"></a>
## Q1. What is the Collections framework?

### 📝 One-Liner
A unified architecture of **interfaces, classes, and algorithms** in `java.util` for storing and manipulating groups of objects.

### 🔑 Quick Answer
The Collections framework provides: (1) Standard interfaces (`Collection`, `List`, `Set`, `Map`, `Queue`). (2) Implementations (`ArrayList`, `HashMap`, `HashSet`, etc.). (3) Algorithms (`Collections.sort()`, `Collections.binarySearch()`). Benefits: high performance, reduces coding effort, increases speed/quality, interoperability, extensible. *(Collections framework = data structures ka standard library — List, Set, Map, Queue sab iske andar)*

### 📖 How It Works

```
Collection Hierarchy:
┌──────────────┐
│  Iterable     │
│  └── Collection│
│       ├── List  │ → ArrayList, LinkedList, Vector
│       ├── Set   │ → HashSet, TreeSet, LinkedHashSet
│       └── Queue │ → PriorityQueue, Deque
├──────────────┤
│  Map (separate)│ → HashMap, TreeMap, LinkedHashMap, Hashtable
└──────────────┘
```

### ⚡ Remember
`Collection framework = interfaces + implementations + algorithms | java.util package`

---

<a id="q2"></a>
## Q2. What is a collection (lowercase)?

### 📝 One-Liner
A collection is a **container** that groups multiple objects into a single unit for easy management.

### 🔑 Quick Answer
A collection stores, retrieves, manipulates, and communicates aggregate data. Basic operations: add, remove, retrieve, iterate, check contains, get size. Examples: list of names, set of unique IDs, queue of tasks. *(collection = objects ka group, ek unit ke roop me manage karo)*

### ⚡ Remember
`collection = group of objects | Add, remove, iterate, search`

---

<a id="q3"></a>
## Q3. Explain Collection interface?

### 📝 One-Liner
`Collection<E>` is the **root interface** of the collection hierarchy — extends `Iterable` and defines basic operations.

### 🔑 Quick Answer
`Collection` extends `Iterable<E>` (inherits `iterator()`). Key methods: `add(E)`, `remove(Object)`, `contains(Object)`, `size()`, `isEmpty()`, `iterator()`, `toArray()`, `addAll(Collection)`, `removeAll(Collection)`, `retainAll(Collection)`, `clear()`. Sub-interfaces: `List`, `Set`, `Queue`, `Deque`. Map does NOT extend Collection. *(Collection = root interface | add, remove, contains, size, iterator — sab yahi define karta hai)*

### ⚡ Remember
`Collection extends Iterable | Root of List, Set, Queue | Map is separate`

---

<a id="q4"></a>
## Q4. Which interfaces extend Collection?

### 📝 One-Liner
`List`, `Set`, `Queue`, and `Deque` (Java 6+) — but **not** `Map`.

### 🔑 Quick Answer
(1) **List** — ordered, allows duplicates. (2) **Set** — no duplicates. (3) **Queue** — FIFO ordering. (4) **Deque** — double-ended queue (from Java 6). `Map` is a separate interface — stores key-value pairs, not part of Collection hierarchy. *(List, Set, Queue, Deque extend Collection | Map alag hai)*

### ⚡ Remember
`Collection → List + Set + Queue + Deque | Map is NOT a Collection`

---

<a id="q5"></a>
## Q5. Explain List interface?

### 📝 One-Liner
`List` is an **ordered collection** that allows **duplicates** and supports **index-based** access.

### 🔑 Quick Answer
List maintains insertion order. Allows duplicate elements. Provides positional access via index (`get(i)`, `set(i, e)`, `add(i, e)`, `remove(i)`). Main implementations: `ArrayList`, `LinkedList`, `Vector`. Extends `Collection` with position-based methods. *(List = ordered + duplicates allowed + index-based access)*

### ⚡ Remember
`List = ordered + duplicates + index access | ArrayList, LinkedList, Vector`

---

<a id="q6"></a>
## Q6. Explain methods specific to List interface?

### 📝 One-Liner
Position-based methods: `get(index)`, `set(index, element)`, `add(index, element)`, `remove(index)`, `indexOf()`, `listIterator()`, `subList()`.

### 🔑 Quick Answer
List-specific methods beyond Collection: (1) `E get(int index)` — element at position. (2) `E set(int index, E element)` — replace at position. (3) `void add(int index, E element)` — insert at position. (4) `E remove(int index)` — remove at position. (5) `int indexOf(Object)` — first occurrence index. (6) `ListIterator<E> listIterator()` — bidirectional iterator. (7) `List<E> subList(int from, int to)` — range view. *(List = index-based methods — get, set, add, remove with index)*

### ⚡ Remember
`get(i), set(i,e), add(i,e), remove(i), indexOf(), listIterator(), subList()`

---

<a id="q7"></a>
## Q7. List implementations of List interface?

### 📝 One-Liner
`ArrayList`, `LinkedList`, and `Vector` (legacy, synchronized).

### 🔑 Quick Answer
(1) **ArrayList** — resizable array, fast random access, slow insert/delete in middle. (2) **LinkedList** — doubly linked list, fast insert/delete, slow random access. (3) **Vector** — like ArrayList but synchronized (legacy, rarely used). Choose based on: random access → ArrayList, frequent insert/delete → LinkedList. *(ArrayList = fast read | LinkedList = fast insert/delete | Vector = synchronized legacy)*

### ⚡ Remember
`ArrayList (fast access) | LinkedList (fast insert/delete) | Vector (synchronized, legacy)`

---

<a id="q8"></a>
## Q8. Difference between Array and ArrayList?

### 📝 One-Liner
Array is **fixed-size** and holds **primitives + objects**; ArrayList is **dynamic** and holds **only objects**.

### 🔑 Quick Answer

| Feature | Array | ArrayList |
|---|---|---|
| Size | Fixed at creation | Dynamic (grows/shrinks) |
| Types | Primitives + objects | Objects only (autobox for primitives) |
| Syntax | `array[i]` | `list.get(i)` |
| Insert/Remove | Manual shift logic | `add()`, `remove()` methods |
| Performance | Faster (no overhead) | Slightly slower (wrapper) |
| Type safety | Fixed type at creation | Generics support |

*(Array = fixed, primitives allowed | ArrayList = dynamic, objects only, methods for add/remove)*

### ⚡ Remember
`Array = fixed + primitives | ArrayList = dynamic + objects | ArrayList uses array internally`

---

<a id="q9"></a>
## Q9. What is Vector?

### 📝 One-Liner
Vector is a **legacy synchronized** dynamic array (since Java 1.0) — similar to ArrayList but thread-safe.

### 🔑 Quick Answer
Vector: (1) Dynamic array that grows/shrinks. (2) **Synchronized** by default (thread-safe but slower). (3) Legacy class — existed before Collections framework (1.0). (4) Implements `List`, `RandomAccess`, `Cloneable`, `Serializable`. Prefer `ArrayList` + `Collections.synchronizedList()` or `CopyOnWriteArrayList` over Vector. *(Vector = thread-safe ArrayList but slow — legacy, use karne ki zarurat nahi)*

### ⚡ Remember
`Vector = synchronized ArrayList | Legacy (Java 1.0) | Prefer ArrayList instead`

---

<a id="q10"></a>
## Q10. Difference between ArrayList and Vector?

### 📝 One-Liner
ArrayList is **not synchronized** (fast); Vector is **synchronized** (slow). ArrayList is modern; Vector is legacy.

### 🔑 Quick Answer

| Feature | ArrayList | Vector |
|---|---|---|
| Synchronization | Not synchronized | Synchronized |
| Performance | Faster | Slower (lock overhead) |
| Version | Java 2.0 | Java 1.0 (legacy) |
| Growth | 50% size increase | 100% (doubles) |

*(ArrayList = fast, not sync | Vector = slow, synchronized, legacy)*

### ⚡ Remember
`ArrayList = not sync + fast + grows 50% | Vector = sync + slow + grows 100%`

---

<a id="q11"></a>
## Q11. Define LinkedList and its features?

### 📝 One-Liner
LinkedList is a **doubly-linked list** implementation — fast insert/delete in middle but slow random access.

### 🔑 Quick Answer
LinkedList: (1) Doubly-linked list (each node has prev + next pointers). (2) Implements `List`, `Deque`, `Cloneable`, `Serializable`. (3) Fast addition/removal in middle (just update pointers). (4) Slow random access (`get(i)` is O(n) — must traverse). (5) Specific methods: `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `removeLast()`. *(LinkedList = doubly-linked | Fast insert/delete | Slow random access | Deque bhi implement karta hai)*

### ⚡ Remember
`LinkedList = doubly-linked | O(1) insert/delete | O(n) random access | Also implements Deque`

---

<a id="q12"></a>
## Q12. Define Iterator and methods in Iterator?

### 📝 One-Liner
Iterator is a standard way to **traverse a collection** one element at a time — `hasNext()`, `next()`, `remove()`.

### 🔑 Quick Answer
Obtained via `collection.iterator()`. Three methods: (1) `boolean hasNext()` — checks if more elements exist. (2) `E next()` — returns next element (throws `NoSuchElementException` if none). (3) `void remove()` — removes last element returned by `next()`. Always call `hasNext()` before `next()`. No guarantee on iteration order (depends on collection type). *(Iterator = collection traverse karne ka standard tarika — hasNext(), next(), remove())*

### 💻 Code Example

```java
List<String> list = List.of("A", "B", "C");
Iterator<String> itr = list.iterator();
while (itr.hasNext()) {
    String s = itr.next();
    System.out.println(s);
}
```

### ⚡ Remember
`hasNext() → next() → remove() | Always check hasNext before next | NoSuchElementException if not`

---

<a id="q13"></a>
## Q13. In which order does Iterator iterate?

### 📝 One-Liner
Depends on the collection: **List** = insertion order, **HashSet** = unpredictable, **TreeSet** = sorted, **LinkedHashSet** = insertion order.

### 🔑 Quick Answer
Iterator follows the collection's traversal order: (1) `List` (ArrayList/LinkedList) → sequential/insertion order. (2) `HashSet` → no guaranteed order. (3) `TreeSet` → natural/sorted order. (4) `LinkedHashSet` → insertion order. (5) `TreeMap` → sorted by keys. *(Iteration order collection pe depend karta hai — List sequential, TreeSet sorted, HashSet random)*

### ⚡ Remember
`List=insertion | HashSet=random | TreeSet=sorted | LinkedHashSet=insertion`

---

<a id="q14"></a>
## Q14. Explain ListIterator and methods?

### 📝 One-Liner
ListIterator is a **bidirectional** iterator for Lists — can traverse **forward and backward**, and modify elements.

### 🔑 Quick Answer
Extends Iterator with backward traversal. Methods: `hasNext()`, `next()`, `hasPrevious()`, `previous()`, `nextIndex()`, `previousIndex()`, `remove()`, `set(E)`, `add(E)`. Position lies between two elements (previous and next). Only works with `List` implementations. *(ListIterator = aage-peeche dono taraf traverse kar sakte ho + modify bhi)*

### 💻 Code Example

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
ListIterator<String> li = list.listIterator();
while (li.hasNext()) {
    String s = li.next();
    if (s.equals("B")) li.set("BB");  // ⭐ modify in place
}
// list = ["A", "BB", "C"]
```

### ⚡ Remember
`ListIterator = bidirectional | hasPrevious + previous + set + add | List only`

---

<a id="q15"></a>
## Q15. Explain about Sets?

### 📝 One-Liner
A Set is a collection that **does not allow duplicate** elements — uses `equals()` internally to enforce uniqueness.

### 🔑 Quick Answer
Set: (1) No duplicates — adding duplicate is silently ignored. (2) At most **one null** (HashSet, LinkedHashSet). (3) Unordered (HashSet) or sorted (TreeSet) or insertion-ordered (LinkedHashSet). (4) No index-based access. (5) Implements `equals()` and `hashCode()` for dedup. *(Set = duplicates nahi, unique elements only — HashSet, TreeSet, LinkedHashSet)*

### ⚡ Remember
`Set = no duplicates | Max 1 null | No index access | HashSet/TreeSet/LinkedHashSet`

---

<a id="q16"></a>
## Q16. Implementations of Set interface?

### 📝 One-Liner
`HashSet` (unordered, fast), `LinkedHashSet` (insertion order), `TreeSet` (sorted).

### 🔑 Quick Answer
(1) **HashSet** — Hash table backed, no order, O(1) operations, best performance. (2) **LinkedHashSet** — Maintains insertion order, slightly slower than HashSet. (3) **TreeSet** — Sorted (natural or Comparator), O(log n) operations, implements `NavigableSet`. *(HashSet = fastest, no order | LinkedHashSet = insertion order | TreeSet = sorted)*

### ⚡ Remember
`HashSet (fast, no order) | LinkedHashSet (insertion order) | TreeSet (sorted, O(log n))`

---

<a id="q17"></a>
## Q17. Explain HashSet and its features?

### 📝 One-Liner
HashSet is an **unordered, unsorted Set** backed by HashMap — O(1) add/remove/contains, allows one null.

### 🔑 Quick Answer
Features: (1) No duplicates. (2) No guaranteed order. (3) O(1) for add, remove, contains (uses hashing). (4) Allows one null. (5) Not synchronized. (6) Backed by HashMap internally. (7) Objects must implement `hashCode()` and `equals()` for proper behavior. *(HashSet = fastest Set, HashMap ke andar, order nahi, duplicates nahi)*

### ⚡ Remember
`HashSet = HashMap internally | O(1) | No order | No duplicates | Needs hashCode+equals`

---

<a id="q18"></a>
## Q18. Explain TreeSet and its features?

### 📝 One-Liner
TreeSet is a **sorted, navigable Set** backed by TreeMap — elements sorted by natural order or Comparator.

### 🔑 Quick Answer
Features: (1) No duplicates. (2) **Sorted** — natural ordering (Comparable) or custom (Comparator). (3) O(log n) for add, remove, contains. (4) Does NOT allow null (throws NPE in Java 7+). (5) Implements `NavigableSet`. (6) Backed by TreeMap (Red-Black tree). *(TreeSet = sorted Set, TreeMap ke andar, O(log n), null nahi allowed)*

### ⚡ Remember
`TreeSet = sorted | O(log n) | No null | Red-Black tree | NavigableSet`

---

<a id="q19"></a>
## Q19. When to use HashSet over TreeSet?

### 📝 One-Liner
Use HashSet when you need **fast operations without ordering**; TreeSet when you need **sorted** elements.

### 🔑 Quick Answer
HashSet: O(1) add/remove/contains — best for performance when order doesn't matter. TreeSet: O(log n) — use when you need elements in sorted order or need NavigableSet operations (floor, ceiling, headSet, tailSet). Same elements, different guarantees. *(HashSet = fast + no order | TreeSet = sorted + slower)*

### ⚡ Remember
`HashSet = O(1) + no order = performance | TreeSet = O(log n) + sorted = ordering`

---

<a id="q20"></a>
## Q20. What is LinkedHashSet?

### 📝 One-Liner
LinkedHashSet is a HashSet that **maintains insertion order** using a doubly-linked list running through its entries.

### 🔑 Quick Answer
Extends HashSet. Same performance as HashSet for add/remove/contains (O(1)). Difference: iteration order = insertion order. Slightly more overhead than HashSet (maintains linked list). Use when you need Set behavior (no duplicates) with predictable iteration order. *(LinkedHashSet = HashSet + insertion order maintain karta hai)*

### ⚡ Remember
`LinkedHashSet = HashSet + insertion order | Slightly more memory | O(1) operations`

---

<a id="q21"></a>
## Q21. Explain Map interface?

### 📝 One-Liner
Map is a **key-value pair** collection — keys are unique, values can be duplicated. Not part of Collection hierarchy.

### 🔑 Quick Answer
Map: (1) Stores key-value pairs. (2) No duplicate keys (duplicate key overwrites value). (3) Values can be duplicated. (4) Not extends Collection — separate hierarchy. (5) Key methods: `put(K,V)`, `get(K)`, `remove(K)`, `containsKey(K)`, `keySet()`, `values()`, `entrySet()`. Implementations: HashMap, LinkedHashMap, TreeMap, Hashtable. *(Map = key-value pair, keys unique, values duplicate allowed)*

### ⚡ Remember
`Map = key→value | Keys unique | Not Collection | HashMap, TreeMap, LinkedHashMap`

---

<a id="q22"></a>
## Q22. What is LinkedHashMap?

### 📝 One-Liner
LinkedHashMap is HashMap that **maintains insertion order** (or access order) using a doubly-linked list.

### 🔑 Quick Answer
Extends HashMap. Guarantees: iteration in insertion order. Can also be configured for **access order** (LRU cache). Slightly slower than HashMap for insertion/deletion. Faster for iteration (linked list). *(LinkedHashMap = HashMap + insertion order | LRU cache ke liye access order bhi set kar sakte ho)*

### 💻 Code Example

```java
Map<String, Integer> map = new LinkedHashMap<>();
map.put("B", 2);
map.put("A", 1);
map.put("C", 3);
map.keySet();  // [B, A, C] — insertion order maintained

// LRU cache with access order:
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true);  // accessOrder=true
```

### ⚡ Remember
`LinkedHashMap = HashMap + insertion/access order | Good for LRU cache`

---

<a id="q23"></a>
## Q23. What is SortedMap interface?

### 📝 One-Liner
SortedMap extends Map and maintains keys in **sorted order** — natural ordering or via Comparator.

### 🔑 Quick Answer
Methods: `firstKey()`, `lastKey()`, `headMap(K toKey)`, `tailMap(K fromKey)`, `subMap(K from, K to)`, `comparator()`. Implementation: `TreeMap`. Provides range-view operations. *(SortedMap = keys sorted order me rehte hain | TreeMap implement karta hai)*

### ⚡ Remember
`SortedMap = sorted keys | firstKey, lastKey, headMap, tailMap | TreeMap implements it`

---

<a id="q24"></a>
## Q24. What is Hashtable?

### 📝 One-Liner
Hashtable is a **legacy synchronized** Map (since Java 1.0) — does **not allow null** keys or values.

### 🔑 Quick Answer
Hashtable: (1) Synchronized (thread-safe). (2) No null keys or values (`NullPointerException`). (3) Legacy (predates Collections framework). (4) Extends `Dictionary` + implements `Map`. Prefer `ConcurrentHashMap` for thread-safety or `HashMap` for single-threaded use. *(Hashtable = synchronized, no nulls, legacy — ConcurrentHashMap use karo instead)*

### ⚡ Remember
`Hashtable = synchronized + no nulls + legacy | Use ConcurrentHashMap instead`

---

<a id="q25"></a>
## Q25. Difference between HashMap and Hashtable?

### 📝 One-Liner
HashMap = **not synchronized, allows nulls**; Hashtable = **synchronized, no nulls, legacy**.

### 🔑 Quick Answer

| Feature | HashMap | Hashtable |
|---|---|---|
| Synchronization | Not synchronized | Synchronized |
| Null keys/values | 1 null key, many null values | No nulls allowed |
| Performance | Faster | Slower (lock overhead) |
| Version | Java 2 (Collection framework) | Java 1.0 (legacy) |
| Iterator | Fail-fast | Fail-fast (Enumerator is fail-safe) |

*(HashMap = fast, nulls allowed, not sync | Hashtable = slow, no nulls, synchronized, legacy)*

### ⚡ Remember
`HashMap = fast + nulls + not sync | Hashtable = slow + no nulls + sync + legacy`
