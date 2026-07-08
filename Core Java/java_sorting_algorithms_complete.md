# 🧠 Complete Sorting Algorithms Guide (Java)
### From Brute Force → Optimized | With Visuals, Examples & Practice Problems

---

## 🧠 THE GOLDEN RULE BEFORE ANYTHING

> **Before you memorize code — understand the IDEA.**
> Every sorting algorithm answers: *"How do I decide where each element belongs?"*

| Algorithm | Core Idea in One Line | Remember As |
|---|---|---|
| **Bubble Sort** | Repeatedly swap adjacent elements if out of order | "Heavy bubbles sink to the right" |
| **Selection Sort** | Find the minimum, place it at the front | "Select the smallest, put it in its spot" |
| **Insertion Sort** | Pick one element, slide it left until it fits | "Sort like holding playing cards in hand" |
| **Merge Sort** | Split into halves, sort each, merge back | "Divide army into squads, win each battle, reunite" |
| **Quick Sort** | Pick a pivot, put smaller left, larger right | "Choose a leader, everyone smaller goes left" |
| **Heap Sort** | Build a max-heap, extract max repeatedly | "Build a priority queue, drain it" |
| **Counting Sort** | Count frequency of each element, rebuild | "Count how many of each, fill slots" |
| **Radix Sort** | Sort digit by digit, from least to most significant | "Sort by last digit first, then tens, then hundreds" |
| **Shell Sort** | Insertion sort with decreasing gap sizes | "Insertion sort on steroids — shrinking jumps" |
| **Tim Sort** | Merge sort + Insertion sort hybrid | "Smart divide + hand-of-cards for small pieces" |
| **Bucket Sort** | Scatter elements into buckets, sort each bucket | "Sort mail into mailboxes, sort each box" |

---

# 🫧 1. BUBBLE SORT

## 🧠 Mental Picture
> Imagine bubbles in water. Heavy bubbles (large numbers) SINK to the right/bottom.
> After each **pass**, the LARGEST unsorted element is in its correct position.

```
Pass 1 of [5, 3, 8, 4, 2]:
Step 1: Compare 5 & 3  → 5 > 3 → SWAP  → [3, 5, 8, 4, 2]
Step 2: Compare 5 & 8  → 5 < 8 → NO SWAP → [3, 5, 8, 4, 2]
Step 3: Compare 8 & 4  → 8 > 4 → SWAP  → [3, 5, 4, 8, 2]
Step 4: Compare 8 & 2  → 8 > 2 → SWAP  → [3, 5, 4, 2, 8] ← 8 is HOME ✅

After Pass 1: [3, 5, 4, 2, | 8]
After Pass 2: [3, 4, 2, | 5, 8]
After Pass 3: [2, 3, | 4, 5, 8]
After Pass 4: [2, | 3, 4, 5, 8] ← SORTED ✅
```

---

## 🐌 Brute Force

```java
public static void bubbleSortBrute(int[] arr) {
    int n = arr.length;
    
    // Do n passes
    for (int i = 0; i < n; i++) {
        // Compare all adjacent pairs every single time (wasteful!)
        for (int j = 0; j < n - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                // Swap
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}
// Problem: Even if sorted, still does all n² comparisons
```

## ⚡ Optimized

```java
public static void bubbleSortOptimized(int[] arr) {
    int n = arr.length;
    
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;                // 🔑 Key optimization
        
        // After i passes, last i elements are sorted
        // No need to check them again → n - i - 1
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        
        // 🔑 No swaps in this pass = ALREADY SORTED → Stop early!
        if (!swapped) break;
    }
}

// Example:
// Input: [1, 2, 3, 4, 5] → Already sorted
// Pass 1: No swaps → swapped = false → BREAK immediately!
// Time: O(n) for nearly-sorted arrays instead of O(n²)
```

## 📊 Complexity
| | Time | Why? |
|---|---|---|
| Best | O(n) | Already sorted → 1 pass, swapped=false → break |
| Average | O(n²) | ~n²/2 comparisons |
| Worst | O(n²) | Reverse sorted → max swaps |
| Space | O(1) | In-place, only temp variable |
| Stable | ✅ YES | Equal elements never swap |

---

# 🎯 2. SELECTION SORT

