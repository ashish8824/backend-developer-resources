# 🔢 The Complete Matrix Problems Guide
## 2D Grid BFS/DFS Patterns — Crystal Clear from Zero to Interview-Ready — Java Edition

---

> **Goal:** By the end, you will understand every type of matrix/grid problem, know exactly when to use BFS vs DFS, and solve any LeetCode matrix problem confidently. Matrix problems trip people up because they look different on the surface but almost all follow the same 6 patterns underneath.

---

## Table of Contents

### Part 1 — Foundations
1. How to Think About a 2D Grid
2. The Direction Array — The Most Important Trick
3. Visited Tracking — 3 Ways
4. BFS vs DFS — When to Use Which
5. Common Matrix Pitfalls

### Part 2 — The 6 Matrix Patterns
6. Pattern 1 — Flood Fill / Island Counting (DFS/BFS)
7. Pattern 2 — Shortest Path in Grid (BFS)
8. Pattern 3 — Multi-Source BFS
9. Pattern 4 — Boundary DFS (Mark from edges inward)
10. Pattern 5 — Dynamic Programming on Grid
11. Pattern 6 — Matrix as Graph (Union-Find)

### Part 3 — Advanced Matrix Techniques
12. 0-1 BFS (Deque)
13. Dijkstra on Grid (Weighted cells)
14. Binary Search + BFS/DFS
15. In-Place Modification

### Part 4 — LeetCode Problems by Category
16. Easy Problems
17. Medium Problems
18. Hard Problems
19. Complete LeetCode Matrix Problem List

### Part 5 — Interview Mastery
20. How to Identify Which Pattern to Use
21. Matrix Interview Cheat Sheet

---

# Part 1 — Foundations

# 1. How to Think About a 2D Grid

## The Mental Model

```
A matrix (2D grid) is just a GRAPH where:
  - Each cell (row, col) is a NODE
  - Adjacent cells are CONNECTED by EDGES

int[][] grid = {
    {1, 0, 1},
    {1, 1, 0},
    {0, 1, 1}
};

Visualize it:
(0,0)=1 — (0,1)=0 — (0,2)=1
  |              |         |
(1,0)=1 — (1,1)=1 — (1,2)=0
  |              |         |
(2,0)=0 — (2,1)=1 — (2,2)=1

Each cell connects to up/down/left/right neighbors.
"1" cells = land, "0" cells = water (in island problems).
Once you see it as a graph, BFS/DFS apply directly.
```

## Indexing Rules

```java
int[][] grid = new int[m][n];
// m = number of rows    → grid.length
// n = number of columns → grid[0].length

// Cell at row r, col c:
grid[r][c]

// Valid cell check:
boolean inBounds(int r, int c, int m, int n) {
    return r >= 0 && r < m && c >= 0 && c < n;
}

// Row r contains: grid[r][0], grid[r][1], ..., grid[r][n-1]
// Col c contains: grid[0][c], grid[1][c], ..., grid[m-1][c]

// COMMON MISTAKE: confusing rows and columns
// grid[r][c] → first index is ROW, second is COLUMN
// When you move "right", column increases: grid[r][c+1]
// When you move "down",  row increases:    grid[r+1][c]
```

---

# 2. The Direction Array — The Most Important Trick

```java
// 4-directional movement (up, down, left, right)
int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};

// 8-directional movement (includes diagonals)
int[][] DIRS8 = {
    {-1,0},{1,0},{0,-1},{0,1},    // 4 cardinal
    {-1,-1},{-1,1},{1,-1},{1,1}   // 4 diagonal
};

// HOW TO USE:
for (int[] dir : DIRS) {
    int nr = r + dir[0];  // New row
    int nc = c + dir[1];  // New col
    if (nr >= 0 && nr < m && nc >= 0 && nc < n) {
        // (nr, nc) is a valid neighbor
        process(nr, nc);
    }
}

// EQUIVALENT but less clean (avoid this):
// if r > 0: process(r-1, c)  // up
// if r < m-1: process(r+1, c)  // down
// if c > 0: process(r, c-1)  // left
// if c < n-1: process(r, c+1)  // right

// WHY DIRECTION ARRAY?
// Cleaner code, easier to switch between 4/8 directions,
// less chance of missing a direction or having a typo.
```

## Complete BFS Template with Direction Array

```java
// BFS from a starting cell (r, c)
void bfs(int[][] grid, int startR, int startC) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    boolean[][] visited = new boolean[m][n];

    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[]{startR, startC});
    visited[startR][startC] = true;

    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        int r = curr[0], c = curr[1];

        // Process current cell
        System.out.println("Visiting: " + r + "," + c);

        // Explore neighbors
        for (int[] dir : DIRS) {
            int nr = r + dir[0];
            int nc = c + dir[1];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n && !visited[nr][nc]) {
                visited[nr][nc] = true;  // Mark BEFORE adding to queue
                queue.offer(new int[]{nr, nc});
            }
        }
    }
}
```

## Complete DFS Template with Direction Array

```java
// DFS from a starting cell (r, c) — recursive
void dfs(int[][] grid, int r, int c, boolean[][] visited) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};

    visited[r][c] = true;

    // Process current cell
    System.out.println("Visiting: " + r + "," + c);

    for (int[] dir : DIRS) {
        int nr = r + dir[0];
        int nc = c + dir[1];
        if (nr >= 0 && nr < m && nc >= 0 && nc < n && !visited[nr][nc]) {
            dfs(grid, nr, nc, visited);
        }
    }
}

// DFS — iterative (using stack instead of recursion)
void dfsIterative(int[][] grid, int startR, int startC) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    boolean[][] visited = new boolean[m][n];

    Deque<int[]> stack = new ArrayDeque<>();
    stack.push(new int[]{startR, startC});
    visited[startR][startC] = true;

    while (!stack.isEmpty()) {
        int[] curr = stack.pop();
        int r = curr[0], c = curr[1];

        for (int[] dir : DIRS) {
            int nr = r + dir[0];
            int nc = c + dir[1];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n && !visited[nr][nc]) {
                visited[nr][nc] = true;
                stack.push(new int[]{nr, nc});
            }
        }
    }
}
```

---

# 3. Visited Tracking — 3 Ways

## Method 1: Separate boolean[][] Array

```java
boolean[][] visited = new boolean[m][n];
// visited[r][c] = true means cell (r,c) has been seen

// PROS: Clean, doesn't modify input
// CONS: Extra O(m*n) space
// USE WHEN: Input shouldn't be modified, or need to restore
```

## Method 2: In-Place Modification (Mark and Restore)

```java
// For "1"/"0" grids: mark visited cell as "0" or '#' or '2'
grid[r][c] = '0';  // Mark as visited by changing value

// After DFS, optionally restore:
// grid[r][c] = '1';  // Restore (if needed for backtracking)

// PROS: O(1) extra space, very common in interviews
// CONS: Modifies input (may not be allowed)
// USE WHEN: Input can be modified (most island problems)
```

## Method 3: Set of Visited Coordinates

```java
Set<String> visited = new HashSet<>();
// visited.add(r + "," + c);
// visited.contains(r + "," + c)

// PROS: Works when grid cells aren't simple values
// CONS: String hashing has overhead
// USE WHEN: Grid values can't be modified, coordinates are key

// Better: encode as integer
Set<Integer> visited = new HashSet<>();
visited.add(r * n + c);  // Unique integer per cell
```

---

# 4. BFS vs DFS — When to Use Which

```
BFS (Breadth-First Search):
  → Use when you need SHORTEST PATH or MINIMUM STEPS
  → Explores level by level (distance 0, then 1, then 2...)
  → First time you reach a cell = shortest path to it
  → Uses QUEUE (FIFO)
  → Time: O(m*n), Space: O(m*n) for queue

DFS (Depth-First Search):
  → Use when you need to EXPLORE all cells in a connected component
  → "Visit everything reachable from here"
  → Good for: counting islands, flood fill, finding paths (not shortest)
  → Uses STACK (recursion or explicit)
  → Time: O(m*n), Space: O(m*n) for call stack

DECISION TABLE:
┌──────────────────────────────────┬───────────────┐
│ Problem Goal                     │ Algorithm     │
├──────────────────────────────────┼───────────────┤
│ Shortest path / minimum steps    │ BFS           │
│ Count connected components       │ DFS or BFS    │
│ Flood fill (paint region)        │ DFS or BFS    │
│ Detect cycle                     │ DFS           │
│ All cells reachable from source  │ DFS or BFS    │
│ Multi-source shortest distance   │ Multi-BFS     │
│ Path existence (not length)      │ DFS (simpler) │
│ DP on grid                       │ Neither (DP)  │
└──────────────────────────────────┴───────────────┘

GOLDEN RULE:
  "Shortest" or "minimum steps" → BFS
  "All" or "connected" → DFS
```

---

# 5. Common Matrix Pitfalls

