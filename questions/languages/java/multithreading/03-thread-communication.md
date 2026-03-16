# 📡 Thread Communication (Q29–Q36)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q29"></a>
## Q29. What is inter-thread communication?

### 📝 One-Liner
> Threads signaling each other using wait/notify or BlockingQueue — "I'm done, your turn."

### 🔑 Quick Answer
> Inter-thread communication is **threads coordinating** by signaling each other. Java provides `wait()/notify()` (low-level) and `BlockingQueue` (high-level). Without it, threads would **busy-wait** — wasting CPU. *(Threads ek dusre ko batate hain — "mera kaam ho gaya, ab tu kaam kar")*

### 📖 How It Works
```
Without communication (busy-waiting — BAD):
  Consumer: while (!dataReady) { } // burns CPU doing nothing! 💀
  *(CPU jal raha hai — kuch nahi kar raha bus check kar raha hai)*

With wait/notify (GOOD):
  Consumer: wait();              // sleeps, zero CPU ✅
  Producer: data = produce(); notify();  // wakes consumer
  Consumer: (wakes up) process(data);
  *(Soya hua hai — jab data ready ho tab jagao)*

With BlockingQueue (BEST):
  Consumer: data = queue.take();  // blocks until data available ✅
  Producer: queue.put(data);       // wakes consumer automatically
  *(Queue handle karta hai sab — hum sirf put/take karo)*
```

### 🗣️ How to Say in Interview
> *"Inter-thread communication is how threads coordinate work. The classic mechanism is wait/notify — a thread calls wait() to release the lock and sleep until another thread calls notify(). In production, I always use BlockingQueue instead — it handles all the synchronization internally. put() blocks when full, take() blocks when empty. No manual wait/notify needed."*

### ⚠️ Pitfalls / Gotchas
- Busy-waiting (`while(!ready)`) wastes CPU *(100% CPU bekaar mein)*
- wait/notify is error-prone — easy to miss a signal or get wrong
- **BlockingQueue** = production standard *(interview mein bhi BlockingQueue bolo — senior approach dikhta hai)*

### ⚡ Remember
1. **Busy-waiting** = CPU waste ❌
2. **wait/notify** = low-level, error-prone
3. **BlockingQueue** = production standard ⭐
4. Always in **while loop** with wait() *(spurious wakeup)*

