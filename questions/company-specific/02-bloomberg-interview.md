# 🏢 Bloomberg — Senior SWE Interview Experience (5 Rounds)

> Complete 5-round Bloomberg Senior Software Engineer interview. Rounds: DS&Algo (×2), System Design, Team Lead, Managerial.

---

## Q1. Design a MinStack — push(), pop(), getMin() in O(1)

### 📝 One-Liner
Design a stack that supports push, pop, and retrieving the minimum element, all in constant time.

### 🔑 Quick Answer
Use two stacks: main stack for values, auxiliary stack tracking the current minimum at each level. On push, push min(value, currentMin) to aux stack. On pop, pop both. `getMin()` = peek aux stack. *(do stacks rakho — ek values ke liye, ek minimum track karne ke liye)*

### 📖 How It Works
- **Push(x)**: Push `x` to main stack. Push `min(x, auxStack.peek())` to aux stack.
- **Pop()**: Pop from both stacks. Return main stack's popped value.
- **getMin()**: Return `auxStack.peek()` — always the current minimum.
- All operations: **O(1) time, O(n) space**

**Space Optimization**: Only push to aux when new value ≤ current min. On pop, only pop aux if main's popped value equals aux's peek. *(space bachane ke liye sirf minimum change hone pe aux mein push karo)*

### 💻 Code
```java
class MinStack {
    private Deque<Integer> stack = new ArrayDeque<>();
    private Deque<Integer> minStack = new ArrayDeque<>();
    
    public void push(int val) {
        stack.push(val);
        int min = minStack.isEmpty() ? val : Math.min(val, minStack.peek());
        minStack.push(min);
    }
    
    public int pop() {
        minStack.pop();
        return stack.pop();
    }
    
    public int top() {
        return stack.peek();
    }
    
    public int getMin() {
        return minStack.peek();
    }
}

// Space-optimized: single stack with encoded values
class MinStackOptimized {
    private Deque<Long> stack = new ArrayDeque<>();
    private long min;
    
    public void push(int val) {
        if (stack.isEmpty()) {
            stack.push(0L);
            min = val;
        } else {
            stack.push((long) val - min); // store difference
            if (val < min) min = val;
        }
    }
    
    public int pop() {
        long top = stack.pop();
        if (top < 0) { // this was the min
            int oldMin = (int) min;
            min = min - top; // restore previous min
            return oldMin;
        }
        return (int) (top + min);
    }
    
    public int getMin() { return (int) min; }
}
```

### ⚡ Remember
- Two-stack approach: simplest, O(n) extra space
- Single-stack with difference: O(1) extra space but uses long to avoid overflow
- Follow-up: "What if we need getMax() too?" → third stack or two-deque approach
- LeetCode #155 — classic stack problem

---

## Q2. Design a Phonebook Data Structure (Multi-Lookup)

### 📝 One-Liner
Design a phonebook supporting lookups by name, phone number, and name prefix — efficiently.

### 🔑 Quick Answer
Use multiple data structures: HashMap for O(1) by-name and by-number lookup, Trie for prefix search. Or a single TreeMap for sorted name access + prefix range queries. *(alag alag lookups ke liye alag data structures — HashMap + Trie combine karo)*

### 📖 How It Works
1. **By Name (exact)**: `HashMap<String, Contact>` → O(1)
2. **By Number (exact)**: `HashMap<String, Contact>` (reverse index) → O(1)
3. **By Prefix**: Trie storing names → O(prefix length) to find, then collect all matches
4. **Alternative**: `TreeMap<String, Contact>` → `subMap(prefix, prefix + '\uffff')` for prefix range

