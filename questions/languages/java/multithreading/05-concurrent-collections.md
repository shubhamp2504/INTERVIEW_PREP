# 📦 Concurrent Collections (Q51–Q56)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q51"></a>
## Q51. What are concurrent collections?

### 📝 One-Liner
> Thread-safe collections from `java.util.concurrent` — fine-grained locking or lock-free, no external synchronization needed.

### 🔑 Quick Answer
> Concurrent collections allow **multiple threads to access simultaneously** without wrapping in `Collections.synchronizedXxx()`. Instead of locking the **entire** collection, they use **bucket-level locking** or **CAS (lock-free)** — much faster. *(Regular collection mein poora lock hota hai — concurrent mein sirf chhota hissa lock hota hai)*

### 📖 How It Works
```
Regular Collections + synchronized:
  Collections.synchronizedMap(hashMap)
  → Locks ENTIRE map for every operation ❌
  → One thread at a time → slow

Concurrent Collections:
  ConcurrentHashMap
  → Locks only the BUCKET being accessed ✅
  → Multiple threads on different buckets → fast
  *(Poora map lock nahi — sirf ek bucket lock)*
```

| Collection | Concurrent Alternative | Strategy |
|-----------|----------------------|----------|
| HashMap | **ConcurrentHashMap** | Bucket-level CAS |
| ArrayList | **CopyOnWriteArrayList** | Copy on write |
| LinkedList | **ConcurrentLinkedQueue** | Lock-free CAS |
| TreeMap | **ConcurrentSkipListMap** | Lock-free skip list |
| Queue | **ArrayBlockingQueue** / **LinkedBlockingQueue** | Lock-based blocking |

### 🗣️ How to Say in Interview
> *"Concurrent collections from java.util.concurrent are designed for multi-threaded access without external synchronization. Unlike Collections.synchronizedMap() which locks the entire map, ConcurrentHashMap uses bucket-level locking and CAS — so multiple threads can read and write to different buckets simultaneously. In my project, I replaced a synchronizedMap that was causing thread contention with ConcurrentHashMap, which gave us significantly better throughput."*

### ⚠️ Pitfalls / Gotchas
- **Never** use `Collections.synchronizedMap()` in production — too slow *(poora map lock — bahut slow)*
- Concurrent collections have **weakly consistent** iteration — won't throw ConcurrentModificationException but may not reflect latest updates
- **CopyOnWriteArrayList** is terrible for write-heavy workloads *(har write pe poora array copy — expensive)*

### ⚡ Remember
1. **Fine-grained** locking or **lock-free** algorithms
2. **ConcurrentHashMap** = most commonly used ⭐
3. **CopyOnWriteArrayList** = read-heavy, write-rare
4. **BlockingQueue** = producer-consumer pattern
5. **Never** Collections.synchronizedMap in production