## Pitfall 1: Visiting Cells Multiple Times

```java
// ❌ WRONG: Mark visited AFTER adding to queue → cells added multiple times
while (!queue.isEmpty()) {
    int[] curr = queue.poll();
    visited[curr[0]][curr[1]] = true;  // TOO LATE!
    for (int[] dir : DIRS) {
        int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
        if (inBounds(nr,nc) && !visited[nr][nc]) {
            queue.offer(new int[]{nr, nc});  // Same cell added many times
        }
    }
}

// ✅ CORRECT: Mark visited BEFORE or WHEN adding to queue
queue.offer(new int[]{startR, startC});
visited[startR][startC] = true;  // Mark immediately!
while (!queue.isEmpty()) {
    int[] curr = queue.poll();
    for (int[] dir : DIRS) {
        int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
        if (inBounds(nr,nc) && !visited[nr][nc]) {
            visited[nr][nc] = true;  // Mark BEFORE adding
            queue.offer(new int[]{nr, nc});
        }
    }
}
```

## Pitfall 2: Forgetting to Check Bounds

```java
// ❌ WRONG: ArrayIndexOutOfBoundsException
for (int[] dir : DIRS) {
    process(grid[r+dir[0]][c+dir[1]]);  // Crashes if out of bounds!
}

// ✅ CORRECT: Always check bounds first
for (int[] dir : DIRS) {
    int nr = r + dir[0], nc = c + dir[1];
    if (nr >= 0 && nr < m && nc >= 0 && nc < n) {
        process(grid[nr][nc]);
    }
}
```

## Pitfall 3: Counting Islands Off by One

```java
// ❌ WRONG: Starting DFS but not incrementing count
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++)
        if (grid[r][c] == '1') dfs(grid, r, c);  // Forgot to count!

// ✅ CORRECT
int count = 0;
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++)
        if (grid[r][c] == '1') { dfs(grid, r, c); count++; }
```

## Pitfall 4: Stack Overflow on Large Grids (DFS)

```java
// For very large grids, recursive DFS may cause StackOverflowError.
// Convert to iterative DFS using explicit stack.
// Or use BFS (no recursion).

// Java default stack depth: ~500-1000 recursive calls
// Grid 500x500 = 250,000 cells → definitely need iterative
```

---

# Part 2 — The 6 Matrix Patterns

# 6. Pattern 1 — Flood Fill / Island Counting (DFS/BFS)

## The Core Idea

```
ISLAND COUNTING: Scan every cell. When you find an unvisited "land" cell,
  start DFS/BFS to mark all connected land cells as visited.
  Each DFS/BFS call = one island.

FLOOD FILL: Given a starting cell, paint all connected cells of the
  same color with a new color.

KEY STEPS:
  1. Outer loop: scan all cells
  2. If cell qualifies (land, unvisited, target color):
     - Increment count (if counting)
     - DFS/BFS to mark entire component
```

---

## 🟡 LC #200 — Number of Islands

```java
public int numIslands(char[][] grid) {
    int m = grid.length, n = grid[0].length;
    int count = 0;

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == '1') {
                dfs(grid, r, c, m, n);
                count++;
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int r, int c, int m, int n) {
    // Base cases: out of bounds OR water OR already visited
    if (r < 0 || r >= m || c < 0 || c >= n || grid[r][c] != '1') return;

    grid[r][c] = '0';  // Mark as visited (sink the island)

    // Explore all 4 directions
    dfs(grid, r-1, c, m, n);  // Up
    dfs(grid, r+1, c, m, n);  // Down
    dfs(grid, r, c-1, m, n);  // Left
    dfs(grid, r, c+1, m, n);  // Right
}
```

**Dry run:**
```
grid:
1 1 0 0
1 1 0 0
0 0 1 0
0 0 0 1

Scan r=0,c=0: '1' found → DFS sinks all connected '1's
  Sinks (0,0),(0,1),(1,0),(1,1). count=1
Scan r=2,c=2: '1' found → DFS sinks (2,2). count=2
Scan r=3,c=3: '1' found → DFS sinks (3,3). count=3

Answer: 3 islands ✓
```

---

## 🟢 LC #733 — Flood Fill

```java
public int[][] floodFill(int[][] image, int sr, int sc, int color) {
    int oldColor = image[sr][sc];
    if (oldColor == color) return image;  // Already the target color, avoid infinite loop
    dfs(image, sr, sc, oldColor, color);
    return image;
}

private void dfs(int[][] image, int r, int c, int oldColor, int newColor) {
    int m = image.length, n = image[0].length;
    if (r < 0 || r >= m || c < 0 || c >= n) return;    // Out of bounds
    if (image[r][c] != oldColor) return;                 // Different color

    image[r][c] = newColor;  // Paint this cell

    dfs(image, r-1, c, oldColor, newColor);
    dfs(image, r+1, c, oldColor, newColor);
    dfs(image, r, c-1, oldColor, newColor);
    dfs(image, r, c+1, oldColor, newColor);
}
```

---

## 🟡 LC #695 — Max Area of Island

```java
public int maxAreaOfIsland(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int maxArea = 0;

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == 1) {
                maxArea = Math.max(maxArea, dfs(grid, r, c, m, n));
            }
        }
    }
    return maxArea;
}

private int dfs(int[][] grid, int r, int c, int m, int n) {
    if (r < 0 || r >= m || c < 0 || c >= n || grid[r][c] != 1) return 0;

    grid[r][c] = 0;  // Mark visited

    return 1 + dfs(grid, r-1, c, m, n)
             + dfs(grid, r+1, c, m, n)
             + dfs(grid, r, c-1, m, n)
             + dfs(grid, r, c+1, m, n);
}
```

---

## 🟡 LC #130 — Surrounded Regions (Boundary DFS — preview)

```java
// Cells not connected to border 'O's get flipped to 'X'
// (Full solution in Pattern 4 below)
public void solve(char[][] board) {
    int m = board.length, n = board[0].length;
    // Mark all 'O's connected to border as safe ('S')
    for (int r = 0; r < m; r++) {
        if (board[r][0] == 'O') dfs(board, r, 0, m, n);
        if (board[r][n-1] == 'O') dfs(board, r, n-1, m, n);
    }
    for (int c = 0; c < n; c++) {
        if (board[0][c] == 'O') dfs(board, 0, c, m, n);
        if (board[m-1][c] == 'O') dfs(board, m-1, c, m, n);
    }
    // Flip: 'O' → 'X' (surrounded), 'S' → 'O' (restore safe)
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            board[r][c] = board[r][c] == 'S' ? 'O' : 'X';
}

private void dfs(char[][] board, int r, int c, int m, int n) {
    if (r < 0 || r >= m || c < 0 || c >= n || board[r][c] != 'O') return;
    board[r][c] = 'S';  // Mark as safe
    dfs(board, r-1, c, m, n);
    dfs(board, r+1, c, m, n);
    dfs(board, r, c-1, m, n);
    dfs(board, r, c+1, m, n);
}
```

---

## 🟡 LC #463 — Island Perimeter

```java
// No DFS needed! Count edges exposed to water or border.
public int islandPerimeter(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int perimeter = 0;

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == 1) {
                perimeter += 4;  // Start with 4 sides
                // Subtract shared edges with adjacent land cells
                if (r > 0 && grid[r-1][c] == 1) perimeter -= 2;  // Shared with above
                if (c > 0 && grid[r][c-1] == 1) perimeter -= 2;  // Shared with left
            }
        }
    }
    return perimeter;
}
```

---

## 🟡 LC #1905 — Count Sub Islands

```java
// Island in grid2 is sub-island if all its cells are also land in grid1
public int countSubIslands(int[][] grid1, int[][] grid2) {
    int m = grid2.length, n = grid2[0].length, count = 0;

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid2[r][c] == 1) {
                // DFS returns true only if entire island in grid2 is in grid1
                if (dfs(grid1, grid2, r, c, m, n)) count++;
            }
        }
    }
    return count;
}

private boolean dfs(int[][] grid1, int[][] grid2, int r, int c, int m, int n) {
    if (r < 0 || r >= m || c < 0 || c >= n || grid2[r][c] == 0) return true;

    grid2[r][c] = 0;  // Mark visited

    boolean isSub = grid1[r][c] == 1;  // This cell must be land in grid1

    // Must explore ALL directions even if one fails (to mark entire island)
    boolean up    = dfs(grid1, grid2, r-1, c, m, n);
    boolean down  = dfs(grid1, grid2, r+1, c, m, n);
    boolean left  = dfs(grid1, grid2, r, c-1, m, n);
    boolean right = dfs(grid1, grid2, r, c+1, m, n);

    return isSub && up && down && left && right;
}
```

---

# 7. Pattern 2 — Shortest Path in Grid (BFS)

## The Core Idea

