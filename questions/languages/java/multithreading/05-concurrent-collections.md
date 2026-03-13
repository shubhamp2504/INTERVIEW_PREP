# 📦 Concurrent Collections (Q51–Q56)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q51"></a>

## Q51. What are concurrent collections?

### 🔑 Quick Answer

> Thread-safe collections from `java.util.concurrent` that allow **concurrent access without external synchronization**. They use fine-grained locking or lock-free algorithms instead of locking the entire collection.

### 📖 Step-by-Step Explanation

```
Regular Collections + synchronized:
  Collections.synchronizedMap(hashMap)
  → Locks ENTIRE map for every operation ❌
  → One thread at a time → slow

Concurrent Collections:
  ConcurrentHashMap
  → Locks only the BUCKET being accessed ✅
  → Multiple threads on different buckets → fast
```

| Collection | Concurrent Alternative | Strategy |
|-----------|----------------------|----------|
| HashMap | **ConcurrentHashMap** | Bucket-level locking (CAS) |
| ArrayList | **CopyOnWriteArrayList** | Copy on write |
| LinkedList | **ConcurrentLinkedQueue** | Lock-free (CAS) |
| TreeMap | **ConcurrentSkipListMap** | Lock-free skip list |
| BlockingQueue | **ArrayBlockingQueue** / **LinkedBlockingQueue** | Lock-based blocking |

### 🗣️ How to Explain in Interview

> *"Concurrent collections from java.util.concurrent are designed for multi-threaded access without needing external synchronization. Unlike Collections.synchronizedMap() which locks the entire map, ConcurrentHashMap uses bucket-level locking and CAS operations — multiple threads can read and write to different buckets simultaneously. CopyOnWriteArrayList creates a new copy of the array on every write — great for read-heavy, write-rare scenarios. BlockingQueue adds blocking operations — put() blocks when full, take() blocks when empty."*

### ⚡ Key Points to Remember

1. **Fine-grained** locking or **lock-free** algorithms
2. **ConcurrentHashMap** = most commonly used ⭐
3. **CopyOnWriteArrayList** = read-heavy, write-rare
4. **BlockingQueue** = producer-consumer pattern
5. **Never** use Collections.synchronizedMap in production

---

<a id="q52"></a>

## Q52. What is ConcurrentHashMap?

### 🔑 Quick Answer

> A thread-safe HashMap that uses **bucket-level locking (CAS + synchronized on node)** for writes and **lock-free reads**. Multiple threads can read/write simultaneously to different buckets. The **go-to concurrent map** in Java.

### 📖 Step-by-Step Explanation

```
HashMap (NOT thread-safe):
  Thread-1: put("A", 1)  → modifies bucket 5 → CORRUPTS internal state
  Thread-2: put("B", 2)  → modifies bucket 5 → at same time! 💀

Collections.synchronizedMap (too slow):
  Thread-1: [LOCK ENTIRE MAP] put("A", 1) [UNLOCK]
  Thread-2: [WAIT...] [LOCK ENTIRE MAP] put("B", 2) [UNLOCK]

ConcurrentHashMap (fast + safe):
  Thread-1: put("A", 1) → lock ONLY bucket 5 → write → unlock bucket 5
  Thread-2: put("B", 2) → lock ONLY bucket 8 → write → unlock bucket 8
  → Both run simultaneously! ✅ (different buckets)
  
  Thread-3: get("A") → NO LOCK needed → reads with volatile guarantees ✅
```

**Java 8+ internal structure:**

```
ConcurrentHashMap:
  [Bucket 0] → Node → Node → ...  (or TreeBin if too many nodes)
  [Bucket 1] → Node → ...
  [Bucket 2] → null
  ...
  [Bucket N] → Node → ...

Write: synchronized(first node of bucket) — only locks ONE bucket
Read:  Lock-free (volatile reads)
Resize: Concurrent (multiple threads help resize)
```

### 💻 Code Example

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Thread-safe put and get
map.put("orders", 100);
int count = map.get("orders");

// Atomic compound operations ⭐
map.putIfAbsent("orders", 0);                    // Put only if key absent
map.compute("orders", (key, val) -> val + 1);    // Atomic read-modify-write
map.merge("orders", 1, Integer::sum);            // Atomic merge

