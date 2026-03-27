
# 🧠 Arrays & Strings — DSA Interview Problems (Q1–Q5)

> **Source**: American Express Java Backend Interview (2-4 YOE, March 2026)  
> **Coverage**: Character frequency manipulation, sliding window, stack-based problems

---

<a id="q1"></a>
## Q1. Minimum deletions to make character frequencies unique

> **LeetCode 1647** — Minimum Deletions to Make Character Frequencies Unique (Medium)

### 📝 Problem
Given a string `s`, return the **minimum number of character deletions** to make the string "good" — a string where **no two different characters have the same frequency**.

### 🔑 Approach
**Greedy + Set**: Count frequencies. For each frequency, if it's already taken, decrement it until it's either 0 or an unused frequency. Track used frequencies in a `Set`.

**Time**: O(n) for counting + O(26²) worst case for resolving conflicts = O(n)  
**Space**: O(1) — at most 26 frequencies

### 🗣️ Answering Approach
"I first count the frequency of each character. Then I iterate through the frequencies and greedily resolve conflicts — if a frequency is already used, I keep decrementing it until I find one that's free or reach zero. Each decrement represents one deletion. I use a HashSet to track which frequencies are already taken. The key insight is greedy works because reducing a frequency always costs exactly 1 deletion per step, and we want to minimize total deletions."

### 💻 Code

```java
public int minDeletions(String s) {
    int[] freq = new int[26];
    for (char c : s.toCharArray()) {
        freq[c - 'a']++;
    }

    Set<Integer> usedFreqs = new HashSet<>();
    int deletions = 0;

    for (int f : freq) {
        while (f > 0 && usedFreqs.contains(f)) {
            f--;            // "delete" one occurrence
            deletions++;
        }
        usedFreqs.add(f);   // 0 is harmless (multiple chars with 0 is fine)
    }
    return deletions;
}
```

### 🔍 Dry Run
```
Input: "aaabbbcc"
Frequencies: a=3, b=3, c=2

Process:
  a: freq=3, usedFreqs={} → add 3 → usedFreqs={3}
  b: freq=3, usedFreqs={3} → conflict! decrement to 2 → usedFreqs={2,3}? No, 2 not used → add 2
     deletions=1, usedFreqs={3,2}
  c: freq=2, usedFreqs={3,2} → conflict! decrement to 1 → add 1
     deletions=2, usedFreqs={3,2,1}

Output: 2
```

### ⚡ Key Points
- Sort frequencies descending for intuition, but Set approach is simpler
- `f = 0` means the character is fully deleted — that's valid
- Greedy is optimal — no need for DP

---

<a id="q2"></a>
## Q2. Longest prefix where removing one element makes frequencies equal

### 📝 Problem
Given an array, find the **longest prefix** such that removing exactly one element makes all remaining element frequencies equal.

### 🔑 Approach
**Frequency of Frequencies**: Maintain running frequency counts. At each index, check if removing one element can equalize all frequencies. Valid removal cases:
1. All elements have frequency 1 (remove any)
2. Only one distinct element (remove one occurrence)
3. All frequencies are `f` except one at `f+1` (remove one from the higher)
4. All frequencies are `f` except one element with frequency 1 (remove that element entirely)

**Time**: O(n)  
**Space**: O(n)

### 💻 Code