```
BFS finds the shortest path in an UNWEIGHTED graph.
In a grid where each step costs 1:
  - Start BFS from source cell
  - Track distance (level in BFS)
  - First time we reach destination = shortest path

KEY PROPERTY OF BFS:
  When you first visit a cell via BFS, you have found
  the MINIMUM number of steps to reach it.

  Level 0: source
  Level 1: all cells 1 step away
  Level 2: all cells 2 steps away
  ...

TEMPLATE:
  queue.offer(start), visited[start]=true, dist[start]=0
  while queue not empty:
    curr = queue.poll()
    for each neighbor:
      if not visited:
        dist[neighbor] = dist[curr] + 1
        if neighbor == target: return dist[neighbor]
        visited[neighbor] = true
        queue.offer(neighbor)
  return -1  // No path
```

---

## 🟡 LC #994 — Rotting Oranges

```java
// Fresh oranges rot when adjacent to rotten ones. How many minutes until all rot?
public int orangesRotting(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    Queue<int[]> queue = new LinkedList<>();
    int fresh = 0;

    // Initialize: add all rotten oranges to queue (multi-source BFS!)
    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == 2) queue.offer(new int[]{r, c});
            else if (grid[r][c] == 1) fresh++;
        }
    }

    if (fresh == 0) return 0;  // No fresh oranges

    int minutes = 0;
    while (!queue.isEmpty() && fresh > 0) {
        minutes++;
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] curr = queue.poll();
            for (int[] dir : DIRS) {
                int nr = curr[0] + dir[0], nc = curr[1] + dir[1];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2;  // Rot the orange
                    fresh--;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
    }

    return fresh == 0 ? minutes : -1;
}
```

**Dry run:**
```
Grid:
2 1 1
1 1 0
0 1 1

Initial: queue=[(0,0)], fresh=6, minutes=0

Minute 1: rot (0,1),(1,0). fresh=4. queue=[(0,1),(1,0)]
Minute 2: from (0,1): rot (0,2). from (1,0): rot (1,1). fresh=2. queue=[(0,2),(1,1)]
Minute 3: from (1,1): rot (2,1). from (0,2): nothing new. fresh=1. queue=[(2,1)]
Minute 4: from (2,1): rot (2,2). fresh=0. done.

Answer: 4 minutes ✓
```

---

## 🟡 LC #1091 — Shortest Path in Binary Matrix

```java
// Shortest path from (0,0) to (n-1,n-1) in binary grid (0=open, 1=blocked)
public int shortestPathBinaryMatrix(int[][] grid) {
    int n = grid.length;
    if (grid[0][0] == 1 || grid[n-1][n-1] == 1) return -1;
    if (n == 1) return 1;

    int[][] DIRS = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}}; // 8 directions!
    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[]{0, 0, 1});  // row, col, distance
    grid[0][0] = 1;  // Mark visited by setting to 1 (blocked)

    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        int r = curr[0], c = curr[1], dist = curr[2];

        for (int[] dir : DIRS) {
            int nr = r + dir[0], nc = c + dir[1];
            if (nr >= 0 && nr < n && nc >= 0 && nc < n && grid[nr][nc] == 0) {
                if (nr == n-1 && nc == n-1) return dist + 1;
                grid[nr][nc] = 1;  // Mark visited
                queue.offer(new int[]{nr, nc, dist + 1});
            }
        }
    }
    return -1;
}
```

---

## 🟡 LC #127 — Word Ladder

```java
// Find shortest transformation sequence from beginWord to endWord
// Each step: change exactly one letter, result must be in wordList
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;

    Queue<String> queue = new LinkedList<>();
    queue.offer(beginWord);
    wordSet.remove(beginWord);
    int steps = 1;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            String curr = queue.poll();
            char[] arr = curr.toCharArray();

            for (int j = 0; j < arr.length; j++) {
                char orig = arr[j];
                for (char ch = 'a'; ch <= 'z'; ch++) {
                    if (ch == orig) continue;
                    arr[j] = ch;
                    String next = new String(arr);
                    if (next.equals(endWord)) return steps + 1;
                    if (wordSet.contains(next)) {
                        wordSet.remove(next);
                        queue.offer(next);
                    }
                }
                arr[j] = orig;
            }
        }
        steps++;
    }
    return 0;
}
```

---

## 🟡 LC #1293 — Shortest Path with Obstacles Elimination

```java
// Shortest path from (0,0) to (m-1,n-1). Can eliminate k obstacles.
public int shortestPath(int[][] grid, int k) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};

    // State: (row, col, remainingEliminations)
    // visited[r][c][k] = true if (r,c) visited with k eliminations remaining
    boolean[][][] visited = new boolean[m][n][k+1];
    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[]{0, 0, k, 0});  // row, col, remaining k, steps
    visited[0][0][k] = true;

    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        int r = curr[0], c = curr[1], rem = curr[2], steps = curr[3];

        if (r == m-1 && c == n-1) return steps;

        for (int[] dir : DIRS) {
            int nr = r + dir[0], nc = c + dir[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;

            int newRem = rem - grid[nr][nc];  // If obstacle: rem-1, else rem
            if (newRem >= 0 && !visited[nr][nc][newRem]) {
                visited[nr][nc][newRem] = true;
                queue.offer(new int[]{nr, nc, newRem, steps + 1});
            }
        }
    }
    return -1;
}
```

---

# 8. Pattern 3 — Multi-Source BFS

## The Core Idea

```
PROBLEM: Find minimum distance from ANY of multiple source cells
         to every other cell.

NAIVE: Run BFS from each source separately → O(sources * m * n)

BETTER (Multi-Source BFS): Add ALL sources to queue simultaneously.
  BFS expands all sources at once, level by level.
  First time any cell is visited = minimum distance to nearest source.

EXAMPLES:
  - Nearest distance to any rotten orange (LC #994)
  - Distance from any gate in a dungeon (LC #286)
  - Distance from any land cell to nearest ocean (LC #1162)

TEMPLATE:
  for each source cell: queue.offer(source), visited[source]=true
  Then run regular BFS — exactly the same as single-source!
```

---

## 🟡 LC #286 — Walls and Gates

```java
// Fill dungeon with distance to nearest gate
// -1=wall, 0=gate, INF=empty room
public void wallsAndGates(int[][] rooms) {
    int m = rooms.length, n = rooms[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    int INF = Integer.MAX_VALUE;
    Queue<int[]> queue = new LinkedList<>();

    // Multi-source: add ALL gates to queue
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (rooms[r][c] == 0) queue.offer(new int[]{r, c});

    // BFS expands from all gates simultaneously
    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        for (int[] dir : DIRS) {
            int nr = curr[0] + dir[0], nc = curr[1] + dir[1];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n && rooms[nr][nc] == INF) {
                rooms[nr][nc] = rooms[curr[0]][curr[1]] + 1;  // Distance
                queue.offer(new int[]{nr, nc});
            }
        }
    }
}
```

---

## 🟡 LC #1162 — As Far from Land as Possible

```java
// Find ocean cell with maximum distance to nearest land cell
public int maxDistance(int[][] grid) {
    int n = grid.length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    Queue<int[]> queue = new LinkedList<>();

    // Multi-source BFS from all land cells
    for (int r = 0; r < n; r++)
        for (int c = 0; c < n; c++)
            if (grid[r][c] == 1) queue.offer(new int[]{r, c});

    if (queue.size() == 0 || queue.size() == n*n) return -1;  // All land or all water

    int maxDist = -1;
    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        for (int[] dir : DIRS) {
            int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
            if (nr >= 0 && nr < n && nc >= 0 && nc < n && grid[nr][nc] == 0) {
                grid[nr][nc] = grid[curr[0]][curr[1]] + 1;  // Distance from land
                maxDist = Math.max(maxDist, grid[nr][nc] - 1);
                queue.offer(new int[]{nr, nc});
            }
        }
    }
    return maxDist;
}
```

---

## 🟡 LC #542 — 01 Matrix (Distance to Nearest 0)

```java
// For each cell, find distance to nearest 0
public int[][] updateMatrix(int[][] mat) {
    int m = mat.length, n = mat[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    int[][] dist = new int[m][n];
    Queue<int[]> queue = new LinkedList<>();

    // Initialize: 0-cells have distance 0, add to queue. 1-cells = infinity.
    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (mat[r][c] == 0) {
                queue.offer(new int[]{r, c});
            } else {
                dist[r][c] = Integer.MAX_VALUE;  // Unknown distance
            }
        }
    }

    // Multi-source BFS from all 0-cells
    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        for (int[] dir : DIRS) {
            int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n) {
                if (dist[nr][nc] > dist[curr[0]][curr[1]] + 1) {
                    dist[nr][nc] = dist[curr[0]][curr[1]] + 1;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
    }
    return dist;
}
```

---

# 9. Pattern 4 — Boundary DFS (Mark from Edges Inward)

## The Core Idea

