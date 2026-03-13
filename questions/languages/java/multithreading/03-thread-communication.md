# 🔔 Thread Communication (Q29–Q36)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q29"></a>

## Q29. What is inter-thread communication?

### 🔑 Quick Answer

> Threads **coordinate** by using `wait()`, `notify()`, and `notifyAll()` on shared objects. One thread waits for a condition; another thread fulfills it and signals. This avoids busy-waiting (polling) and is how producer-consumer works.

### 📖 Step-by-Step Explanation

```
WITHOUT inter-thread communication (busy-waiting):
  Consumer: Is data ready? No. Is data ready? No. Is data ready? No...
  → Wastes CPU cycles constantly checking ❌

WITH inter-thread communication (wait/notify):
  Consumer: wait()  → Sleeps (zero CPU) → notify() → Wakes up → Processes data
  Producer: Produces data → notify()
  → Efficient, no wasted CPU ✅
```

### 🗣️ How to Explain in Interview

> *"Inter-thread communication lets threads coordinate without busy-waiting. Instead of a consumer thread looping 'is data ready? is data ready?', it calls wait() and goes to sleep — zero CPU usage. When the producer thread has data ready, it calls notify(), waking the consumer. This is built on Java's monitor mechanism — wait() and notify() must be called within a synchronized block on the same object. The pattern is fundamental to producer-consumer, blocking queues, and many concurrent designs."*

### ⚡ Key Points to Remember

1. **wait/notify** = efficient coordination (no busy-waiting)
2. Must be inside **synchronized** block on the same object
3. **wait()** → releases lock + sleeps
4. **notify()** → wakes one waiting thread
5. Foundation of **producer-consumer** pattern

---

<a id="q30"></a>

## Q30. What are wait(), notify(), notifyAll()?

### 🔑 Quick Answer

> `wait()` — current thread **releases lock and sleeps** until notified. `notify()` — wakes up **one** waiting thread. `notifyAll()` — wakes up **all** waiting threads. All three must be called inside `synchronized`.

### 📖 Step-by-Step Explanation

```
obj.wait():
  1. Thread RELEASES lock on obj
  2. Thread enters WAITING state (no CPU usage)
  3. Thread waits until: notify(), notifyAll(), or interrupt

obj.notify():
  1. Wakes ONE arbitrary waiting thread (on obj's wait-set)
  2. Awakened thread must RE-ACQUIRE lock before continuing
  3. Calling thread keeps the lock until it exits synchronized

obj.notifyAll():
  1. Wakes ALL threads waiting on obj
  2. All compete to re-acquire the lock
  3. Only one succeeds at a time — others go back to BLOCKED
```

### 💻 Code Example

```java
Object monitor = new Object();

// Thread A: Waits for signal
new Thread(() -> {
    synchronized (monitor) {
        while (!dataReady) {       // Always use WHILE, not IF!
            monitor.wait();        // Release lock + sleep
        }
        // Process data (lock re-acquired automatically)
    }
}).start();

// Thread B: Sends signal
new Thread(() -> {
    synchronized (monitor) {
        dataReady = true;
        monitor.notify();          // Wake Thread A
    }                              // Lock released when exiting synchronized
}).start();
```

### 🗣️ How to Explain in Interview

> *"wait() puts the current thread to sleep and releases the lock — other threads can then enter the synchronized block. notify() wakes up one waiting thread, and notifyAll() wakes all of them. A critical best practice: always call wait() inside a WHILE loop, not an IF statement, because of spurious wakeups — the thread can wake up without being notified. The while loop rechecks the condition, and if it's not actually true, the thread goes back to waiting."*

### ⚡ Key Points to Remember

1. **wait()** = release lock + sleep
2. **notify()** = wake ONE, **notifyAll()** = wake ALL
3. ALL must be inside **synchronized** block
4. Always use **while loop** (not if) for wait — spurious wakeups
5. After notify, awakened thread must **re-acquire lock**

---

<a id="q31"></a>

## Q31. Why must wait() be called inside a synchronized block?

### 🔑 Quick Answer

> Because wait() **releases the monitor lock** — it can only release what it holds. Without synchronized, there's no lock to release, and we'd have a **race condition** between checking the condition and calling wait().

### 📖 Step-by-Step Explanation

```
WITHOUT synchronized (RACE CONDITION):
  Thread A:  if (!dataReady)        // Checks: false
  Thread B:       dataReady = true; notify();  // Signal LOST!
  Thread A:              wait();    // Waits FOREVER (missed notify)

WITH synchronized (SAFE):
  Thread A:  synchronized(m) {
                if (!dataReady)     // Checks: false (holds lock)
                    m.wait();       // Atomically: release lock + wait
             }
  Thread B:  synchronized(m) {     // Must wait for A to call wait()
                dataReady = true;
                m.notify();         // A will receive this
             }
```

### 🗣️ How to Explain in Interview

