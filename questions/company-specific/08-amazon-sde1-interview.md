# 🏢 Amazon — SDE-1 Interview Experience (4 Rounds)

> Complete interview process for Software Development Engineer 1 role. 2 onsite rounds (DSA + LLD) and 2 virtual rounds (Leadership Principles + Bar Raiser). Focus on problem-solving, design thinking, and Amazon's Leadership Principles.

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

## Round 1 — Onsite: DSA (45 minutes, 2 problems)

---

<a id="q1"></a>
## Q1. Multi-source BFS — Solve a grid problem where you start BFS from multiple sources simultaneously

### 📝 One-Liner
Multi-source BFS initializes the queue with **all source nodes at once** (distance 0), then expands level-by-level — used for "nearest distance from any source" problems like 01 Matrix, Rotting Oranges, or Walls and Gates.

### 🔑 Quick Answer
**Standard BFS** starts from 1 source. **Multi-source BFS** adds ALL sources to the queue before starting, effectively treating them as a "super source" at distance 0. Each cell gets the minimum distance to any source naturally as BFS expands. **Time**: O(m×n) for a grid. *(Ek source se BFS karte ho, multi-source mein saare sources ko ek saath queue mein daal do — sab se simultaneously expand hota hai)*

### 💻 Code
```java
// Example: 01 Matrix — find distance of each cell to nearest 0
public int[][] updateMatrix(int[][] mat) {
    int m = mat.length, n = mat[0].length;
    int[][] dist = new int[m][n];
    Queue<int[]> queue = new LinkedList<>();

    // Step 1: Add ALL sources (0-cells) to queue
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (mat[i][j] == 0) {
                queue.offer(new int[]{i, j});
                dist[i][j] = 0;
            } else {
                dist[i][j] = Integer.MAX_VALUE;
            }
        }
    }

    // Step 2: BFS from all sources simultaneously
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] cell = queue.poll();
        for (int[] d : dirs) {
            int ni = cell[0] + d[0], nj = cell[1] + d[1];
            if (ni >= 0 && ni < m && nj >= 0 && nj < n
                    && dist[ni][nj] > dist[cell[0]][cell[1]] + 1) {
                dist[ni][nj] = dist[cell[0]][cell[1]] + 1;
                queue.offer(new int[]{ni, nj});
            }
        }
    }
    return dist;
}

// Example: Rotting Oranges (LC #994)
// Add all rotten oranges to queue → BFS → track minutes → check if fresh remain
```

### ⚠️ Pitfalls
| Mistake | Fix |
|---------|-----|
| Starting BFS from each source separately | O(k × m × n) — multi-source is O(m × n) |
| Not marking visited before adding to queue | Can add same cell multiple times |
| Using DFS instead of BFS for shortest distance | BFS guarantees level-order = shortest path |

### ⚡ Remember
> **All sources → queue at once** → BFS level-by-level | O(m×n) not O(k×m×n) | Used in: 01 Matrix, Rotting Oranges, Walls & Gates, Nearest Exit | Classic Amazon grid problem

---

<a id="q2"></a>
## Q2. Sliding Window — Solve an optimization problem using the sliding window technique

### 📝 One-Liner
Sliding window maintains a **dynamic window** over a sequence — expand right to include, shrink left to exclude — tracking the optimal (max/min) window that satisfies constraints, in O(n) time.

### 🔑 Quick Answer
**Pattern**: Two pointers (`left`, `right`). Expand `right` → update state → while constraint violated, shrink `left` → update answer. Works for: max/min subarray, substring with conditions, at-most-K problems. **Amazon favorites**: Max Sum Subarray of Size K, Longest Substring Without Repeating Chars, Minimum Window Substring. *(Sliding window mein right pointer expand karta hai, left pointer shrink — O(n) mein optimal subarray/substring dhundh lete ho)*