```
PROBLEM TYPE: "Find cells that CANNOT reach the boundary"
              OR "Find cells connected to the border"

APPROACH: Instead of checking FROM each cell TO boundary (expensive),
  start FROM the boundary and mark everything reachable.
  Cells NOT marked = answer.

CLASSIC EXAMPLE: LC #130 — Surrounded Regions
  'O' cells surrounded by 'X' on all 4 sides → flip to 'X'
  'O' cells connected to border → keep

STEPS:
  1. DFS/BFS from all BORDER cells that qualify
  2. Mark all reachable cells as "safe"
  3. Scan entire grid: unseen qualifiers → answer, safe → restore

WHY THIS WORKS:
  Any cell reachable from border = NOT surrounded.
  Any cell NOT reachable from border = surrounded = answer.
```

---

## 🟡 LC #130 — Surrounded Regions

```java
public void solve(char[][] board) {
    int m = board.length, n = board[0].length;

    // Step 1: DFS from all border 'O' cells, mark as 'S' (safe)
    for (int r = 0; r < m; r++) {
        dfs(board, r, 0, m, n);      // Left border
        dfs(board, r, n-1, m, n);   // Right border
    }
    for (int c = 0; c < n; c++) {
        dfs(board, 0, c, m, n);      // Top border
        dfs(board, m-1, c, m, n);   // Bottom border
    }

    // Step 2: Convert
    // 'O' → 'X' (surrounded, not safe)
    // 'S' → 'O' (restore safe cells)
    // 'X' → 'X' (already correct)
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            board[r][c] = board[r][c] == 'S' ? 'O' : 'X';
}

private void dfs(char[][] board, int r, int c, int m, int n) {
    if (r < 0 || r >= m || c < 0 || c >= n || board[r][c] != 'O') return;
    board[r][c] = 'S';  // Mark safe
    dfs(board, r-1, c, m, n);
    dfs(board, r+1, c, m, n);
    dfs(board, r, c-1, m, n);
    dfs(board, r, c+1, m, n);
}
```

**Dry run:**
```
Before:              After boundary DFS:    Final:
X X X X              X X X X               X X X X
X O O X              X S S X               X X X X
X X O X     →        X X S X      →        X X X X
X O X X              X O X X               X O X X
                     (O on row 3 col 1      (bottom-connected O stays)
                      is border-connected)
```

---

## 🟡 LC #417 — Pacific Atlantic Water Flow

```java
// Water can flow to adjacent cells with height <= current cell
// Find cells that can flow to BOTH Pacific and Atlantic oceans
// Pacific: top/left borders; Atlantic: bottom/right borders
public List<List<Integer>> pacificAtlantic(int[][] heights) {
    int m = heights.length, n = heights[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    boolean[][] pacific  = new boolean[m][n];
    boolean[][] atlantic = new boolean[m][n];

    Queue<int[]> pacQ = new LinkedList<>();
    Queue<int[]> atlQ = new LinkedList<>();

    // Seeds: border cells for each ocean
    for (int r = 0; r < m; r++) {
        pacQ.offer(new int[]{r, 0});     pacific[r][0] = true;    // Left border → Pacific
        atlQ.offer(new int[]{r, n-1});   atlantic[r][n-1] = true; // Right border → Atlantic
    }
    for (int c = 0; c < n; c++) {
        pacQ.offer(new int[]{0, c});     pacific[0][c] = true;    // Top → Pacific
        atlQ.offer(new int[]{m-1, c});   atlantic[m-1][c] = true; // Bottom → Atlantic
    }

    // BFS "uphill" from each ocean border (reverse flow direction)
    bfsOcean(heights, pacQ, pacific, m, n, DIRS);
    bfsOcean(heights, atlQ, atlantic, m, n, DIRS);

    // Cells reachable from both oceans
    List<List<Integer>> result = new ArrayList<>();
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (pacific[r][c] && atlantic[r][c])
                result.add(Arrays.asList(r, c));
    return result;
}

private void bfsOcean(int[][] h, Queue<int[]> q, boolean[][] visited, int m, int n, int[][] DIRS) {
    while (!q.isEmpty()) {
        int[] curr = q.poll();
        for (int[] dir : DIRS) {
            int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
            if (nr>=0 && nr<m && nc>=0 && nc<n && !visited[nr][nc]
                && h[nr][nc] >= h[curr[0]][curr[1]]) {  // Can flow from nr,nc to curr
                visited[nr][nc] = true;
                q.offer(new int[]{nr, nc});
            }
        }
    }
}
```

---

## 🟡 LC #1020 — Number of Enclaves

```java
// Count land cells that cannot reach border
public int numEnclaves(int[][] grid) {
    int m = grid.length, n = grid[0].length;

    // Step 1: DFS from border land cells, mark them 0
    for (int r = 0; r < m; r++) {
        dfs(grid, r, 0, m, n);
        dfs(grid, r, n-1, m, n);
    }
    for (int c = 0; c < n; c++) {
        dfs(grid, 0, c, m, n);
        dfs(grid, m-1, c, m, n);
    }

    // Step 2: Count remaining land cells (enclosed)
    int count = 0;
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (grid[r][c] == 1) count++;
    return count;
}

private void dfs(int[][] grid, int r, int c, int m, int n) {
    if (r<0||r>=m||c<0||c>=n||grid[r][c]!=1) return;
    grid[r][c] = 0;
    dfs(grid,r-1,c,m,n); dfs(grid,r+1,c,m,n);
    dfs(grid,r,c-1,m,n); dfs(grid,r,c+1,m,n);
}
```

---

# 10. Pattern 5 — Dynamic Programming on Grid

## The Core Idea

```
DP on grids: dp[r][c] = answer at cell (r, c) based on neighboring cells.

COMMON PATTERNS:
  dp[r][c] = dp[r-1][c] + dp[r][c-1]  (paths from top-left)
  dp[r][c] = min(dp[r-1][c], dp[r][c-1]) + cost[r][c]  (min cost path)
  dp[r][c] = 1 + max(neighbors)  (longest path in increasing order)

WHEN TO USE DP vs BFS:
  BFS: unweighted shortest path, all edges cost 1
  DP:  when answer at each cell depends ONLY on cells already computed
       (usually top and left, or when grid has DAG structure)
       Also: counting paths (BFS can't count all paths efficiently)

KEY INSIGHT FOR DP:
  Process cells in the ORDER that guarantees dependencies are resolved.
  - Top-to-bottom, left-to-right: dp depends on [r-1][c] and [r][c-1]
  - Bottom-to-top: dp depends on [r+1][c]
  - Topological order for complex dependencies
```

---

## 🟢 LC #62 — Unique Paths

```java
// Count unique paths from (0,0) to (m-1,n-1), moving only right/down
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];

    // First row and column: only one way to reach each cell
    for (int r = 0; r < m; r++) dp[r][0] = 1;
    for (int c = 0; c < n; c++) dp[0][c] = 1;

    // Fill rest: paths from top + paths from left
    for (int r = 1; r < m; r++)
        for (int c = 1; c < n; c++)
            dp[r][c] = dp[r-1][c] + dp[r][c-1];

    return dp[m-1][n-1];
}
```

---

## 🟡 LC #64 — Minimum Path Sum

```java
// Find minimum cost path from (0,0) to (m-1,n-1), moving right/down only
public int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];
    dp[0][0] = grid[0][0];

    // First row: can only come from left
    for (int c = 1; c < n; c++) dp[0][c] = dp[0][c-1] + grid[0][c];
    // First col: can only come from above
    for (int r = 1; r < m; r++) dp[r][0] = dp[r-1][0] + grid[r][0];

    // Fill rest: min of top or left + current cost
    for (int r = 1; r < m; r++)
        for (int c = 1; c < n; c++)
            dp[r][c] = Math.min(dp[r-1][c], dp[r][c-1]) + grid[r][c];

    return dp[m-1][n-1];
}
```

---

## 🟡 LC #63 — Unique Paths II (with obstacles)

```java
public int uniquePathsWithObstacles(int[][] obstacleGrid) {
    int m = obstacleGrid.length, n = obstacleGrid[0].length;
    if (obstacleGrid[0][0] == 1) return 0;

    int[][] dp = new int[m][n];
    dp[0][0] = 1;

    // First column: blocked by obstacle
    for (int r = 1; r < m; r++)
        dp[r][0] = (obstacleGrid[r][0] == 0 && dp[r-1][0] == 1) ? 1 : 0;
    // First row: blocked by obstacle
    for (int c = 1; c < n; c++)
        dp[0][c] = (obstacleGrid[0][c] == 0 && dp[0][c-1] == 1) ? 1 : 0;

    for (int r = 1; r < m; r++)
        for (int c = 1; c < n; c++)
            dp[r][c] = obstacleGrid[r][c] == 1 ? 0 : dp[r-1][c] + dp[r][c-1];

    return dp[m-1][n-1];
}
```