> *"wait() must be in synchronized because of two reasons. First, technically, wait() needs a lock to release — calling wait() without holding the lock throws IllegalMonitorStateException. Second, and more importantly, it prevents a race condition called 'lost wakeup'. Without synchronized, Thread A might check the condition, then Thread B sets it and calls notify(), then Thread A calls wait() — missing the notify forever. Synchronized ensures the check-and-wait is atomic with respect to the lock."*

### ⚡ Key Points to Remember

1. **wait() releases the lock** — must hold one first
2. Without synchronized → **IllegalMonitorStateException**
3. Prevents **lost wakeup** race condition
4. Check + wait must be **atomic** (inside same synchronized)
5. Same applies to notify() — must hold the monitor

---

<a id="q32"></a>

## Q32. What happens when notify() is called?

### 🔑 Quick Answer

> One **arbitrary** thread from the object's wait-set moves from WAITING to BLOCKED state. It doesn't run immediately — it must **re-acquire the lock** first. The notifying thread keeps the lock until it exits the synchronized block.

### 📖 Step-by-Step Explanation

```
Timeline:
  T1: synchronized(obj) { obj.wait(); }      → Releases lock, enters WAITING
  T2: synchronized(obj) { obj.wait(); }      → Releases lock, enters WAITING
  T3: synchronized(obj) { obj.notify(); }    → ONE of T1/T2 wakes up
                                              → T3 STILL holds lock!
  T3: }                                       → T3 releases lock
  T1: (if chosen) → re-acquires lock → continues after wait()

Important:
  ✅ notify() wakes ONE thread (which one? unpredictable)
  ✅ Awakened thread goes WAITING → BLOCKED (still needs lock)
  ✅ Notifying thread keeps lock until synchronized block ends
  ❌ Awakened thread does NOT run immediately
```

### 🗣️ How to Explain in Interview

> *"When notify() is called, one arbitrary thread from the wait-set wakes up and moves from WAITING to BLOCKED state. It doesn't execute immediately — the notifying thread still holds the lock. Only when the notifying thread exits the synchronized block does the awakened thread get a chance to re-acquire the lock and continue. Which waiting thread gets woken is not specified — it's JVM-implementation dependent. That's why for production code, I prefer notifyAll() or higher-level constructs like BlockingQueue."*

### ⚡ Key Points to Remember

1. Wakes **ONE arbitrary** thread (not predictable which)
2. Awakened thread → **BLOCKED** (needs to re-acquire lock)
3. Notifying thread **keeps lock** until exiting synchronized
4. **Not immediate** — only runs after lock is available
5. Prefer **notifyAll()** or **BlockingQueue** in production

---

<a id="q33"></a>

## Q33. Difference between notify() and notifyAll()?

### 🔑 Quick Answer

> `notify()` wakes **one** arbitrary waiting thread. `notifyAll()` wakes **all** waiting threads (they then compete for the lock). Use **notifyAll()** in most cases to avoid threads being permanently stuck.

### 📖 Step-by-Step Explanation

```
3 threads waiting on obj.wait():

obj.notify():
  T1: WAITING ──→ BLOCKED (chosen to wake)
  T2: WAITING ──→ WAITING (still sleeping)
  T3: WAITING ──→ WAITING (still sleeping)
  
  ⚠️ If T1 rechecks condition and it's still false → calls wait() again
  T2 and T3 NEVER wake up → STUCK!

obj.notifyAll():
  T1: WAITING ──→ BLOCKED
  T2: WAITING ──→ BLOCKED
  T3: WAITING ──→ BLOCKED
  
  All compete for lock, only one runs at a time
  Each rechecks condition in while loop
  ✅ No thread gets permanently stuck
```

### 🗣️ How to Explain in Interview

> *"notify() wakes one thread, notifyAll() wakes all. The problem with notify() is that it might wake a thread whose condition isn't met — that thread goes back to waiting, and no one else wakes up. For example, in a producer-consumer, if I have multiple consumers waiting and I notify() one that can't consume — maybe its specific condition isn't met — the item goes unprocessed and other consumers that could handle it stay asleep. notifyAll() avoids this because all threads wake up and recheck their conditions. It has more overhead but is much safer."*

### ⚡ Key Points to Remember

1. **notify()** = one thread, **notifyAll()** = all threads
2. **notifyAll() is safer** — no risk of stuck threads
3. After wakeup, threads **compete** for lock (one at a time)
4. Each thread **rechecks condition** (while loop)
5. Use **notify()** only if exactly ONE thread can proceed

---

<a id="q34"></a>

## Q34. What is the producer-consumer problem?

### 🔑 Quick Answer

> A classic concurrency problem: **producer** threads create data and put it in a shared buffer, **consumer** threads take data from the buffer. The challenge: producer must wait when buffer is full, consumer must wait when buffer is empty.

### 📖 Step-by-Step Explanation

```
Shared Buffer (capacity = 3):

Producer → [Item1, Item2, Item3] → Consumer
                 FULL!
Producer must WAIT until consumer takes an item

Consumer → [  empty  ] → Nothing to consume
Consumer must WAIT until producer adds an item

RACE CONDITIONS to prevent:
  1. Both threads accessing buffer simultaneously → data corruption
  2. Producer adding to full buffer → overflow
  3. Consumer reading from empty buffer → underflow
```