```java
public int maxEqualFreqPrefix(int[] nums) {
    Map<Integer, Integer> count = new HashMap<>();   // element → frequency
    Map<Integer, Integer> freqCount = new HashMap<>(); // frequency → how many elements have this freq
    int maxFreq = 0, result = 0;

    for (int i = 0; i < nums.length; i++) {
        int c = count.getOrDefault(nums[i], 0);
        if (c > 0) {
            freqCount.merge(c, -1, Integer::sum);
            if (freqCount.get(c) == 0) freqCount.remove(c);
        }
        count.merge(nums[i], 1, Integer::sum);
        int newC = count.get(nums[i]);
        freqCount.merge(newC, 1, Integer::sum);
        maxFreq = Math.max(maxFreq, newC);

        int n = i + 1; // prefix length
        int uniqueElements = count.size();

        // Case 1: all elements appear once → remove any
        if (maxFreq == 1)
            result = n;
        // Case 2: one distinct element → remove one occurrence
        else if (uniqueElements == 1)
            result = n;
        // Case 3: all freq = maxFreq-1, one at maxFreq → remove from max
        else if (freqCount.getOrDefault(maxFreq, 0) == 1
                && maxFreq * 1 + (maxFreq - 1) * (uniqueElements - 1) == n)
            result = n;
        // Case 4: all freq = maxFreq, one element has freq 1 → remove it
        else if (freqCount.getOrDefault(maxFreq, 0) == uniqueElements - 1
                && freqCount.getOrDefault(1, 0) == 1
                && maxFreq * (uniqueElements - 1) + 1 == n)
            result = n;
        // Case 5: all same freq, and removing one empties a bucket (freq=1, n=unique+1)
        else if (maxFreq * uniqueElements + 1 == n
                && freqCount.getOrDefault(maxFreq, 0) == uniqueElements)
            result = n;
    }
    return result;
}
```

### ⚡ Key Points
- Track both `count` (element → freq) and `freqCount` (freq → count of elements with that freq)
- Check multiple edge cases at each prefix length
- O(n) single pass with constant-time checks

---

<a id="q3"></a>
## Q3. Longest substring without repeating characters

> **LeetCode 3** — Longest Substring Without Repeating Characters (Medium)

### 📝 Problem
Given string `s`, find the length of the **longest substring** without repeating characters.

### 🔑 Approach
**Sliding Window + HashMap**: Two pointers `left` and `right`. Expand `right`, track last seen index of each character. When a repeat is found, move `left` past the previous occurrence.

**Time**: O(n)  
**Space**: O(min(n, 26)) = O(1) for lowercase letters

### 🗣️ Answering Approach
"I use a sliding window with two pointers. I expand the right pointer and store each character's last-seen index in a HashMap. When I encounter a character that's already in the current window, I move the left pointer to one past its previous occurrence. At each step, I update the max length as right - left + 1. This gives O(n) time because each character is processed at most twice."

### 💻 Code

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int maxLen = 0, left = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1;   // shrink window past duplicate
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

### 🔍 Dry Run
```
Input: "abcabcbb"

right=0, c='a': lastSeen={a:0}, left=0, maxLen=1
right=1, c='b': lastSeen={a:0,b:1}, left=0, maxLen=2
right=2, c='c': lastSeen={a:0,b:1,c:2}, left=0, maxLen=3
right=3, c='a': lastSeen[a]=0 ≥ left(0) → left=1
            lastSeen={a:3,b:1,c:2}, maxLen=3
right=4, c='b': lastSeen[b]=1 ≥ left(1) → left=2
            lastSeen={a:3,b:4,c:2}, maxLen=3
right=5, c='c': lastSeen[c]=2 ≥ left(2) → left=3
            lastSeen={a:3,b:4,c:5}, maxLen=3
right=6, c='b': lastSeen[b]=4 ≥ left(3) → left=5
            lastSeen={a:3,b:6,c:5}, maxLen=3 (not 2)
right=7, c='b': lastSeen[b]=6 ≥ left(5) → left=7
            maxLen=3

Output: 3 ("abc")
```

### ⚠️ Edge Cases
- Empty string → 0
- All same characters `"aaaa"` → 1
- All unique `"abcdef"` → 6
- **Critical check**: `lastSeen.get(c) >= left` — without `>= left`, stale entries from before the window cause wrong `left` moves

### ⚡ Key Points
- Classic sliding window — O(n) time, O(1) space
- HashMap stores last index, not just presence
- `>= left` check prevents jumping backward

---

<a id="q4"></a>
## Q4. Fruit Into Baskets (sliding window)

> **LeetCode 904** — Fruit Into Baskets (Medium)

