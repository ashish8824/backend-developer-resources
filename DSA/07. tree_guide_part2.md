
---

# 21. Complete LeetCode Problem List

## 🟢 Easy Problems

| # | Problem | Pattern | Key Insight |
|---|---------|---------|-------------|
| 94 | Binary Tree Inorder Traversal | Traversal | Left-Root-Right |
| 100 | Same Tree | Recursion | Compare node by node |
| 101 | Symmetric Tree | Recursion | Mirror check |
| 104 | Maximum Depth of Binary Tree | Recursion | 1 + max(left, right) |
| 107 | Binary Tree Level Order Bottom-Up | BFS | Reverse level order |
| 108 | Convert Sorted Array to BST | BST Build | Mid = root |
| 110 | Balanced Binary Tree | Recursion | Height diff ≤ 1 |
| 111 | Minimum Depth | Recursion | Min to LEAF node |
| 112 | Path Sum | DFS | Reduce target by node value |
| 144 | Binary Tree Preorder Traversal | Traversal | Root-Left-Right |
| 145 | Binary Tree Postorder Traversal | Traversal | Left-Right-Root |
| 226 | Invert Binary Tree | Recursion | Swap left and right |
| 235 | Lowest Common Ancestor BST | BST | Both go same side? |
| 257 | Binary Tree Paths | DFS + Backtrack | Root-to-leaf paths |
| 404 | Sum of Left Leaves | DFS | Track if left child |
| 501 | Find Mode in BST | BST Inorder | Track consecutive |
| 530 | Minimum Absolute Difference BST | BST Inorder | Adjacent in inorder |
| 543 | Diameter of Binary Tree | Recursion | left + right heights |
| 563 | Binary Tree Tilt | Recursion | Sum diffs postorder |
| 572 | Subtree of Another Tree | Recursion | isSameTree at each node |
| 606 | Construct String from Tree | Preorder | Add parens for children |
| 617 | Merge Two Binary Trees | Recursion | Add values, recurse |
| 637 | Average of Levels | BFS | Sum / levelSize |
| 669 | Trim a BST | BST | Skip out-of-range subtrees |
| 671 | Second Minimum Node | DFS | Track global min |
| 700 | Search in BST | BST | Go left or right |
| 783 | Minimum Distance Between BST Nodes | BST Inorder | Track previous |
| 872 | Leaf-Similar Trees | DFS | Compare leaf sequences |
| 938 | Range Sum of BST | BST | Skip out-of-range |
| 965 | Univalued Binary Tree | DFS | All same as root? |
| 1022 | Sum of Root To Leaf Binary Numbers | DFS | Shift bits |
| 1038 | BST to Greater Sum Tree | BST Reverse Inorder | Accumulate from right |
| 1302 | Deepest Leaves Sum | BFS | Sum last level |
| 1448 | Count Good Nodes | DFS | Track path max |
| 1469 | Find All The Lonely Nodes | DFS | Single child nodes |

---

## 🟡 Medium Problems