### 💻 Code
```java
class Phonebook {
    private Map<String, Contact> byName = new HashMap<>();
    private Map<String, Contact> byNumber = new HashMap<>();
    private TrieNode prefixTrie = new TrieNode();
    
    public void addContact(String name, String number) {
        Contact c = new Contact(name, number);
        byName.put(name, c);
        byNumber.put(number, c);
        insertTrie(name, c);
    }
    
    public Contact lookupByName(String name) {
        return byName.get(name); // O(1)
    }
    
    public Contact lookupByNumber(String number) {
        return byNumber.get(number); // O(1)
    }
    
    public List<Contact> lookupByPrefix(String prefix) {
        return searchTrie(prefix); // O(prefix + matches)
    }
    
    // TreeMap alternative for prefix
    // TreeMap<String, Contact> sorted = new TreeMap<>();
    // sorted.subMap(prefix, prefix + Character.MAX_VALUE).values()
}

record Contact(String name, String number) {}
```

### 🆚 vs. Comparison
| Lookup Type | HashMap | TreeMap | Trie |
|------------|---------|---------|------|
| Exact name | O(1) ✅ | O(log n) | O(name.length) |
| Exact number | O(1) ✅ | O(log n) | N/A |
| Prefix search | O(n) scan | O(log n + k) ✅ | O(prefix + k) ✅ |
| Memory | Low | Medium | High |

### ⚡ Remember
- Multiple indexes = fast multi-type lookup (trade space for speed)
- Trie: best for prefix/autocomplete queries
- TreeMap.subMap: elegant prefix search without Trie overhead
- Follow-up: "What about fuzzy search?" → edit distance, BK-tree

---

## Q3. Design a Transit System — Entry, Exit, Average Time

### 📝 One-Liner
Track rider entry/exit at stations and compute average transit time between any two stations.

### 🔑 Quick Answer
Use a HashMap for active riders (entry station + time), and a HashMap of station-pair stats (total time + count) for averages. *(rider ka entry time track karo, exit pe calculate karo, average nikalte jao)*

### 📖 How It Works
1. **entry(rider, station)**: Store `{station, timestamp}` for rider
2. **exit(rider, station)**: Look up entry → compute duration → add to station-pair stats → remove rider entry
3. **getAverageTransitTime(from, to)**: Return `totalTime / count` for the station pair
4. All operations: **O(1) time**

### 💻 Code
```java
class TransitSystem {
    // Active riders: riderId → {entryStation, entryTime}
    private Map<String, StationTime> activeRiders = new HashMap<>();
    // Station pair stats: "stationA→stationB" → {totalTime, count}
    private Map<String, double[]> stationStats = new HashMap<>();
    
    public void entry(String rider, String station) {
        activeRiders.put(rider, new StationTime(station, System.currentTimeMillis()));
    }
    
    public void exit(String rider, String station) {
        StationTime entry = activeRiders.remove(rider);
        if (entry == null) throw new IllegalStateException("No entry for rider: " + rider);
        
        double duration = System.currentTimeMillis() - entry.time;
        String key = entry.station + "→" + station;
        
        stationStats.merge(key, new double[]{duration, 1},
            (old, val) -> new double[]{old[0] + val[0], old[1] + val[1]});
    }
    
    public double getAverageTransitTime(String entryStation, String exitStation) {
        String key = entryStation + "→" + exitStation;
        double[] stats = stationStats.get(key);
        if (stats == null || stats[1] == 0) return 0;
        return stats[0] / stats[1];
    }
    
    record StationTime(String station, long time) {}
}
```

### ⚠️ Pitfalls / Gotchas
- Handle rider calling exit without entry (defensive check)
- Thread safety: `ConcurrentHashMap` + `LongAdder` for concurrent access
- Floating point precision: use `long` for milliseconds, convert at query time

### ⚡ Remember
- Two Maps: active riders + station-pair aggregates
- Running average: store `{totalTime, count}`, compute on query
- All operations O(1)
- Follow-up: "What about top-3 slowest routes?" → maintain a sorted structure

---

## Q4. Design a Rules Management System

### 📝 One-Liner
A system where users create conditions-based rules that evaluate events and trigger actions.