### 💻 Code
```java
// Template: Maximum sum subarray of size K
public int maxSumK(int[] arr, int k) {
    int windowSum = 0, maxSum = Integer.MIN_VALUE;
    for (int i = 0; i < arr.length; i++) {
        windowSum += arr[i];
        if (i >= k - 1) {
            maxSum = Math.max(maxSum, windowSum);
            windowSum -= arr[i - k + 1]; // shrink left
        }
    }
    return maxSum;
}

// Template: Variable-size window (shrink when constraint violated)
public int longestWithCondition(int[] arr) {
    int left = 0, maxLen = 0;
    // state tracking (HashMap, counter, etc.)
    for (int right = 0; right < arr.length; right++) {
        // expand: update state with arr[right]
        while (/* constraint violated */) {
            // shrink: remove arr[left] from state
            left++;
        }
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

### ⚡ Remember
> Fixed window: size K, slide by 1 | Variable window: expand right, shrink left | O(n) time | Common patterns: max sum, longest substring, minimum window | Amazon loves sliding window + two pointers

---

## Round 2 — Onsite: Low Level Design (LLD)

---

<a id="q3"></a>
## Q3. Design a Notification System — classes, interfaces, design patterns, and scalability discussion

### 📝 One-Liner
Use **Strategy pattern** for notification channels (Email, SMS, Push), **Observer pattern** for event subscription, **Factory** for channel creation — SOLID principles with clean interface segregation.

### 🔑 Quick Answer
**Core design**: `NotificationService` orchestrates. `NotificationChannel` interface (Strategy) with `EmailChannel`, `SMSChannel`, `PushChannel` implementations. `UserPreference` stores which channels each user subscribes to. `NotificationTemplate` for message formatting. `RetryHandler` for delivery failures. **Scalability**: Queue-based (Kafka/SQS) for async delivery, batch processing for bulk notifications. *(Notification system mein Strategy pattern use karo channels ke liye, Observer pattern events ke liye, Queue se async delivery)*

### 💻 Code
```java
// Strategy — Notification Channel
public interface NotificationChannel {
    void send(Notification notification, User user);
    boolean supports(ChannelType type);
}

@Component
public class EmailChannel implements NotificationChannel {
    @Override
    public void send(Notification notification, User user) {
        // Send email via SMTP/SES
    }
    @Override
    public boolean supports(ChannelType type) { return type == ChannelType.EMAIL; }
}

@Component
public class SMSChannel implements NotificationChannel { /* ... */ }

@Component
public class PushChannel implements NotificationChannel { /* ... */ }

// Service — orchestrates delivery
@Service
public class NotificationService {
    private final List<NotificationChannel> channels; // injected
    private final UserPreferenceRepository prefRepo;

    public void notify(String userId, NotificationEvent event) {
        User user = userRepo.findById(userId);
        UserPreference prefs = prefRepo.getPreferences(userId);
        Notification notification = templateEngine.render(event);

        prefs.getEnabledChannels().stream()
            .map(type -> channels.stream()
                .filter(c -> c.supports(type)).findFirst().orElseThrow())
            .forEach(channel -> channel.send(notification, user));
    }
}

