# 🧠 DSA — Classic Interview Coding Problems

> Common coding problems asked at Atlassian, JPMorgan, PhonePe & US product companies.

---

## Q1. Check if a String is a Palindrome

### 📝 One-Liner
Check whether a string reads the same forwards and backwards.

### 🔑 Quick Answer
Compare the string with its reverse — if equal, it's a palindrome. *(ulta seedha same ho toh palindrome)*

### 📖 How It Works
- Use two pointers: one from start, one from end
- Compare characters moving inward *(dono taraf se compare karo)*
- Time: O(n), Space: O(1) with two pointers

### 🗣️ How to Say in Interview
"I'd use the two-pointer technique — one pointer at the start and one at the end, comparing characters as they move toward the center. This gives O(n) time and O(1) space."

### 💻 Code
```java
public static boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) return false;
        left++;
        right--;
    }
    return true;
}
// Java 8 one-liner:
// new StringBuilder(s).reverse().toString().equals(s)
```

### ⚠️ Pitfalls / Gotchas
- Handle null and empty strings
- Case sensitivity — normalize with `toLowerCase()`
- Ignore non-alphanumeric characters for phrase palindromes

### ⚡ Remember
- Two-pointer: O(n) time, O(1) space
- StringBuilder reverse: O(n) time, O(n) space
- Edge cases: empty string, single char → both palindromes

---

## Q2. Combination Sum II (Backtracking)

### 📝 One-Liner
Find all unique combinations of candidates that sum to a target, each number used at most once.

### 🔑 Quick Answer
Sort the array, then use backtracking with skip-duplicate logic. *(sort karo, phir backtrack karo — duplicate skip)*

### 📖 How It Works
- Sort candidates to handle duplicates
- Backt rack: at each index, include or skip the number
- Skip consecutive same values at the same recursion depth *(same level pe same value skip karo)*
- Time: O(2^n), Space: O(target)

### 🗣️ How to Say in Interview
"I'd sort the candidates first, then use backtracking. At each step I choose to include or exclude the current number. To avoid duplicates, I skip consecutive same values at the same recursion depth."

### 💻 Code
```java
public List<List<Integer>> combinationSum2(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int remain, int start,
                       List<Integer> path, List<List<Integer>> result) {
    if (remain == 0) { result.add(new ArrayList<>(path)); return; }
    for (int i = start; i < nums.length; i++) {
        if (nums[i] > remain) break;          // pruning
        if (i > start && nums[i] == nums[i-1]) continue; // skip dups
        path.add(nums[i]);
        backtrack(nums, remain - nums[i], i + 1, path, result);
        path.remove(path.size() - 1);
    }
}
```

### ⚡ Remember
- Sort first → enables duplicate skip + pruning
- `i > start` — not `i > 0` — for same-level duplicate check
- Different from Combination Sum I where you CAN reuse

---

## Q3. Find Missing Integer in Consecutive Array

### 📝 One-Liner
Given n-1 integers from 1 to n, find the one missing number.

### 🔑 Quick Answer
Use the formula `n*(n+1)/2` minus the array sum — the difference is the missing number. *(formula se total nikalo, array sum ghatao)*

### 📖 How It Works
- Mathematical: Sum of 1..n = n*(n+1)/2, subtract actual sum
- XOR approach: XOR all numbers 1..n with all array elements — remaining value is the missing one *(XOR same numbers cancel ho jaate hain)*
- Both are O(n) time, O(1) space

### 🗣️ How to Say in Interview
"I'd use the mathematical approach — compute the expected sum using n*(n+1)/2 and subtract the actual array sum. For overflow safety, I could use XOR instead."

### 💻 Code
```java
// Math approach
public static int findMissing(int[] arr, int n) {
    int expectedSum = n * (n + 1) / 2;
    int actualSum = Arrays.stream(arr).sum();
    return expectedSum - actualSum;
}

// XOR approach (overflow-safe)
public static int findMissingXOR(int[] arr, int n) {
    int xor = 0;
    for (int i = 1; i <= n; i++) xor ^= i;
    for (int num : arr) xor ^= num;
    return xor;
}
```

