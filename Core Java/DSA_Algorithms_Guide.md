# DSA Algorithms — Complete Guide for LeetCode

> From zero to solving LeetCode problems confidently.

---

## Table of Contents

1. [Two Pointers](#1-two-pointers)
2. [Sliding Window](#2-sliding-window)
3. [Kadane's Algorithm](#3-kadanes-algorithm)
4. [Binary Search](#4-binary-search)
5. [Fast & Slow Pointers (Floyd's Cycle)](#5-fast--slow-pointers-floyds-cycle)
6. [Prefix Sum](#6-prefix-sum)
7. [Monotonic Stack](#7-monotonic-stack)
8. [BFS (Breadth-First Search)](#8-bfs-breadth-first-search)
9. [DFS (Depth-First Search)](#9-dfs-depth-first-search)
10. [Dynamic Programming](#10-dynamic-programming)
11. [Merge Intervals](#11-merge-intervals)
12. [Bit Manipulation](#12-bit-manipulation)
13. [Algorithm Selection Guide](#algorithm-selection-guide)
14. [Time Complexity Summary](#time-complexity-summary)

---

## 1. Two Pointers

### What is it?

The Two Pointers technique uses **two index variables** that traverse a data structure — usually from opposite ends, or one faster than the other. Instead of brute-forcing O(n²) nested loops, you reduce it to **O(n)**.

### When to use?

- Array/string is **sorted** (or can be sorted)
- You need to find a **pair**, **triplet**, or **subarray** satisfying a condition
- You need to **reverse**, **palindrome check**, or **merge** arrays
- Removing duplicates or elements in-place

### Two Variants

**1. Opposite Direction (converging):** Left pointer starts at the beginning, right pointer at the end. They move toward each other.

**2. Same Direction (sliding):** A slow pointer tracks where to write, a fast pointer scans ahead. Common for filtering or compressing arrays.

### How it works (intuition)

```
Array: [-3, -1, 1, 2, 4, 6]   Target sum = 5
        ↑                 ↑
       left             right

Step 1: left + right = -3 + 6 = 3  → too small → move left →
Step 2: left + right = -1 + 6 = 5  → FOUND! ✅
```

The key insight: if the sum is **too small**, we need a bigger left value → move `left` right.
If the sum is **too big**, we need a smaller right value → move `right` left.

### Why this works

Sorting guarantees that moving the left pointer right always increases the sum, and moving the right pointer left always decreases it. This means we never need to revisit pairs — we can always make a meaningful decision at each step.

### Templates

```java
// ── Opposite direction (most common) ────────────────────
int left = 0, right = arr.length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) {
        // found!
        left++;
        right--;
    } else if (sum < target) {
        left++;   // need bigger value
    } else {
        right--;  // need smaller value
    }
}

// ── Same direction (fast/slow or partition) ──────────────
int slow = 0;
for (int fast = 0; fast < arr.length; fast++) {
    if (condition) {
        arr[slow] = arr[fast];
        slow++;
    }
}
```

---

### Problem 1 — Two Sum II (LC #167) ⭐ Easy

**Problem:** Given a sorted array, find two numbers that sum to target. Return 1-indexed positions.

```
Input:  numbers = [2, 7, 11, 15], target = 9
Output: [1, 2]   (numbers[0] + numbers[1] = 2 + 7 = 9)
```

```java
public int[] twoSum(int[] numbers, int target) {
    int left = 0, right = numbers.length - 1;
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        if (sum == target)   return new int[]{left + 1, right + 1};
        else if (sum < target) left++;
        else                   right--;
    }
    return new int[]{-1, -1};
}
// Time: O(n)  Space: O(1)
```

---

### Problem 2 — Valid Palindrome (LC #125) ⭐ Easy

**Key idea:** Skip non-alphanumeric characters and compare characters from both ends.

```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        while (left < right && !Character.isLetterOrDigit(s.charAt(left)))  left++;
        while (left < right && !Character.isLetterOrDigit(s.charAt(right))) right--;
        if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right)))
            return false;
        left++; right--;
    }
    return true;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 3 — Container With Most Water (LC #11) ⭐⭐ Medium

**Key insight:** Width decreases as we move pointers inward. So we must maximize height. Always move the **shorter** side inward — keeping the taller side gives us the best chance of finding a taller wall later, which could compensate for the reduced width.

```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxWater = 0;
    while (left < right) {
        int width  = right - left;
        int h      = Math.min(height[left], height[right]);
        maxWater   = Math.max(maxWater, width * h);
        if (height[left] < height[right]) left++;
        else                              right--;
    }
    return maxWater;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 4 — 3Sum (LC #15) ⭐⭐ Medium

**Strategy:** Fix one number with an outer loop, then use two pointers on the rest. Sort first to enable deduplication.

```java
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;  // skip duplicates
        int left = i + 1, right = nums.length - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                while (left < right && nums[left]  == nums[left+1])  left++;
                while (left < right && nums[right] == nums[right-1]) right--;
                left++; right--;
            } else if (sum < 0) left++;
            else                right--;
        }
    }
    return result;
}
// Time: O(n²)  Space: O(1)
```

---

### Problem 5 — Remove Duplicates from Sorted Array (LC #26) ⭐ Easy

**Key idea:** `slow` tracks the write position; `fast` scans ahead. Whenever `fast` finds a new value (different from the previous element), write it to `slow`.

```java
public int removeDuplicates(int[] nums) {
    int slow = 1;
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[fast - 1]) {
            nums[slow++] = nums[fast];
        }
    }
    return slow;
}
// Time: O(n)  Space: O(1)
```

---

### Two Pointers — LeetCode Problem List

| # | Problem | Difficulty | Key Idea |
|---|---------|-----------|----------|
| 125 | Valid Palindrome | Easy | Opposite pointers |
| 167 | Two Sum II | Easy | Sorted array, opposite ends |
| 26 | Remove Duplicates | Easy | Slow/fast same direction |
| 27 | Remove Element | Easy | Slow/fast same direction |
| 977 | Squares of Sorted Array | Easy | Merge from both ends |
| 11 | Container With Most Water | Medium | Move shorter side |
| 15 | 3Sum | Medium | Sort + fix + two pointers |
| 16 | 3Sum Closest | Medium | Variation of 3Sum |
| 18 | 4Sum | Medium | Fix two, two pointers |
| 75 | Sort Colors | Medium | Dutch flag (3 pointers) |
| 42 | Trapping Rain Water | Hard | Left/right max arrays |
| 632 | Smallest Range | Hard | Multi-pointer |

---

## 2. Sliding Window

### What is it?

A **window** is a contiguous subarray or substring. Instead of recalculating from scratch for every window position, you **slide** the window right — adding one element on the right and removing one on the left — so each transition is O(1). Total: O(n).

### Two types

**Fixed-size window:** Window size `k` is constant. Use for problems like "max sum of k consecutive elements."

**Variable-size window:** Window grows by moving right, shrinks by moving left. Use when you need the longest/shortest window satisfying some condition.

### How it works (intuition)

```
Fixed window (k=3) on [1, 3, 2, 6, 4, 1]:

[1  3  2] 6  4  1   → sum = 6
 1 [3  2  6] 4  1   → sum = 6 + 6 - 1 = 11  (add right, remove left)
 1  3 [2  6  4] 1   → sum = 11 + 4 - 3 = 12
 1  3  2 [6  4  1]  → sum = 12 + 1 - 2 = 11

Answer: max sum = 12
```

**Variable window — shrink when condition breaks:**
```
Find smallest subarray with sum ≥ 7:
 [2  3  1  2  4  3]

window = [2,3,1,2]   sum=8  ≥ 7 ✅ length=4, try shrinking →
window = [3,1,2,4]   sum=10 ≥ 7 ✅ length=4, try shrinking →
window = [1,2,4]     sum=7  ≥ 7 ✅ length=3, try shrinking →
window = [4,3]       sum=7  ≥ 7 ✅ length=2  ← final answer!
```

### Templates

```java
// ── Fixed-size window ───────────────────────────────────
int windowSum = 0;
for (int i = 0; i < k; i++) windowSum += arr[i]; // build initial window
int maxSum = windowSum;
for (int i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];             // slide: add right, remove left
    maxSum = Math.max(maxSum, windowSum);
}

// ── Variable-size window ────────────────────────────────
int left = 0, result = Integer.MAX_VALUE;
int windowSum = 0;
for (int right = 0; right < arr.length; right++) {
    windowSum += arr[right];                       // expand right
    while (windowSum >= target) {                  // condition met — try shrinking
        result = Math.min(result, right - left + 1);
        windowSum -= arr[left++];                  // shrink from left
    }
}
```

---

### Problem 1 — Maximum Average Subarray I (LC #643) ⭐ Easy

```java
public double findMaxAverage(int[] nums, int k) {
    double sum = 0;
    for (int i = 0; i < k; i++) sum += nums[i];
    double maxSum = sum;
    for (int i = k; i < nums.length; i++) {
        sum += nums[i] - nums[i - k];
        maxSum = Math.max(maxSum, sum);
    }
    return maxSum / k;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 2 — Longest Substring Without Repeating Characters (LC #3) ⭐⭐ Medium

**Key idea:** Use a `HashSet` to track characters in the current window. When a duplicate is found, shrink the window from the left until the duplicate is gone.

```java
public int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        while (window.contains(c)) {
            window.remove(s.charAt(left++)); // shrink until no duplicate
        }
        window.add(c);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Time: O(n)  Space: O(min(n, alphabet))
```

---

### Problem 3 — Minimum Size Subarray Sum (LC #209) ⭐⭐ Medium

```java
public int minSubArrayLen(int target, int[] nums) {
    int left = 0, sum = 0, minLen = Integer.MAX_VALUE;
    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];
        while (sum >= target) {
            minLen = Math.min(minLen, right - left + 1);
            sum -= nums[left++];
        }
    }
    return minLen == Integer.MAX_VALUE ? 0 : minLen;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 4 — Longest Repeating Character Replacement (LC #424) ⭐⭐ Medium

**Key insight:** A window of length `L` is valid if `L - (count of most frequent char) ≤ k`. This means we only need to replace the minority characters. If the window becomes invalid, we slide left.

```java
public int characterReplacement(String s, int k) {
    int[] count = new int[26];
    int left = 0, maxCount = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        count[s.charAt(right) - 'A']++;
        maxCount = Math.max(maxCount, count[s.charAt(right) - 'A']);
        // Window size - most frequent char count > k → invalid, shrink
        if ((right - left + 1) - maxCount > k) {
            count[s.charAt(left) - 'A']--;
            left++;
        }
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Time: O(n)  Space: O(26) = O(1)
```

---

### Problem 5 — Minimum Window Substring (LC #76) ⭐⭐⭐ Hard

**Key idea:** Use two frequency maps. Track `formed` — how many unique chars of `t` have been satisfied in the current window. Expand until all chars are covered, then shrink to minimize.

```java
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);

    int left = 0, formed = 0, required = need.size();
    Map<Character, Integer> window = new HashMap<>();
    int[] ans = {-1, 0, 0}; // [window_length, left, right]

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.merge(c, 1, Integer::sum);
        if (need.containsKey(c) && window.get(c).equals(need.get(c))) formed++;

        while (formed == required) {
            if (ans[0] == -1 || right - left + 1 < ans[0]) {
                ans[0] = right - left + 1;
                ans[1] = left; ans[2] = right;
            }
            char lc = s.charAt(left++);
            window.merge(lc, -1, Integer::sum);
            if (need.containsKey(lc) && window.get(lc) < need.get(lc)) formed--;
        }
    }
    return ans[0] == -1 ? "" : s.substring(ans[1], ans[2] + 1);
}
// Time: O(|s| + |t|)  Space: O(|s| + |t|)
```

---

### Sliding Window — LeetCode Problem List

| # | Problem | Difficulty | Type |
|---|---------|-----------|------|
| 643 | Max Average Subarray I | Easy | Fixed |
| 1343 | Num of Sub-arrays of Size K | Easy | Fixed |
| 1456 | Max Vowels in Substring | Easy | Fixed |
| 3 | Longest Substring Without Repeating | Medium | Variable |
| 209 | Minimum Size Subarray Sum | Medium | Variable |
| 424 | Longest Repeating Char Replacement | Medium | Variable |
| 567 | Permutation in String | Medium | Fixed |
| 438 | Find All Anagrams in a String | Medium | Fixed |
| 1004 | Max Consecutive Ones III | Medium | Variable |
| 904 | Fruit Into Baskets | Medium | Variable (at most 2 types) |
| 76 | Minimum Window Substring | Hard | Variable |
| 992 | Subarrays with K Different Integers | Hard | Variable |

---

## 3. Kadane's Algorithm

### What is it?

Kadane's algorithm finds the **maximum sum subarray** in O(n) time. It's a classic DP-style greedy approach that makes one pass through the array.

### Core Insight

At each position, you have two choices:
1. **Extend** the previous subarray by adding the current element
2. **Start fresh** from the current element (if the previous running sum was negative)

Formally: `maxEndingHere = max(current, maxEndingHere + current)`

If the running sum ever becomes negative, it is a liability for any future subarray — so we "reset" and start fresh from the current element.

### Why negative running sums hurt

If your current window sum is -5 and you add any element `x`, your new sum is `x - 5`, which is always worse than just taking `x` alone. So you abandon the previous window.

### Walkthrough

```
Array: [-2,  1, -3,  4, -1,  2,  1, -5,  4]

Step 1: cur = -2           maxEndHere = -2    globalMax = -2
Step 2: cur =  1           maxEndHere = max(1, -2+1)= 1   globalMax = 1
Step 3: cur = -3           maxEndHere = max(-3,1-3)= -2   globalMax = 1
Step 4: cur =  4           maxEndHere = max(4,-2+4)= 4    globalMax = 4
Step 5: cur = -1           maxEndHere = max(-1,4-1)= 3    globalMax = 4
Step 6: cur =  2           maxEndHere = max(2,3+2) = 5    globalMax = 5
Step 7: cur =  1           maxEndHere = max(1,5+1) = 6    globalMax = 6  ✅
Step 8: cur = -5           maxEndHere = max(-5,6-5)= 1    globalMax = 6
Step 9: cur =  4           maxEndHere = max(4,1+4) = 5    globalMax = 6

Answer: 6   (subarray [4,-1,2,1])
```

### Templates

```java
// Basic Kadane's
int maxSum(int[] nums) {
    int currentMax = nums[0];
    int globalMax  = nums[0];
    for (int i = 1; i < nums.length; i++) {
        currentMax = Math.max(nums[i], currentMax + nums[i]);
        globalMax  = Math.max(globalMax, currentMax);
    }
    return globalMax;
}

// Track indices (to return the actual subarray)
int[] kadaneWithIndices(int[] nums) {
    int currentMax = nums[0], globalMax = nums[0];
    int start = 0, end = 0, tempStart = 0;
    for (int i = 1; i < nums.length; i++) {
        if (nums[i] > currentMax + nums[i]) {
            currentMax = nums[i];
            tempStart  = i;      // potential new start
        } else {
            currentMax += nums[i];
        }
        if (currentMax > globalMax) {
            globalMax = currentMax;
            start = tempStart;
            end   = i;
        }
    }
    return new int[]{start, end, globalMax};
}
```

---

### Problem 1 — Maximum Subarray (LC #53) ⭐ Easy

```java
public int maxSubArray(int[] nums) {
    int curr = nums[0], best = nums[0];
    for (int i = 1; i < nums.length; i++) {
        curr = Math.max(nums[i], curr + nums[i]);
        best = Math.max(best, curr);
    }
    return best;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 2 — Maximum Product Subarray (LC #152) ⭐⭐ Medium

**Twist:** Negative × negative = positive, so a large negative can become the best option after another negative. Track both `max` AND `min` at each step, because the current minimum (most negative) could become the future maximum after multiplication.

```java
public int maxProduct(int[] nums) {
    int maxProd = nums[0], minProd = nums[0], result = nums[0];
    for (int i = 1; i < nums.length; i++) {
        int n = nums[i];
        int tempMax = Math.max(n, Math.max(maxProd * n, minProd * n));
        int tempMin = Math.min(n, Math.min(maxProd * n, minProd * n));
        maxProd = tempMax;
        minProd = tempMin;
        result = Math.max(result, maxProd);
    }
    return result;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 3 — Maximum Sum Circular Subarray (LC #918) ⭐⭐ Medium

**Key insight:** The answer is either:
- A **non-wrapping** subarray → standard Kadane's gives max sum, OR
- A **wrapping** subarray → this equals `total_sum - (min non-wrapping subarray sum)`, because wrapping around means we exclude some middle portion

```java
public int maxSubarraySumCircular(int[] nums) {
    int totalSum = 0, maxSum = nums[0], minSum = nums[0];
    int currMax = nums[0], currMin = nums[0];
    for (int i = 1; i < nums.length; i++) {
        currMax = Math.max(nums[i], currMax + nums[i]);
        currMin = Math.min(nums[i], currMin + nums[i]);
        maxSum  = Math.max(maxSum, currMax);
        minSum  = Math.min(minSum, currMin);
        totalSum += nums[i];
    }
    totalSum += nums[0];
    // If all negative, maxSum is the answer (totalSum - minSum would be 0)
    return maxSum > 0 ? Math.max(maxSum, totalSum - minSum) : maxSum;
}
// Time: O(n)  Space: O(1)
```

---

### Kadane's — LeetCode Problem List

| # | Problem | Difficulty | Twist |
|---|---------|-----------|-------|
| 53 | Maximum Subarray | Easy | Classic Kadane's |
| 121 | Best Time to Buy and Sell Stock | Easy | Max profit = max subarray of differences |
| 152 | Maximum Product Subarray | Medium | Track min and max |
| 918 | Maximum Sum Circular Subarray | Medium | total - min subarray |
| 1749 | Max Absolute Sum of Any Subarray | Medium | Max subarray + Min subarray |
| 2321 | Maximum Score of Spliced Array | Hard | Kadane on difference array |

---

## 4. Binary Search

### What is it?

Binary search finds a target in a **sorted** array by repeatedly halving the search space. Each step eliminates half the remaining candidates. Result: O(log n) instead of O(n) linear search.

### How it works

```
Target = 7  in  [1, 3, 5, 7, 9, 11, 13]
                  0  1  2  3  4   5   6

Round 1: left=0, right=6  mid=3   nums[3]=7  == target ✅  Done!

Target = 9:
Round 1: left=0, right=6  mid=3   nums[3]=7  < 9  → left=4
Round 2: left=4, right=6  mid=5   nums[5]=11 > 9  → right=4
Round 3: left=4, right=4  mid=4   nums[4]=9  == 9 ✅
```

### The tricky part — boundary conditions

There are three common variants of binary search, and choosing the wrong boundary causes off-by-one bugs:

```java
// ── Standard: find exact target ─────────────────────────
int left = 0, right = arr.length - 1;
while (left <= right) {                    // ← inclusive: <= because right = length-1
    int mid = left + (right - left) / 2;  // avoids integer overflow
    if (arr[mid] == target)      return mid;
    else if (arr[mid] < target)  left  = mid + 1;
    else                         right = mid - 1;
}
return -1;

// ── Lower bound: first position where arr[mid] >= target ─
int left = 0, right = arr.length;          // ← right = length (not length-1!)
while (left < right) {                     // ← strict < because right = length
    int mid = left + (right - left) / 2;
    if (arr[mid] < target) left  = mid + 1;
    else                   right = mid;    // mid could be the answer, don't skip it
}
return left;  // insertion position

// ── Upper bound: last position of target ─────────────────
int left = 0, right = arr.length;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] <= target) left  = mid + 1;
    else                    right = mid;
}
return left - 1;  // last occurrence of target
```

### Binary search on the ANSWER

A powerful advanced pattern: instead of searching an array, binary search on the **range of possible answers**. This converts "minimize X" problems into "is X feasible?" problems.

```
"Find the minimum speed so that you can finish eating bananas in h hours."
→ Binary search speed in range [1, max_pile].
→ For each candidate speed, check if it's feasible (takes ≤ h hours).
→ Find the smallest feasible speed.
```

---

### Problem 1 — Binary Search (LC #704) ⭐ Easy

```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;
        if (nums[mid] < target)  left  = mid + 1;
        else                     right = mid - 1;
    }
    return -1;
}
// Time: O(log n)  Space: O(1)
```

---

### Problem 2 — Search in Rotated Sorted Array (LC #33) ⭐⭐ Medium

**Key insight:** Even after rotation, one half is always fully sorted. Use the sorted half to decide whether the target is in it or in the other half.

```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;
        // Left half is sorted
        if (nums[left] <= nums[mid]) {
            if (nums[left] <= target && target < nums[mid])
                right = mid - 1;
            else
                left = mid + 1;
        } else { // Right half is sorted
            if (nums[mid] < target && target <= nums[right])
                left = mid + 1;
            else
                right = mid - 1;
        }
    }
    return -1;
}
// Time: O(log n)  Space: O(1)
```

---

### Problem 3 — Koko Eating Bananas (LC #875) ⭐⭐ Medium (Binary search on answer)

**Key idea:** Binary search on speed `k` (1 to max pile). For each candidate speed, simulate eating and check if it finishes within `h` hours. Find the minimum valid speed.

```java
public int minEatingSpeed(int[] piles, int h) {
    int left = 1, right = Arrays.stream(piles).max().getAsInt();
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canFinish(piles, mid, h)) right = mid;  // valid, try slower
        else                          left  = mid + 1; // too slow, go faster
    }
    return left;
}

private boolean canFinish(int[] piles, int speed, int h) {
    int hours = 0;
    for (int p : piles) hours += (p + speed - 1) / speed; // ceil division
    return hours <= h;
}
// Time: O(n log m)  Space: O(1)
```

---

### Problem 4 — Find Minimum in Rotated Sorted Array (LC #153) ⭐⭐ Medium

**Key insight:** The minimum is the only element smaller than its predecessor. If `nums[mid] > nums[right]`, the minimum is in the right half.

```java
public int findMin(int[] nums) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] > nums[right]) left  = mid + 1; // min is in right half
        else                         right = mid;      // mid could be the min
    }
    return nums[left];
}
// Time: O(log n)  Space: O(1)
```

---

### Binary Search — LeetCode Problem List

| # | Problem | Difficulty | Pattern |
|---|---------|-----------|---------|
| 704 | Binary Search | Easy | Classic |
| 35 | Search Insert Position | Easy | Lower bound |
| 278 | First Bad Version | Easy | Binary search on range |
| 69 | Sqrt(x) | Easy | Binary search on answer |
| 153 | Find Min in Rotated Sorted Array | Medium | Rotated array |
| 33 | Search in Rotated Sorted Array | Medium | Rotated array |
| 34 | Find First and Last Position | Medium | Lower/upper bound |
| 875 | Koko Eating Bananas | Medium | Search on answer |
| 1011 | Capacity to Ship Packages | Medium | Search on answer |
| 74 | Search a 2D Matrix | Medium | Flatten to 1D BS |
| 162 | Find Peak Element | Medium | Binary search on property |
| 4 | Median of Two Sorted Arrays | Hard | Binary search partition |

---

## 5. Fast & Slow Pointers (Floyd's Cycle)

### What is it?

Two pointers move through a linked list (or sequence) at **different speeds** — the fast pointer moves 2 steps per iteration, the slow pointer moves 1. If there is a cycle, the fast pointer will eventually "lap" the slow pointer, and they will meet. If there is no cycle, fast reaches null.

### Why they meet inside a cycle

Imagine the slow pointer is at distance `d` from the cycle start when the fast pointer enters the cycle. The fast pointer is `C - d` steps behind (where C is the cycle length). Since fast gains 1 step per iteration, they meet in `C - d` more steps. Mathematical proof ensures they always meet within the cycle.

### Finding the cycle start (Floyd's Phase 2)

After meeting inside the cycle: reset slow to the head. Move both pointers one step at a time. They will meet exactly at the cycle start. Why? The distance from head to cycle-start equals the distance from the meeting point to cycle-start (a provable property of the math).

### How it works

```
Linked list with cycle:
1 → 2 → 3 → 4 → 5
            ↑       |
            └───────┘ (5 points back to 3)

Start: slow=1, fast=1
Step1: slow=2, fast=3
Step2: slow=3, fast=5
Step3: slow=4, fast=4  ← MEET! Cycle detected ✅

Finding cycle start:
Reset slow to head. Move both one step at a time.
slow=1, fast=4
slow=2, fast=5
slow=3, fast=3  ← Cycle start! ✅
```

---

### Problem 1 — Linked List Cycle (LC #141) ⭐ Easy

```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 2 — Find Duplicate Number (LC #287) ⭐⭐ Medium

**Brilliant insight:** Treat the array `nums` as a function where index `i` maps to value `nums[i]`. Since values are in range `[1, n]` and indices are `[0, n]`, this defines a linked list where `nums[i]` is the next pointer. A duplicate value creates a cycle (two indices point to the same next node).

```java
public int findDuplicate(int[] nums) {
    int slow = nums[0], fast = nums[0];
    // Phase 1: detect cycle
    do {
        slow = nums[slow];
        fast = nums[nums[fast]];
    } while (slow != fast);
    // Phase 2: find cycle entry = duplicate number
    slow = nums[0];
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 3 — Middle of Linked List (LC #876) ⭐ Easy

**Why it works:** When fast (moving 2 steps) reaches the end, slow (moving 1 step) has covered exactly half the distance — the middle.

```java
public ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow; // when fast reaches end, slow is at middle
}
// Time: O(n)  Space: O(1)
```

---

### Fast/Slow — LeetCode Problem List

| # | Problem | Difficulty |
|---|---------|-----------|
| 141 | Linked List Cycle | Easy |
| 876 | Middle of Linked List | Easy |
| 202 | Happy Number | Easy |
| 142 | Linked List Cycle II (find start) | Medium |
| 287 | Find the Duplicate Number | Medium |
| 457 | Circular Array Loop | Medium |

---

## 6. Prefix Sum

### What is it?

Precompute cumulative sums so any range sum query `[i, j]` is answered in O(1) instead of O(n). Build once in O(n), query many times in O(1).

```
prefix[i] = arr[0] + arr[1] + ... + arr[i-1]
rangeSum(i, j) = prefix[j+1] - prefix[i]
```

The `prefix[0] = 0` sentinel makes the indexing clean: the sum of the whole array is `prefix[n] - prefix[0]`.

### Walkthrough

```
Array:   [1,  2,  3,  4,  5]
Prefix:  [0,  1,  3,  6, 10, 15]
          ↑ (prefix[0] = 0, makes indexing clean)

Range sum [1..3] (indices 1 to 3, values 2+3+4):
= prefix[4] - prefix[1]
= 10 - 1
= 9 ✅
```

### Advanced: Prefix Sum + HashMap

For problems like "count subarrays with sum = k", use a HashMap to store how many times each prefix sum has appeared. If `prefix[j] - prefix[i] == k`, then `prefix[i] = prefix[j] - k`. So for each new prefix sum, check how many previous prefix sums equal `current - k`.

### Template

```java
// Build prefix sum
int[] prefix = new int[nums.length + 1];
for (int i = 0; i < nums.length; i++)
    prefix[i + 1] = prefix[i] + nums[i];

// Query range [l, r] (0-indexed, inclusive)
int rangeSum = prefix[r + 1] - prefix[l];
```

---

### Problem 1 — Range Sum Query (LC #303) ⭐ Easy

```java
class NumArray {
    int[] prefix;
    NumArray(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++)
            prefix[i + 1] = prefix[i] + nums[i];
    }
    public int sumRange(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
}
// Build: O(n)  Query: O(1)
```

---

### Problem 2 — Subarray Sum Equals K (LC #560) ⭐⭐ Medium

**Key trick:** Use a HashMap to count prefix sums seen so far. For each new prefix sum `p`, we want to find how many past prefix sums equal `p - k` — those form a subarray of sum `k` with the current position.

```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1); // empty prefix: sum of 0 seen once
    int prefixSum = 0, count = 0;
    for (int num : nums) {
        prefixSum += num;
        count += prefixCount.getOrDefault(prefixSum - k, 0);
        prefixCount.merge(prefixSum, 1, Integer::sum);
    }
    return count;
}
// Time: O(n)  Space: O(n)
```

---

### Prefix Sum — LeetCode Problem List

| # | Problem | Difficulty |
|---|---------|-----------|
| 303 | Range Sum Query - Immutable | Easy |
| 1480 | Running Sum of 1d Array | Easy |
| 724 | Find Pivot Index | Easy |
| 560 | Subarray Sum Equals K | Medium |
| 525 | Contiguous Array | Medium |
| 974 | Subarray Sums Divisible by K | Medium |
| 238 | Product of Array Except Self | Medium |
| 304 | Range Sum Query 2D | Medium |

---

## 7. Monotonic Stack

### What is it?

A stack that maintains elements in **monotonically increasing or decreasing order**. When a new element would break the monotonic property, we pop elements — and the pop event itself reveals the answer (e.g., "this popped element has its next greater element found").

### When to use?

- **Next Greater Element (NGE):** For each element, what is the first larger element to its right?
- **Previous Smaller Element (PSE):** For each element, what is the first smaller element to its left?
- **Largest Rectangle in Histogram**
- **Trapping Rain Water**

### How it works — Next Greater Element

Maintain a **decreasing** stack (each element ≤ the one below it). When a new element is greater than the stack top, the stack top has found its NGE.

```
Array:  [2, 1, 5, 6, 2, 3]

Maintain a DECREASING stack (indices):

i=0: stack=[]    push 0.  stack=[0]  (val=2)
i=1: val=1 < stack top 2 → push. stack=[0,1]  (vals 2,1)
i=2: val=5 > 1 → POP 1: NGE[1]=5; val=5 > 2 → POP 0: NGE[0]=5; push 2. stack=[2]
i=3: val=6 > 5 → POP 2: NGE[2]=6; push 3. stack=[3]
i=4: val=2 < 6 → push 4. stack=[3,4]
i=5: val=3 > 2 → POP 4: NGE[4]=3; push 5. stack=[3,5]
End: remaining in stack → no NGE → -1

NGE: [5, 5, 6, -1, 3, -1]
```

### Why O(n)?

Even though there's a while loop inside the for loop, each element is pushed once and popped at most once. Total operations: 2n = O(n).

### Template

```java
// Next Greater Element
int[] nge = new int[nums.length];
Deque<Integer> stack = new ArrayDeque<>(); // stores indices
for (int i = 0; i < nums.length; i++) {
    while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
        nge[stack.pop()] = nums[i]; // nums[i] is the NGE for popped element
    }
    stack.push(i);
}
while (!stack.isEmpty()) nge[stack.pop()] = -1; // no NGE found
```

---

### Problem 1 — Daily Temperatures (LC #739) ⭐⭐ Medium

**Find how many days until a warmer temperature.** Same as NGE but we store the index difference.

```java
public int[] dailyTemperatures(int[] temperatures) {
    int[] result = new int[temperatures.length];
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < temperatures.length; i++) {
        while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
            int idx = stack.pop();
            result[idx] = i - idx; // days until warmer temperature
        }
        stack.push(i);
    }
    return result;
}
// Time: O(n)  Space: O(n)
```

---

### Problem 2 — Largest Rectangle in Histogram (LC #84) ⭐⭐⭐ Hard

**Key insight:** For each bar as the "height" of a rectangle, find the farthest left and right it can extend while remaining the shortest bar. Use an **increasing** stack — when we pop, the popped bar is shorter than the current bar, so we calculate the max rectangle with the popped bar's height.

```java
public int largestRectangleArea(int[] heights) {
    Deque<Integer> stack = new ArrayDeque<>();
    int maxArea = 0;
    for (int i = 0; i <= heights.length; i++) {
        int h = (i == heights.length) ? 0 : heights[i]; // sentinel 0 flushes stack
        while (!stack.isEmpty() && h < heights[stack.peek()]) {
            int height = heights[stack.pop()];
            // Width: from current position back to the new stack top
            int width  = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, height * width);
        }
        stack.push(i);
    }
    return maxArea;
}
// Time: O(n)  Space: O(n)
```

---

### Monotonic Stack — LeetCode Problem List

| # | Problem | Difficulty |
|---|---------|-----------|
| 496 | Next Greater Element I | Easy |
| 739 | Daily Temperatures | Medium |
| 503 | Next Greater Element II | Medium |
| 901 | Online Stock Span | Medium |
| 907 | Sum of Subarray Minimums | Medium |
| 42 | Trapping Rain Water | Hard |
| 84 | Largest Rectangle in Histogram | Hard |
| 85 | Maximal Rectangle | Hard |

---

## 8. BFS (Breadth-First Search)

### What is it?

BFS explores a graph or tree **level by level** using a queue (FIFO). All nodes at distance 1 are visited before nodes at distance 2, and so on. This makes BFS **optimal for finding shortest paths** in unweighted graphs.

### Why BFS finds shortest paths

Because BFS visits nodes in order of increasing distance from the source, the first time a node is reached is guaranteed to be via the shortest path. Once a node is marked visited, any future path to it would be longer.

### Key properties

- Uses a **queue** (not a stack — that would be DFS)
- Must track **visited** nodes to avoid revisiting
- For grids, expand in 4 (or 8) directions
- Process level-by-level using a size variable to know when one level ends and the next begins

### Templates

```java
// ── Graph BFS (shortest path) ────────────────────────────
int bfs(int[][] graph, int start, int target) {
    Queue<Integer> queue = new LinkedList<>();
    Set<Integer>   visited = new HashSet<>();
    queue.offer(start);
    visited.add(start);
    int level = 0;
    while (!queue.isEmpty()) {
        int size = queue.size(); // process all nodes at current level
        for (int i = 0; i < size; i++) {
            int node = queue.poll();
            if (node == target) return level;
            for (int neighbor : graph[node]) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
        level++;
    }
    return -1;
}

// ── Grid BFS (4-directional) ─────────────────────────────
int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
Queue<int[]> queue = new LinkedList<>();
queue.offer(new int[]{startRow, startCol});
visited[startRow][startCol] = true;
int steps = 0;
while (!queue.isEmpty()) {
    int size = queue.size();
    for (int i = 0; i < size; i++) {
        int[] cell = queue.poll();
        int r = cell[0], c = cell[1];
        if (r == targetR && c == targetC) return steps;
        for (int[] d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                    && !visited[nr][nc] && grid[nr][nc] != '#') {
                visited[nr][nc] = true;
                queue.offer(new int[]{nr, nc});
            }
        }
    }
    steps++;
}
```

---

### Problem 1 — Number of Islands (LC #200) ⭐⭐ Medium

**Strategy:** Every time we find an unvisited `'1'`, we start a BFS to flood-fill the entire island (marking all connected `'1'`s as visited). Count how many BFS calls we make.

```java
public int numIslands(char[][] grid) {
    int count = 0;
    int rows = grid.length, cols = grid[0].length;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (grid[r][c] == '1') {
                count++;
                Queue<int[]> q = new LinkedList<>();
                q.offer(new int[]{r, c});
                grid[r][c] = '0'; // mark visited by changing to '0'
                while (!q.isEmpty()) {
                    int[] cur = q.poll();
                    for (int[] d : dirs) {
                        int nr = cur[0]+d[0], nc = cur[1]+d[1];
                        if (nr>=0 && nr<rows && nc>=0 && nc<cols && grid[nr][nc]=='1') {
                            grid[nr][nc] = '0';
                            q.offer(new int[]{nr, nc});
                        }
                    }
                }
            }
        }
    }
    return count;
}
// Time: O(m*n)  Space: O(min(m,n))
```

---

### BFS — LeetCode Problem List

| # | Problem | Difficulty |
|---|---------|-----------|
| 102 | Binary Tree Level Order Traversal | Medium |
| 200 | Number of Islands | Medium |
| 542 | 01 Matrix | Medium |
| 994 | Rotting Oranges | Medium |
| 127 | Word Ladder | Hard |
| 126 | Word Ladder II | Hard |
| 1926 | Nearest Exit from Entrance | Medium |
| 1091 | Shortest Path in Binary Matrix | Medium |

---

## 9. DFS (Depth-First Search)

### What is it?

DFS explores as **deep as possible** before backtracking. It follows one path all the way to the end before trying another. Implemented recursively (using the call stack implicitly) or iteratively with an explicit stack.

### DFS vs BFS

| | DFS | BFS |
|--|-----|-----|
| Data structure | Stack (or recursion) | Queue |
| Finds | Any path | Shortest path |
| Memory | O(depth) | O(width) |
| Use for | Backtracking, all paths, tree traversals | Shortest path, level order |

### Traversal orders (for trees)

- **Pre-order:** Process node → recurse left → recurse right (used to copy or serialize a tree)
- **In-order:** Recurse left → process node → recurse right (gives sorted order for BSTs)
- **Post-order:** Recurse left → recurse right → process node (used to delete or compute bottom-up values)

### Backtracking pattern

Backtracking is DFS that explores all possibilities by trying each option, recursing, and **undoing** the choice before trying the next. The undo step is the "backtrack."

### Templates

```java
// ── Recursive DFS (tree) ─────────────────────────────────
void dfs(TreeNode node) {
    if (node == null) return;
    // preorder: process node here
    dfs(node.left);
    // inorder: process node here
    dfs(node.right);
    // postorder: process node here
}

// ── DFS with backtracking (permutations/subsets) ─────────
void backtrack(List<Integer> current, int[] nums, boolean[] used) {
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current)); // found valid solution
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        current.add(nums[i]);
        backtrack(current, nums, used);    // explore
        current.remove(current.size() - 1); // ← backtrack (undo)
        used[i] = false;
    }
}
```

---

### Problem 1 — Number of Connected Components (LC #323) ⭐⭐ Medium

```java
public int countComponents(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        graph.get(e[1]).add(e[0]);
    }
    boolean[] visited = new boolean[n];
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (!visited[i]) { dfs(graph, visited, i); count++; }
    }
    return count;
}

void dfs(List<List<Integer>> graph, boolean[] visited, int node) {
    visited[node] = true;
    for (int neighbor : graph.get(node))
        if (!visited[neighbor]) dfs(graph, visited, neighbor);
}
// Time: O(V+E)  Space: O(V+E)
```

---

### Problem 2 — Subsets (LC #78) ⭐⭐ Medium (DFS + Backtracking)

**Key idea:** At each index, we choose to include the element or not, creating a binary decision tree. All leaves (end of array) are valid subsets. Backtracking lets us explore all branches.

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(result, new ArrayList<>(), nums, 0);
    return result;
}

void backtrack(List<List<Integer>> result, List<Integer> current, int[] nums, int start) {
    result.add(new ArrayList<>(current)); // every state is a valid subset
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                     // include nums[i]
        backtrack(result, current, nums, i + 1);  // explore
        current.remove(current.size() - 1);       // backtrack (exclude nums[i])
    }
}
// Time: O(2^n)  Space: O(n)
```

---

### DFS — LeetCode Problem List

| # | Problem | Difficulty |
|---|---------|-----------|
| 104 | Max Depth of Binary Tree | Easy |
| 543 | Diameter of Binary Tree | Easy |
| 78 | Subsets | Medium |
| 46 | Permutations | Medium |
| 79 | Word Search | Medium |
| 39 | Combination Sum | Medium |
| 695 | Max Area of Island | Medium |
| 417 | Pacific Atlantic Water Flow | Medium |
| 51 | N-Queens | Hard |
| 37 | Sudoku Solver | Hard |

---

## 10. Dynamic Programming

### What is it?

DP solves problems by breaking them into **overlapping subproblems** and caching results to avoid recomputation. It applies when:

1. **Optimal substructure:** The optimal solution to the problem contains optimal solutions to subproblems.
2. **Overlapping subproblems:** The same subproblem is solved multiple times if we use naive recursion.

### Two approaches

**Top-down (memoization):** Write a recursive solution, add a cache to store results of subproblems already solved. Natural to think about, easier to write.

**Bottom-up (tabulation):** Fill a DP table iteratively, starting from the smallest subproblems and building up to the answer. No recursion overhead, often more efficient.

### Identifying DP problems

- "How many ways...?" (count)
- "What is the maximum/minimum...?" (optimize)
- "Can you reach...?" (decision/feasibility)
- Subproblems **overlap** (the same subproblem recurs in a tree of recursive calls)
- Optimal substructure holds (the problem can be assembled from optimal sub-solutions)

### DP design steps

1. Define the state: What does `dp[i]` (or `dp[i][j]`) represent?
2. Write the transition: How does `dp[i]` relate to smaller subproblems?
3. Set base cases.
4. Determine evaluation order (usually bottom-up, left-to-right).

---

### Problem 1 — Climbing Stairs (LC #70) ⭐ Easy

**State:** `dp[i]` = number of ways to reach step `i`

**Transition:** `dp[i] = dp[i-1] + dp[i-2]` — you can arrive from one step below (1-step jump) or two steps below (2-step jump).

**This is exactly Fibonacci.** Optimize space by only keeping the last two values.

```java
public int climbStairs(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 2;
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 2 — Coin Change (LC #322) ⭐⭐ Medium (Unbounded Knapsack)

**State:** `dp[i]` = minimum number of coins needed to make amount `i`

**Transition:** For each coin, `dp[i] = min(dp[i], dp[i - coin] + 1)` — try each coin and take the option that uses the fewest coins.

**Key insight:** Start `dp` as `amount + 1` (a value larger than any valid answer, acting as infinity). Only reachable amounts get meaningful values.

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // "infinity"
    dp[0] = 0;                   // 0 coins needed to make 0
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i)
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
// Time: O(amount * coins)  Space: O(amount)
```

---

### Problem 3 — Longest Common Subsequence (LC #1143) ⭐⭐ Medium (2D DP)

**State:** `dp[i][j]` = length of LCS of `text1[0..i-1]` and `text2[0..j-1]`

**Transition:**
- If characters match: `dp[i][j] = dp[i-1][j-1] + 1`
- If not: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])` (best without one or the other character)

```java
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i-1) == text2.charAt(j-1))
                dp[i][j] = dp[i-1][j-1] + 1;
            else
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
        }
    }
    return dp[m][n];
}
// Time: O(m*n)  Space: O(m*n)
```

---

### DP — LeetCode Problem List

| # | Problem | Difficulty | Pattern |
|---|---------|-----------|---------|
| 70 | Climbing Stairs | Easy | 1D DP |
| 198 | House Robber | Easy | 1D DP |
| 746 | Min Cost Climbing Stairs | Easy | 1D DP |
| 322 | Coin Change | Medium | Unbounded knapsack |
| 300 | Longest Increasing Subsequence | Medium | LIS |
| 1143 | Longest Common Subsequence | Medium | 2D DP |
| 416 | Partition Equal Subset Sum | Medium | 0/1 knapsack |
| 72 | Edit Distance | Medium | 2D DP |
| 312 | Burst Balloons | Hard | Interval DP |
| 124 | Binary Tree Max Path Sum | Hard | Tree DP |

---

## 11. Merge Intervals

### What is it?

Sort intervals by start time, then merge overlapping ones. Two intervals `[a, b]` and `[c, d]` overlap if `c <= b` (the new interval starts before the current one ends). When they overlap, merge by extending the end to `max(b, d)`.

### Why sort by start?

Sorting by start time ensures that when we process intervals left to right, we only need to compare each new interval against the last merged interval — earlier intervals can't possibly overlap with later ones.

### Common variants

- **Merge overlapping:** Combine all overlapping intervals into one
- **Insert interval:** Add a new interval into a sorted, non-overlapping list
- **Find non-overlapping:** Greedily select maximum non-overlapping intervals (sort by end time)
- **Min rooms for meetings:** Count max overlapping intervals at any point

### Template

```java
public int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]); // sort by start time
    List<int[]> result = new ArrayList<>();
    int[] current = intervals[0];
    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] <= current[1]) {        // overlap: new start ≤ current end
            current[1] = Math.max(current[1], intervals[i][1]); // extend end
        } else {
            result.add(current);
            current = intervals[i];
        }
    }
    result.add(current); // don't forget the last interval
    return result.toArray(new int[0][]);
}
// Time: O(n log n)  Space: O(n)
```

---

### LeetCode Problems

| # | Problem | Difficulty |
|---|---------|-----------|
| 56 | Merge Intervals | Medium |
| 57 | Insert Interval | Medium |
| 435 | Non-overlapping Intervals | Medium |
| 252 | Meeting Rooms | Easy |
| 253 | Meeting Rooms II | Medium |
| 1851 | Minimum Interval to Include Each Query | Hard |

---

## 12. Bit Manipulation

### What is it?

Operating directly on binary representations of integers using bitwise operators. Often achieves O(1) space solutions that would otherwise need hashmaps or sorting.

### Core operations

```java
n & 1          // check if n is odd (last bit is 1)
n >> 1         // divide by 2 (arithmetic right shift)
n << 1         // multiply by 2 (left shift)
n & (n-1)      // clear lowest set bit — each time you do this, one '1' bit disappears
               // useful to count set bits: loop until n == 0
n & (-n)       // isolate lowest set bit (only that bit remains)
n ^ n == 0     // XOR of identical numbers = 0
n ^ 0 == n     // XOR with 0 = n (identity)
a ^ b ^ a == b // XOR is commutative and associative — order doesn't matter
```

### XOR trick intuition

XOR is like addition without carrying. Two identical bits XOR to 0; different bits XOR to 1. So XOR-ing a number with itself cancels it out. This is why XOR-ing all numbers in an array where every number appears twice (except one) gives you the unique number — all pairs cancel.

---

### Problem 1 — Single Number (LC #136) ⭐ Easy

**Trick:** XOR all numbers. Every pair cancels to 0, leaving only the unique number.

```
[4, 1, 2, 1, 2]
4 ^ 1 ^ 2 ^ 1 ^ 2
= 4 ^ (1 ^ 1) ^ (2 ^ 2)
= 4 ^ 0 ^ 0
= 4 ✅
```

```java
public int singleNumber(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;
    return result;
}
// Time: O(n)  Space: O(1)
```

---

### Problem 2 — Count Bits (LC #338) ⭐ Easy

**Key insight:** `dp[i] = dp[i >> 1] + (i & 1)`. The number of set bits in `i` equals the number of set bits in `i/2`, plus 1 if `i` is odd (the last bit). Since we build `dp` in order, `dp[i/2]` is always already computed.

```java
public int[] countBits(int n) {
    int[] dp = new int[n + 1];
    for (int i = 1; i <= n; i++)
        dp[i] = dp[i >> 1] + (i & 1); // dp[i/2] + last bit
    return dp;
}
// Time: O(n)  Space: O(n)
```

---

### LeetCode Problems

| # | Problem | Difficulty |
|---|---------|-----------|
| 136 | Single Number | Easy |
| 338 | Counting Bits | Easy |
| 191 | Number of 1 Bits | Easy |
| 190 | Reverse Bits | Easy |
| 268 | Missing Number | Easy |
| 137 | Single Number II | Medium |
| 260 | Single Number III | Medium |

---

## Algorithm Selection Guide

```
Is the array sorted (or can it be)?
  Yes → Two Pointers or Binary Search
  No  → Consider sorting first

Do you need a subarray/substring satisfying a condition?
  Fixed length K → Sliding Window (fixed)
  Variable, shrink/expand → Sliding Window (variable)
  Maximum sum subarray → Kadane's Algorithm
  Range sum queries → Prefix Sum
  Count subarrays with property → Prefix Sum + HashMap

Is it a linked list problem?
  Cycle detection / middle / find duplicate → Fast & Slow Pointers

Is it a graph/tree problem?
  Shortest path / level order → BFS
  Explore all paths / backtrack → DFS

Need optimal substructure? Overlapping subproblems?
  → Dynamic Programming

Working with intervals?
  → Sort by start + Merge Intervals

Looking for next greater/smaller element?
  → Monotonic Stack

Binary (0/1) data or XOR trick?
  → Bit Manipulation
```

---

## Time Complexity Summary

| Algorithm | Time | Space | Notes |
|-----------|------|-------|-------|
| Two Pointers | O(n) | O(1) | Requires sorted input for most uses |
| Sliding Window | O(n) | O(k) | k = window size or alphabet size |
| Kadane's | O(n) | O(1) | Single pass |
| Binary Search | O(log n) | O(1) | Requires sorted or monotonic condition |
| Fast & Slow Pointers | O(n) | O(1) | Linked list / cycle detection |
| Prefix Sum (build) | O(n) | O(n) | One-time preprocessing |
| Prefix Sum (query) | O(1) | — | After preprocessing |
| Monotonic Stack | O(n) | O(n) | Each element pushed/popped once |
| BFS | O(V+E) | O(V) | V = vertices, E = edges |
| DFS | O(V+E) | O(V) | Depth = stack space |
| DP (1D) | O(n) | O(n) | Often optimizable to O(1) space |
| DP (2D) | O(m×n) | O(m×n) | Sometimes optimizable to O(n) space |
| Merge Intervals | O(n log n) | O(n) | Dominated by sorting |
| Bit Manipulation | O(n) | O(1) | Per-element operations are O(1) |

---

*Practice consistently. Start with Easy problems to master the pattern, then progress to Medium. Each algorithm here covers dozens of LeetCode problems — learning the pattern is more valuable than memorizing solutions.* 🚀