// Thread-safe iteration (weakly consistent)
map.forEach((key, value) -> System.out.println(key + ": " + value));
```

### 🗣️ How to Explain in Interview

> *"ConcurrentHashMap is the production standard for thread-safe maps. In Java 8+, it uses CAS and per-bucket synchronized blocks. Reads are lock-free — they use volatile semantics. Writes lock only the first node of the affected bucket — so writes to different buckets are fully concurrent. It also provides atomic compound operations: putIfAbsent(), compute(), merge() — these are critical because doing get-then-put on a regular map is a race condition. Iteration is weakly consistent — it reflects concurrent changes but doesn't throw ConcurrentModificationException."*

### ⚡ Key Points to Remember

1. **Lock-free reads**, **per-bucket locking** for writes
2. **Atomic operations**: putIfAbsent, compute, merge ⭐
3. **No null keys or values** (unlike HashMap)
4. **Weakly consistent** iteration (no ConcurrentModificationException)
5. Default initial capacity = 16, load factor = 0.75

---

<a id="q53"></a>

## Q53. Difference between HashMap and ConcurrentHashMap?

### 🔑 Quick Answer

| Feature | HashMap | ConcurrentHashMap |
|---------|---------|------------------|
| **Thread-safe** | ❌ No | ✅ Yes |
| **Null keys/values** | ✅ Allowed | ❌ Not allowed |
| **Performance (single-thread)** | Faster | Slightly slower |
| **Performance (multi-thread)** | UNSAFE | Designed for concurrency |
| **Iteration** | Fail-fast (CME) | Weakly consistent |
| **Locking** | None | Bucket-level CAS + sync |

### 📖 Step-by-Step Explanation

```
HashMap in multi-threaded environment:
  ⚠️ Infinite loop during resize (Java 7 — linked list cycle)
  ⚠️ Lost updates (two threads put, one overwrites the other)
  ⚠️ ConcurrentModificationException during iteration
  ⚠️ Corrupted internal state
  → NEVER use HashMap in multi-threaded code!

ConcurrentHashMap:
  ✅ Safe concurrent reads and writes
  ✅ Atomic compound operations
  ✅ No ConcurrentModificationException
  ✅ High throughput under contention
```

### 🗣️ How to Explain in Interview

> *"HashMap is not thread-safe — using it from multiple threads can cause infinite loops during resize, lost updates, and data corruption. ConcurrentHashMap is specifically designed for concurrency. Three key differences: First, ConcurrentHashMap doesn't allow null keys or values because null would be ambiguous — does get() returning null mean 'not found' or 'value is null'? Second, iteration is weakly consistent — it won't throw ConcurrentModificationException but might not reflect the latest updates. Third, ConcurrentHashMap provides atomic operations like compute() and merge() that eliminate the check-then-act race condition."*

### ⚡ Key Points to Remember

1. **HashMap** = NEVER in multi-threaded code
2. **ConcurrentHashMap** = designed for concurrency
3. **No nulls** in ConcurrentHashMap (ambiguity)
4. **Weakly consistent** iteration (safe, may miss recent updates)
5. Use **compute/merge** instead of get-then-put

---

<a id="q54"></a>

## Q54. What is CopyOnWriteArrayList?

### 🔑 Quick Answer

> A thread-safe ArrayList where every **write creates a new copy** of the underlying array. Reads are **lock-free** and fast. Ideal for **read-heavy, write-rare** scenarios like listener lists and configuration.

### 📖 Step-by-Step Explanation

```
Internal mechanism:
  Array: [A, B, C]

  Read (Thread-1): reads from current array → no lock, fast ✅
  Read (Thread-2): reads from current array → no lock, fast ✅

  Write (Thread-3): add("D")
    1. Lock (ReentrantLock)
    2. Copy: [A, B, C] → new [A, B, C, D]
    3. Replace reference: array = newArray
    4. Unlock
    
  After write:
    Thread-1 (iterating): still sees [A, B, C] ← snapshot, no CME ✅
    Thread-4 (new read): sees [A, B, C, D] ← latest
```

### 💻 Code Example

```java
CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

// Writes are rare — adding/removing listeners
listeners.add("listener1");
listeners.add("listener2");

// Reads are frequent — notifying all listeners (no lock!)
for (String listener : listeners) {
    notify(listener);  // Safe — iterates over snapshot
}