---

## 🟡 LC #221 — Maximal Square

```java
// Find the largest square containing only 1s. Return its area.
public int maximalSquare(char[][] matrix) {
    int m = matrix.length, n = matrix[0].length, maxSide = 0;
    int[][] dp = new int[m][n];

    // dp[r][c] = side length of largest square with bottom-right corner at (r,c)
    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (matrix[r][c] == '0') continue;

            if (r == 0 || c == 0) {
                dp[r][c] = 1;
            } else {
                // Largest square limited by smallest of 3 neighbors
                dp[r][c] = Math.min(dp[r-1][c], Math.min(dp[r][c-1], dp[r-1][c-1])) + 1;
            }
            maxSide = Math.max(maxSide, dp[r][c]);
        }
    }
    return maxSide * maxSide;
}
```

**Why min of 3 neighbors?**
```
dp[r-1][c-1] dp[r-1][c]
dp[r][c-1]   dp[r][c] ← we're filling this

For dp[r][c] to be k, we need:
  dp[r-1][c]   >= k-1 (square above has side k-1)
  dp[r][c-1]   >= k-1 (square left has side k-1)
  dp[r-1][c-1] >= k-1 (square diagonal has side k-1)
All three together form a k×k square.
So dp[r][c] = min of three + 1.
```

---

## 🟡 LC #329 — Longest Increasing Path in a Matrix

```java
// Find longest strictly increasing path in matrix (any direction)
// Use DFS + memoization (matrix has DAG structure when strictly increasing)
public int longestIncreasingPath(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] memo = new int[m][n];
    int maxLen = 0;

    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            maxLen = Math.max(maxLen, dfs(matrix, r, c, m, n, memo));

    return maxLen;
}

int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};

private int dfs(int[][] mat, int r, int c, int m, int n, int[][] memo) {
    if (memo[r][c] != 0) return memo[r][c];  // Already computed

    int maxPath = 1;
    for (int[] dir : DIRS) {
        int nr = r+dir[0], nc = c+dir[1];
        if (nr>=0 && nr<m && nc>=0 && nc<n && mat[nr][nc] > mat[r][c]) {
            maxPath = Math.max(maxPath, 1 + dfs(mat, nr, nc, m, n, memo));
        }
    }
    return memo[r][c] = maxPath;
}
```

**Why no visited array needed?** Because we only move to STRICTLY LARGER values, we can never revisit a cell. No cycles possible.

---

## 🔴 LC #174 — Dungeon Game

```java
// Knight starts at (0,0), must reach (m-1,n-1) alive.
// Find minimum initial health.
// Fill from bottom-right to top-left (reverse DP)
public int calculateMinimumHP(int[][] dungeon) {
    int m = dungeon.length, n = dungeon[0].length;
    int[][] dp = new int[m][n];  // dp[r][c] = min health needed entering (r,c)

    // Start from destination, work backwards
    for (int r = m-1; r >= 0; r--) {
        for (int c = n-1; c >= 0; c--) {
            if (r == m-1 && c == n-1) {
                dp[r][c] = Math.max(1, 1 - dungeon[r][c]);
            } else if (r == m-1) {
                dp[r][c] = Math.max(1, dp[r][c+1] - dungeon[r][c]);
            } else if (c == n-1) {
                dp[r][c] = Math.max(1, dp[r+1][c] - dungeon[r][c]);
            } else {
                int minNext = Math.min(dp[r+1][c], dp[r][c+1]);
                dp[r][c] = Math.max(1, minNext - dungeon[r][c]);
            }
        }
    }
    return dp[0][0];
}
```

---

# 11. Pattern 6 — Matrix as Graph (Union-Find)

## When to Use Union-Find on Grid

```
USE WHEN:
  - Dynamically connecting cells (cells added over time)
  - Need to know if two cells are in the same component
  - Number of components changes as cells are added

CONVERT GRID TO UNION-FIND:
  Each cell (r, c) → node ID = r * n + c
  Connect adjacent land cells

CLASSIC: LC #305 — Number of Islands II
  Cells become land one by one. After each, report number of islands.
```

```java
class UnionFind {
    int[] parent, rank;
    int count;

    UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);  // Path compression
        return parent[x];
    }

    boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;
        if (rank[px] < rank[py]) { int t=px; px=py; py=t; }
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
        count--;
        return true;
    }
}
```

---

## 🔴 LC #305 — Number of Islands II

```java
public List<Integer> numIslands2(int m, int n, int[][] positions) {
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    boolean[][] land = new boolean[m][n];
    UnionFind uf = new UnionFind(m * n);
    List<Integer> result = new ArrayList<>();

    for (int[] pos : positions) {
        int r = pos[0], c = pos[1];
        if (!land[r][c]) {
            land[r][c] = true;
            uf.count++;  // New island

            for (int[] dir : DIRS) {
                int nr = r+dir[0], nc = c+dir[1];
                if (nr>=0&&nr<m&&nc>=0&&nc<n&&land[nr][nc]) {
                    uf.union(r*n+c, nr*n+nc);  // Merge if adjacent land
                }
            }
        }
        result.add(uf.count);
    }
    return result;
}
```

---

## 🟡 LC #990 — Satisfiability of Equality Equations

```java
// a==b: merge a and b. a!=b: check if a and b are already merged.
public boolean equationsPossible(String[] equations) {
    int[] parent = new int[26];
    for (int i = 0; i < 26; i++) parent[i] = i;

    // Process all == equations first
    for (String eq : equations) {
        if (eq.charAt(1) == '=') {
            int a = eq.charAt(0)-'a', b = eq.charAt(3)-'a';
            union(parent, a, b);
        }
    }
    // Then check all != equations
    for (String eq : equations) {
        if (eq.charAt(1) == '!') {
            int a = eq.charAt(0)-'a', b = eq.charAt(3)-'a';
            if (find(parent, a) == find(parent, b)) return false;
        }
    }
    return true;
}

int find(int[] p, int x) { return p[x]==x ? x : (p[x]=find(p,p[x])); }
void union(int[] p, int x, int y) { p[find(p,x)] = find(p,y); }
```

---

# Part 3 — Advanced Matrix Techniques

# 12. 0-1 BFS (Deque)

```
PROBLEM: Grid where edge weights are 0 or 1 (not all 1).
NAIVE BFS: Wrong — BFS assumes all edges cost 1.
DIJKSTRA: Works but O(m*n*log(m*n)) — overkill.
0-1 BFS: O(m*n) using deque!

IDEA:
  Cost 0 edge → add to FRONT of deque (same "level")
  Cost 1 edge → add to BACK of deque (next "level")
  Always process minimum cost cell first (front of deque).

USE CASES:
  - Move with cost 0 or 1
  - "Can pass through k walls for free"
  - Cell values 0=free, 1=cost
```

```java
// LC #1368 — Minimum Cost to Make at Least One Valid Path
// Grid cells have directions. Override direction costs 1. Following costs 0.
public int minCost(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{0,1},{0,-1},{1,0},{-1,0}};  // Right, Left, Down, Up (1,2,3,4)
    int[][] dist = new int[m][n];
    for (int[] row : dist) Arrays.fill(row, Integer.MAX_VALUE);
    dist[0][0] = 0;

    Deque<int[]> deque = new ArrayDeque<>();
    deque.offerFirst(new int[]{0, 0});

    while (!deque.isEmpty()) {
        int[] curr = deque.pollFirst();
        int r = curr[0], c = curr[1];

        for (int d = 0; d < 4; d++) {
            int nr = r+DIRS[d][0], nc = c+DIRS[d][1];
            if (nr<0||nr>=m||nc<0||nc>=n) continue;

            // Cost 0 if following grid direction, else cost 1
            int cost = (grid[r][c]-1 == d) ? 0 : 1;
            if (dist[r][c] + cost < dist[nr][nc]) {
                dist[nr][nc] = dist[r][c] + cost;
                if (cost == 0) deque.offerFirst(new int[]{nr, nc});
                else           deque.offerLast(new int[]{nr, nc});
            }
        }
    }
    return dist[m-1][n-1];
}
```

---

# 13. Dijkstra on Grid (Weighted Cells)

```
PROBLEM: Grid cells have different costs to enter. Find minimum cost path.
USE: Dijkstra with min-heap of (cost, row, col).

WHY NOT BFS: BFS only works when all edges have equal cost.
             Here cell weights differ, so BFS gives wrong answer.
```