// Domain models
public record Notification(String title, String body, Map<String, Object> data) {}
public enum ChannelType { EMAIL, SMS, PUSH, IN_APP }
public record UserPreference(String userId, Set<ChannelType> enabledChannels) {}
```

### 🗣️ Answering Approach
"I'd model this with the Strategy pattern — a `NotificationChannel` interface with implementations for Email, SMS, and Push. A `NotificationService` looks up the user's channel preferences and sends through each enabled channel. For scalability, I'd make the send operation async by publishing to a Kafka topic per channel type. Each channel has a consumer group that processes messages and handles retries with exponential backoff. Templates are stored separately and rendered with user-specific data. I'd use a `NotificationLog` table for audit trails and delivery status tracking. For the class design, I follow SOLID — each channel is a separate class (SRP), new channels are added without modifying existing code (OCP), and the service depends on the interface not concrete classes (DIP)."

### ⚡ Remember
> **Strategy** for channels | **Observer** for event subscription | Async via Kafka/SQS | Retry with backoff | User preferences drive routing | SOLID throughout

---

## Round 3 — Virtual: Leadership Principles + Implementation

---

<a id="q4"></a>
## Q4. Leadership Principles discussion + OOP-based implementation problem with class design focus

### 📝 One-Liner
Amazon LP round evaluates **ownership, bias for action, customer obsession, and dive deep** through STAR-format behavioral questions, combined with a coding/design problem testing OOP fundamentals.

### 🔑 Quick Answer
**LP preparation**: Use STAR format (Situation, Task, Action, Result) for each LP. **Most tested LPs**: (1) Customer Obsession — going beyond requirements for user. (2) Ownership — taking responsibility beyond your scope. (3) Bias for Action — making decisions with incomplete data. (4) Dive Deep — debugging a complex production issue. (5) Disagree and Commit — pushing back on a decision, then committing. **Implementation**: Expect a class design problem where interviewer evaluates encapsulation, inheritance, interfaces, and clean API design.

### 🗣️ Answering Approach (Example STAR)
"**Ownership example**: In my previous project, we had a recurring data inconsistency between our payment service and the notification service. It wasn't my team's responsibility, but I noticed it was causing customer complaints. I took ownership — *Situation*. I analyzed the event flow and found that occasional Kafka message loss during broker restarts was the root cause — *Task*. I proposed and implemented idempotent consumers with a deduplication table, and added a reconciliation job that runs every hour — *Action*. This reduced data inconsistency incidents from 15/month to zero, and the pattern was adopted across 3 other services — *Result*."

### ⚡ Remember
> **STAR format** for every LP answer | Quantify results (numbers, percentages) | 2-3 stories per LP | Be specific, not generic | "I" not "we" — show YOUR contribution

---

## Round 4 — Virtual: Bar Raiser

---

<a id="q5"></a>
## Q5. Bar Raiser round — deep LP discussion, project deep-dive, and raising-the-bar evaluation

### 📝 One-Liner
The Bar Raiser is an **independent evaluator** from a different team who ensures the candidate raises the hiring bar — focus is on LP depth, ability to handle ambiguity, and long-term thinking.

### 🔑 Quick Answer
**What to expect**: (1) Deep-dive into 1-2 projects with follow-up questions that go 3-4 levels deep. (2) Scenario-based LP questions testing judgment under pressure. (3) Focus on **Learn and Be Curious**, **Earn Trust**, **Think Big**, **Insist on Highest Standards**. (4) May ask about failures and what you learned. **Key**: Be honest, show growth, demonstrate impact with metrics.

### 🗣️ Answering Approach (Handling "Tell me about a failure")
"In one project, I designed a caching layer that initially used TTL-based invalidation. Under load testing, we found that stale data was being served for up to 30 seconds after updates — unacceptable for our financial data use case. I had underestimated the cache invalidation complexity. I redesigned using event-driven cache invalidation with Kafka — whenever the source data changed, an event triggered cache eviction. This reduced stale data window from 30 seconds to under 100 milliseconds. The lesson I learned was to always validate cache consistency under realistic load conditions before shipping, not just functional tests."

### ⚡ Remember
> Bar Raiser is INDEPENDENT — can veto a hire | Go 3-4 levels deep on projects | Own your failures + show learnings | Quantify everything | **"Raise the bar"** = would this person make the team better?

---

<a id="q6"></a>
## Q6. GenAI-related questions — expectations around AI/ML awareness for SDE roles at Amazon

### 📝 One-Liner
Amazon expects SDE-1 candidates to have **awareness** of GenAI concepts — LLMs, prompt engineering, RAG patterns, and how AI integrates into products — not deep ML expertise, but ability to build AI-powered features.

### 🔑 Quick Answer
**Topics to know**: (1) What are LLMs (GPT, Claude, Bedrock). (2) Prompt engineering basics — system prompts, few-shot learning. (3) RAG (Retrieval-Augmented Generation) — combine search with LLM. (4) Amazon Bedrock — managed service for foundation models. (5) AI safety — hallucination, bias, responsible AI. **SDE angle**: How would you integrate an LLM API into a product? How would you handle rate limiting, caching responses, fallback when AI is unavailable?

### ⚡ Remember
> SDE ≠ ML engineer, but must understand GenAI integration | Know LLMs, RAG, prompt engineering | Amazon Bedrock for AWS | Focus on building WITH AI, not building AI models