## 🧠 Mental Picture
> You have 5 unsorted cards on a table.
> **Round 1:** Scan ALL cards → find MINIMUM (say it's 2) → put it in position 0.
> **Round 2:** Scan remaining 4 cards → find MINIMUM → put it in position 1.
> Repeat until done.

```
[5, 3, 8, 4, 2]
 ^____________^  Scan → min = 2 (index 4) → Swap with index 0
[2, 3, 8, 4, 5]
 ✅ ^_________^  Scan → min = 3 (index 1) → Already there, swap with itself
[2, 3, 8, 4, 5]
 ✅ ✅ ^______^  Scan → min = 4 (index 3) → Swap with index 2
[2, 3, 4, 8, 5]
 ✅ ✅ ✅ ^___^  Scan → min = 5 (index 4) → Swap with index 3
[2, 3, 4, 5, 8] ← SORTED ✅
```

---

## 🐌 Brute Force

```java
public static void selectionSortBrute(int[] arr) {
    int n = arr.length;
    
    for (int i = 0; i < n - 1; i++) {
        // Find minimum in arr[i...n-1]
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) {
                minIdx = j;
            }
        }
        // Swap minimum with first unsorted position
        int temp = arr[minIdx];
        arr[minIdx] = arr[i];
        arr[i] = temp;
    }
}
// Key insight: At most n-1 swaps total (one per outer loop)
// Even if already sorted → still does n² comparisons (no early exit!)
```

## ⚡ Optimized (Find Min AND Max in one pass)

```java
public static void selectionSortOptimized(int[] arr) {
    int left = 0;
    int right = arr.length - 1;
    
    while (left < right) {
        int minIdx = left;
        int maxIdx = left;
        
        // In ONE scan, find BOTH min and max
        for (int k = left; k <= right; k++) {
            if (arr[k] < arr[minIdx]) minIdx = k;
            if (arr[k] > arr[maxIdx]) maxIdx = k;
        }
        
        // Place minimum at left end
        swap(arr, left, minIdx);
        
        // IMPORTANT: if max was at 'left', it just moved to minIdx!
        if (maxIdx == left) maxIdx = minIdx;
        
        // Place maximum at right end
        swap(arr, right, maxIdx);
        
        left++;
        right--;
    }
}

private static void swap(int[] arr, int i, int j) {
    int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
}

// [5, 3, 8, 4, 2]:
// Pass 1: min=2(idx4), max=8(idx2) → [2, 3, 5, 4, 8]  (2 elements placed!)
// Pass 2: min=3(idx1), max=5(idx2) → [2, 3, 4, 5, 8]  DONE in 2 passes!
```

## 📊 Complexity
| | Time | Why? |
|---|---|---|
| Best | O(n²) | NO early exit — always scans entire unsorted part |
| Average | O(n²) | Same comparisons always |
| Worst | O(n²) | Same |
| Space | O(1) | In-place |
| Stable | ❌ NO | Distant swap can disrupt equal elements |

> 💡 **When to use Selection Sort?** When writing to memory is expensive (like flash storage).
> It does at most **n-1 swaps** — minimum possible swaps!

---

# 🃏 3. INSERTION SORT

## 🧠 Mental Picture
> You're picking up playing cards one by one.
> Each new card → you insert it in the CORRECT POSITION among cards already in your hand.
> Your hand is always sorted.

```
Pick up: 5      → Hand: [5]
Pick up: 3      → 3 < 5? YES → shift 5 right → Hand: [3, 5]
Pick up: 8      → 8 > 5? NO  → insert here   → Hand: [3, 5, 8]
Pick up: 4      → 4 < 8? YES → shift 8
                  4 < 5? YES → shift 5
                  4 > 3? YES → insert → Hand: [3, 4, 5, 8]
Pick up: 2      → shift 8, 5, 4, 3 → insert at start → [2, 3, 4, 5, 8] ✅
```

---

## 🐌 Brute Force

```java
public static void insertionSortBrute(int[] arr) {
    int n = arr.length;
    
    for (int i = 1; i < n; i++) {
        int key = arr[i];    // The card we just picked up
        int j = i - 1;
        
        // Shift all elements GREATER than key to the right
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];  // slide element right
            j--;
        }
        arr[j + 1] = key;   // Insert key in correct spot
    }
}

// Trace for key=4 in [3, 5, 8, 4]:
// j=2: arr[2]=8 > 4 → shift: [3, 5, 8, 8]
// j=1: arr[1]=5 > 4 → shift: [3, 5, 5, 8]
// j=0: arr[0]=3 < 4 → STOP
// Insert at j+1=1: [3, 4, 5, 8] ✅
```

## ⚡ Optimized (Binary Search for insert position)

```java
public static void insertionSortBinary(int[] arr) {
    int n = arr.length;
    
    for (int i = 1; i < n; i++) {
        int key = arr[i];
        
        // Binary search: find WHERE to insert key in arr[0..i-1]
        // Instead of scanning O(n), use O(log n) comparisons
        int pos = binarySearchPosition(arr, key, 0, i - 1);
        
        // Shift elements from pos to i-1 one step right
        int j = i - 1;
        while (j >= pos) {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[pos] = key;
    }
}

private static int binarySearchPosition(int[] arr, int key, int low, int high) {
    while (low <= high) {
        int mid = (low + high) / 2;
        if (arr[mid] == key) return mid;
        else if (arr[mid] < key) low = mid + 1;
        else high = mid - 1;
    }
    return low;  // Insertion point
}

// Comparisons: O(n log n) — much fewer "is this the right spot?" checks
// BUT shifting is still O(n²) — overall worst case unchanged
// Great when comparisons are expensive (e.g., comparing large strings)
```

## 📊 Complexity
| | Basic | Binary |
|---|---|---|
| Best | O(n) | O(n log n) |
| Average | O(n²) | O(n²) |
| Worst | O(n²) | O(n²) |
| Space | O(1) | O(1) |
| Stable | ✅ YES | ✅ YES |
| Adaptive | ✅ YES | Partial |

> 💡 **Python and Java use Insertion Sort internally (TimSort)** for arrays < 32 elements.
> Nearly sorted array? Insertion Sort is the FASTEST of the O(n²) sorts!

---

# 🔀 4. MERGE SORT

## 🧠 Mental Picture
> You have 8 unsorted cards. Divide into 2 groups of 4 → each into 2 groups of 2 → each pair sorted individually.
> Then MERGE sorted groups back: compare heads of each group, take the smaller one.

```
[5, 3, 8, 4, 2, 7, 1, 6]
        SPLIT PHASE:
[5,3,8,4]          [2,7,1,6]
[5,3]  [8,4]      [2,7]  [1,6]
[5][3] [8][4]    [2][7]  [1][6]
        MERGE PHASE:
[3,5]  [4,8]      [2,7]  [1,6]
   [3,4,5,8]          [1,2,6,7]
        [1,2,3,4,5,6,7,8] ✅
```

---

## Implementation

```java
public static void mergeSort(int[] arr, int left, int right) {
    if (left >= right) return;  // Base case: 1 element = already sorted
    
    int mid = left + (right - left) / 2;  // Avoid integer overflow vs (left+right)/2
    
    mergeSort(arr, left, mid);       // Sort left half
    mergeSort(arr, mid + 1, right);  // Sort right half
    merge(arr, left, mid, right);    // Merge both halves
}

private static void merge(int[] arr, int left, int mid, int right) {
    // Create temp arrays for left and right halves
    int[] L = Arrays.copyOfRange(arr, left, mid + 1);
    int[] R = Arrays.copyOfRange(arr, mid + 1, right + 1);
    
    int i = 0, j = 0, k = left;
    
    // Merge: always pick the SMALLER of the two heads
    while (i < L.length && j < R.length) {
        if (L[i] <= R[j]) {      // <= keeps it STABLE
            arr[k++] = L[i++];
        } else {
            arr[k++] = R[j++];
        }
    }
    
    // Copy remaining elements
    while (i < L.length) arr[k++] = L[i++];
    while (j < R.length) arr[k++] = R[j++];
}

// Call: mergeSort(arr, 0, arr.length - 1);
```

## ⚡ Optimized (Bottom-Up Merge Sort — No Recursion)

```java
public static void mergeSortBottomUp(int[] arr) {
    int n = arr.length;
    
    // Start with size=1, then 2, 4, 8, 16...
    for (int size = 1; size < n; size *= 2) {
        for (int left = 0; left < n - size; left += 2 * size) {
            int mid = left + size - 1;
            int right = Math.min(left + 2 * size - 1, n - 1);
            merge(arr, left, mid, right);
        }
    }
}
// Advantage: No recursion overhead (no call stack), same O(n log n) time
```

## 📊 Complexity
| | Time | Why? |
|---|---|---|
| Best | O(n log n) | Always divides and merges |
| Average | O(n log n) | log n levels × n merge work |
| Worst | O(n log n) | Consistent — never degrades! |
| Space | O(n) | Needs temp arrays during merge |
| Stable | ✅ YES | ≤ in merge keeps order |

> 💡 **Best choice when stability matters AND consistent performance needed.**
> Used in Java's `Arrays.sort()` for Object arrays!

---

# ⚡ 5. QUICK SORT

## 🧠 Mental Picture
> Pick a "pivot" (a leader). Everyone SMALLER goes to the LEFT group.
> Everyone LARGER goes to the RIGHT group.
> The pivot is now in its FINAL position forever!
> Repeat for left and right groups.

```
[5, 3, 8, 4, 2, 7, 1, 6]  pivot = 5 (last element for simplicity)
After partition:
[3, 4, 2, 1, 5, 8, 7, 6]
 ^__________^  ↑  ^_____^
  smaller     pivot  larger
  
Recurse on [3,4,2,1] and [8,7,6] → pivot 1, pivot 6
Eventually: [1,2,3,4,5,6,7,8] ✅
```

---

## Implementation (Lomuto Partition)

```java
public static void quickSort(int[] arr, int low, int high) {
    if (low >= high) return;
    
    int pivotIdx = partition(arr, low, high);
    quickSort(arr, low, pivotIdx - 1);   // Sort left of pivot
    quickSort(arr, pivotIdx + 1, high);  // Sort right of pivot
}

private static int partition(int[] arr, int low, int high) {
    int pivot = arr[high];  // Pick last element as pivot
    int i = low - 1;        // i tracks boundary of "smaller" region
    
    for (int j = low; j < high; j++) {
        if (arr[j] <= pivot) {
            i++;  // Expand "smaller" region
            swap(arr, i, j);  // Move element into smaller region
        }
    }
    
    swap(arr, i + 1, high);  // Place pivot at correct position
    return i + 1;
}

// Trace: [3, 6, 8, 10, 1, 2, 1]  pivot=1
// j=0: arr[0]=3 > 1 → skip
// j=1: arr[1]=6 > 1 → skip
// ...
// j=4: arr[4]=1 ≤ 1 → i=0, swap(arr,0,4): [1, 6, 8, 10, 3, 2, 1]
// Final: swap(arr, 1, 6): pivot 1 at index 1 ✅
```

## ⚡ Optimized (3-Way Partition — Handles Duplicates)

```java
// Dutch National Flag by Dijkstra
// Perfect for arrays with many duplicate elements
public static void quickSort3Way(int[] arr, int low, int high) {
    if (low >= high) return;
    
    int pivot = arr[low];
    int lt = low;     // arr[low..lt-1] < pivot
    int gt = high;    // arr[gt+1..high] > pivot
    int i = low + 1; // arr[lt..i-1] == pivot
    
    while (i <= gt) {
        if (arr[i] < pivot) {
            swap(arr, lt++, i++);
        } else if (arr[i] > pivot) {
            swap(arr, i, gt--);
        } else {
            i++;  // Equal to pivot, skip
        }
    }
    
    quickSort3Way(arr, low, lt - 1);
    quickSort3Way(arr, gt + 1, high);
}

// [3,3,3,1,4,1,5,3,3] → after one pass all 3s are in the middle
// No need to recurse on the 3s! Huge speedup for duplicates.
```

## ⚡ Optimized (Randomized Pivot — Avoids Worst Case)

```java
private static Random rand = new Random();

public static void quickSortRandom(int[] arr, int low, int high) {
    if (low >= high) return;
    
    // Randomly swap pivot to avoid O(n²) on sorted input
    int randIdx = low + rand.nextInt(high - low + 1);
    swap(arr, randIdx, high);
    
    int pivotIdx = partition(arr, low, high);
    quickSortRandom(arr, low, pivotIdx - 1);
    quickSortRandom(arr, pivotIdx + 1, high);
}
// Why? Without this, sorted input [1,2,3,4,5] → O(n²) worst case!
// Randomization makes worst case extremely unlikely: expected O(n log n)
```

## 📊 Complexity
| | Time | Why? |
|---|---|---|
| Best | O(n log n) | Balanced partitions |
| Average | O(n log n) | Expected with random pivot |
| Worst | O(n²) | Already sorted + bad pivot choice |
| Space | O(log n) | Recursion stack (avg) |
| Stable | ❌ NO | Swaps can disrupt order |

> 💡 **Fastest in practice for large random datasets.** Cache-friendly, low constant factor.
> Java's `Arrays.sort()` for primitive arrays uses a variant of Quicksort (Dual-Pivot QuickSort)!

---

# 🏔️ 6. HEAP SORT

## 🧠 Mental Picture
> A Max-Heap is like a tournament bracket where the WINNER (largest) is always at the top.
> **Step 1:** Build the tournament bracket (max-heap) from your array.
> **Step 2:** Extract the winner → put at end → re-run the tournament for remaining players.

```
Build Max-Heap from [4, 10, 3, 5, 1]:
        10
       /  \
      5    3
     / \
    4   1
Array: [10, 5, 3, 4, 1]

Extract 10 → put at end: [5, 4, 3, 1, | 10]
Extract  5 → put at end: [4, 1, 3, | 5, 10]
Extract  4 → put at end: [3, 1, | 4, 5, 10]
Extract  3 → put at end: [1, | 3, 4, 5, 10]
SORTED: [1, 3, 4, 5, 10] ✅
```

---

## Implementation

```java
public static void heapSort(int[] arr) {
    int n = arr.length;
    
    // Step 1: Build max-heap (heapify from bottom up)
    // Start from last non-leaf node: (n/2 - 1)
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapify(arr, n, i);
    }
    
    // Step 2: Extract max one by one
    for (int i = n - 1; i > 0; i--) {
        // Move current root (max) to end
        swap(arr, 0, i);
        
        // Heapify reduced heap (ignore sorted elements at end)
        heapify(arr, i, 0);
    }
}

private static void heapify(int[] arr, int n, int i) {
    int largest = i;      // Root
    int left = 2 * i + 1;  // Left child
    int right = 2 * i + 2; // Right child
    
    // Is left child larger than root?
    if (left < n && arr[left] > arr[largest])
        largest = left;
    
    // Is right child larger than current largest?
    if (right < n && arr[right] > arr[largest])
        largest = right;
    
    // If root is not largest, swap and continue heapifying
    if (largest != i) {
        swap(arr, i, largest);
        heapify(arr, n, largest);  // Recursively fix the affected subtree
    }
}
```

## 📊 Complexity
| | Time | Why? |
|---|---|---|
| Best | O(n log n) | Building heap O(n) + n extractions × O(log n) each |
| Average | O(n log n) | Same |
| Worst | O(n log n) | Guaranteed — never degrades! |
| Space | O(1) | In-place! (unlike merge sort) |
| Stable | ❌ NO | Heap operations disrupt order |

> 💡 **Best of both worlds:** O(n log n) guaranteed + O(1) space.
> But poor cache performance makes it slower in practice than QuickSort.

---

# 🔢 7. COUNTING SORT

## 🧠 Mental Picture
> You have 20 students with scores 0-10.
> Instead of comparing, just **COUNT** how many got each score.
> Then **REBUILD** the array: 2 zeros, 3 ones, 1 two, etc.

```
Input: [1, 4, 1, 2, 7, 5, 2]
Scores: 0  1  2  3  4  5  6  7
Count:  0  2  2  0  1  1  0  1

Prefix Sum (for stable sort):
       0  2  4  4  5  6  6  7

Rebuild output from right to left (for stability):
arr[6]=2 → goes to position 3, output[3]=2, count[2]→3
arr[5]=5 → goes to position 5, output[5]=5, count[5]→5
...
Result: [1, 1, 2, 2, 4, 5, 7] ✅
```

---

## Implementation

```java
public static void countingSort(int[] arr, int maxVal) {
    int[] count = new int[maxVal + 1];
    
    // Step 1: Count frequency of each element
    for (int num : arr) count[num]++;
    
    // Step 2: Rebuild array in sorted order
    int idx = 0;
    for (int i = 0; i <= maxVal; i++) {
        while (count[i]-- > 0) {
            arr[idx++] = i;
        }
    }
}

// Stable version (needed for Radix Sort):
public static int[] countingSortStable(int[] arr, int maxVal) {
    int n = arr.length;
    int[] count = new int[maxVal + 1];
    int[] output = new int[n];
    
    for (int num : arr) count[num]++;
    
    // Prefix sum: count[i] = number of elements ≤ i
    for (int i = 1; i <= maxVal; i++) count[i] += count[i - 1];
    
    // Build output (right to left for stability)
    for (int i = n - 1; i >= 0; i--) {
        output[--count[arr[i]]] = arr[i];
    }
    
    return output;
}
```

## 📊 Complexity
| | Time | Space |
|---|---|---|
| All cases | O(n + k) | O(k) |

Where k = range of input (max - min).

> ⚠️ **Limitation:** Only works for **integers** with a known, small range.
> Input [1, 1000000] → needs a million-size count array! Use Radix Sort instead.

---

# 📐 8. RADIX SORT

## 🧠 Mental Picture
> Sort phone numbers in a directory:
> **First pass:** Sort by LAST digit only → [170, 90, 802, 2, 24, 45, 66, 75, 11]
> **Second pass:** Sort by TENS digit → stable sort preserves last digit order
> **Third pass:** Sort by HUNDREDS digit → DONE
> Key: Use a STABLE sort (Counting Sort) at each digit level!

```
Input: [170, 45, 75, 90, 802, 24, 2, 66]

Pass 1 (ones digit):
170→0, 90→0, 802→2, 2→2, 24→4, 45→5, 75→5, 66→6
Sorted: [170, 90, 802, 2, 24, 45, 75, 66]

Pass 2 (tens digit):
802→0, 2→0, 24→2, 45→4, 66→6, 170→7, 75→7, 90→9
Sorted: [802, 2, 24, 45, 66, 170, 75, 90]

Pass 3 (hundreds digit):
2→0, 24→0, 45→0, 66→0, 75→0, 90→0, 170→1, 802→8
Sorted: [2, 24, 45, 66, 75, 90, 170, 802] ✅
```

---

## Implementation

```java
public static void radixSort(int[] arr) {
    int max = Arrays.stream(arr).max().getAsInt();
    
    // Sort by each digit: 1s place, 10s place, 100s place...
    for (int exp = 1; max / exp > 0; exp *= 10) {
        countingSortByDigit(arr, exp);
    }
}

private static void countingSortByDigit(int[] arr, int exp) {
    int n = arr.length;
    int[] output = new int[n];
    int[] count = new int[10];  // Digits 0-9
    
    // Count occurrences of each digit at 'exp' place
    for (int num : arr) count[(num / exp) % 10]++;
    
    // Prefix sum
    for (int i = 1; i < 10; i++) count[i] += count[i - 1];
    
    // Build output (right to left for stability)
    for (int i = n - 1; i >= 0; i--) {
        int digit = (arr[i] / exp) % 10;
        output[--count[digit]] = arr[i];
    }
    
    System.arraycopy(output, 0, arr, 0, n);
}
```

## 📊 Complexity
| | Time | Space |
|---|---|---|
| All cases | O(d × (n + b)) | O(n + b) |

Where d = number of digits, b = base (10 for decimal).

> 💡 For d=3, b=10: O(3(n+10)) ≈ **O(n)** — linear time!
> Fastest for large datasets of fixed-length integers/strings.

---

# 🐚 9. SHELL SORT

## 🧠 Mental Picture
> **Insertion Sort's Problem:** If 2 is at the end of [5, 8, 7, 6, 4, 3, 2], it needs 6 shifts.
> **Shell Sort's Solution:** Start with LARGE gaps (compare far-apart elements), reduce gap each time.
> Final pass (gap=1) is regular insertion sort — but by then, array is ALMOST sorted → very few shifts!

```
Array: [8, 7, 6, 5, 4, 3, 2, 1]  n=8

Gap = 4: Compare arr[i] with arr[i-4]
[8,7,6,5,4,3,2,1] → [4,3,2,1,8,7,6,5]

Gap = 2: Compare arr[i] with arr[i-2]
[4,3,2,1,8,7,6,5] → [2,1,4,3,6,5,8,7]

Gap = 1: Regular insertion sort (almost sorted now!)
[2,1,4,3,6,5,8,7] → [1,2,3,4,5,6,7,8] ✅ (few shifts needed!)
```

---

## Implementation

```java
public static void shellSort(int[] arr) {
    int n = arr.length;
    
    // Start with large gap, reduce by half each time
    for (int gap = n / 2; gap > 0; gap /= 2) {
        
        // Do gapped insertion sort
        for (int i = gap; i < n; i++) {
            int key = arr[i];
            int j = i;
            
            // Shift elements gap positions ahead if they're greater
            while (j >= gap && arr[j - gap] > key) {
                arr[j] = arr[j - gap];
                j -= gap;
            }
            arr[j] = key;
        }
    }
}

// Better gap sequences for improved performance:
// Ciura's sequence: 1, 4, 10, 23, 57, 132, 301, 701
// Hibbard's: 1, 3, 7, 15, 31...  → O(n^(3/2)) guaranteed
```

## 📊 Complexity
| Gap Sequence | Best | Worst |
|---|---|---|
| n/2, n/4... (÷2) | O(n log n) | O(n²) |
| Hibbard (2^k - 1) | O(n^(3/2)) | O(n^(3/2)) |
| Ciura (empirical) | O(n log² n) | Unknown |

---

# 🪣 10. BUCKET SORT

## 🧠 Mental Picture
> Sorting students' exam scores (0.0 to 1.0):
> Create 10 buckets: [0–0.1), [0.1–0.2), ..., [0.9–1.0)
> Toss each score into its bucket.
> Sort each bucket (with insertion sort).
> Concatenate all buckets.

```
Input: [0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12]

Buckets:
[0.1]: [0.17, 0.12]
[0.2]: [0.26, 0.21]
[0.3]: [0.39]
[0.7]: [0.78, 0.72]
[0.9]: [0.94]

Sort each bucket:
[0.1]: [0.12, 0.17]  [0.2]: [0.21, 0.26] ...

Concatenate: [0.12, 0.17, 0.21, 0.26, 0.39, 0.72, 0.78, 0.94] ✅
```

---

## Implementation

```java
public static void bucketSort(float[] arr) {
    int n = arr.length;
    List<Float>[] buckets = new ArrayList[n];
    
    // Initialize buckets
    for (int i = 0; i < n; i++) buckets[i] = new ArrayList<>();
    
    // Distribute elements into buckets
    for (float num : arr) {
        int bucketIdx = (int)(num * n);  // Map [0,1) → bucket index
        buckets[bucketIdx].add(num);
    }
    
    // Sort each bucket
    for (List<Float> bucket : buckets) Collections.sort(bucket);
    
    // Concatenate
    int idx = 0;
    for (List<Float> bucket : buckets)
        for (float num : bucket) arr[idx++] = num;
}
```

## 📊 Complexity
| | Time | Space |
|---|---|---|
| Best (uniform) | O(n + k) | O(n + k) |
| Average | O(n + k) | O(n + k) |
| Worst (all in one bucket) | O(n²) | O(n) |

---

# 🧩 11. TIM SORT (Java's Default!)

## 🧠 Mental Picture
> **Observation:** Real-world data often has RUNS — sequences already sorted.
> TimSort finds these runs, uses Insertion Sort to build them up to min size,
> then merges them with a smart Merge Sort.
> It's the **best of both worlds.**

```
Input: [3, 1, 2, | 8, 6, 7, 9, | 5, 4]
         run1        run2         partial

Extend runs with Insertion Sort (min run size = 32):
Run 1: [1, 2, 3]
Run 2: [6, 7, 8, 9]
Run 3: [4, 5]

Merge runs: [1,2,3] + [6,7,8,9] = [1,2,3,6,7,8,9]
Final merge: [1,2,3,6,7,8,9] + [4,5] = [1,2,3,4,5,6,7,8,9] ✅
```

## 📊 Complexity
| | Time | Space |
|---|---|---|
| Best (already sorted) | O(n) | O(n) |
| Average | O(n log n) | O(n) |
| Worst | O(n log n) | O(n) |
| Stable | ✅ YES | |

> 💡 **This is `Arrays.sort()` for Objects and `Collections.sort()` in Java.**
> You don't implement it — but know how it works!

---

# 📋 MASTER COMPARISON TABLE

```
╔══════════════════╦══════════════╦══════════════╦══════════════╦═════════╦════════╦═══════════╗
║   Algorithm      ║     Best     ║   Average    ║    Worst     ║  Space  ║ Stable ║ Use When  ║
╠══════════════════╬══════════════╬══════════════╬══════════════╬═════════╬════════╬═══════════╣
║  Bubble Sort     ║    O(n)      ║    O(n²)     ║    O(n²)     ║  O(1)   ║  YES   ║ Teaching  ║
║  Selection Sort  ║    O(n²)     ║    O(n²)     ║    O(n²)     ║  O(1)   ║   NO   ║ Min swaps ║
║  Insertion Sort  ║    O(n)      ║    O(n²)     ║    O(n²)     ║  O(1)   ║  YES   ║ Small/    ║
║                  ║              ║              ║              ║         ║        ║ near-sort ║
║  Merge Sort      ║  O(n log n)  ║  O(n log n)  ║  O(n log n)  ║  O(n)   ║  YES   ║ Stable +  ║
║                  ║              ║              ║              ║         ║        ║ Linked List║
║  Quick Sort      ║  O(n log n)  ║  O(n log n)  ║    O(n²)     ║O(log n) ║   NO   ║ General   ║
║                  ║              ║              ║              ║ avg     ║        ║ purpose   ║
║  Heap Sort       ║  O(n log n)  ║  O(n log n)  ║  O(n log n)  ║  O(1)   ║   NO   ║ Guaranteed║
║                  ║              ║              ║              ║         ║        ║ + in-place║
║  Counting Sort   ║   O(n + k)   ║   O(n + k)   ║   O(n + k)   ║  O(k)   ║  YES   ║ Small int ║
║                  ║              ║              ║              ║         ║        ║ range     ║
║  Radix Sort      ║  O(d(n+b))   ║  O(d(n+b))   ║  O(d(n+b))   ║ O(n+b)  ║  YES   ║ Large int ║
║  Shell Sort      ║  O(n log n)  ║  O(n log² n) ║    O(n²)     ║  O(1)   ║   NO   ║ Medium    ║
║                  ║              ║              ║              ║         ║        ║ data      ║
║  Bucket Sort     ║   O(n + k)   ║   O(n + k)   ║    O(n²)     ║ O(n+k)  ║  YES   ║ Uniform   ║
║                  ║              ║              ║              ║         ║        ║ floats    ║
║  Tim Sort        ║    O(n)      ║  O(n log n)  ║  O(n log n)  ║  O(n)   ║  YES   ║ Real-world║
║                  ║              ║              ║              ║         ║        ║ (DEFAULT) ║
╚══════════════════╩══════════════╩══════════════╩══════════════╩═════════╩════════╩═══════════╝
```

---

## 🎯 DECISION FLOWCHART: Which Sort to Use?

```
Is data already nearly sorted?
├── YES → Insertion Sort (O(n) best case)
└── NO
    ├── Are elements integers with small range (0 to 10000)?
    │   ├── YES → Counting Sort (O(n+k))
    │   └── NO
    │       ├── Are elements integers with many digits?
    │       │   └── YES → Radix Sort (O(dn))
    │       └── Are elements floats uniformly distributed 0-1?
    │           └── YES → Bucket Sort
    └── General case:
        ├── Need stable sort + O(n) extra memory OK?
        │   └── YES → Merge Sort
        ├── Need in-place O(1) memory + stability not needed?
        │   └── YES → Heap Sort (guaranteed) or Quick Sort (faster avg)
        └── Just use Java's Arrays.sort() / Collections.sort()
            (It uses Tim Sort internally — best of everything!)
```

---

# 💻 PRACTICE PROBLEMS

## 🟢 LeetCode

### Sorting Fundamentals:
| # | Problem | Difficulty | Sort Concept |
|---|---|---|---|
| [912](https://leetcode.com/problems/sort-an-array/) | Sort an Array | Medium | Implement merge/quick/heap sort |
| [75](https://leetcode.com/problems/sort-colors/) | Sort Colors | Medium | 3-way partition (Quick Sort idea) |
| [283](https://leetcode.com/problems/move-zeroes/) | Move Zeroes | Easy | Partition (like Quick Sort) |
| [88](https://leetcode.com/problems/merge-sorted-array/) | Merge Sorted Array | Easy | Merge step of Merge Sort |
| [147](https://leetcode.com/problems/insertion-sort-list/) | Insertion Sort List | Medium | Insertion Sort on LL |
| [148](https://leetcode.com/problems/sort-list/) | Sort List | Medium | Merge Sort on LL |

### Counting/Radix:
| # | Problem | Difficulty | Sort Concept |
|---|---|---|---|
| [1122](https://leetcode.com/problems/relative-sort-array/) | Relative Sort Array | Easy | Counting Sort |
| [274](https://leetcode.com/problems/h-index/) | H-Index | Medium | Counting Sort |
| [164](https://leetcode.com/problems/maximum-gap/) | Maximum Gap | Hard | Radix/Bucket Sort |
| [220](https://leetcode.com/problems/contains-duplicate-iii/) | Contains Duplicate III | Hard | Bucket Sort |

### Inversions and Applications:
| # | Problem | Difficulty | Sort Concept |
|---|---|---|---|
| [493](https://leetcode.com/problems/reverse-pairs/) | Reverse Pairs | Hard | Merge Sort + counting |
| [315](https://leetcode.com/problems/count-of-smaller-numbers-after-self/) | Count Smaller Numbers | Hard | Merge Sort |
| [327](https://leetcode.com/problems/count-of-range-sum/) | Count of Range Sum | Hard | Merge Sort |

### Quick Select (Quick Sort cousin):
| # | Problem | Difficulty |
|---|---|---|
| [215](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Kth Largest Element | Medium |
| [347](https://leetcode.com/problems/top-k-frequent-elements/) | Top K Frequent Elements | Medium |
| [973](https://leetcode.com/problems/k-closest-points-to-origin/) | K Closest Points to Origin | Medium |

---

## 🟠 GeeksForGeeks

| Problem | Link | Concept |
|---|---|---|
| Bubble Sort | [GFG](https://www.geeksforgeeks.org/problems/bubble-sort/1) | Classic |
| Selection Sort | [GFG](https://www.geeksforgeeks.org/problems/selection-sort/1) | Classic |
| Insertion Sort | [GFG](https://www.geeksforgeeks.org/problems/insertion-sort/0) | Classic |
| Merge Sort | [GFG](https://www.geeksforgeeks.org/problems/merge-sort/1) | Classic |
| Quick Sort | [GFG](https://www.geeksforgeeks.org/problems/quick-sort/1) | Classic |
| Count Inversions | [GFG](https://www.geeksforgeeks.org/problems/inversion-of-array-1587115620/1) | Merge Sort |
| Kth Smallest Element | [GFG](https://www.geeksforgeeks.org/problems/kth-smallest-element5635/1) | Quick Select |
| Sort 0s, 1s and 2s | [GFG](https://www.geeksforgeeks.org/problems/sort-an-array-of-0s-1s-and-2s4231/1) | Dutch Flag |
| Minimum Swaps to Sort | [GFG](https://www.geeksforgeeks.org/minimum-number-of-swaps-required-to-sort-an-array/) | Selection Sort insight |
| Sort by Frequency | [GFG](https://www.geeksforgeeks.org/problems/sorting-elements-of-an-array-by-frequency/0) | Custom comparator |
| Merge Without Extra Space | [GFG](https://www.geeksforgeeks.org/problems/merge-two-sorted-arrays-1587115620/1) | Merge Sort variant |
| Nuts and Bolts | [GFG](https://www.geeksforgeeks.org/nuts-and-bolts-problem-lock-key-problem/) | Quick Sort intuition |

---

## 📈 4-WEEK PROGRESSION PATH

```
Week 1 — O(n²) Sorts:
  Day 1-2: Implement Bubble, Selection, Insertion Sort
  Day 3-4: LC #283, #88, LC #147
  Day 5-7: GFG classics + Count Inversions (brute: O(n²) with Bubble Sort idea)

Week 2 — O(n log n) Sorts:
  Day 1-2: Merge Sort + LC #148 (Merge Sort List)
  Day 3-4: Quick Sort + LC #75, #912
  Day 5-7: Heap Sort + LC #215, #347

Week 3 — Linear Sorts + Advanced:
  Day 1-2: Counting Sort + Radix Sort + LC #274, #164
  Day 3-4: Bucket Sort + Shell Sort
  Day 5-7: LC #220, #493

Week 4 — Hard Problems:
  Day 1-2: LC #315, #327 (Merge Sort advanced)
  Day 3-4: LC #973 + GFG Kth Smallest
  Day 5-7: Mock interviews combining sort concepts
```

---

## 🔑 TOP INTERVIEW QUESTIONS

1. **Why is Quick Sort faster than Merge Sort in practice?** → Cache locality (in-place), lower constant factor
2. **When does Quick Sort become O(n²)?** → Already sorted array with last element as pivot
3. **Why is Merge Sort preferred for Linked Lists?** → No random access needed; no extra space required
4. **Can you make Selection Sort stable?** → Yes, shift instead of swap (makes it O(n²) shifts)
5. **What is an inversion?** → Pair (i,j) where i<j but arr[i]>arr[j]. Bubble sort's swap count = inversions.
6. **Why does TimSort use Insertion Sort for small arrays?** → Low overhead, O(n) for nearly-sorted, cache-friendly
7. **What's the lower bound for comparison-based sorting?** → Ω(n log n) — proven by decision tree argument
8. **When can we beat O(n log n)?** → Non-comparison sorts (Counting, Radix, Bucket) by exploiting value structure
9. **Dual-Pivot QuickSort in Java?** → Java's `Arrays.sort()` uses two pivots → fewer comparisons on average
10. **What is external sorting?** → Merge Sort variant for data too large to fit in memory

---

*"Tell me and I forget. Teach me and I remember. Involve me and I learn." — Benjamin Franklin*
*Code it, trace it, visualize it. That's how sorting algorithms become second nature.*