```java
// LC #1631 — Path With Minimum Effort
// Effort = maximum absolute difference of adjacent cells on path. Minimize effort.
public int minimumEffortPath(int[][] heights) {
    int m = heights.length, n = heights[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    int[][] effort = new int[m][n];
    for (int[] row : effort) Arrays.fill(row, Integer.MAX_VALUE);
    effort[0][0] = 0;

    // Min-heap: (effort, row, col)
    PriorityQueue<int[]> pq = new PriorityQueue<>((a,b) -> a[0]-b[0]);
    pq.offer(new int[]{0, 0, 0});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int e = curr[0], r = curr[1], c = curr[2];

        if (r == m-1 && c == n-1) return e;
        if (e > effort[r][c]) continue;  // Outdated entry

        for (int[] dir : DIRS) {
            int nr = r+dir[0], nc = c+dir[1];
            if (nr<0||nr>=m||nc<0||nc>=n) continue;

            int newEffort = Math.max(e, Math.abs(heights[nr][nc]-heights[r][c]));
            if (newEffort < effort[nr][nc]) {
                effort[nr][nc] = newEffort;
                pq.offer(new int[]{newEffort, nr, nc});
            }
        }
    }
    return effort[m-1][n-1];
}
```

---

## 🟡 LC #778 — Swim in Rising Water

```java
// Grid values = time when water rises to that cell
// Find minimum time to swim from (0,0) to (n-1,n-1)
public int swimInWater(int[][] grid) {
    int n = grid.length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    int[][] minTime = new int[n][n];
    for (int[] row : minTime) Arrays.fill(row, Integer.MAX_VALUE);
    minTime[0][0] = grid[0][0];

    PriorityQueue<int[]> pq = new PriorityQueue<>((a,b) -> a[0]-b[0]);
    pq.offer(new int[]{grid[0][0], 0, 0});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int t = curr[0], r = curr[1], c = curr[2];
        if (r==n-1 && c==n-1) return t;
        if (t > minTime[r][c]) continue;

        for (int[] dir : DIRS) {
            int nr = r+dir[0], nc = c+dir[1];
            if (nr<0||nr>=n||nc<0||nc>=n) continue;
            int newTime = Math.max(t, grid[nr][nc]);
            if (newTime < minTime[nr][nc]) {
                minTime[nr][nc] = newTime;
                pq.offer(new int[]{newTime, nr, nc});
            }
        }
    }
    return minTime[n-1][n-1];
}
```

---

# 14. In-Place Modification

```
TRICK: Use the grid itself to store state, avoiding extra space.
Common modifications:
  '1' → '0' or '#' or '2': mark as visited
  Negate value: grid[r][c] = -grid[r][c] (mark without losing value)
  
WHEN TO RESTORE:
  Backtracking: restore after recursive call
  No restore needed: islands (visited cells stay modified)

ENCODING MULTIPLE STATES:
  If original values are {0,1}: use '2' for visited
  If original values are positive: negate to mark visited
```

```java
// LC #73 — Set Matrix Zeroes
// If element is 0, set its entire row and column to 0
// O(1) space using first row/col as markers
public void setZeroes(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    boolean firstRowZero = false, firstColZero = false;

    // Check if first row/col should be zeroed
    for (int c = 0; c < n; c++) if (matrix[0][c] == 0) firstRowZero = true;
    for (int r = 0; r < m; r++) if (matrix[r][0] == 0) firstColZero = true;

    // Use first row and col as markers
    for (int r = 1; r < m; r++)
        for (int c = 1; c < n; c++)
            if (matrix[r][c] == 0) { matrix[r][0] = 0; matrix[0][c] = 0; }

    // Zero out marked rows and cols
    for (int r = 1; r < m; r++)
        for (int c = 1; c < n; c++)
            if (matrix[r][0] == 0 || matrix[0][c] == 0) matrix[r][c] = 0;

    // Handle first row and col
    if (firstRowZero) for (int c = 0; c < n; c++) matrix[0][c] = 0;
    if (firstColZero) for (int r = 0; r < m; r++) matrix[r][0] = 0;
}
```

---

# Part 4 — LeetCode Problems

# 15. Easy Problems

## 🟢 LC #566 — Reshape the Matrix

```java
public int[][] matrixReshape(int[][] mat, int r, int c) {
    int m = mat.length, n = mat[0].length;
    if (m * n != r * c) return mat;

    int[][] result = new int[r][c];
    for (int i = 0; i < m * n; i++) {
        result[i / c][i % c] = mat[i / n][i % n];
    }
    return result;
}
```

---

## 🟢 LC #867 — Transpose Matrix

```java
public int[][] transpose(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] result = new int[n][m];
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            result[c][r] = matrix[r][c];
    return result;
}
```

---

## 🟢 LC #1572 — Matrix Diagonal Sum

```java
public int diagonalSum(int[][] mat) {
    int n = mat.length, sum = 0;
    for (int i = 0; i < n; i++) {
        sum += mat[i][i];           // Primary diagonal
        sum += mat[i][n-1-i];       // Secondary diagonal
    }
    // Subtract center if n is odd (counted twice)
    if (n % 2 == 1) sum -= mat[n/2][n/2];
    return sum;
}
```

---

# 16. Medium Problems

## 🟡 LC #48 — Rotate Image

```java
// Rotate n×n matrix 90 degrees clockwise IN PLACE
public void rotate(int[][] matrix) {
    int n = matrix.length;

    // Step 1: Transpose (swap matrix[r][c] with matrix[c][r])
    for (int r = 0; r < n; r++)
        for (int c = r+1; c < n; c++) {
            int temp = matrix[r][c];
            matrix[r][c] = matrix[c][r];
            matrix[c][r] = temp;
        }

    // Step 2: Reverse each row
    for (int r = 0; r < n; r++) {
        int l = 0, right = n-1;
        while (l < right) {
            int temp = matrix[r][l];
            matrix[r][l++] = matrix[r][right];
            matrix[r][right--] = temp;
        }
    }
}
// Why transpose then reverse? Rotating 90° clockwise = transpose + horizontal flip
```

---

## 🟡 LC #54 — Spiral Matrix

```java
public List<Integer> spiralOrder(int[][] matrix) {
    List<Integer> result = new ArrayList<>();
    int top = 0, bottom = matrix.length-1;
    int left = 0, right = matrix[0].length-1;

    while (top <= bottom && left <= right) {
        // Left to right along top row
        for (int c = left; c <= right; c++) result.add(matrix[top][c]);
        top++;

        // Top to bottom along right column
        for (int r = top; r <= bottom; r++) result.add(matrix[r][right]);
        right--;

        // Right to left along bottom row (if still valid)
        if (top <= bottom) {
            for (int c = right; c >= left; c--) result.add(matrix[bottom][c]);
            bottom--;
        }

        // Bottom to top along left column (if still valid)
        if (left <= right) {
            for (int r = bottom; r >= top; r--) result.add(matrix[r][left]);
            left++;
        }
    }
    return result;
}
```

---

## 🟡 LC #289 — Game of Life

```java
// Update all cells simultaneously based on neighbor live counts
// Use bit encoding: bit0=current, bit1=next state
public void gameOfLife(int[][] board) {
    int m = board.length, n = board[0].length;
    int[][] DIRS = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            int live = 0;
            for (int[] dir : DIRS) {
                int nr = r+dir[0], nc = c+dir[1];
                if (nr>=0&&nr<m&&nc>=0&&nc<n&&(board[nr][nc]&1)==1) live++;
            }
            // Rules: live cell with 2-3 live neighbors stays alive
            //        dead cell with exactly 3 live neighbors becomes alive
            if (board[r][c]==1 && (live==2||live==3)) board[r][c] |= 2;
            if (board[r][c]==0 && live==3) board[r][c] |= 2;
        }
    }

    // Extract next state (bit1)
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            board[r][c] >>= 1;
}
```

---

## 🟡 LC #79 — Word Search

```java
// Find if word exists in grid following adjacent cells (can't reuse)
public boolean exist(char[][] board, String word) {
    int m = board.length, n = board[0].length;
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            if (dfs(board, r, c, word, 0, m, n)) return true;
    return false;
}

private boolean dfs(char[][] board, int r, int c, String word, int idx, int m, int n) {
    if (idx == word.length()) return true;
    if (r<0||r>=m||c<0||c>=n||board[r][c]!=word.charAt(idx)) return false;

    char temp = board[r][c];
    board[r][c] = '#';  // Mark as visited (in-place)

    boolean found = dfs(board,r-1,c,word,idx+1,m,n)
                 || dfs(board,r+1,c,word,idx+1,m,n)
                 || dfs(board,r,c-1,word,idx+1,m,n)
                 || dfs(board,r,c+1,word,idx+1,m,n);

    board[r][c] = temp;  // Restore for backtracking
    return found;
}
```

---

## 🟡 LC #934 — Shortest Bridge