### ⚡ Remember
- Math: simple but risk integer overflow for very large n
- XOR: no overflow risk, same O(n)
- Variant: if range is 0..n, adjust formula to `n*(n+1)/2`

---

## Q4. Move All Zeroes to End of Array

### 📝 One-Liner
Move all 0s to the end while maintaining relative order of non-zero elements.

### 🔑 Quick Answer
Use a write-pointer: copy non-zero elements forward, then fill remaining with zeros. *(non-zero aage copy karo, baaki zero bhar do)*

### 📖 How It Works
- Two-pointer: `writeIdx` starts at 0
- Iterate: if current != 0, write to `writeIdx` and increment
- After loop, fill `writeIdx..end` with 0
- Time: O(n), Space: O(1), in-place

### 🗣️ How to Say in Interview
"I'd use a two-pointer approach — one pointer for reading and one for writing. I copy every non-zero element to the write position, then fill the rest with zeros. This is O(n) and in-place."

### 💻 Code
```java
public static void moveZeroes(int[] nums) {
    int writeIdx = 0;
    for (int num : nums) {
        if (num != 0) nums[writeIdx++] = num;
    }
    while (writeIdx < nums.length) {
        nums[writeIdx++] = 0;
    }
}
```

### ⚡ Remember
- Two-pointer pattern — same as "remove element" problem
- Maintains relative order of non-zero elements
- Single pass + fill = O(n)

---

## Q5. Check if Two Strings are Anagrams

### 📝 One-Liner
Determine if two strings contain the exact same characters with same frequency.

### 🔑 Quick Answer
Sort both strings and compare, OR use a frequency array/map. *(dono ka character count same hona chahiye)*

### 📖 How It Works
- **Sort approach**: Sort both → compare. O(n log n)
- **Frequency approach**: Count each char in string1 (increment), count in string2 (decrement), check all zeros. O(n)
- Handle edge case: different lengths → not anagram

### 🗣️ How to Say in Interview
"I'd first check if lengths differ — if so, they can't be anagrams. Then I'd use a frequency array of size 26 for lowercase letters, increment for the first string and decrement for the second, then verify all counts are zero."

### 💻 Code
```java
public static boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] freq = new int[26];
    for (char c : s.toCharArray()) freq[c - 'a']++;
    for (char c : t.toCharArray()) freq[c - 'a']--;
    for (int count : freq) {
        if (count != 0) return false;
    }
    return true;
}

// Java 8 approach
public static boolean isAnagramStreams(String s, String t) {
    return Arrays.equals(
        s.chars().sorted().toArray(),
        t.chars().sorted().toArray()
    );
}
```

### ⚡ Remember
- Frequency array: O(n) time, O(1) space (fixed 26)
- Sort: O(n log n) time — simpler but slower
- For Unicode: use `HashMap<Character, Integer>` instead of `int[26]`

---

## Q6. Longest Common Prefix Among Strings

### 📝 One-Liner
Find the longest prefix string shared by all strings in an array.

### 🔑 Quick Answer
Compare characters column by column across all strings — stop when mismatch found. *(sab strings ko column-wise compare karo)*

### 📖 How It Works
- Start with first string as prefix
- For each subsequent string, shrink prefix until it matches
- `indexOf(prefix) != 0` → chop last char from prefix *(prefix ko chhota karte jao jab tak match na ho)*
- Time: O(S) where S = sum of all characters

### 🗣️ How to Say in Interview
"I'd start with the first string as the assumed prefix, then iterate through remaining strings, shrinking the prefix until it's a valid prefix of each string."

### 💻 Code
```java
public static String longestCommonPrefix(String[] strs) {
    if (strs == null || strs.length == 0) return "";
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (strs[i].indexOf(prefix) != 0) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    }
    return prefix;
}
```

### ⚡ Remember
- Worst case: all strings identical → O(S)
- Best case: first comparison has no match → O(minLen × n)
- Sort-first optimization: only compare first and last after sort

---

## Q7. Longest Increasing Subsequence (LIS)

### 📝 One-Liner
Find the length of the longest strictly increasing subsequence in an array.