### 🔑 Quick Answer
Rule = conditions + actions. Engine evaluates incoming events against rule conditions using expression parsing. Matched rules trigger associated actions. Store rules in DB, evaluate with Rete algorithm or simple pattern matching. *(rules conditions ke saath store karo, events aaye toh match karo, action trigger karo)*

### 📖 How It Works
1. **Rule Definition**: Conditions (e.g., `A = B + C AND B * C > 1000`) + Actions (e.g., send alert)
2. **Expression Parser**: Parse conditions into AST (Abstract Syntax Tree)
3. **Event Ingestion**: Events arrive with data (field-value pairs)
4. **Rule Evaluation**: For each event, evaluate all applicable rules against event data
5. **Action Execution**: Matched rules trigger their actions (email, API call, state change)

**Architecture**:
```
┌──────────┐     ┌──────────────┐     ┌───────────────┐
│ Rule API │────→│ Rule Store   │     │ Event Stream  │
│ (CRUD)   │     │ (DB + Cache) │←────│ (Kafka)       │
└──────────┘     └──────┬───────┘     └───────┬───────┘
                        │                     │
                 ┌──────▼─────────────────────▼──────┐
                 │        Rule Engine                 │
                 │  ├── Load rules (cached)           │
                 │  ├── Parse conditions (AST)        │
                 │  ├── Evaluate against event data   │
                 │  └── Trigger matched actions       │
                 └──────────────┬─────────────────────┘
                                │
                 ┌──────────────▼──────────────┐
                 │ Action Executor (async)      │
                 │ └── Email, Webhook, DB write  │
                 └─────────────────────────────┘
```