| # | Problem | Pattern | Key Insight |
|---|---------|---------|-------------|
| 95 | Unique Binary Search Trees II | BST Build | Generate all BSTs |
| 96 | Unique Binary Search Trees | DP | Catalan numbers |
| 98 | Validate Binary Search Tree | BST | Pass min/max bounds |
| 102 | Binary Tree Level Order Traversal | BFS | Process levelSize per loop |
| 103 | Binary Tree Zigzag Level Order | BFS | Toggle direction |
| 105 | Construct from Preorder + Inorder | Construction | Pre[0]=root |
| 106 | Construct from Inorder + Postorder | Construction | Post[-1]=root |
| 113 | Path Sum II | DFS+Backtrack | Collect all valid paths |
| 114 | Flatten Binary Tree to Linked List | Tree Modify | In-place preorder |
| 116 | Populating Next Right Pointers | BFS | Connect level nodes |
| 117 | Populating Next Right Pointers II | BFS | General case |
| 129 | Sum Root to Leaf Numbers | DFS | Carry current number |
| 144 | Binary Tree Preorder Iterative | Traversal | Stack + right before left |
| 173 | Binary Search Tree Iterator | BST Inorder | Lazy inorder with stack |
| 199 | Binary Tree Right Side View | BFS | Last node per level |
| 200 | Number of Islands | BFS/DFS | Flood fill |
| 208 | Implement Trie | Trie | 26-array children |
| 211 | Design Add and Search Words | Trie + DFS | '.' matches any char |
| 222 | Count Complete Tree Nodes | Binary Search | O(log²n) |
| 230 | Kth Smallest in BST | BST Inorder | Stop at kth |
| 236 | Lowest Common Ancestor | DFS | Split left/right |
| 285 | Inorder Successor in BST | BST | Smallest > p |
| 297 | Serialize and Deserialize | DFS | Preorder + null markers |
| 298 | Binary Tree Longest Consecutive Sequence | DFS | Track length |
| 307 | Range Sum Query Mutable | Segment Tree | Point update + range query |
| 310 | Minimum Height Trees | BFS | Peel leaves |
| 314 | Binary Tree Vertical Order | BFS + sort | Column index |
| 322 | Coin Change | DP | Classic DP |
| 337 | House Robber III | Tree DP | Rob or not rob |
| 404 | Sum of Left Leaves | DFS | isLeft parameter |
| 427 | Construct Quad Tree | Divide+Conquer | 4-ary tree |
| 429 | N-ary Tree Level Order | BFS | Add all children |
| 437 | Path Sum III | Prefix Sum + DFS | Any path, any start |
| 449 | Serialize and Deserialize BST | BST | Compact encoding |
| 450 | Delete Node in BST | BST | 3 cases |
| 508 | Most Frequent Subtree Sum | DFS + Map | Postorder sum |
| 513 | Find Bottom Left Tree Value | BFS | First of last level |
| 515 | Find Largest Value in Row | BFS | Max per level |
| 538 | Convert BST to Greater Tree | Reverse Inorder | Accumulate right→left |
| 545 | Boundary of Binary Tree | DFS | Left + leaves + right |
| 560 | Subarray Sum Equals K | Prefix Sum | Map of prefix sums |
| 623 | Add One Row to Tree | BFS | Insert at depth d |
| 637 | Average of Levels | BFS | Sum / count |
| 648 | Replace Words | Trie | Shortest root match |
| 652 | Find Duplicate Subtrees | DFS + Serialization | Serialize + map |
| 653 | Two Sum IV Input is BST | DFS + Set | Check complement |
| 654 | Maximum Binary Tree | Recursion | Max = root |
| 662 | Maximum Width of Binary Tree | BFS + Index | last - first + 1 |
| 669 | Trim a BST | BST | Return different subtree |
| 671 | Second Minimum Node | DFS | Track two values |
| 701 | Insert into BST | BST | Recursive insert |
| 814 | Binary Tree Pruning | DFS | Remove subtrees with no 1 |
| 863 | All Nodes Distance K | BFS + Parent Map | Find all at distance k |
| 889 | Construct from Preorder + Postorder | Construction | Both need unique values |
| 894 | All Possible Full Binary Trees | Recursion | Even nodes → null |
| 987 | Vertical Order Traversal | BFS | Sort by col, row, val |
| 993 | Cousins in Binary Tree | BFS | Same level, different parent |
| 998 | Maximum Binary Tree II | BST-like | Insert on right side |
| 1008 | Construct BST from Preorder | BST + Bounds | Min/max bounds |
| 1038 | BST to Greater Sum Tree | Reverse Inorder | Right→Root→Left |
| 1123 | Lowest Common Ancestor Deepest Leaves | DFS | Return node + height |
| 1161 | Maximum Level Sum | BFS | Track max sum level |
| 1245 | Tree Diameter | DFS | Max path through any node |
| 1315 | Sum of Nodes with Even-Valued Grandparent | DFS | Pass grandparent value |
| 1325 | Delete Leaves With Given Value | DFS | Postorder deletion |
| 1339 | Maximum Product of Splitted Binary Tree | DFS | Total - subtree |
| 1372 | Longest ZigZag Path | DFS | Track direction |
| 1382 | Balance a BST | Inorder + Rebuild | Sort then build |
| 1448 | Count Good Nodes | DFS | Max on path |
| 1457 | Pseudo-Palindromic Paths | DFS + Bitmask | XOR of digits |
| 1530 | Number of Good Leaf Node Pairs | DFS | Distance between leaves |

