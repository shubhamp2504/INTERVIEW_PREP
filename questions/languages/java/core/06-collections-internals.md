# ☕ Core Java — ArrayList vs LinkedList, Collection vs Collections & CAS (Q1–Q3)

> **Source**: Capgemini + Java Developer Interview (4+ years)  
> **Coverage**: List implementations, utility class distinction, lock-free concurrency primitive

---

<a id="q1"></a>
## Q1. What is the difference between ArrayList and LinkedList?

### 📝 One-Liner
`ArrayList` uses a **dynamic array** (fast random access O(1), slow insert/delete in middle O(n)); `LinkedList` uses a **doubly-linked list** (fast insert/delete O(1) at ends, slow random access O(n)).

### 🔑 Quick Answer
`ArrayList` — backed by `Object[]` array. **Get by index = O(1)** (direct array access). **Add/remove in middle = O(n)** (shifts elements). **Add at end = amortized O(1)** (array resize when full — doubles capacity). `LinkedList` — backed by doubly-linked nodes (prev/next pointers). **Add/remove at head/tail = O(1)** (pointer updates). **Get by index = O(n)** (traverses from head or tail). Also implements `Deque` — usable as queue/stack. **In practice, ArrayList wins almost always** — CPU cache locality of contiguous array beats LinkedList pointer chasing. *(ArrayList = array hai — index se fast access; LinkedList = nodes hain — beech mein insert fast lekin access slow)*

### 📖 How It Works (Detailed Explanation)

```
ArrayList (contiguous memory):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │   │   │  ← array (capacity=8, size=6)
└───┴───┴───┴───┴───┴───┴───┴───┘
  get(3) = O(1)  ← direct index
  add(2, X) = O(n) ← shifts [2,3,4,5] right

LinkedList (scattered memory):
  HEAD ↔ [A] ↔ [B] ↔ [C] ↔ [D] ↔ TAIL
  get(2) = O(n)  ← traverse from head: A→B→C
  addFirst(X) = O(1) ← update HEAD pointer
  remove(node) = O(1) ← update prev/next pointers
```

**ArrayList resizing**: default capacity = 10. When full, grows by 50% (`newCapacity = oldCapacity + (oldCapacity >> 1)`). Copies entire array → expensive. **Pre-size** with `new ArrayList<>(expectedSize)` to avoid resizing. **LinkedList overhead**: each node stores element + prev pointer + next pointer → ~3x memory overhead per element. **Cache performance**: ArrayList array is contiguous in memory → CPU cache prefetching works well. LinkedList nodes are scattered → frequent cache misses → slower in practice even for sequential access.

### 🗣️ Interview Script
"ArrayList is backed by a contiguous array, so random access by index is O(1). But insertion or removal in the middle is O(n) because elements must be shifted. LinkedList uses doubly-linked nodes, so adding or removing at the head or tail is O(1) with pointer updates, but random access requires traversal from one end, making it O(n). Despite LinkedList's theoretical advantage for insertions, in real-world applications I almost always use ArrayList. The reason is CPU cache locality — contiguous memory in ArrayList means the CPU can prefetch data efficiently, while LinkedList nodes are scattered in heap memory causing cache misses. The only time I'd consider LinkedList is as a Queue or Deque where all operations are at the ends."

### 💻 Code Example

```java
// ✅ ArrayList — default choice
List<String> names = new ArrayList<>(1000);  // pre-size to avoid resizing
names.add("Alice");          // O(1) amortized
names.get(500);              // O(1) — direct array index
names.add(50, "Bob");        // O(n) — shifts elements right
names.remove(50);            // O(n) — shifts elements left

// ✅ LinkedList — as Deque/Queue
Deque<Task> taskQueue = new LinkedList<>();
taskQueue.addFirst(urgentTask);   // O(1) — pointer update
taskQueue.addLast(normalTask);    // O(1) — pointer update
Task next = taskQueue.pollFirst(); // O(1) — pointer update
// ❌ Don't do: taskQueue.get(500); → O(n) traversal!

// ✅ Performance comparison
List<Integer> arrayList = new ArrayList<>();
List<Integer> linkedList = new LinkedList<>();

// Sequential add at end — both fast
for (int i = 0; i < 100_000; i++) {
    arrayList.add(i);   // O(1) amortized — 5ms
    linkedList.add(i);  // O(1) — but 15ms (node creation + allocation)
}

// Random access — ArrayList wins massively
for (int i = 0; i < 100_000; i++) {
    arrayList.get(i);   // O(1) — 2ms total
    // linkedList.get(i); // O(n) — would take minutes!
}

// ✅ When LinkedList actually helps: frequent add/remove at head
Deque<LogEntry> recentLogs = new LinkedList<>();
recentLogs.addFirst(newLog);       // O(1)
if (recentLogs.size() > 100) {
    recentLogs.removeLast();       // O(1)
}
// ArrayList equivalent: add(0, newLog) → O(n) shift every time!
```