### 📝 Problem
You have a row of trees, each with a fruit type `fruits[i]`. You have **2 baskets** — each basket can hold one type of fruit. Starting from any tree, pick fruits going right, but you can only pick if the type fits in one of your baskets. Find the **maximum number of fruits** you can collect.

**Translation**: Find the longest contiguous subarray with **at most 2 distinct values**.

### 🔑 Approach
**Sliding Window + HashMap**: Maintain a window `[left, right]` with a frequency map. When distinct keys exceed 2, shrink from the left until we have ≤ 2 types.

**Time**: O(n)  
**Space**: O(1) — at most 3 keys in map

### 🗣️ Answering Approach
"This is the 'longest subarray with at most K distinct elements' pattern where K=2. I expand the right pointer, adding each fruit type to a frequency map. When the map has more than 2 distinct types, I shrink from the left — decrementing counts and removing types that hit zero — until I'm back to 2 types. The answer is the maximum window size seen."

### 💻 Code

```java
public int totalFruit(int[] fruits) {
    Map<Integer, Integer> basket = new HashMap<>();  // fruit type → count
    int maxFruits = 0, left = 0;

    for (int right = 0; right < fruits.length; right++) {
        basket.merge(fruits[right], 1, Integer::sum);

        while (basket.size() > 2) {
            int leftFruit = fruits[left];
            basket.merge(leftFruit, -1, Integer::sum);
            if (basket.get(leftFruit) == 0) {
                basket.remove(leftFruit);
            }
            left++;
        }
        maxFruits = Math.max(maxFruits, right - left + 1);
    }
    return maxFruits;
}
```

### 🔍 Dry Run
```
Input: [1, 2, 1, 2, 3]

right=0: basket={1:1}, max=1
right=1: basket={1:1,2:1}, max=2
right=2: basket={1:2,2:1}, max=3
right=3: basket={1:2,2:2}, max=4
right=4: basket={1:2,2:2,3:1} → size=3!
  shrink: left=0 (fruit=1), basket={1:1,2:2,3:1}, left=1
  still 3: left=1 (fruit=2), basket={1:1,2:1,3:1}, left=2
  still 3: left=2 (fruit=1), basket={2:1,3:1}, left=3 ✅
  max=max(4, 4-3+1)=max(4,2)=4

Output: 4 → subarray [1,2,1,2]
```

### ⚡ Key Points
- Generalized pattern: "at most K distinct" — just change `2` to `K`
- Each element enters/leaves window at most once → O(n)
- HashMap acts as the "basket" tracking fruit types

---

<a id="q5"></a>
## Q5. Celebrity Problem (stack-based approach)

> **Classic Interview Problem** — Find the Celebrity (Medium)

### 📝 Problem
In a party of `n` people, a **celebrity** is someone who:
- Is **known by everyone** else
- **Knows nobody**

Given a function `knows(a, b)` → true if `a` knows `b`, find the celebrity or return -1. Minimize calls to `knows()`.

### 🔑 Approach
**Stack-based elimination**: Push all people onto a stack. Pop two, eliminate one — if `a` knows `b`, `a` is NOT the celebrity (push `b` back); if `a` doesn't know `b`, `b` is NOT the celebrity (push `a` back). Last person standing is the **candidate**. Verify with a second pass.

**Time**: O(n) — (n-1) comparisons to find candidate + 2(n-1) to verify  
**Space**: O(n) for stack

### 🗣️ Answering Approach
"I use a stack-based elimination approach. First, I push all n people onto a stack. Then I repeatedly pop two people and compare — if A knows B, A can't be the celebrity so I push B back; if A doesn't know B, B can't be the celebrity so I push A back. After n-1 comparisons, one candidate remains. But this only gives a candidate — I must verify with a full pass to confirm everyone knows them and they know nobody. Total: O(n) time with 3(n-1) knows() calls at most."

### 💻 Code