---

## 🔴 Hard Problems

| # | Problem | Pattern | Key Insight |
|---|---------|---------|-------------|
| 23 | Merge K Sorted Lists | Heap / Divide+Conquer | K-way merge |
| 32 | Longest Valid Parentheses | Stack/DP | — |
| 99 | Recover Binary Search Tree | Morris / Inorder | Find two swapped |
| 124 | Binary Tree Maximum Path Sum | DFS | Max gain per node |
| 212 | Word Search II | Trie + DFS | Prune with trie |
| 214 | Shortest Palindrome | KMP / Trie | — |
| 297 | Serialize and Deserialize Binary Tree | DFS | Preorder + # |
| 315 | Count of Smaller Numbers After Self | BIT/Merge Sort | Rank compress |
| 327 | Count of Range Sum | Merge Sort / BIT | Prefix + range |
| 428 | Serialize Deserialize N-ary Tree | DFS | Child count encoding |
| 440 | K-th Smallest in Lexicographical Order | Trie Traversal | Count nodes |
| 472 | Concatenated Words | Trie + DP | Word break with trie |
| 493 | Reverse Pairs | Merge Sort / BIT | Count inversions |
| 502 | IPO | Two Heaps | Max profit greedy |
| 600 | Non-negative Integers without Consecutive Ones | DP on Trie | Digit DP |
| 623 | Add One Row to Tree | BFS | — |
| 685 | Redundant Connection II | Union-Find / DFS | Directed graph |
| 726 | Number of Atoms | Stack + Map | Nested parsing |
| 745 | Prefix and Suffix Search | Trie | Combined prefix-suffix key |
| 759 | Employee Free Time | Heap / Sort | Interval merge |
| 834 | Sum of Distances in Tree | DFS × 2 | Rerooting technique |
| 968 | Binary Tree Cameras | Greedy DFS | 3 states |
| 979 | Distribute Coins in Binary Tree | DFS | Excess coins |
| 1032 | Stream of Characters | Trie | Reverse Trie |
| 1110 | Delete Nodes and Return Forest | DFS | Mark deletions |
| 1145 | Binary Tree Coloring Game | DFS | Count subtree sizes |
| 1373 | Maximum Sum BST in Binary Tree | DFS | Validate + track |
| 1483 | Kth Ancestor of a Tree Node | Binary Lifting | Sparse table |
| 1569 | Number of Ways to Reorder Array to Get BST | DFS + Comb | Count orderings |
| 1600 | Throne Inheritance | N-ary + DFS | Eulerian path |
| 1612 | Check If Two Expression Trees are Equivalent | Canonicalize | Normalize tree |
| 1932 | Merge BSTs to Create Single BST | BST merge | Match leaves to roots |
| 2003 | Smallest Missing Genetic Value in Each Subtree | DFS | Union-Find |
| 2322 | Minimum Score of Path Between Two Cities | DFS | Track min edge |

---

# 22. How to Identify Tree Problem Patterns

## 🔍 The Master Decision Tree

```
Is the input a TREE structure?
│
├── Does order of visiting matter?
│   ├── Visit level by level → BFS (Queue)
│   └── Visit depth first   → DFS (Recursion/Stack)
│
├── Is it a BST?
│   ├── Use BST property (left<root<right) for O(log n)
│   ├── Inorder gives sorted order
│   └── Pass min/max bounds for validation
│
├── What are you computing?
│   ├── Height/Depth     → Postorder: 1 + max(L, R)
│   ├── Sum/Count        → Postorder: left + right + node
│   ├── Path/Max Path    → Global variable + local return
│   ├── Level info       → BFS
│   ├── Ancestor         → LCA pattern
│   └── Build tree       → Inorder/Preorder/Postorder construction
│
├── Does it involve PREFIX matching?
│   └── → Trie
│
├── Does it involve RANGE QUERIES with updates?
│   └── → Segment Tree or Fenwick Tree
│
└── Does it involve SELF-BALANCING?
    └── → AVL Tree or Red-Black Tree
```

## ⚡ Pattern Quick-Lookup