### ⚠️ Common Pitfalls
- **Looping with `get(i)` on LinkedList** — O(n²) total! Use `Iterator` instead
- **Not pre-sizing ArrayList** — default capacity 10 resizes many times when adding thousands of elements
- **Choosing LinkedList for "frequent inserts"** — unless inserting at head/tail, you still need O(n) traversal to find the position
- **Memory overhead** — LinkedList uses ~3x more memory per element (node + 2 pointers)

### 🆚 Comparison Table

| Aspect | ArrayList | LinkedList |
|--------|----------|------------|
| Backing | `Object[]` array | Doubly-linked nodes |
| get(index) | **O(1)** ⭐ | O(n) |
| add(end) | O(1) amortized | O(1) |
| add(middle) | O(n) (shift) | O(1) if at node* |
| remove(middle) | O(n) (shift) | O(1) if at node* |
| Memory/element | ~1x (just reference) | ~3x (node + 2 pointers) |
| Cache locality | **Excellent** ⭐ | Poor (scattered) |
| Implements | List, RandomAccess | List, Deque, Queue |
| Default choice | **Yes** ⭐ | Only for Queue/Deque |

*LinkedList add/remove at a known node is O(1), but finding the node is O(n).

### ⚡ Remember (Quick Recall)
- **ArrayList = default choice** (99% of cases)
- ArrayList: O(1) get, O(n) insert/remove middle
- LinkedList: O(1) add/remove at ends, O(n) get by index
- Cache locality makes ArrayList faster even for iteration
- Pre-size ArrayList: `new ArrayList<>(expectedSize)`
- LinkedList only for Queue/Deque usage