```java
// Two islands in grid. Find shortest bridge (minimum 0-cells to flip) connecting them.
// Step 1: DFS to find first island, add all its cells to BFS queue
// Step 2: BFS from first island to find shortest path to second island
public int shortestBridge(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    Queue<int[]> queue = new LinkedList<>();
    boolean found = false;

    // Step 1: Find first island, mark it, add to queue
    outer:
    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == 1) {
                dfs(grid, r, c, m, n, queue);
                found = true;
                break outer;
            }
        }
    }

    // Step 2: BFS outward from first island
    int steps = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] curr = queue.poll();
            for (int[] dir : DIRS) {
                int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
                if (nr<0||nr>=m||nc<0||nc>=n||grid[nr][nc]==-1) continue;
                if (grid[nr][nc] == 1) return steps;  // Reached second island!
                grid[nr][nc] = -1;
                queue.offer(new int[]{nr, nc});
            }
        }
        steps++;
    }
    return -1;
}

private void dfs(int[][] grid, int r, int c, int m, int n, Queue<int[]> queue) {
    if (r<0||r>=m||c<0||c>=n||grid[r][c]!=1) return;
    grid[r][c] = -1;  // Mark island 1 as visited
    queue.offer(new int[]{r, c});
    dfs(grid,r-1,c,m,n,queue); dfs(grid,r+1,c,m,n,queue);
    dfs(grid,r,c-1,m,n,queue); dfs(grid,r,c+1,m,n,queue);
}
```

---

# 17. Hard Problems

## 🔴 LC #212 — Word Search II

```java
// Find all words from dictionary that exist in board
// Use Trie for efficient multi-word search
class Solution {
    class TrieNode {
        TrieNode[] children = new TrieNode[26];
        String word = null;  // Non-null if this node ends a word
    }

    public List<String> findWords(char[][] board, String[] words) {
        TrieNode root = new TrieNode();
        for (String w : words) {
            TrieNode node = root;
            for (char c : w.toCharArray()) {
                int i = c - 'a';
                if (node.children[i] == null) node.children[i] = new TrieNode();
                node = node.children[i];
            }
            node.word = w;
        }

        List<String> result = new ArrayList<>();
        int m = board.length, n = board[0].length;
        for (int r = 0; r < m; r++)
            for (int c = 0; c < n; c++)
                dfs(board, r, c, m, n, root, result);
        return result;
    }

    private void dfs(char[][] board, int r, int c, int m, int n, TrieNode node, List<String> res) {
        if (r<0||r>=m||c<0||c>=n||board[r][c]=='#') return;
        char ch = board[r][c];
        TrieNode next = node.children[ch - 'a'];
        if (next == null) return;

        if (next.word != null) {
            res.add(next.word);
            next.word = null;  // Avoid duplicates
        }

        board[r][c] = '#';
        int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
        for (int[] dir : DIRS) dfs(board, r+dir[0], c+dir[1], m, n, next, res);
        board[r][c] = ch;
    }
}
```

---

## 🔴 LC #85 — Maximal Rectangle

```java
// Find largest rectangle containing only 1s in binary matrix
public int maximalRectangle(char[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[] heights = new int[n];
    int maxArea = 0;

    for (int r = 0; r < m; r++) {
        // Build histogram for this row
        for (int c = 0; c < n; c++)
            heights[c] = matrix[r][c] == '1' ? heights[c]+1 : 0;

        maxArea = Math.max(maxArea, largestInHistogram(heights));
    }
    return maxArea;
}

private int largestInHistogram(int[] h) {
    Deque<Integer> stack = new ArrayDeque<>();
    int max = 0;
    for (int i = 0; i <= h.length; i++) {
        int curr = i == h.length ? 0 : h[i];
        while (!stack.isEmpty() && h[stack.peek()] > curr) {
            int height = h[stack.pop()];
            int width = stack.isEmpty() ? i : i-stack.peek()-1;
            max = Math.max(max, height*width);
        }
        stack.push(i);
    }
    return max;
}
```

---

## 🔴 LC #407 — Trapping Rain Water II

```java
// 3D version of trapping rain water
// Use min-heap: water level determined by minimum boundary height
public int trapRainWater(int[][] heightMap) {
    int m = heightMap.length, n = heightMap[0].length;
    if (m < 3 || n < 3) return 0;

    boolean[][] visited = new boolean[m][n];
    // Min-heap: (height, row, col)
    PriorityQueue<int[]> pq = new PriorityQueue<>((a,b)->a[0]-b[0]);

    // Add all border cells
    for (int r = 0; r < m; r++) {
        pq.offer(new int[]{heightMap[r][0], r, 0}); visited[r][0] = true;
        pq.offer(new int[]{heightMap[r][n-1], r, n-1}); visited[r][n-1] = true;
    }
    for (int c = 1; c < n-1; c++) {
        pq.offer(new int[]{heightMap[0][c], 0, c}); visited[0][c] = true;
        pq.offer(new int[]{heightMap[m-1][c], m-1, c}); visited[m-1][c] = true;
    }

    int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
    int water = 0;

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int h = curr[0], r = curr[1], c = curr[2];
        for (int[] dir : DIRS) {
            int nr = r+dir[0], nc = c+dir[1];
            if (nr<0||nr>=m||nc<0||nc>=n||visited[nr][nc]) continue;
            visited[nr][nc] = true;
            water += Math.max(0, h - heightMap[nr][nc]);  // Trapped water
            pq.offer(new int[]{Math.max(h, heightMap[nr][nc]), nr, nc});
        }
    }
    return water;
}
```

---

# 18. Complete LeetCode Matrix Problem List

## 🟢 Easy
| # | Problem | Pattern |
|---|---------|---------|
| 463 | Island Perimeter | Count edges |
| 566 | Reshape the Matrix | Index math |
| 661 | Image Smoother | Neighbor average |
| 733 | Flood Fill | DFS flood fill |
| 832 | Flipping an Image | Row manipulation |
| 867 | Transpose Matrix | Index swap |
| 1572 | Matrix Diagonal Sum | Diagonal scan |
| 1672 | Richest Customer Wealth | Row sum |

## 🟡 Medium
| # | Problem | Pattern |
|---|---------|---------|
| 48 | Rotate Image | Transpose + reverse |
| 54 | Spiral Matrix | Boundary pointers |
| 59 | Spiral Matrix II | Boundary fill |
| 62 | Unique Paths | Grid DP |
| 63 | Unique Paths II | Grid DP + obstacles |
| 64 | Minimum Path Sum | Grid DP |
| 73 | Set Matrix Zeroes | In-place marking |
| 74 | Search a 2D Matrix | Binary search |
| 79 | Word Search | DFS backtrack |
| 130 | Surrounded Regions | Boundary DFS |
| 200 | Number of Islands | DFS/BFS island |
| 221 | Maximal Square | Grid DP |
| 240 | Search a 2D Matrix II | Top-right pointer |
| 286 | Walls and Gates | Multi-source BFS |
| 289 | Game of Life | In-place simulation |
| 329 | Longest Increasing Path | DFS + memo |
| 417 | Pacific Atlantic | Boundary BFS |
| 542 | 01 Matrix | Multi-source BFS |
| 695 | Max Area of Island | DFS island |
| 778 | Swim in Rising Water | Dijkstra grid |
| 934 | Shortest Bridge | DFS + BFS |
| 994 | Rotting Oranges | Multi-source BFS |
| 1020 | Number of Enclaves | Boundary DFS |
| 1091 | Shortest Path Binary Matrix | BFS 8-dir |
| 1162 | As Far From Land | Multi-source BFS |
| 1254 | Count Closed Islands | Boundary DFS |
| 1293 | Shortest Path k Obstacles | BFS + state |
| 1368 | Min Cost Valid Path | 0-1 BFS |
| 1631 | Path Min Effort | Dijkstra |
| 1905 | Count Sub Islands | DFS compare |
| 2101 | Detonate the Maximum Bombs | Graph DFS |

## 🔴 Hard
| # | Problem | Pattern |
|---|---------|---------|
| 37 | Sudoku Solver | DFS backtrack |
| 51 | N-Queens | DFS backtrack |
| 84 | Largest Rectangle Histogram | Stack |
| 85 | Maximal Rectangle | Histogram DP |
| 174 | Dungeon Game | Reverse DP |
| 212 | Word Search II | DFS + Trie |
| 305 | Number of Islands II | Union-Find |
| 329 | Longest Increasing Path | DFS + memo |
| 407 | Trapping Rain Water II | Dijkstra heap |
| 753 | Cracking the Safe | DFS Eulerian |

---

# Part 5 — Interview Mastery

# 20. How to Identify Which Pattern to Use