### 🔑 Quick Answer
DP approach: O(n²) with `dp[i]` = LIS ending at i. Optimal: O(n log n) using patience sorting with binary search. *(binary search se optimal solution milta hai)*

### 📖 How It Works
- **DP O(n²)**: For each `i`, check all `j < i` where `nums[j] < nums[i]`, `dp[i] = max(dp[j] + 1)`
- **Binary Search O(n log n)**: Maintain a "tails" array — for each number, replace the first tail that's >= it, or append if larger than all *(tails array maintain karo, binary search se position dhundho)*

### 🗣️ How to Say in Interview
"For optimal O(n log n), I'd maintain a tails array where tails[i] is the smallest tail element of an increasing subsequence of length i+1. For each element, I use binary search to find its position."

### 💻 Code
```java
// O(n log n) approach
public static int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int num : nums) {
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1);
        if (pos == tails.size()) tails.add(num);
        else tails.set(pos, num);
    }
    return tails.size();
}
```

### ⚡ Remember
- O(n²) DP: simple, good enough for n ≤ 10⁴
- O(n log n) patience sort: interview-impressive
- Tails array length = LIS length (but tails ≠ the actual LIS)

---

## Q8. Best Time to Buy and Sell Stock

### 📝 One-Liner
Find maximum profit from one buy-sell transaction given daily prices.

### 🔑 Quick Answer
Track minimum price so far and maximum profit at each step. *(minimum price track karo, har step pe profit check karo)*

### 📖 How It Works
- Single pass: maintain `minPrice` and `maxProfit`
- At each price: update `maxProfit = max(maxProfit, price - minPrice)`, then update `minPrice` *(ek baar mein hi dono track karo)*
- Time: O(n), Space: O(1)

### 🗣️ How to Say in Interview
"I'd do a single pass — track the minimum price seen so far and at each step calculate the potential profit. The maximum of all potential profits is the answer."

### 💻 Code
```java
public static int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE;
    int maxProfit = 0;
    for (int price : prices) {
        if (price < minPrice) minPrice = price;
        else maxProfit = Math.max(maxProfit, price - minPrice);
    }
    return maxProfit;
}
```

### ⚡ Remember
- Can't sell before buying → track min on left
- Single transaction only (Buy & Sell Stock I)
- Variant II (unlimited transactions): sum all upward segments

---

## Q9. Dijkstra's Algorithm — Shortest Path

### 📝 One-Liner
Find the shortest path from a source to all vertices in a weighted graph with non-negative edges.

### 🔑 Quick Answer
Greedy BFS with a priority queue — always process the nearest unvisited vertex. *(sabse paas wala vertex pehle process karo, priority queue se)*

### 📖 How It Works
- Initialize distances: source = 0, all others = ∞
- Priority queue (min-heap) with (distance, vertex)
- Extract min, relax all neighbors: if `dist[u] + weight(u,v) < dist[v]` → update *(agar naya raasta chhota hai toh update karo)*
- Time: O((V + E) log V) with binary heap

### 🗣️ How to Say in Interview
"Dijkstra's uses a min-heap priority queue. Starting from the source with distance 0, I extract the vertex with the smallest distance, relax its neighbors, and push updated distances. Time complexity is O((V+E) log V)."

### 💻 Code
```java
public static int[] dijkstra(int[][] graph, int src) {
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    // PQ: {distance, vertex}
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, src});
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];
        if (d > dist[u]) continue; // stale entry
        for (int v = 0; v < n; v++) {
            if (graph[u][v] > 0 && dist[u] + graph[u][v] < dist[v]) {
                dist[v] = dist[u] + graph[u][v];
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```

### ⚡ Remember
- Only works with non-negative weights (use Bellman-Ford for negative)
- Skip stale PQ entries with `if (d > dist[u]) continue`
- Adjacency list version is better for sparse graphs: O((V+E) log V)

---

## Q10. Coin Change — Minimum Coins

### 📝 One-Liner
Find the minimum number of coins needed to make a given amount.