| You see this... | Use this pattern |
|-----------------|-----------------|
| "level order", "level by level" | BFS with Queue |
| "right/left side view" | BFS, take last/first of each level |
| "maximum depth", "height" | Postorder DFS, return 1+max(L,R) |
| "minimum depth" | BFS (finds shortest path to leaf) |
| "diameter" | DFS, global max = L+R height |
| "balanced" | DFS returning -1 if imbalanced |
| "path sum" | DFS, reduce target |
| "all root-to-leaf paths" | DFS + backtracking |
| "max path sum (any→any)" | DFS with global max, return 1-directional |
| "lowest common ancestor" | LCA pattern, check if found in L or R |
| "serialize/deserialize" | Preorder DFS with null markers |
| "inorder = sorted" | BST inorder traversal |
| "validate BST" | DFS with min/max bounds |
| "kth smallest in BST" | BST inorder, stop at k |
| "construct from traversals" | Find root in preorder/postorder, use inorder to split |
| "prefix", "autocomplete" | Trie |
| "range sum with updates" | Segment Tree / Fenwick Tree |
| "duplicate subtrees" | Serialize subtree + HashMap |
| "vertical order" | BFS with column index |
| "distance between nodes" | Parent pointer map + BFS |
| "all possible BSTs" | Catalan number recursion |

---

# 23. Interview Cheat Sheet

## ⚡ TreeNode Definition (Always write first)

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode() {}
    TreeNode(int val) { this.val = val; }
    TreeNode(int val, TreeNode left, TreeNode right) {
        this.val = val; this.left = left; this.right = right;
    }
}
```

---

## ⚡ All Traversals (Recursive — 30 seconds each)

```java
// INORDER (L-Root-R)
void inorder(TreeNode node, List<Integer> res) {
    if (node == null) return;
    inorder(node.left, res);
    res.add(node.val);
    inorder(node.right, res);
}

// PREORDER (Root-L-R)
void preorder(TreeNode node, List<Integer> res) {
    if (node == null) return;
    res.add(node.val);
    preorder(node.left, res);
    preorder(node.right, res);
}

// POSTORDER (L-R-Root)
void postorder(TreeNode node, List<Integer> res) {
    if (node == null) return;
    postorder(node.left, res);
    postorder(node.right, res);
    res.add(node.val);
}

// LEVEL ORDER (BFS)
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int sz = q.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < sz; i++) {
            TreeNode n = q.poll();
            level.add(n.val);
            if (n.left  != null) q.offer(n.left);
            if (n.right != null) q.offer(n.right);
        }
        res.add(level);
    }
    return res;
}
```

---

## ⚡ Core Recursive Patterns

```java
// HEIGHT
int height(TreeNode node) {
    if (node == null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
}

// COUNT NODES
int count(TreeNode node) {
    if (node == null) return 0;
    return 1 + count(node.left) + count(node.right);
}

// TREE SUM
int sum(TreeNode node) {
    if (node == null) return 0;
    return node.val + sum(node.left) + sum(node.right);
}

// IS SAME
boolean isSame(TreeNode p, TreeNode q) {
    if (p == null && q == null) return true;
    if (p == null || q == null) return false;
    return p.val == q.val && isSame(p.left, q.left) && isSame(p.right, q.right);
}

// IS BALANCED (returns -1 if not balanced)
int checkBalanced(TreeNode node) {
    if (node == null) return 0;
    int L = checkBalanced(node.left);
    int R = checkBalanced(node.right);
    if (L == -1 || R == -1 || Math.abs(L-R) > 1) return -1;
    return 1 + Math.max(L, R);
}

// DIAMETER PATTERN
int maxDiameter = 0;
int heightForDiameter(TreeNode node) {
    if (node == null) return 0;
    int L = heightForDiameter(node.left);
    int R = heightForDiameter(node.right);
    maxDiameter = Math.max(maxDiameter, L + R);
    return 1 + Math.max(L, R);
}

// MAX PATH SUM PATTERN
int globalMax = Integer.MIN_VALUE;
int maxGain(TreeNode node) {
    if (node == null) return 0;
    int L = Math.max(0, maxGain(node.left));
    int R = Math.max(0, maxGain(node.right));
    globalMax = Math.max(globalMax, node.val + L + R);
    return node.val + Math.max(L, R);
}
```

---

## ⚡ BST Core Operations

```java
// SEARCH BST
TreeNode searchBST(TreeNode root, int val) {
    while (root != null) {
        if (val == root.val) return root;
        root = val < root.val ? root.left : root.right;
    }
    return null;
}

// INSERT BST
TreeNode insertBST(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    if (val < root.val) root.left  = insertBST(root.left,  val);
    else                root.right = insertBST(root.right, val);
    return root;
}

// VALIDATE BST
boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}
boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val)
        && validate(node.right, node.val, max);
}