```
See a MATRIX problem
        ↓
What does the problem ask?
        ↓
┌──────────────────────────────────────────────────────────────┐
│ "Count islands / components / connected regions"            │
│ "Paint all connected cells / flood fill"                    │
│ "Find area of each island"                                  │
│       → PATTERN 1: FLOOD FILL / DFS                         │
│       → Outer loop + DFS/BFS to sink each component         │
│                                                             │
│ "Shortest path from A to B"                                 │
│ "Minimum steps to reach"                                    │
│ "Minimum operations"                                        │
│       → PATTERN 2: BFS (if cost=1 per step)                 │
│       → DIJKSTRA (if cells have different weights)          │
│                                                             │
│ "Distance from ANY of multiple sources"                     │
│ "Nearest gate / rotten orange / 0-cell"                     │
│ "Time for all cells to get infected"                        │
│       → PATTERN 3: MULTI-SOURCE BFS                         │
│       → Add ALL sources to queue first, then BFS            │
│                                                             │
│ "Cells connected to border / edge / boundary"               │
│ "Surrounded / enclosed by"                                  │
│ "Can flow to ocean / border"                                │
│       → PATTERN 4: BOUNDARY DFS                             │
│       → DFS from border cells, mark safe, invert rest       │
│                                                             │
│ "Count paths from top-left to bottom-right"                 │
│ "Minimum cost path (right/down only)"                       │
│ "Largest square of 1s"                                      │
│ "Longest increasing path"                                   │
│       → PATTERN 5: DP ON GRID                               │
│       → dp[r][c] based on neighbors already computed        │
│                                                             │
│ "Connect cells dynamically (added one by one)"              │
│ "Are these two cells in the same group?"                    │
│       → PATTERN 6: UNION-FIND                               │
│       → Each cell = node ID = r*n+c                         │
└──────────────────────────────────────────────────────────────┘

THEN ASK:
  All edge weights = 1? → BFS
  Weights 0 or 1?       → 0-1 BFS (deque)
  Arbitrary weights?    → Dijkstra
  No weights, explore all? → DFS
```

---

# 21. Matrix Interview Cheat Sheet

## ⚡ Essential Setup (Write This First)

```java
int m = grid.length, n = grid[0].length;
int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};  // 4-directional
// or 8-directional: {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}}

// Bounds check helper:
boolean inBounds(int r, int c) {
    return r >= 0 && r < m && c >= 0 && c < n;
}
```

## ⚡ Core Templates

```java
// ===================================================
// 1. DFS FLOOD FILL (count/mark connected component)
// ===================================================
void dfs(int[][] grid, int r, int c) {
    if (r<0||r>=m||c<0||c>=n||grid[r][c]!=TARGET) return;
    grid[r][c] = VISITED;  // Mark in-place
    for (int[] d : DIRS) dfs(grid, r+d[0], c+d[1]);
}
// Outer loop:
int count = 0;
for (int r=0;r<m;r++) for (int c=0;c<n;c++)
    if (grid[r][c]==TARGET) { dfs(grid,r,c); count++; }

// ===================================================
// 2. BFS SHORTEST PATH
// ===================================================
Queue<int[]> q = new LinkedList<>();
boolean[][] vis = new boolean[m][n];
q.offer(new int[]{sr, sc, 0});  // row, col, distance
vis[sr][sc] = true;
while (!q.isEmpty()) {
    int[] cur = q.poll();
    if (cur[0]==er && cur[1]==ec) return cur[2];
    for (int[] d : DIRS) {
        int nr=cur[0]+d[0], nc=cur[1]+d[1];
        if (inBounds(nr,nc) && !vis[nr][nc] && grid[nr][nc]==OPEN) {
            vis[nr][nc]=true; q.offer(new int[]{nr,nc,cur[2]+1});
        }
    }
}

// ===================================================
// 3. MULTI-SOURCE BFS
// ===================================================
Queue<int[]> q = new LinkedList<>();
for (int r=0;r<m;r++) for (int c=0;c<n;c++)
    if (grid[r][c]==SOURCE) { q.offer(new int[]{r,c}); grid[r][c]=VISITED; }
while (!q.isEmpty()) {
    int[] cur = q.poll();
    for (int[] d : DIRS) {
        int nr=cur[0]+d[0], nc=cur[1]+d[1];
        if (inBounds(nr,nc) && grid[nr][nc]==TARGET) {
            grid[nr][nc] = grid[cur[0]][cur[1]]+1;  // Distance
            q.offer(new int[]{nr,nc});
        }
    }
}

// ===================================================
// 4. BOUNDARY DFS (mark safe, then invert)
// ===================================================
// From each border cell that qualifies, DFS and mark 'S'
for (int r=0;r<m;r++) { dfs(board,r,0); dfs(board,r,n-1); }
for (int c=0;c<n;c++) { dfs(board,0,c); dfs(board,m-1,c); }
// Then: 'S'→original, target→answer

// ===================================================
// 5. GRID DP (top-left to bottom-right)
// ===================================================
int[][] dp = new int[m][n];
dp[0][0] = grid[0][0];
for (int r=0;r<m;r++) for (int c=0;c<n;c++) {
    if (r==0 && c==0) continue;
    int fromTop  = r>0 ? dp[r-1][c] : INF;
    int fromLeft = c>0 ? dp[r][c-1] : INF;
    dp[r][c] = Math.min(fromTop,fromLeft) + grid[r][c];
}
return dp[m-1][n-1];

// ===================================================
// 6. DIJKSTRA ON GRID
// ===================================================
int[][] dist = new int[m][n];
for (int[] row : dist) Arrays.fill(row, Integer.MAX_VALUE);
dist[0][0] = 0;
PriorityQueue<int[]> pq = new PriorityQueue<>((a,b)->a[0]-b[0]);
pq.offer(new int[]{0,0,0});  // cost, row, col
while (!pq.isEmpty()) {
    int[] cur = pq.poll();
    if (cur[0] > dist[cur[1]][cur[2]]) continue;
    for (int[] d : DIRS) {
        int nr=cur[1]+d[0], nc=cur[2]+d[1];
        if (!inBounds(nr,nc)) continue;
        int newCost = cur[0] + weight(nr,nc);
        if (newCost < dist[nr][nc]) {
            dist[nr][nc]=newCost; pq.offer(new int[]{newCost,nr,nc});
        }
    }
}
```

## ⚡ BFS Level-by-Level Template (when steps matter)

```java
Queue<int[]> queue = new LinkedList<>();
queue.offer(new int[]{startR, startC});
visited[startR][startC] = true;
int steps = 0;

while (!queue.isEmpty()) {
    int size = queue.size();  // Process entire level at once
    for (int i = 0; i < size; i++) {
        int[] curr = queue.poll();
        // ... process curr ...
        for (int[] dir : DIRS) {
            int nr = curr[0]+dir[0], nc = curr[1]+dir[1];
            if (inBounds(nr,nc) && !visited[nr][nc] && condition) {
                visited[nr][nc] = true;
                queue.offer(new int[]{nr, nc});
            }
        }
    }
    steps++;
}
```

## ⚡ The 6 Golden Rules of Matrix Problems

```
1. DIRECTION ARRAY: Always use int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}}.
   Never write 4 separate if-statements. Fewer bugs, easier to extend to 8-dir.

2. MARK BEFORE ADDING TO QUEUE (not after polling).
   If you mark after poll: same cell gets added to queue multiple times → wrong.

3. BFS FOR SHORTEST PATH, DFS FOR CONNECTIVITY.
   "Minimum steps" → BFS. "Count connected / flood fill" → DFS.
   Both are O(m*n). Choose based on what you need.

4. MULTI-SOURCE BFS: just add ALL sources to queue before starting.
   Exactly the same code as single-source BFS — only initialization differs.

5. BOUNDARY DFS TRICK: To find cells NOT connected to border,
   first mark everything connected TO border as safe.
   Remaining unmarked cells = answer.

6. GRID DP ORDER: Process in order that guarantees dependencies are ready.
   Top-to-bottom, left-to-right: uses dp[r-1][c] and dp[r][c-1].
   If reverse DP needed (like dungeon), process bottom-to-top.
```

---

> **Your 3-Week Learning Path:**
>
> Week 1 — DFS Foundations:
> Master flood fill, island counting, boundary DFS.
> Solve: LC #733, #200, #695, #463, #130, #417, #1020
>
> Week 2 — BFS Patterns:
> Single-source BFS, multi-source BFS, state BFS.
> Solve: LC #994, #286, #542, #1091, #1162, #934, #127
>
> Week 3 — DP + Advanced:
> Grid DP, Dijkstra, 0-1 BFS, hard problems.
> Solve: LC #62, #64, #221, #329, #1631, #778, #212
>
> The secret: 90% of matrix problems are one of 6 patterns.
> When you see a matrix problem:
>   1. Is it "shortest path"? → BFS or Dijkstra
>   2. Is it "flood fill / connected"? → DFS
>   3. Is it "from multiple sources"? → Multi-source BFS
>   4. Is it "not connected to border"? → Boundary DFS
>   5. Is it "count paths / minimum cost"? → Grid DP
>   6. Is it "dynamic connectivity"? → Union-Find
> Name the pattern in 30 seconds → you already know 70% of the solution.

---

*This guide covers 60+ LeetCode matrix problems across 6 patterns + 4 advanced techniques — from basic flood fill to Dijkstra on weighted grids and 3D rain water trapping.*