### 🗣️ How to Explain in Interview

> *"The producer-consumer problem is about coordinating threads that share a bounded buffer. Producers add items and must wait when the buffer is full. Consumers take items and must wait when it's empty. The challenge is synchronizing access to prevent corruption while maximizing throughput. In Java, the cleanest solution is BlockingQueue — it handles all the wait/notify internally. For interviews, knowing the wait/notify implementation shows you understand the underlying mechanism."*

### ⚡ Key Points to Remember

1. **Buffer**: shared, bounded (fixed capacity)
2. **Producer waits** when full, **consumer waits** when empty
3. Must prevent **concurrent access** to buffer
4. Classic solution: `wait()`/`notifyAll()` in synchronized
5. Modern solution: **BlockingQueue** (preferred)

---

<a id="q35"></a>

## Q35. How do you solve producer-consumer using wait/notify?

### 🔑 Quick Answer

> Shared buffer with `synchronized` access. Producer calls `wait()` when full, consumer calls `wait()` when empty. Both call `notifyAll()` after modifying the buffer.

### 💻 Code Example

```java
public class ProducerConsumer {
    private final Queue<Integer> buffer = new LinkedList<>();
    private final int capacity = 5;
    
    // Producer: Add item, wait if full
    public synchronized void produce(int item) throws InterruptedException {
        while (buffer.size() == capacity) {  // Buffer full?
            wait();                           // Release lock + sleep
        }
        buffer.add(item);
        System.out.println("Produced: " + item + " | Size: " + buffer.size());
        notifyAll();                          // Wake consumers
    }
    
    // Consumer: Take item, wait if empty
    public synchronized int consume() throws InterruptedException {
        while (buffer.isEmpty()) {            // Buffer empty?
            wait();                           // Release lock + sleep
        }
        int item = buffer.poll();
        System.out.println("Consumed: " + item + " | Size: " + buffer.size());
        notifyAll();                          // Wake producers
        return item;
    }
}
```

```java
// Modern solution using BlockingQueue (production standard) ⭐
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

// Producer — blocks automatically when full
new Thread(() -> {
    for (int i = 0; i < 100; i++)
        queue.put(i);  // Blocks if queue is full
}).start();

// Consumer — blocks automatically when empty
new Thread(() -> {
    while (true)
        process(queue.take());  // Blocks if queue is empty
}).start();
```

### 🗣️ How to Explain in Interview

> *"The wait/notify implementation: both produce and consume are synchronized methods. The producer checks if the buffer is full — if so, it calls wait() which releases the lock and sleeps. The consumer does the same when the buffer is empty. After adding or removing an item, both call notifyAll() to wake up waiting threads. The key detail is using while loops, not if statements, to handle spurious wakeups. In production, I use ArrayBlockingQueue which encapsulates all this logic — put() blocks when full, take() blocks when empty."*

### ⚡ Key Points to Remember

1. **while loop** for wait condition (not if)
2. **notifyAll()** after producing AND consuming
3. **synchronized** on the shared buffer object
4. Production: use **BlockingQueue** (handles everything)
5. `put()` = blocking add, `take()` = blocking remove

---

<a id="q36"></a>

## Q36. How do you solve producer-consumer using BlockingQueue?

### 🔑 Quick Answer

> `BlockingQueue.put()` blocks when full, `take()` blocks when empty. No manual synchronization needed — it's the **production standard** approach.

### 💻 Code Example

```java
public class ProducerConsumerModern {
    private final BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
    
    public void startProducer() {
        new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    String item = "Item-" + i;
                    queue.put(item);        // Blocks if queue is full
                    System.out.println("Produced: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Producer").start();
    }
    
    public void startConsumer() {
        new Thread(() -> {
            try {
                while (true) {
                    String item = queue.take();  // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    processItem(item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Consumer").start();
    }
}
```

### 🗣️ How to Explain in Interview

> *"BlockingQueue is the production solution for producer-consumer. put() blocks when the queue is full and take() blocks when it's empty — no manual wait/notify needed. I choose ArrayBlockingQueue for bounded buffers with a fixed capacity, or LinkedBlockingQueue for unbounded or very large buffers. The key advantage is it's thread-safe by design, tested, and optimized. I know how wait/notify works underneath, but in production code I never implement it manually."*

### ⚡ Key Points to Remember

1. **put()** = blocks when full, **take()** = blocks when empty
2. **No manual synchronization** — thread-safe by design
3. **ArrayBlockingQueue** = bounded (fixed size)
4. **LinkedBlockingQueue** = optionally bounded
5. Always the **production choice** over manual wait/notify

---

> **🎯 Navigation:** [← Synchronization (Q18-28)](02-synchronization.md) | [Next → Concurrency Utilities (Q37-50)](04-concurrency-utilities.md) | [📋 All Sections](README.md)