// LCA BST
TreeNode lcaBST(TreeNode root, TreeNode p, TreeNode q) {
    while (root != null) {
        if (p.val < root.val && q.val < root.val) root = root.left;
        else if (p.val > root.val && q.val > root.val) root = root.right;
        else return root;
    }
    return null;
}

// LCA BINARY TREE (General)
TreeNode lca(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode L = lca(root.left, p, q);
    TreeNode R = lca(root.right, p, q);
    if (L != null && R != null) return root;
    return L != null ? L : R;
}

// INORDER SUCCESSOR BST
TreeNode successor(TreeNode root, TreeNode p) {
    TreeNode result = null;
    while (root != null) {
        if (p.val < root.val) { result = root; root = root.left; }
        else root = root.right;
    }
    return result;
}
```

---

## ⚡ BFS Template (Level Order)

```java
// Standard BFS template for any tree problem
void bfs(TreeNode root) {
    if (root == null) return;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size(); // CRITICAL: snapshot BEFORE loop

        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            // Process node here

            if (node.left  != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        // After for loop: processed one complete level
    }
}
```

---

## ⚡ Trie Template

```java
class TrieNode {
    TrieNode[] ch = new TrieNode[26];
    boolean end;
}
class Trie {
    TrieNode root = new TrieNode();

    void insert(String w) {
        TrieNode n = root;
        for (char c : w.toCharArray()) {
            if (n.ch[c-'a'] == null) n.ch[c-'a'] = new TrieNode();
            n = n.ch[c-'a'];
        }
        n.end = true;
    }
    boolean search(String w) {
        TrieNode n = find(w);
        return n != null && n.end;
    }
    boolean startsWith(String p) { return find(p) != null; }
    TrieNode find(String s) {
        TrieNode n = root;
        for (char c : s.toCharArray()) {
            if (n.ch[c-'a'] == null) return null;
            n = n.ch[c-'a'];
        }
        return n;
    }
}
```

---

## ⚡ Segment Tree Template

```java
class SegTree {
    int[] tree;
    int n;
    SegTree(int[] arr) {
        n = arr.length;
        tree = new int[2*n];
        for (int i=0; i<n; i++) tree[n+i] = arr[i];
        for (int i=n-1; i>0; i--) tree[i] = tree[2*i] + tree[2*i+1];
    }
    void update(int i, int v) {
        tree[i+n] = v;
        for (i=(i+n)/2; i>0; i/=2) tree[i] = tree[2*i] + tree[2*i+1];
    }
    int query(int l, int r) { // [l, r) half-open
        int s = 0;
        for (l+=n, r+=n; l<r; l/=2, r/=2) {
            if ((l&1)==1) s += tree[l++];
            if ((r&1)==1) s += tree[--r];
        }
        return s;
    }
}
```

---

## ⚡ Fenwick Tree Template

```java
class BIT {
    int[] bit;
    int n;
    BIT(int n) { this.n = n; bit = new int[n+1]; }
    void update(int i, int d) { for (; i<=n; i+=i&(-i)) bit[i]+=d; }
    int query(int i) { int s=0; for (; i>0; i-=i&(-i)) s+=bit[i]; return s; }
    int query(int l, int r) { return query(r) - query(l-1); }
}
```

---

## 📊 Complexity Summary

| Structure | Search | Insert | Delete | Space |
|-----------|--------|--------|--------|-------|
| Binary Tree | O(n) | O(n) | O(n) | O(n) |
| BST (avg) | O(log n) | O(log n) | O(log n) | O(n) |
| BST (worst) | O(n) | O(n) | O(n) | O(n) |
| AVL Tree | O(log n) | O(log n) | O(log n) | O(n) |
| Trie | O(L) | O(L) | O(L) | O(alphabet×n×L) |
| Segment Tree | O(log n) | O(log n) | O(log n) | O(n) |
| Fenwick Tree | O(log n) | O(log n) | O(log n) | O(n) |

Where L = string/word length

| Traversal | Time | Space (recursive) | Space (iterative) |
|-----------|------|-------------------|-------------------|
| Inorder | O(n) | O(h) | O(h) |
| Preorder | O(n) | O(h) | O(h) |
| Postorder | O(n) | O(h) | O(h) |
| Level Order | O(n) | — | O(w) — max width |

h = height (O(log n) balanced, O(n) skewed)

---

## 🎯 Most Important Problems to Master (Top 20)

```
Absolute Must-Know:
1.  LC #104  — Maximum Depth                 (Base recursion)
2.  LC #226  — Invert Binary Tree            (Simple swap)
3.  LC #94   — Inorder Traversal             (Both recursive + iterative)
4.  LC #102  — Level Order Traversal         (BFS template)
5.  LC #100  — Same Tree                     (Recursive comparison)
6.  LC #98   — Validate BST                  (Pass min/max bounds)
7.  LC #110  — Balanced Binary Tree          (Return -1 if invalid)
8.  LC #543  — Diameter of Binary Tree       (Global max, local height)
9.  LC #236  — LCA of Binary Tree            (Classic LCA)
10. LC #105  — Construct from Preorder+Inorder (Tree construction)