### 💻 Code
```java
// Rule model
record Rule(String id, String conditionExpression, List<Action> actions) {}

// Condition evaluation (simplified)
public class RuleEngine {
    private final ExpressionParser parser = new SpelExpressionParser();
    private final List<Rule> rules;
    
    public List<Action> evaluate(Map<String, Object> eventData) {
        StandardEvaluationContext ctx = new StandardEvaluationContext();
        eventData.forEach(ctx::setVariable);
        
        return rules.stream()
            .filter(rule -> {
                Expression exp = parser.parseExpression(rule.conditionExpression());
                return Boolean.TRUE.equals(exp.getValue(ctx, Boolean.class));
            })
            .flatMap(rule -> rule.actions().stream())
            .collect(Collectors.toList());
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Expression injection**: sanitize user-defined conditions *(user ki expression validate karo — injection ka risk)*
- **Performance**: thousands of rules → Rete algorithm or indexed evaluation
- **Ordering**: rules may conflict — define priority/ordering strategy
- **Testing**: rules must be testable in isolation (dry-run mode)

### ⚡ Remember
- Parse → AST → Evaluate → Act
- SpEL (Spring Expression Language) or MVEL for condition parsing
- Rete algorithm: efficient for many rules (forward chaining)
- Cache compiled rules — don't re-parse on every event
- Real-world: Drools, Easy Rules, custom engines

---

## Q5. Technical Decision-Making Scenarios

### 📝 One-Liner
Demonstrate how you evaluate trade-offs and make technology/architecture decisions under uncertainty.

### 🔑 Quick Answer
Use a structured framework: define the problem → list options → evaluate against criteria (performance, cost, team skill, time) → decide → document → validate with POC if risky. *(problem define karo, options list karo, criteria ke against evaluate karo, decide karo)*

### 📖 How It Works
**Framework**:
1. **Define**: What problem are we solving? What are the constraints?
2. **Options**: List 2-3 viable approaches
3. **Criteria**: Performance, maintainability, team expertise, timeline, cost
4. **Evaluate**: Score each option against criteria
5. **Decide**: Pick one, document the reasoning
6. **Validate**: Spike/POC if the decision is high-risk or irreversible

### 🗣️ Answering Approach
"When I needed to choose between Kafka and RabbitMQ for our event system, I first defined the requirements: ordering guarantees, throughput of 50K events/sec, and replay capability. I evaluated both against these criteria — Kafka won on ordering and replay, RabbitMQ on simplicity. Given our scale needs and the team's familiarity with Kafka, we chose Kafka. I documented the decision in an ADR (Architecture Decision Record) and validated latency requirements with a 2-day POC."

### ⚡ Remember
- Always explain the "WHY" behind your decisions
- Mention trade-offs you explicitly accepted
- Show you considered alternatives, not just picked your favorite
- ADRs (Architecture Decision Records) = professional approach

---

## Q6. How Do You Prioritize Tasks?

### 📝 One-Liner
Show a systematic approach to prioritization based on impact, urgency, dependencies, and stakeholder alignment.

### 🔑 Quick Answer
Eisenhower matrix (urgent vs important), impact/effort scoring, stakeholder alignment. Production issues first, then blocking items, then high-impact features, then tech debt. *(urgent aur important pehle, blocking items uske baad, phir high-impact features)*

### 🗣️ Answering Approach
"I use a combination of impact and urgency. Production issues are always P0 — they come first. Then I look at what's blocking other team members. For feature work, I prioritize by business impact relative to effort. I keep a visible backlog so the team and stakeholders are aligned. In my last project, when we had a P0 incident alongside a release deadline, I split the team — 2 engineers on the incident, 3 on the release — and we delivered both."

### ⚡ Remember
- P0 (production down) > P1 (blocking) > P2 (high-impact) > P3 (nice-to-have)
- Make priorities visible — shared board/backlog
- Re-prioritize when new information arrives
- Communicate changes to stakeholders proactively

---

## Q7. How Do You Handle Production Incidents?

### 📝 One-Liner
Demonstrate a structured incident response: detect, communicate, mitigate, root-cause, and prevent recurrence.

### 🔑 Quick Answer
Follow incident management: detect (alerts/monitoring) → acknowledge → communicate (status page, Slack) → mitigate (rollback, scale, hotfix) → root cause analysis (5 whys) → post-mortem (blameless) → action items. *(pehle detect karo, communicate karo, fix karo, phir root cause nikalo, post-mortem likho)*

### 🗣️ Answering Approach
"In my last incident, our payment service latency spiked to 5 seconds. Here's how I handled it: First, I acknowledged the alert and posted in our #incidents channel with initial status. I pulled thread dumps and found connection pool exhaustion. I mitigated by scaling horizontally and increasing pool size. Root cause was a slow downstream API without a circuit breaker. Post-mortem action items: add circuit breaker, set connection timeout to 3s, add pool exhaustion alerting."

### ⚡ Remember
- **Mitigate first, investigate later** — restore service before finding root cause
- Blameless post-mortems — focus on systems, not people
- 5 Whys: drill down to root cause systematically
- Action items: each must have an owner and deadline

---

## Q8. Conflict with Team Member — Resolution

### 📝 One-Liner
Show emotional intelligence, active listening, and collaborative problem-solving in interpersonal conflicts.

### 🔑 Quick Answer
Listen first, understand their perspective, find common ground, propose a solution that serves the team/project goals. Escalate only if needed, always keep it professional. *(pehle suno, samjho, common ground dhundho, team ke goal ke liye solution nikalo)*

### 🗣️ Answering Approach
"I disagreed with a colleague about using microservices vs monolith for a new project. Instead of arguing in the meeting, I scheduled a 1:1. I listened to their concerns about operational complexity — valid points. I shared my concerns about scaling. We agreed to start as a modular monolith with clean boundaries, with a migration path to microservices when traffic justified it. This compromise addressed both concerns and the team was aligned."

### ⚡ Remember
- Private conversation > public argument
- Active listening: repeat back their concerns to show understanding
- Focus on the GOAL, not on being right
- Document the agreed solution so it doesn't resurface

---

## Q9. Initiative Outside Core Responsibility

### 📝 One-Liner
Demonstrate proactive ownership and impact beyond your assigned role.

### 🗣️ Answering Approach
"I noticed our deployment process took 45 minutes with manual steps and frequent rollback-triggering errors. Even though DevOps wasn't my responsibility, I spent evenings over two weeks building a CI/CD pipeline with GitHub Actions. I automated building, testing, and deploying to staging. After validating with the team, we adopted it for production too. Deployment time dropped to 8 minutes, and deployment failures dropped from 30% to near zero."

### ⚡ Remember
- Show initiative: identify problem → propose solution → deliver
- Quantify impact: time saved, errors reduced, team velocity improved
- Show collaboration: didn't go rogue, got team buy-in
- Shows leadership potential beyond technical skills

---

## Q10. Managing Deadlines and Pressure in Release Cycle

### 📝 One-Liner
Show how you maintain quality and team morale under tight deadlines.

### 🗣️ Answering Approach
"During a tight release, we had 3 features to deliver in 2 weeks with the team at 80% capacity due to a team member being sick. I re-scoped with the PM — we agreed to ship 2 features fully and defer one non-critical feature to the next sprint. I broke remaining work into smaller PRs for faster reviews, set up daily 15-min syncs instead of a lengthy standup, and I personally took the most complex feature. We shipped on time with zero critical bugs."

### ⚡ Remember
- Re-scope honestly vs. silently cutting corners
- Break large tasks into smaller deliverables
- Protect team from burnout — sustainable pace
- Communicate trade-offs to stakeholders EARLY

---

## Q11. How Your Work Aligns with Business Outcomes

### 📝 One-Liner
Connect technical contributions to measurable business impact.

### 🗣️ Answering Approach
"I always tie my work to business metrics. For example, I optimized our search API response time from 800ms to 120ms. The business impact: conversion rate on search pages increased by 15% because users weren't abandoning slow searches. I worked with the product team to identify this correlation. Similarly, when I implemented caching for our product catalog, it reduced infrastructure costs by $2K/month by cutting database load by 60%."

### ⚡ Remember
- Technical metric → business metric mapping
- Latency → conversion, Revenue → ROI
- Infrastructure savings = direct cost impact
- Shows you think beyond code

---

## Q12. A Mistake You Made and What You Learned

### 📝 One-Liner
Show self-awareness, accountability, and growth from a genuine mistake.

### 🗣️ Answering Approach
"Early in my career, I deployed a database migration to production without testing it against production-sized data. The migration locked a critical table for 20 minutes during peak hours, causing partial outage. I immediately rolled back, communicated the impact, and ran the migration during the maintenance window. What I learned: always test migrations against production data volumes, use online schema change tools like pt-online-schema-change, and implement a deployment checklist. I later drove adoption of a migration testing pipeline for the team."

### ⚡ Remember
- Pick a REAL mistake (not a humble brag like "I work too hard")
- Show: What happened → Impact → How you fixed it → What you learned → How it changed your behavior
- Systemic improvement: didn't just fix the symptom, changed the process
- Shows maturity and continuous improvement mindset

---

## Q13. Why Bloomberg?

### 📝 One-Liner
Show genuine research about Bloomberg's tech culture, products, and how your skills align with their mission.

### 🗣️ Answering Approach
"Three things attract me to Bloomberg. First, the engineering challenges — processing billions of real-time financial data points requires systems that are both low-latency and highly reliable, which aligns with my experience in real-time data pipelines. Second, Bloomberg's commitment to engineering culture — the open-source contributions like the BDE libraries and involvement in C++ standards show investment in engineering excellence. Third, the impact — Bloomberg Terminal is the backbone of global finance, and contributing to that infrastructure is meaningful to me."

### ⚡ Remember
- Research: Bloomberg Terminal, BDE libraries, real-time market data
- Connect YOUR skills to THEIR challenges
- Show genuine interest in the domain (finance + tech)
- Don't just say "great company" — be specific

---