// No ConcurrentModificationException even with concurrent modifications!
```

### 🗣️ How to Explain in Interview

> *"CopyOnWriteArrayList creates a new array copy on every write operation. That sounds expensive, and it is — O(n) for every add/remove. But reads are completely lock-free — they just access the current array reference, which is volatile. Iteration uses a snapshot — even if another thread modifies the list, the iterator sees the version at the time it was created. This makes it perfect for listener lists, event handlers, and configuration — scenarios where reads happen thousands of times for every write."*

### ⚡ Key Points to Remember

1. **Write** = copy entire array (expensive O(n))
2. **Read** = lock-free, fast (just volatile reference)
3. **Snapshot iteration** — no ConcurrentModificationException
4. Best for: **read-heavy, write-rare** (listener lists)
5. **Don't** use for write-heavy workloads (use ConcurrentLinkedQueue)

---

<a id="q55"></a>

## Q55. What is BlockingQueue?

### 🔑 Quick Answer

> A thread-safe queue with **blocking operations**: `put()` blocks when full, `take()` blocks when empty. The **standard solution** for producer-consumer pattern. No manual wait/notify needed.

### 📖 Step-by-Step Explanation

```
BlockingQueue (capacity = 3):

  Producer → put("A") → [A _ _]      ← Space available, instant
  Producer → put("B") → [A B _]      ← Space available, instant
  Producer → put("C") → [A B C]      ← Space available, instant
  Producer → put("D") → [BLOCKED!]   ← Queue full, waits...

  Consumer → take()   → [B C] → "A"  ← Returns "A", unblocks producer
  Producer → (unblocked) → [B C D]   ← Now has space
```

| Method | When Full | When Empty |
|--------|-----------|-----------|
| **put()** | Blocks ✅ | — |
| **take()** | — | Blocks ✅ |
| **offer()** | Returns false | — |
| **poll()** | — | Returns null |
| **offer(timeout)** | Blocks with timeout | — |
| **poll(timeout)** | — | Blocks with timeout |

### 🗣️ How to Explain in Interview

> *"BlockingQueue is the production standard for producer-consumer. It provides put() which blocks when the queue is full, and take() which blocks when it's empty — no manual wait/notify needed. I use ArrayBlockingQueue for fixed-size queues — like a task pipeline with bounded memory. LinkedBlockingQueue for optional bounding with potentially higher throughput. The choice of blocking vs non-blocking operations depends on the use case — put/take for reliable delivery, offer/poll for non-critical or timeout-based scenarios."*

### ⚡ Key Points to Remember

1. **put()** = blocking add, **take()** = blocking remove
2. **offer/poll** = non-blocking alternatives
3. Thread-safe — **no manual synchronization**
4. Production standard for **producer-consumer**
5. Main implementations: **ArrayBlockingQueue**, **LinkedBlockingQueue**

---

<a id="q56"></a>

## Q56. Difference between ArrayBlockingQueue and LinkedBlockingQueue?

### 🔑 Quick Answer

| Feature | ArrayBlockingQueue | LinkedBlockingQueue |
|---------|-------------------|-------------------|
| **Internal** | Array (fixed size) | Linked nodes |
| **Bounded** | Always bounded | Optionally bounded |
| **Lock** | Single lock (1 for put+take) | Two locks (put + take separate) |
| **Memory** | Predictable (pre-allocated) | Grows per element (GC pressure) |
| **Throughput** | Lower under contention | Higher (separate locks) |

### 📖 Step-by-Step Explanation

```
ArrayBlockingQueue:
  ONE lock for both put and take → contention between producers and consumers
  Pre-allocated array → no GC pressure
  Always bounded (capacity required at construction)

LinkedBlockingQueue:
  TWO separate locks (putLock + takeLock)
  → Producer and consumer don't block EACH OTHER ✅
  → Higher throughput under high contention
  Linked nodes → GC pressure on each add/remove
  Optional bound (Integer.MAX_VALUE if not specified ⚠️)
```

### 🗣️ How to Explain in Interview

> *"ArrayBlockingQueue uses a single lock for both put and take — producers and consumers contend on the same lock. It uses a fixed-size pre-allocated array, so memory is predictable. LinkedBlockingQueue uses two separate locks — one for put, one for take — so producers and consumers don't block each other, giving higher throughput under contention. However, LinkedBlockingQueue creates a new node per element, causing GC pressure. I use ArrayBlockingQueue for most cases because bounded memory is predictable. LinkedBlockingQueue when I need higher throughput with heavy concurrent producers and consumers."*

### ⚡ Key Points to Remember

1. **ArrayBlockingQueue** = 1 lock, bounded, predictable memory
2. **LinkedBlockingQueue** = 2 locks, higher throughput, GC pressure
3. ArrayBQ: **always bounded** (safe)
4. LinkedBQ: **default unbounded** ⚠️ (specify capacity!)
5. For most cases: **ArrayBlockingQueue** is safer and simpler

---

> **🎯 Navigation:** [← Concurrency Utilities (Q37-50)](04-concurrency-utilities.md) | [Next → Memory Model (Q57-62)](06-memory-model.md) | [📋 All Sections](README.md)