### 🔗 Follow-ups
→ [Q30. wait/notify/notifyAll](#q30) → [Q35. BlockingQueue solution](#q35)

---

<a id="q30"></a>
## Q30. How do wait(), notify(), and notifyAll() work?

### 📝 One-Liner
> wait() releases lock and sleeps; notify() wakes one waiting thread; notifyAll() wakes all.

### 🔑 Quick Answer
> - `wait()` — releases lock, thread goes to **WAITING** state *(soja, lock chhod de)*
> - `notify()` — wakes **one** random waiting thread *(ek ko jagao)*
> - `notifyAll()` — wakes **all** waiting threads *(sab ko jagao)* — preferred ⭐

### 📖 How It Works
```
Object has a "wait set" — list of threads that called wait()

  Thread-1: synchronized(obj) { obj.wait(); }  → enters wait set
  Thread-2: synchronized(obj) { obj.wait(); }  → enters wait set
  Thread-3: synchronized(obj) { obj.wait(); }  → enters wait set

  Thread-4: synchronized(obj) { obj.notify(); }
  → Wakes ONE random thread from wait set (say Thread-2)
  → Thread-2 moves from WAITING → BLOCKED (tries to re-acquire lock)
  
  Thread-4: synchronized(obj) { obj.notifyAll(); }
  → Wakes ALL threads from wait set
  → All move to BLOCKED → compete for lock → one gets in
  *(notify = ek ko jagao; notifyAll = sab ko jagao aur compete karo)*
```

### 💻 Code
```java
// Producer-consumer with wait/notify
synchronized (lock) {
    while (queue.isEmpty()) {      // WHILE not IF! (spurious wakeup ke liye)
        lock.wait();               // release lock, sleep
    }
    item = queue.remove();         // got data!
}

// Producer side
synchronized (lock) {
    queue.add(item);
    lock.notifyAll();              // wake all waiting consumers
}
```

### ⚠️ Pitfalls / Gotchas
- Must be called inside `synchronized` → else `IllegalMonitorStateException`
- Always use **while loop**, never `if` *(spurious wakeup — bina notify ke uth sakta hai)*
- `notify()` wakes random thread — might wake the wrong one *(galat ko jaga diya toh kuch nahi hoga)*
- **notifyAll()** is safer — all threads check their condition

### 🎯 Tricky Interview Qs
**Q: Why while loop and not if with wait()?**
> Spurious wakeup — JVM can wake a thread without notify(). While loop re-checks the condition. *(Bina reason ke uth sakta hai — dobara check karo condition)*

### ⚡ Remember
1. `wait()` = release lock + sleep *(lock chhodo, soja)*
2. `notify()` = wake one (random) *(ek jagao)*
3. `notifyAll()` = wake all (safer) ⭐ *(sab jagao)*
4. **while loop** always *(spurious wakeup)*
5. Must be inside **synchronized** *(warna error)*

### 🔗 Follow-ups
→ [Q31. Why wait needs synchronized](#q31) → [Q33. notify vs notifyAll](#q33)

---

<a id="q31"></a>
## Q31. Why does wait() need to be called inside synchronized?

### 📝 One-Liner
> To prevent "lost wakeup" — if notify() fires before wait(), the signal is missed forever.

### 🔑 Quick Answer
> Without synchronized, there's a **race condition**: notify() can fire between condition check and `wait()` — the signal is **lost forever**, thread sleeps permanently. *(Agar synchronized nahi toh notify miss ho jaayega — thread soya rahega forever)*

### 📖 How It Works
```
WITHOUT synchronized (BROKEN):
  Consumer:                     Producer:
  if (queue.isEmpty())          
                                queue.add(item);
                                obj.notify();    ← LOST! no one is waiting yet
  obj.wait();                   ← sleeps FOREVER (missed the notify)
  *(Notify pehle aa gaya, wait baad mein — signal kho gaya)*

WITH synchronized (CORRECT):
  Consumer:                     Producer:
  synchronized(lock) {          
    while (queue.isEmpty())     synchronized(lock) { ← BLOCKED (consumer has lock)
      lock.wait();              
  }                             queue.add(item);
  // releases lock ──────────→  lock.notifyAll();     ← consumer is now waiting ✅
                                }
```

### 🗣️ How to Say in Interview
> *"wait() must be inside synchronized to prevent the 'lost wakeup' problem. Without synchronization, there's a race: the producer might call notify() between the consumer's condition check and wait() call — the signal is lost, and the consumer waits forever. synchronized ensures the check-and-wait is atomic from the signaling thread's perspective."*

### ⚡ Remember
1. Prevents **lost wakeup** *(signal miss na ho)*
2. Check + wait must be **atomic** *(beech mein koi na aaye)*
3. `IllegalMonitorStateException` if not in synchronized *(Java force karta hai)*

### 🔗 Follow-ups
→ [Q32. What happens on notify()](#q32)

---

<a id="q32"></a>
## Q32. What happens when notify() is called?

### 📝 One-Liner
> One waiting thread moves from WAITING → BLOCKED (must re-acquire lock before running).

### 🔑 Quick Answer
> `notify()` picks **one random** thread from the object's wait set, moves it from WAITING → BLOCKED. The woken thread must **re-acquire the monitor lock** before it can continue. The notifying thread **keeps the lock** until it exits synchronized. *(Ek ko jagaya — par woh turant nahi chalega, pehle lock lena padega)*

### 📖 How It Works
```
  Thread-1: wait() → [WAITING in wait set]
  
  Thread-2: synchronized(lock) {
    lock.notify();        // wakes Thread-1
    // Thread-2 still holds the lock!
    doSomeWork();         // Thread-1 can't run yet (BLOCKED)
  }                       // Thread-2 exits sync → releases lock
  
  Thread-1: [WAITING] → [BLOCKED] → acquires lock → [RUNNABLE] → continues after wait()
```

### ⚡ Remember
1. Woken thread goes WAITING → **BLOCKED** (not RUNNABLE) *(jagaya par lock chahiye)*
2. Notifier **keeps the lock** until sync block exits
3. Woken thread runs only after **re-acquiring** the lock
4. Must re-check condition in **while loop** *(condition phir se check karo)*

### 🔗 Follow-ups
→ [Q33. notify vs notifyAll](#q33)

---

<a id="q33"></a>
## Q33. What is the difference between notify() and notifyAll()?

### 📝 One-Liner
> notify() wakes one random thread; notifyAll() wakes all — notifyAll is safer.

### 🆚 vs. Comparison
| Feature | notify() | notifyAll() |
|---------|----------|-------------|
| Wakes | 1 random thread *(ek ko jagao)* | All waiting threads *(sab ko jagao)* |
| Risk | Wrong thread woken → others starve | All check condition → correct one proceeds |
| Performance | Slightly faster | Slightly more contention |
| **Safety** | Risky ⚠️ | **Safe** ✅ ⭐ |

### 📖 How It Works
```
notify() problem:
  3 consumers waiting for different conditions:
  C1: waiting for item-type-A
  C2: waiting for item-type-B
  C3: waiting for item-type-A
  
  Producer adds type-A item → notify() → wakes C2 (random!)
  C2 checks: "not type-B" → goes back to wait()
  C1 and C3 still sleeping → TYPE-A ITEM NOT PROCESSED! 💀
  *(Galat thread jag gaya — sahi wala soya raha)*

notifyAll() solution:
  Producer adds type-A → notifyAll() → wakes C1, C2, C3
  C1: "type-A! process it" ✅
  C2: "not type-B" → wait()
  C3: "type-A but C1 got it" → wait()
  → Correct thread always picks up ✅
```

### 🗣️ How to Say in Interview
> *"I always use notifyAll() instead of notify(). notify() wakes a random thread — if that thread's condition isn't met, the signal is effectively lost. notifyAll() wakes all threads, each re-checks its condition in a while loop, and the right one proceeds. The slight overhead of waking extra threads is negligible compared to the risk of missed signals."*

### ⚡ Remember
1. **Always use notifyAll()** in production ⭐ *(sab ko jagao, sahi wala kaam karega)*
2. notify() = risky (wrong thread may wake) *(galat thread jag sakta)*
3. notifyAll() + while loop = **safe pattern**

### 🔗 Follow-ups
→ [Q34. Producer-consumer problem](#q34)

---

<a id="q34"></a>
## Q34. What is the producer-consumer problem?

### 📝 One-Liner
> Classic concurrency problem: producer adds data to shared buffer, consumer removes — need coordination when buffer is full/empty.

### 🔑 Quick Answer
> Producer creates data, puts in a **shared buffer**. Consumer takes from that buffer. Issues: buffer **full** (producer must wait), buffer **empty** (consumer must wait). *(Ek daalta hai, ek nikalta hai — jab jagah nahi toh ruko, jab khaali toh ruko)*

### 📖 How It Works
```
Buffer (capacity = 3):
  Producer: put("A") → [A _ _]      ← ok
  Producer: put("B") → [A B _]      ← ok
  Producer: put("C") → [A B C]      ← ok
  Producer: put("D") → WAIT! (full) ← buffer bhar gaya, ruko
  
  Consumer: take()   → [B C] → "A"  ← freed space
  Producer: (wakes)  → [B C D]      ← ab jagah hai, daalo
```

### 🗣️ How to Say in Interview
> *"The producer-consumer is the fundamental concurrency pattern. Producer threads put data into a shared bounded buffer, consumer threads take from it. When the buffer is full, producers wait. When empty, consumers wait. In Java, I solve this with BlockingQueue — put() blocks on full, take() blocks on empty. No manual wait/notify needed. It's the backbone of most async processing — message queues, task pipelines, event processing."*

### ⚡ Remember
1. Producer → buffer → Consumer *(chain)*
2. Buffer full → producer waits *(jagah nahi)*
3. Buffer empty → consumer waits *(data nahi)*
4. **BlockingQueue** = ready-made solution ⭐

### 🔗 Follow-ups
→ [Q35. wait/notify solution](#q35) → [Q36. BlockingQueue solution](#q36)

---

<a id="q35"></a>
## Q35. How to implement producer-consumer with wait/notify?

### 📝 One-Liner
> synchronized + while + wait/notifyAll — producer waits when full, consumer waits when empty.

### 💻 Code
```java
public class SharedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 5;

    // Producer — daalo, agar jagah nahi toh ruko
    public synchronized void produce(int item) throws InterruptedException {
        while (queue.size() == capacity) {  // WHILE not IF!
            wait();  // buffer full → release lock, sleep
        }
        queue.add(item);
        System.out.println("Produced: " + item);
        notifyAll();  // wake consumers (sab ko jagao)
    }

    // Consumer — nikalo, agar khaali toh ruko
    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) {  // WHILE not IF!
            wait();  // buffer empty → release lock, sleep
        }
        int item = queue.remove();
        System.out.println("Consumed: " + item);
        notifyAll();  // wake producers (sab ko jagao)
        return item;
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Use **while**, never **if** for wait() *(spurious wakeup — bina notify ke uth sakta hai)*
- `notifyAll()` not `notify()` *(galat thread jag sakta hai)*
- This is **interview code** — in production, use **BlockingQueue** *(manually mat karo)*

### ⚡ Remember
1. **while** + wait() *(spurious wakeup ke liye)*
2. **notifyAll()** *(sabko jagao)*
3. This code = interview answer | Production = **BlockingQueue** ⭐

### 🔗 Follow-ups
→ [Q36. BlockingQueue solution](#q36)

---

<a id="q36"></a>
## Q36. How to implement producer-consumer with BlockingQueue?

### 📝 One-Liner
> Just use put() and take() — BlockingQueue handles all synchronization internally.

### 🔑 Quick Answer
> `BlockingQueue.put()` blocks when full, `take()` blocks when empty. **Zero manual synchronization**. Production-grade solution in 5 lines. *(put daalo, take nikalo — baaki sab queue sambhal lega)*

### 💻 Code
```java
// Itna simple hai — bas put aur take!
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

// Producer
Runnable producer = () -> {
    for (int i = 0; i < 100; i++) {
        queue.put(i);  // blocks if full (jagah nahi toh ruko)
    }
};

// Consumer
Runnable consumer = () -> {
    while (true) {
        int item = queue.take();  // blocks if empty (data nahi toh ruko)
        process(item);
    }
};

// Start
executor.submit(producer);
executor.submit(consumer);
```

### 🆚 vs. Comparison
| Feature | wait/notify | BlockingQueue |
|---------|-----------|--------------|
| Code complexity | ~30 lines *(jhanjhat)* | ~5 lines *(simple)* |
| Error-prone | Very ⚠️ | No ✅ |
| Production-ready | ❌ | ✅ ⭐ |
| Performance | OK | Optimized |

### 🗣️ How to Say in Interview
> *"In production, I always use BlockingQueue for producer-consumer. put() blocks when the queue is full, take() blocks when empty — no manual synchronization needed. I use ArrayBlockingQueue for fixed-size bounded queues with predictable memory, and LinkedBlockingQueue when I need higher throughput with separate put and take locks. The wait/notify solution is good to know for interviews, but I'd never use it in production code."*

### ⚠️ Pitfalls / Gotchas
- **Always specify capacity** for LinkedBlockingQueue — default is Integer.MAX_VALUE *(unbounded = OOM risk!)*
- `offer()` returns false if full (instead of blocking) — use `put()` for guaranteed delivery

### ⚡ Remember
1. `put()` blocks on full, `take()` blocks on empty *(automatic!)*
2. **ArrayBlockingQueue** = fixed size, predictable *(safe choice)*
3. **LinkedBlockingQueue** = always specify capacity *(default unbounded = danger)*
4. **Zero manual sync** — queue handles everything ⭐
5. This is the **production answer** *(interview mein bolo toh senior lagoge)*

### 🔗 Follow-ups
→ [Q55. BlockingQueue deep dive](#q55) → [Q56. ArrayBQ vs LinkedBQ](#q56)

---

> **🎯 Navigation:** [← Synchronization (Q18-28)](02-synchronization.md) | [Next → Concurrency Utilities (Q37-50)](04-concurrency-utilities.md) | [📋 All Sections](README.md)