### 🔑 Quick Answer
Bottom-up DP: `dp[i]` = min coins for amount `i`. For each coin, `dp[i] = min(dp[i], dp[i - coin] + 1)`. *(har amount ke liye minimum coins track karo)*

### 📖 How It Works
- `dp[0] = 0` (zero coins for zero amount)
- For each amount 1..target, try all coins
- `dp[i] = min(dp[i], dp[i - coin] + 1)` if `i - coin >= 0` and `dp[i-coin] != ∞`
- Time: O(amount × coins), Space: O(amount)

### 🗣️ How to Say in Interview
"I'd use bottom-up DP. Create a dp array of size amount+1, initialized to infinity. For each amount from 1 to target, try subtracting each coin and take the minimum. If dp[amount] is still infinity, return -1."

### 💻 Code
```java
public static int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // acts as infinity
    dp[0] = 0;
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### ⚡ Remember
- Classic unbounded knapsack variant
- `amount + 1` as infinity avoids overflow
- Greedy (always pick largest coin) does NOT work for all coin sets

---

## Q11. Reverse-Add Palindrome Problem

### 📝 One-Liner
Given a number, repeatedly reverse and add until the result is a palindrome.

### 🔑 Quick Answer
Loop: reverse the number, add to original, check if palindrome. Repeat until palindrome found. *(reverse karo, jodo, palindrome check karo — repeat)*

### 📖 How It Works
- Start with the number
- Reverse its digits, add to current number
- Check if result is a palindrome
- If not, repeat with the new number *(har step pe naya sum banta hai)*
- Some numbers take many iterations (e.g., 196 is an unsolved conjecture!)

### 🗣️ How to Say in Interview
"I'd iterate: reverse the current number, add it to itself, check if the result is a palindrome. I'd use BigInteger for safety since numbers can grow very large. I'd also set a max iteration limit."

### 💻 Code
```java
import java.math.BigInteger;

public static BigInteger reverseAddPalindrome(long num) {
    BigInteger n = BigInteger.valueOf(num);
    int maxIterations = 1000;
    for (int i = 0; i < maxIterations; i++) {
        String s = n.toString();
        String rev = new StringBuilder(s).reverse().toString();
        if (s.equals(rev)) return n;
        n = n.add(new BigInteger(rev));
    }
    return BigInteger.valueOf(-1); // not found within limit
}
```

### ⚡ Remember
- Use BigInteger — numbers grow rapidly
- Set iteration limit (some numbers may never converge — Lychrel numbers)
- Famous: 196 → no palindrome found even after millions of iterations

---

## Q12. Word Search in 2D Matrix (Right/Down Only)

### 📝 One-Liner
Check if a word exists in a 2D character matrix by moving only right or down.

### 🔑 Quick Answer
DP or recursive search from each cell — move only right or down, matching one character at a time. *(sirf right ya down ja sakte ho, ek ek character match karo)*

### 📖 How It Works
- For each cell matching the first character, start a path search
- At each step, try moving right (col+1) or down (row+1)
- Match the next character in the word
- If all characters matched → word found *(sab characters match ho gaye toh mil gaya)*
- Time: O(m×n×L) worst case, where L = word length

### 🗣️ How to Say in Interview
"I'd iterate through each cell as a potential start. From each starting cell, I'd use recursion trying right and down moves, matching one character at a time. Since we only move right/down, no visited array is needed."

### 💻 Code
```java
public static boolean exist(char[][] board, String word) {
    for (int i = 0; i < board.length; i++) {
        for (int j = 0; j < board[0].length; j++) {
            if (search(board, word, i, j, 0)) return true;
        }
    }
    return false;
}

private static boolean search(char[][] board, String word, int r, int c, int idx) {
    if (idx == word.length()) return true;
    if (r >= board.length || c >= board[0].length) return false;
    if (board[r][c] != word.charAt(idx)) return false;
    // Try right and down only
    return search(board, word, r, c + 1, idx + 1)
        || search(board, word, r + 1, c, idx + 1);
}
```

### ⚡ Remember
- Right/down only → no need for visited tracking (can't revisit)
- Different from LeetCode Word Search which allows all 4 directions
- Optimization: DFS with early termination on mismatch

---