```java
// Assume: boolean knows(int a, int b) is given
public int findCelebrity(int n) {
    // Step 1: Eliminate using stack
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < n; i++) {
        stack.push(i);
    }

    while (stack.size() > 1) {
        int a = stack.pop();
        int b = stack.pop();
        if (knows(a, b)) {
            stack.push(b);   // a knows b → a is NOT celebrity
        } else {
            stack.push(a);   // a doesn't know b → b is NOT celebrity
        }
    }

    int candidate = stack.pop();

    // Step 2: Verify the candidate
    for (int i = 0; i < n; i++) {
        if (i == candidate) continue;
        // Celebrity must be known by everyone AND know nobody
        if (!knows(i, candidate) || knows(candidate, i)) {
            return -1;   // no celebrity
        }
    }
    return candidate;
}
```

### 🔍 Dry Run
```
Input: n=4, celebrity=2
knows matrix:
  0→1:T, 0→2:T, 0→3:F
  1→0:F, 1→2:T, 1→3:F
  2→0:F, 2→1:F, 2→3:F  ← knows nobody
  3→0:F, 3→2:T, 3→1:F

Stack: [0, 1, 2, 3]

Pop 3,2: knows(3,2)=T → push 2 → [0, 1, 2]
Pop 2,1: knows(2,1)=F → push 2 → [0, 2]
Pop 2,0: knows(2,0)=F → push 2 → [2]

Candidate: 2

Verify:
  i=0: knows(0,2)=T ✅, knows(2,0)=F ✅
  i=1: knows(1,2)=T ✅, knows(2,1)=F ✅
  i=3: knows(3,2)=T ✅, knows(2,3)=F ✅

Output: 2 ✅
```

### 🆚 Alternative: Two-Pointer (no stack)

```java
public int findCelebrity(int n) {
    int candidate = 0;
    for (int i = 1; i < n; i++) {
        if (knows(candidate, i)) {
            candidate = i;   // candidate knows i → candidate is out
        }
    }
    // Verify candidate
    for (int i = 0; i < n; i++) {
        if (i != candidate && (!knows(i, candidate) || knows(candidate, i))) {
            return -1;
        }
    }
    return candidate;
}
// Same logic as stack but O(1) space!
```

### ⚡ Key Points
- **Elimination principle** — each `knows()` call eliminates one person
- Stack and two-pointer approaches are equivalent — both O(n)
- **Verification is mandatory** — elimination only gives a candidate
- Two-pointer is preferred (O(1) space vs O(n))

## 📋 Questions Index

| # | Question | Difficulty | Tags | Status |
|---|----------|-----------|------|--------|
| — | _No questions yet_ | — | — | — |

---

<!-- 
========================================
  QUESTION TEMPLATE — Copy this block  
  for each new question added below
========================================

## Q1. ❓ Question Title Here

🔖 **Tags:** `#arrays` `#two-pointer` `#amazon` `#medium`  
📊 **Difficulty:** 🟡 Medium  
🔥 **Frequency:** ⭐⭐⭐⭐ (Frequently Asked)  
📅 **Added:** YYYY-MM-DD  
📌 **Source:** LinkedIn / LeetCode / Interview Experience

---

### 📝 Question
> Write the full question statement here.

---

### 💡 Hints
<details>
<summary>Hint 1</summary>
Think about using a hash map...
</details>

<details>
<summary>Hint 2</summary>
Two pointer approach can help...
</details>

---

### ✅ Solution Approach

**Approach 1: Brute Force**
- Time: O(n²) | Space: O(1)
- Description of approach

**Approach 2: Optimized (Hash Map)**
- Time: O(n) | Space: O(n)
- Description of approach

---

### 💻 Code

```python
# Python Solution
def solution(nums):
    pass
```

```javascript
// JavaScript Solution
function solution(nums) {
}
```

```java
// Java Solution
class Solution {
    public void solve(int[] nums) {
    }
}
```

---

### 🖼️ Explanation Diagram
![Diagram](../../images/placeholder.png)

---

### 🔗 Related Questions
- [Similar Question 1](link)
- [Similar Question 2](link)

---

### 📌 Key Takeaways
> 💡 Important points to remember for interview

---
-->

---

[← Back to DSA](./README.md) | [← Back to Home](../../README.md)
]]>
