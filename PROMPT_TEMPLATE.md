
# 📋 PROMPT TEMPLATE

### _Use this template to bulk-add questions to the Interview Prep System_


---

## 🚀 How to Use

1. **Copy questions** from LinkedIn, websites, interview experiences, etc.
2. **Paste them in Copilot Chat** using any of the formats below
3. **Copilot will automatically:**
   - ✅ Categorize each question into the right topic file
   - ✅ Add in-depth explanation, hints, and solution approaches
   - ✅ Add code samples in multiple languages
   - ✅ Add tags, difficulty level, and source
   - ✅ Update the index and progress tracker
   - ✅ Add related questions and key takeaways

---

## 📝 Format 1: Simple List (Easiest)

Just paste your questions as a list. Copilot will auto-detect the category:

```
Here are my questions, add them to the interview prep system:

1. What is the difference between == and === in JavaScript?
2. Explain closures in JavaScript with an example.
3. What is the time complexity of binary search?
4. How does HashMap work internally in Java?
5. Design a URL shortener like bit.ly
6. What is the difference between SQL and NoSQL?
7. Explain SOLID principles with examples.
8. What is Docker and why do we use it?
9. Tell me about a time you handled a conflict at work.
10. What is the difference between process and thread?
```

---

## 📝 Format 2: With Categories (Better Organization)

```
Add these to interview prep system:

## JavaScript
- What is event loop?
- Explain promises vs async/await
- What is hoisting?

## React
- What are hooks? Explain useState and useEffect
- What is virtual DOM?
- Explain React lifecycle methods

## DSA
- Two Sum problem
- Reverse a linked list
- Find the longest palindromic substring

## System Design
- Design Instagram
- Design a chat application like WhatsApp

## HR
- Why should we hire you?
- Where do you see yourself in 5 years?
```

---

## 📝 Format 3: With Source & Tags (Most Detailed)

```
Add these to interview prep system:

1. Question: What is the difference between let, var, and const?
   Category: JavaScript
   Source: LinkedIn Post by @xyz
   Difficulty: Easy
   Tags: #javascript #basics #must-know

2. Question: Design a rate limiter
   Category: System Design
   Source: Amazon Interview Experience
   Difficulty: Hard
   Tags: #system-design #amazon #hard

3. Question: Explain the event loop in Node.js
   Category: Node.js / Web Dev
   Source: LinkedIn
   Difficulty: Medium
   Tags: #nodejs #event-loop #frequently-asked
```

---

## 📝 Format 4: Raw Paste (Laziest — Still Works!)

Just paste anything — screenshot text, LinkedIn post content, random notes:

```
Bro LinkedIn var ek post hoti about React interview questions:
- virtual dom kasa kaam karta
- hooks mhnje kai
- useEffect chi dependency array
- context api vs redux
- server side rendering

plus ek system design question hota:
- design Netflix recommendation system

ani ek DSA:
- find kth largest element in array
```

> 💡 **Copilot will understand any format and organize everything properly!**

---

## 📝 Format 5: Topic/Concept Deep Dive

When you want in-depth notes on a topic (not just questions):

```
Add a deep dive on these topics:

1. Topic: Event Loop in JavaScript
   - Full explanation with diagrams
   - Call stack, callback queue, microtask queue
   - Common interview questions about it

2. Topic: Database Indexing
   - How indexes work internally (B-Tree, Hash)
   - When to use and when not to
   - Performance impact

3. Topic: Microservices Architecture
   - Pros and cons
   - Communication patterns (REST, gRPC, Message Queue)
   - Real world examples
```

---

## 📝 Format 6: Company Interview Experience

```
Add this interview experience:

Company: Amazon
Role: SDE-1
Date: March 2026
Rounds: 4

Round 1 (Online Assessment):
- Two Sum variation
- Longest substring without repeating characters

Round 2 (Technical):
- Design a parking lot (LLD)
- Implement LRU Cache

Round 3 (Technical):
- Design Amazon's order tracking system (HLD)
- Questions about microservices

Round 4 (Behavioral):
- Tell me about a time you disagreed with your manager
- Leadership principles discussion
```

---

## ⚡ Quick Commands

You can also use these quick commands in chat:

| Command | What it does |
|---------|-------------|
| `Add these questions: [list]` | Adds questions with full answers |
| `Deep dive: [topic]` | Creates detailed topic notes |
| `Add interview experience: [details]` | Adds company interview experience |
| `Add tips: [tips list]` | Adds to tips & tricks section |
| `Add to cheat sheet: [topic]` | Creates/updates cheat sheet |
| `Update progress` | Updates the progress tracker |
| `Add roadmap: [role]` | Creates a learning roadmap |

---

## ✅ Post-Batch Sync Checklist
### _Run this after EVERY batch. Takes < 5 minutes. Nothing missed._

**STEP 1 — Folder README** _(the folder where file was added)_
- [ ] Add new file row to the table: `| N | Topic Name | Q count | [Go →](./filename.md) |`
- [ ] Update total count in heading/badge

**STEP 2 — Parent README(s)**
- [ ] `java/README.md` → update subtotal for the changed topic (Core / Spring / MT / SB / Arch / Prod)
- [ ] `languages/README.md` → update Java total (if a Java folder changed)

**STEP 3 — Root README.md**
- [ ] Update matching row in the Table of Contents (question count)
- [ ] Update Grand Total row in Progress Tracker table
- [ ] Add entry to 🔄 Update Log: `| YYYY-MM-DD | Batch N | Short summary |`

**STEP 4 — daily-updates/README.md**
- [ ] Add row under `## 📅 Month YYYY` (add heading if new month)
- [ ] Format: `| Date | Batch N | X Qs — topic (count), topic (count) | Category | file links |`

**STEP 5 — STUDY_PLAN.md**
- [ ] Add new file to the correct Phase table (see "Phase Assignment Rules" in STUDY_PLAN.md)
- [ ] Update memory note: new total question count

**STEP 6 — Memory Note** _(tell GitHub Copilot to update)_
- [ ] "Update my memory: question count is now [N] after Batch [N]"

---

> 💡 **Phase Assignment Quick Reference**
> Java basics/OOP/collections → Phase 1 | Java 8/JVM/Streams → Phase 2 |
> Multithreading → Phase 3 | Spring Boot/JPA/DB → Phase 4 |
> System Design/Architecture → Phase 5 | Production/Spring Batch/DevOps → Phase 6 |
> Company interviews/HR/Testing/Frontend → Phase 7

---


### 🎯 Just paste your content — the system handles the rest!

]]>