### 🔗 Follow-up Topics
- [Q1 in core/01 → HashMap internals (array + linked list/tree)](01-collections-jvm-threading.md#q1)
- CopyOnWriteArrayList (thread-safe ArrayList)
- ArrayDeque vs LinkedList as Queue (ArrayDeque is faster)

---

<a id="q2"></a>
## Q2. What is the difference between Collection and Collections in Java?

### 📝 One-Liner
`Collection` is the root **interface** of the collections hierarchy (`List`, `Set`, `Queue` extend it); `Collections` is a **utility class** with static methods to operate on collections.

### 🔑 Quick Answer
`java.util.Collection<E>` — **interface** that defines the contract for all collections: `add()`, `remove()`, `contains()`, `size()`, `iterator()`, `stream()`. Extended by `List`, `Set`, `Queue`. NOT extended by `Map`. `java.util.Collections` — **utility class** (all static methods, private constructor, cannot be instantiated). Provides: `sort()`, `unmodifiableList()`, `synchronizedMap()`, `singleton()`, `emptyList()`, `frequency()`, `disjoint()`, `reverse()`, `shuffle()`. *(Collection = interface hai jo List/Set/Queue define karti hai; Collections = helper class hai static methods ke saath)*

### 📖 How It Works (Detailed Explanation)

```
Collection (interface hierarchy):
┌─────────────────────────────────┐
│ Iterable<E>                      │
│  └── Collection<E>  ← ROOT      │
│       ├── List<E>                │
│       │    ├── ArrayList         │
│       │    └── LinkedList        │
│       ├── Set<E>                 │
│       │    ├── HashSet           │
│       │    └── TreeSet           │
│       └── Queue<E>              │
│            ├── PriorityQueue    │
│            └── Deque (LinkedList)│
└─────────────────────────────────┘

Collections (utility class):
┌──────────────────────────────────┐
│ Collections (final class)        │
│  .sort(list)                     │
│  .unmodifiableList(list)         │
│  .synchronizedMap(map)           │
│  .singletonList(element)        │
│  .emptyList()                    │
│  .reverse(list)                  │
│  .frequency(collection, obj)    │
└──────────────────────────────────┘
```

### 🗣️ Interview Script
"Collection with a capital C and no 's' is the root interface of the Java Collections Framework — it defines the core operations like add, remove, contains, and iterator that all collections implement. List, Set, and Queue extend it. Note that Map does NOT extend Collection — it's a separate hierarchy. Collections with an 's' is a utility class full of static helper methods. I use it for things like creating unmodifiable views of lists, wrapping collections for thread safety with synchronizedMap, sorting, and creating empty or singleton collections. With Java 9+, I now prefer List.of() and Map.of() over Collections.unmodifiableList() and Collections.emptyList() for immutable collections."

### 💻 Code Example

```java
// ✅ Collection — the interface
Collection<String> items = new ArrayList<>();   // polymorphism
items.add("A");
items.contains("A");                            // true
items.size();                                   // 1
items.stream().filter(s -> s.length() > 0);    // stream API

// ✅ Collections — the utility class
List<Integer> numbers = new ArrayList<>(List.of(3, 1, 4, 1, 5));

Collections.sort(numbers);                     // [1, 1, 3, 4, 5]
Collections.reverse(numbers);                  // [5, 4, 3, 1, 1]
Collections.shuffle(numbers);                  // random order
int freq = Collections.frequency(numbers, 1);  // 2

// Unmodifiable wrapper
List<String> readOnly = Collections.unmodifiableList(names);
// readOnly.add("X"); → UnsupportedOperationException!

// Thread-safe wrapper
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// Empty/singleton (pre-Java 9)
List<String> empty = Collections.emptyList();
List<String> single = Collections.singletonList("only");

// ✅ Java 9+ preferred alternatives
List<String> empty9 = List.of();
List<String> single9 = List.of("only");
Map<String, Integer> map9 = Map.of("a", 1, "b", 2);

// ✅ Common pattern: return unmodifiable from method
public List<Item> getItems() {
    return Collections.unmodifiableList(this.items);
}
```

### 🆚 Comparison Table

| Aspect | Collection | Collections |
|--------|-----------|-------------|
| Type | **Interface** | **Utility class** (final) |
| Package | `java.util` | `java.util` |
| Purpose | Define collection contract | Helper methods |
| Instantiate | No (interface) | No (private constructor) |
| Key methods | `add`, `remove`, `contains`, `stream` | `sort`, `unmodifiableList`, `synchronizedMap` |
| Extended by | List, Set, Queue | Nothing (final class) |

### ⚡ Remember (Quick Recall)
- **Collection** = interface (root of List/Set/Queue hierarchy)
- **Collections** = utility class (static helper methods)
- **Map** does NOT extend Collection
- Java 9+: prefer `List.of()` / `Map.of()` over `Collections.emptyList()` / `Collections.singletonList()`

---

<a id="q3"></a>
## Q3. What is CAS (Compare And Swap) in Java?

### 📝 One-Liner
CAS is a **lock-free atomic CPU instruction** that updates a value only if it currently equals the expected value — foundation of `java.util.concurrent.atomic` classes and `ConcurrentHashMap`.

### 🔑 Quick Answer
CAS takes three parameters: **memory location**, **expected old value**, **new value**. If current value == expected → write new value (success). If current value ≠ expected → do nothing (failure, retry). Entire operation is **atomic at the hardware level** (CPU instruction like `CMPXCHG`). Used by `AtomicInteger`, `AtomicReference`, `ConcurrentHashMap` (bucket insertion), `LongAdder`. **Advantage over locks**: no thread blocking, no context switches, no deadlocks. **Disadvantage**: **ABA problem** (value changes A→B→A, CAS thinks unchanged) → fixed with `AtomicStampedReference`. *(CAS = CPU level pe atomic check-and-update — lock lagane ki zaroorat nahi)*

### 📖 How It Works (Detailed Explanation)

```
CAS Operation (hardware-level atomic):
┌─────────────────────────────────────┐
│ CAS(memory, expected=5, new=6)      │
│                                     │
│ if (memory == 5) {    ← compare    │
│   memory = 6;         ← swap       │
│   return true;        ← success    │
│ } else {                            │
│   return false;       ← retry!     │
│ }                                   │
│ ⚡ Entire block is ONE CPU instruction│
└─────────────────────────────────────┘

AtomicInteger.incrementAndGet() loop:
  do {
    oldValue = get();           // read current
    newValue = oldValue + 1;    // compute new
  } while (!CAS(old, new));     // retry if someone else changed it
```

**No lock needed** — threads spin-retry on failure instead of blocking. Under low contention, CAS is much faster than `synchronized` (no OS-level context switch). Under **high contention**, many threads retrying wastes CPU → `LongAdder` solves this by distributing counters across cells (each thread updates its own cell, sum at read time). **ConcurrentHashMap** uses CAS for inserting into empty buckets and `synchronized` on bucket head for collisions — hybrid approach.

### 🗣️ Interview Script
"CAS is a lock-free concurrency primitive that performs an atomic compare-and-swap at the CPU instruction level. It checks if a memory location holds an expected value, and only then updates it — the entire operation is atomic. If another thread changed the value in between, the CAS fails and the thread retries. This is the foundation of all Atomic classes in Java — AtomicInteger, AtomicLong, AtomicReference — and it's how ConcurrentHashMap achieves lock-free reads and bucket insertions. The main advantage is no thread blocking — no deadlocks, no context switches, excellent performance under low contention. The gotcha is the ABA problem: a value changes from A to B and back to A — CAS thinks nothing changed. For pointer-based operations where this matters, I use AtomicStampedReference which adds a version stamp."

### 💻 Code Example

```java
// ✅ AtomicInteger — uses CAS internally
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();       // CAS loop: read → increment → CAS → retry if failed
counter.compareAndSet(1, 2);     // explicit CAS: if current==1, set to 2

// ✅ How incrementAndGet works (simplified)
// public int incrementAndGet() {
//     int oldValue, newValue;
//     do {
//         oldValue = get();                    // read current
//         newValue = oldValue + 1;             // compute
//     } while (!compareAndSet(oldValue, newValue)); // CAS retry loop
//     return newValue;
// }

// ✅ CAS in ConcurrentHashMap (simplified)
// Inserting into empty bucket:
//   if (CAS(bucket[i], null, newNode)) → success (no lock!)
// Inserting into occupied bucket:
//   synchronized(bucketHead) { ... }  → lock only this bucket

// ✅ AtomicReference — CAS on objects
AtomicReference<Config> configRef = new AtomicReference<>(currentConfig);
Config oldConfig = configRef.get();
Config newConfig = loadNewConfig();
configRef.compareAndSet(oldConfig, newConfig);  // atomic swap

// ✅ ABA problem demonstration
AtomicInteger val = new AtomicInteger(1);
// Thread A reads 1, gets suspended
// Thread B changes 1 → 2 → 1
// Thread A resumes, CAS(expected=1, new=3) → succeeds! (but value was modified)

// ✅ Fix ABA with AtomicStampedReference
AtomicStampedReference<Integer> stamped = new AtomicStampedReference<>(1, 0);
int[] stampHolder = new int[1];
int value = stamped.get(stampHolder);  // value=1, stamp=0
// Even if value goes 1→2→1, stamp changes 0→1→2
stamped.compareAndSet(1, 3, 0, 1);    // checks BOTH value AND stamp

// ✅ LongAdder — better than AtomicLong under high contention
LongAdder adder = new LongAdder();
adder.increment();  // each thread updates its own cell (no CAS contention)
long total = adder.sum();  // aggregates all cells
```

### ⚠️ Common Pitfalls
- **ABA problem** — value reverts to original → CAS doesn't detect intermediate changes → use `AtomicStampedReference`
- **High contention spin waste** — many threads retrying CAS burns CPU → use `LongAdder` instead of `AtomicLong` for counters
- **Not a replacement for locks** in all cases — complex multi-variable operations still need `synchronized` or `ReentrantLock`
- **CAS on objects** = reference comparison, not `.equals()` → `AtomicReference.compareAndSet` compares `==`

### 🆚 Comparison Table

| Aspect | CAS (Atomic) | synchronized | ReentrantLock |
|--------|-------------|-------------|--------------|
| Blocking | No (spin-retry) | Yes (blocks) | Yes (blocks) |
| Deadlock risk | None | Yes | Yes |
| Performance (low contention) | **Fastest** ⭐ | Good (biased locking) | Good |
| Performance (high contention) | Degrades (spin) | Good (queue) | Good (fair option) |
| Multi-variable ops | ❌ Single variable | ✅ Any code block | ✅ Any code block |
| Use case | Counters, flags | General mutual exclusion | Timeout, fairness needed |

### ⚡ Remember (Quick Recall)
- CAS = **compare expected + swap atomically** (CPU instruction)
- Foundation of `AtomicInteger`, `AtomicReference`, `ConcurrentHashMap`
- **No locks → no deadlocks → no blocking**
- ABA problem → `AtomicStampedReference`
- High contention counters → `LongAdder` > `AtomicLong`
- ConcurrentHashMap: CAS for empty buckets, synchronized for collisions

### 🔗 Follow-up Topics
- [Q51-52 in multithreading/05 → ConcurrentHashMap internals (uses CAS)](../multithreading/05-concurrent-collections.md)
- [Q58 in multithreading/06 → volatile keyword (visibility without atomicity)](../multithreading/06-memory-model.md#q58)
- Lock-free data structures (Michael-Scott queue, Treiber stack)