### 🔗 Follow-ups
→ [Q52. ConcurrentHashMap](#q52) → [Q55. BlockingQueue](#q55)

---

<a id="q52"></a>
## Q52. What is ConcurrentHashMap?

### 📝 One-Liner
> Thread-safe HashMap with bucket-level CAS + synchronized — lock-free reads, per-bucket writes, the go-to concurrent map.

### 🔑 Quick Answer
> ConcurrentHashMap uses **lock-free reads** (volatile) and **per-bucket synchronized** for writes. Multiple threads can read/write to different buckets **simultaneously**. Provides **atomic** compound operations: `putIfAbsent`, `compute`, `merge`. *(HashMap ka thread-safe version — alag alag bucket pe alag thread kaam kar sakta hai)*

### 📖 How It Works
```
HashMap (NOT safe):
  Thread-1: put("A", 1) → modifies bucket 5 → CORRUPTS 💀
  Thread-2: put("B", 2) → modifies bucket 5 → at same time!

synchronizedMap (too slow):
  Thread-1: [LOCK ENTIRE MAP] → put("A") → [UNLOCK]
  Thread-2: [WAIT...] → [LOCK ENTIRE MAP] → put("B") → [UNLOCK]

ConcurrentHashMap (fast + safe):
  Thread-1: put("A") → lock ONLY bucket 5 → unlock bucket 5
  Thread-2: put("B") → lock ONLY bucket 8 → both run! ✅
  Thread-3: get("A") → NO LOCK (volatile read) ✅
  *(Alag bucket pe koi rok nahi — sab parallel)*

Java 8+ internals:
  [Bucket 0] → Node → Node → ...  (TreeBin if >8 nodes)
  [Bucket 1] → Node → ...
  Write: synchronized(first node of bucket) — ONE bucket locked
  Read:  Lock-free (volatile reads)
  Resize: Concurrent (multiple threads help!)
```

### 💻 Code
```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Thread-safe put/get
map.put("orders", 100);
int count = map.get("orders");

// Atomic compound operations ⭐ (no race condition!)
map.putIfAbsent("orders", 0);                    // put only if absent
map.compute("orders", (key, val) -> val + 1);    // atomic read-modify-write
map.merge("orders", 1, Integer::sum);            // atomic merge

// Thread-safe iteration (weakly consistent)
map.forEach((key, value) -> System.out.println(key + ": " + value));
```

### 🗣️ How to Say in Interview
> *"ConcurrentHashMap is the production standard for thread-safe maps. In Java 8+, reads are lock-free using volatile semantics, and writes lock only the first node of the affected bucket. It provides atomic compound operations like putIfAbsent(), compute(), and merge() — these eliminate the check-then-act race condition. In my project, I use compute() for atomic counter updates and putIfAbsent() for cache initialization."*

### ⚠️ Pitfalls / Gotchas
- **No null keys or values** — unlike HashMap *(null ambiguous hoga — "not found" ya "value null"?)*
- **Weakly consistent** iteration — may not see latest updates
- `size()` is an **estimate** under concurrency — use `mappingCount()` for long

### 🎯 Tricky Interview Qs
**Q: Why doesn't ConcurrentHashMap allow null keys/values?**
> Ambiguity: if `get(key)` returns null, does it mean "key not present" or "value is null"? In single-threaded HashMap, you can call `containsKey()` to check — but in concurrent access, another thread might remove the key between `containsKey` and `get`. *(Null diya toh pata nahi — key nahi hai ya value null hai?)*

### ⚡ Remember
1. **Lock-free reads**, **per-bucket locking** writes
2. **putIfAbsent, compute, merge** = atomic compound ops ⭐
3. **No null** keys or values
4. **Weakly consistent** iteration (safe but may miss updates)
5. Default capacity 16, load factor 0.75

### 🔗 Follow-ups
→ [Q53. HashMap vs ConcurrentHashMap](#q53)

---

<a id="q53"></a>
## Q53. Difference between HashMap and ConcurrentHashMap?

### 📝 One-Liner
> HashMap = not thread-safe, allows null; ConcurrentHashMap = thread-safe, no nulls, atomic operations.

### 🆚 vs. Comparison
| Feature | HashMap | ConcurrentHashMap |
|---------|---------|------------------|
| **Thread-safe** | ❌ No | ✅ Yes |
| **Null keys/values** | ✅ Allowed | ❌ Not allowed |
| **Single-thread perf** | Faster | Slightly slower |
| **Multi-thread** | UNSAFE 💀 | Designed for it ✅ |
| **Iteration** | Fail-fast (CME) | Weakly consistent |
| **Locking** | None | Bucket-level CAS+sync |
| **Atomic ops** | ❌ | ✅ compute, merge |

### ⚠️ Pitfalls / Gotchas
- HashMap in multi-threaded = **infinite loop** during resize (Java 7), **lost updates**, **corrupted state** *(HashMap multi-thread mein use kiya toh data corrupt — infinite loop bhi ho sakta hai)*
- ConcurrentModificationException from HashMap iterator ≠ thread-safety — it's just fail-fast

### 🗣️ How to Say in Interview
> *"HashMap is not thread-safe — using it from multiple threads can cause infinite loops during resize, lost updates, and data corruption. ConcurrentHashMap uses bucket-level locking for writes and lock-free reads. Three key differences: no null keys/values in CHM because null is ambiguous under concurrency, weakly consistent iteration instead of fail-fast, and critically — atomic compound operations like compute() and merge() that eliminate check-then-act race conditions."*

### ⚡ Remember
1. **HashMap** = NEVER in multi-threaded code *(data corrupt hoga)*
2. **ConcurrentHashMap** = designed for concurrency
3. **No nulls** in CHM (ambiguity problem)
4. **Weakly consistent** iteration = safe, may miss recent
5. Use **compute/merge** not get-then-put

### 🔗 Follow-ups
→ [Q54. CopyOnWriteArrayList](#q54)

---

<a id="q54"></a>
## Q54. What is CopyOnWriteArrayList?

### 📝 One-Liner
> Thread-safe ArrayList where every write creates a new array copy — lock-free reads, perfect for read-heavy/write-rare.

### 🔑 Quick Answer
> Every **write** (add/remove) creates a **new copy** of the underlying array. Reads are **completely lock-free** — just access the current volatile reference. Ideal for **listener lists, event handlers, configuration**. *(Har write pe naya array banta hai — read free, write expensive)*

### 📖 How It Works
```
Array: [A, B, C]

Read (Thread-1): reads current array → no lock, fast ✅
Read (Thread-2): reads current array → no lock, fast ✅

Write (Thread-3): add("D")
  1. Lock (ReentrantLock)
  2. Copy: [A, B, C] → new [A, B, C, D]
  3. Replace reference: array = newArray (volatile write)
  4. Unlock

After write:
  Thread-1 (iterating): still sees [A, B, C] ← snapshot ✅
  Thread-4 (new read): sees [A, B, C, D] ← latest ✅
  *(Purana thread purana data dekhega, naya thread naya dekhega)*
```

### 💻 Code
```java
CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

// Writes are rare — adding/removing listeners
listeners.add("listener1");
listeners.add("listener2");

// Reads are frequent — notifying all (no lock!)
for (String listener : listeners) {
    notify(listener);  // snapshot iteration — safe even if modified
}
// No ConcurrentModificationException ever! ✅
```

### ⚠️ Pitfalls / Gotchas
- **O(n) per write** — copies entire array *(write bahut costly — bada array toh aur slow)*
- **Never** for write-heavy workloads *(100 writes/second = 100 array copies = terrible)*
- Iterator is snapshot — won't see changes made after iterator creation

### 🗣️ How to Say in Interview
> *"CopyOnWriteArrayList creates a new array copy on every write — O(n) per add/remove. But reads are completely lock-free. Iteration uses a snapshot — even concurrent modifications don't cause ConcurrentModificationException. I use it for event listener lists and configuration that rarely change but are read thousands of times per second."*

### ⚡ Remember
1. **Write** = copy entire array (expensive O(n))
2. **Read** = lock-free, fast (volatile reference)
3. **Snapshot iteration** — no CME ever
4. Best for: **read-heavy, write-rare** *(listeners, config)*
5. **Never** for write-heavy workloads

### 🔗 Follow-ups
→ [Q55. BlockingQueue](#q55)

---

<a id="q55"></a>
## Q55. What is BlockingQueue?

### 📝 One-Liner
> Thread-safe queue with blocking put() and take() — producer waits when full, consumer waits when empty.

### 🔑 Quick Answer
> `BlockingQueue` adds **blocking operations** to Queue: `put()` blocks when full, `take()` blocks when empty. **Standard solution** for producer-consumer — no manual wait/notify needed. *(Queue bhari hai toh producer ruk jaaye, khaali hai toh consumer ruk jaaye — automatic)*

### 📖 How It Works
```
BlockingQueue (capacity = 3):

  Producer → put("A") → [A _ _]      ← space, instant
  Producer → put("B") → [A B _]      ← space, instant
  Producer → put("C") → [A B C]      ← space, instant
  Producer → put("D") → [BLOCKED!]   ← full, waits...
  *(Queue bhar gayi — producer ruk gaya)*

  Consumer → take()   → [B C] → "A"  ← removes "A", unblocks producer
  Producer → (unblocked) → [B C D]   ← now has space
```

| Method | When Full | When Empty |
|--------|-----------|-----------|
| **put()** | Blocks ✅ | — |
| **take()** | — | Blocks ✅ |
| **offer()** | Returns false | — |
| **poll()** | — | Returns null |
| **offer(timeout)** | Blocks with timeout | — |
| **poll(timeout)** | — | Blocks with timeout |

### 💻 Code
```java
BlockingQueue<Order> queue = new ArrayBlockingQueue<>(100);

// Producer thread
executor.submit(() -> {
    while (running) {
        Order order = receiveOrder();
        queue.put(order);  // blocks if queue full (natural backpressure)
    }
});

// Consumer thread
executor.submit(() -> {
    while (running) {
        Order order = queue.take();  // blocks if queue empty (waits for work)
        processOrder(order);
    }
});
```

### 🗣️ How to Say in Interview
> *"BlockingQueue is the production standard for producer-consumer. put() blocks when full, take() blocks when empty — no manual wait/notify needed. I use ArrayBlockingQueue for bounded queues with predictable memory. The blocking behavior provides natural backpressure — if the consumer is slow, the producer automatically slows down."*

### ⚡ Remember
1. **put()** = blocking add | **take()** = blocking remove
2. **offer/poll** = non-blocking alternatives
3. Thread-safe — **no manual synchronization**
4. **Producer-consumer** standard solution ⭐
5. Implementations: **ArrayBlockingQueue**, **LinkedBlockingQueue**

### 🔗 Follow-ups
→ [Q56. ArrayBQ vs LinkedBQ](#q56)

---

<a id="q56"></a>
## Q56. Difference between ArrayBlockingQueue and LinkedBlockingQueue?

### 📝 One-Liner
> ArrayBQ = single lock, bounded, predictable memory; LinkedBQ = two locks (higher throughput), optionally bounded, GC pressure.

### 🆚 vs. Comparison
| Feature | ArrayBlockingQueue | LinkedBlockingQueue |
|---------|-------------------|-------------------|
| **Internal** | Array (pre-allocated) | Linked nodes |
| **Bounded** | Always (capacity required) | Optional (default ∞ ⚠️) |
| **Lock** | 1 lock (put+take share) | 2 locks (separate put/take) ✅ |
| **Memory** | Predictable | GC pressure per node |
| **Throughput** | Lower under contention | Higher (separate locks) ✅ |

### 📖 How It Works
```
ArrayBlockingQueue:
  ONE lock for both put and take
  → Producer and consumer contend on SAME lock ❌
  → Pre-allocated array → no GC
  *(Ek hi taala — producer consumer ek dusre ko roke)*

LinkedBlockingQueue:
  TWO separate locks (putLock + takeLock)
  → Producer and consumer DON'T block each other ✅
  → New node per element → GC pressure
  *(Do taale — producer aur consumer saath saath chal sakte hain)*
```

### ⚠️ Pitfalls / Gotchas
- LinkedBlockingQueue **default = Integer.MAX_VALUE** → effectively unbounded → **OOM risk** *(hamesha capacity do — warna memory khatam)*
- ArrayBlockingQueue **always requires capacity** — safer by design

### 🗣️ How to Say in Interview
> *"ArrayBlockingQueue uses a single lock for both put and take — producers compete with consumers. It's bounded and memory-predictable. LinkedBlockingQueue uses two separate locks — producers and consumers don't block each other, giving higher throughput under contention. But it creates a new node per element, adding GC pressure. I use ArrayBlockingQueue for most cases for simplicity and safety. LinkedBlockingQueue when I need higher throughput with heavy concurrent producers and consumers — always specifying a capacity."*

### ⚡ Remember
1. **ArrayBQ** = 1 lock, bounded, predictable memory
2. **LinkedBQ** = 2 locks, higher throughput, GC pressure
3. ArrayBQ always **bounded** (safe) ✅
4. LinkedBQ default **unbounded** ⚠️ (specify capacity!)
5. Most cases: **ArrayBQ** is safer and simpler

### 🔗 Follow-ups
→ [Q36. Producer-Consumer with BlockingQueue](03-thread-communication.md#q36)

---

> **🎯 Navigation:** [← Concurrency Utilities (Q37-50)](04-concurrency-utilities.md) | [Next → Memory Model (Q57-62)](06-memory-model.md) | [📋 All Sections](README.md)