Intermediate Must-Know:
11. LC #124  — Maximum Path Sum              (Hardest DFS pattern)
12. LC #297  — Serialize/Deserialize         (Preorder + null markers)
13. LC #437  — Path Sum III                  (Prefix sum on tree)
14. LC #230  — Kth Smallest in BST           (BST inorder)
15. LC #199  — Right Side View               (BFS last-per-level)
16. LC #208  — Implement Trie               (Trie fundamentals)
17. LC #307  — Range Sum Query Mutable       (Segment tree)
18. LC #337  — House Robber III              (Tree DP)
19. LC #114  — Flatten to Linked List        (Tree modification)
20. LC #572  — Subtree of Another Tree       (Nested isSameTree)
```

---

## 🧠 Mindset for Tree Problems in Interview

```
Step 1: IDENTIFY the tree type
  → Binary tree? BST? N-ary? Trie?

Step 2: IDENTIFY what to compute
  → Height? Count? Path? Level info? Range query?

Step 3: CHOOSE traversal
  → Need parent before children? → PREORDER
  → Need children before parent? → POSTORDER (most common!)
  → Need sorted BST output?      → INORDER
  → Need level by level?         → BFS

Step 4: DEFINE the recursive function
  → What does it return? (int, boolean, TreeNode)
  → What is the base case? (null node)
  → How do you combine left+right+root?

Step 5: HANDLE EDGE CASES
  → null root
  → single node
  → skewed tree (all left or all right)
  → negative values (for path sums)

The pattern: 90% of tree problems = null check + recurse left + recurse right + combine
```

---

## 📘 Learning Path

**Week 1 — Foundations:**
- Understand all tree types and terminology
- Implement all 4 traversals (recursive + iterative)
- LC #104, #226, #100, #101, #112, #543

**Week 2 — BST:**
- Implement BST operations from scratch
- LC #98, #700, #701, #450, #230, #235, #108

**Week 3 — DFS Patterns:**
- Master the global variable + local return trick
- LC #110, #124, #437, #572, #113, #236

**Week 4 — BFS + Construction:**
- Master level order and all its variants
- LC #102, #103, #199, #199, #105, #106, #297

**Week 5 — Advanced:**
- Trie, Segment Tree, Fenwick Tree
- LC #208, #212, #307, #315, #834

**Week 6 — Hard + Consolidation:**
- LC #124, #99, #968, #1373, #212 (Word Search II)

---

> **The Ultimate Secret:**
>
> Trees look intimidating, but they're not. 90% of all tree problems are solved by answering ONE question:
>
> **"What should this function return to its parent so the parent can make the right decision?"**
>
> Once you answer that — the rest writes itself. Practice this thinking on every problem and you'll solve any tree problem in an interview.

---

*This guide covers 120+ LeetCode problems and every tree concept needed for any DSA interview — from basics to AVL, Trie, Segment Tree, and Fenwick Tree.*
