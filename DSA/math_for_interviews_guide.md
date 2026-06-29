# 🔢 The Complete Math for Interviews Guide
## GCD, Primes, Modular Arithmetic, Combinatorics — Java Edition with LeetCode Problems

---

> **Goal:** By the end, you will understand every math concept that appears in coding interviews, know the exact Java implementations, and solve any math-based LeetCode problem confidently. Math problems look scary but they all reduce to a small set of patterns.

---

## Table of Contents

### Part 1 — Number Theory Foundations
1. Divisibility, Factors, and Multiples
2. GCD and LCM — Euclidean Algorithm
3. Prime Numbers — Sieve of Eratosthenes
4. Prime Factorization
5. Modular Arithmetic

### Part 2 — Modular Arithmetic Deep Dive
6. Why Modular Arithmetic in Interviews
7. Modular Addition, Subtraction, Multiplication
8. Modular Exponentiation (Fast Power)
9. Modular Inverse
10. Fermat's Little Theorem

### Part 3 — Combinatorics
11. Permutations and Combinations
12. Pascal's Triangle
13. Counting Paths and Arrangements
14. Pigeonhole Principle
15. Stars and Bars

### Part 4 — Common Math Patterns in Interviews
16. Pattern 1 — Digit Manipulation
17. Pattern 2 — Number Base Conversion
18. Pattern 3 — Bit Manipulation as Math
19. Pattern 4 — Geometry and Grid Math
20. Pattern 5 — Sequence and Series

### Part 5 — LeetCode Problems by Category
21. Easy Problems
22. Medium Problems
23. Hard Problems
24. Complete LeetCode Math Problem List

### Part 6 — Interview Mastery
25. How to Recognize Math Problems
26. Math Interview Cheat Sheet

---

# Part 1 — Number Theory Foundations

# 1. Divisibility, Factors, and Multiples

## Core Definitions

```
DIVISOR / FACTOR:
  a divides b (written a | b) if b % a == 0
  Example: 3 | 12 because 12 % 3 == 0
  "3 is a factor of 12"

MULTIPLE:
  b is a multiple of a if a | b
  Example: 12 is a multiple of 3

FINDING ALL FACTORS of n:
  Naive: check every number from 1 to n → O(n)
  Better: only check up to √n → O(√n)
  
  WHY √n?
  Factors come in PAIRS: if d divides n, then n/d also divides n.
  One of each pair is ≤ √n (if both > √n, their product > n — contradiction).
  So check up to √n, and for each d found, n/d is the paired factor.

Example: factors of 36
  √36 = 6. Check 1,2,3,4,5,6:
  1 → pair 36. 2 → pair 18. 3 → pair 12. 4 → pair 9. 6 → pair 6 (same)
  Factors: {1, 2, 3, 4, 6, 9, 12, 18, 36}
```

```java
// Find all factors of n in O(√n)
public List<Integer> getFactors(int n) {
    List<Integer> factors = new ArrayList<>();
    for (int i = 1; (long) i * i <= n; i++) {
        if (n % i == 0) {
            factors.add(i);
            if (i != n / i) factors.add(n / i);  // Don't add √n twice
        }
    }
    Collections.sort(factors);
    return factors;
}

// Count number of factors of n (number of divisors)
public int countFactors(int n) {
    int count = 0;
    for (int i = 1; (long) i * i <= n; i++) {
        if (n % i == 0) {
            count += (i == n / i) ? 1 : 2;
        }
    }
    return count;
}

// Check if a divides b
boolean divides(int a, int b) { return b % a == 0; }
```

---

# 2. GCD and LCM — Euclidean Algorithm

## GCD (Greatest Common Divisor)

```
GCD(a, b) = largest integer that divides both a and b.

EUCLIDEAN ALGORITHM:
  GCD(a, b) = GCD(b, a % b)
  GCD(a, 0) = a   ← base case

WHY IT WORKS:
  Any common divisor of a and b also divides (a mod b) = a - k*b.
  So GCD(a, b) = GCD(b, a mod b).

Example: GCD(48, 18)
  GCD(48, 18) = GCD(18, 48%18) = GCD(18, 12)
  GCD(18, 12)  = GCD(12, 18%12) = GCD(12, 6)
  GCD(12, 6)   = GCD(6, 12%6)   = GCD(6, 0)
  GCD(6, 0)    = 6

TIME: O(log(min(a,b))) — incredibly fast!

INTERESTING PROPERTIES:
  GCD(a, 0) = a
  GCD(a, a) = a
  GCD(a, b) = GCD(b, a)  ← commutative
  GCD(a*b, a*c) = a * GCD(b, c)
  If GCD(a, b) = 1: a and b are CO-PRIME (no common factors except 1)
```

```java
// Recursive GCD — clean and fast
public int gcd(int a, int b) {
    return b == 0 ? a : gcd(b, a % b);
}

// Iterative GCD — avoids stack overflow for large numbers
public int gcdIterative(int a, int b) {
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

// Java built-in (Java 9+):
// import java.math.BigInteger;
// BigInteger.valueOf(a).gcd(BigInteger.valueOf(b)).intValue()

// GCD of array
public int gcdArray(int[] nums) {
    int result = nums[0];
    for (int n : nums) result = gcd(result, n);
    return result;
}

// Extended Euclidean: finds x, y such that ax + by = GCD(a, b)
// Useful for modular inverse
public int[] extendedGcd(int a, int b) {
    if (b == 0) return new int[]{a, 1, 0};  // GCD=a, x=1, y=0
    int[] rec = extendedGcd(b, a % b);
    int g = rec[0], x1 = rec[1], y1 = rec[2];
    return new int[]{g, y1, x1 - (a / b) * y1};
}
```

## LCM (Least Common Multiple)

```
LCM(a, b) = smallest positive integer divisible by both a and b.

KEY FORMULA: LCM(a, b) = a * b / GCD(a, b)

WHY: a = GCD * p, b = GCD * q where GCD(p,q)=1
     LCM = GCD * p * q = (a/GCD) * b = a * b / GCD

CAREFUL: a * b can OVERFLOW for large integers!
         Use: LCM = a / GCD(a,b) * b   (divide FIRST to avoid overflow)

Example: LCM(12, 18)
  GCD(12, 18) = 6
  LCM = 12 * 18 / 6 = 36

PROPERTIES:
  LCM(a, b) * GCD(a, b) = a * b
  LCM(a, b) = a   if a is a multiple of b (b divides a)
```

```java
public long lcm(long a, long b) {
    return a / gcd(a, b) * b;  // Divide FIRST to avoid overflow
}

// LCM of array
public long lcmArray(int[] nums) {
    long result = nums[0];
    for (int n : nums) result = lcm(result, n);
    return result;
}
```

## LeetCode Problems Using GCD/LCM

```java
// LC #1979 — Find Greatest Common Divisor of Array
public int findGCD(int[] nums) {
    int min = Arrays.stream(nums).min().getAsInt();
    int max = Arrays.stream(nums).max().getAsInt();
    return gcd(min, max);
}

// LC #2447 — Number of Subarrays With GCD Equal to K
public int subarrayGCD(int[] nums, int k) {
    int count = 0;
    for (int i = 0; i < nums.length; i++) {
        int g = 0;
        for (int j = i; j < nums.length; j++) {
            g = gcd(g, nums[j]);
            if (g == k) count++;
            if (g < k) break;  // GCD can only decrease
        }
    }
    return count;
}

// LC #365 — Water and Jug Problem
// Can we measure exactly target liters using jugs of size x and y?
// By Bezout's theorem: possible iff target is a multiple of GCD(x, y)
//                      and target <= x + y
public boolean canMeasureWater(int x, int y, int target) {
    if (target > x + y) return false;
    if (target == 0) return true;
    return target % gcd(x, y) == 0;
}

// LC #149 — Max Points on a Line
// Use GCD to normalize slope representation
public int maxPoints(int[][] points) {
    int n = points.length, max = 1;
    for (int i = 0; i < n; i++) {
        Map<String, Integer> slopes = new HashMap<>();
        for (int j = i+1; j < n; j++) {
            int dx = points[j][0]-points[i][0];
            int dy = points[j][1]-points[i][1];
            int g = gcd(Math.abs(dx), Math.abs(dy));
            if (g > 0) { dx /= g; dy /= g; }
            if (dx < 0) { dx=-dx; dy=-dy; }
            else if (dx==0) dy = Math.abs(dy);
            String key = dx + "," + dy;
            slopes.merge(key, 1, Integer::sum);
            max = Math.max(max, slopes.get(key) + 1);
        }
    }
    return max;
}
```

---

# 3. Prime Numbers — Sieve of Eratosthenes

## What is a Prime?

```
PRIME: A number > 1 with exactly 2 factors: 1 and itself.
  Primes: 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, ...

COMPOSITE: A number with more than 2 factors (not prime, > 1).

PRIMALITY TEST:
  Naive: check divisibility by all numbers 1..n → O(n)
  Better: check only up to √n → O(√n)
  
  WHY √n? If n has a factor > √n, it must have a paired factor < √n.
  So if no factor found up to √n, n is prime.

  ALSO: Only need to check primes as factors (skip even numbers after 2).
  Check 2, then only odd numbers: 3, 5, 7, 9, 11, ...
  This gives O(√n / 2) in practice.
```

```java
// Check if n is prime — O(√n)
public boolean isPrime(int n) {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;
    for (int i = 3; (long) i * i <= n; i += 2) {  // Only odd divisors
        if (n % i == 0) return false;
    }
    return true;
}
```

## Sieve of Eratosthenes — All Primes Up to N

```
GOAL: Find ALL primes up to N efficiently.
NAIVE: Check each number with isPrime() → O(n√n)
SIEVE: O(n log log n) — almost linear!

ALGORITHM:
  1. Create boolean array isPrime[0..N], all true initially.
  2. Set isPrime[0] = isPrime[1] = false.
  3. For each i from 2 to √N:
       If isPrime[i] is true (i is prime):
         Mark all multiples of i (starting from i*i) as NOT prime.
  4. Remaining true values = primes.

WHY start from i*i?
  All smaller multiples of i (i.e., 2i, 3i, ..., (i-1)i) have already
  been marked by smaller primes. So we start at i*i.

Example: Sieve up to 20
  Start: [F F T T T T T T T T T T T T T T T T T T T]
          0 1 2 3 4 5 6 7 8 9 ...
  i=2: mark 4,6,8,10,12,14,16,18,20 as false
  i=3: mark 9,15 as false (6,12,18 already marked by 2)
  i=4: already false, skip
  Result primes: 2,3,5,7,11,13,17,19
```

```java
// Sieve of Eratosthenes — O(n log log n)
public boolean[] sieve(int n) {
    boolean[] isPrime = new boolean[n + 1];
    Arrays.fill(isPrime, true);
    isPrime[0] = isPrime[1] = false;

    for (int i = 2; (long) i * i <= n; i++) {
        if (isPrime[i]) {
            // Mark multiples of i starting from i*i
            for (int j = i * i; j <= n; j += i) {
                isPrime[j] = false;
            }
        }
    }
    return isPrime;
}

// Get list of all primes up to n
public List<Integer> getPrimes(int n) {
    boolean[] isPrime = sieve(n);
    List<Integer> primes = new ArrayList<>();
    for (int i = 2; i <= n; i++) {
        if (isPrime[i]) primes.add(i);
    }
    return primes;
}

// Count primes up to n (for LC #204)
public int countPrimes(int n) {
    if (n < 2) return 0;
    boolean[] isPrime = sieve(n - 1);
    int count = 0;
    for (boolean p : isPrime) if (p) count++;
    return count;
}
```

## LeetCode Problems Using Primes

```java
// LC #204 — Count Primes
public int countPrimes(int n) {
    if (n < 2) return 0;
    boolean[] notPrime = new boolean[n];
    int count = 0;
    for (int i = 2; i < n; i++) {
        if (!notPrime[i]) {
            count++;
            for (long j = (long)i*i; j < n; j += i) notPrime[(int)j] = true;
        }
    }
    return count;
}

// LC #2523 — Closest Prime Numbers in Range
public int[] closestPrimes(int left, int right) {
    boolean[] isPrime = sieve(right);
    int prev = -1, minGap = Integer.MAX_VALUE;
    int[] ans = {-1, -1};
    for (int i = left; i <= right; i++) {
        if (isPrime[i]) {
            if (prev != -1 && i - prev < minGap) {
                minGap = i - prev;
                ans = new int[]{prev, i};
            }
            prev = i;
        }
    }
    return ans;
}

// LC #866 — Prime Palindrome
public int primePalindrome(int n) {
    while (true) {
        if (isPalindrome(n) && isPrime(n)) return n;
        n++;
        // Skip even-length numbers (all even-length palindromes > 11 are divisible by 11)
        if (n >= 10000000 && n <= 99999999) n = 100000000;
    }
}
boolean isPalindrome(int n) {
    String s = String.valueOf(n);
    int l=0, r=s.length()-1;
    while (l<r) if (s.charAt(l++)!=s.charAt(r--)) return false;
    return true;
}
```

---

# 4. Prime Factorization

```
PRIME FACTORIZATION: Express n as product of prime powers.
  Example: 360 = 2³ × 3² × 5¹

ALGORITHM: Trial division up to √n.
  - Divide by 2 as many times as possible.
  - Then check odd numbers 3, 5, 7, ... up to √remaining.
  - If remaining > 1 after loop: it's a prime factor itself.

TIME: O(√n)

APPLICATIONS:
  - Count divisors: if n = p1^a1 * p2^a2 * ... → divisors = (a1+1)(a2+1)...
  - Find LCM of many numbers using prime factorizations
  - Check if n is a perfect square (all exponents even)
  - Check if n is a perfect cube (all exponents divisible by 3)
```

```java
// Prime factorization: return map of prime → exponent
public Map<Integer, Integer> primeFactors(int n) {
    Map<Integer, Integer> factors = new TreeMap<>();

    // Divide by 2 first
    while (n % 2 == 0) {
        factors.merge(2, 1, Integer::sum);
        n /= 2;
    }

    // Then check odd factors from 3
    for (int i = 3; (long) i * i <= n; i += 2) {
        while (n % i == 0) {
            factors.merge(i, 1, Integer::sum);
            n /= i;
        }
    }

    // If n > 1, n itself is a prime factor
    if (n > 1) factors.merge(n, 1, Integer::sum);

    return factors;
}

// Number of divisors using prime factorization
// n = p1^a1 * p2^a2 * ... → divisors count = (a1+1)(a2+1)...
public int numDivisors(int n) {
    int count = 1;
    for (Map.Entry<Integer, Integer> entry : primeFactors(n).entrySet()) {
        count *= (entry.getValue() + 1);
    }
    return count;
}

// Smallest prime factor sieve — useful for quick factorization of many numbers
public int[] smallestPrimeFactor(int n) {
    int[] spf = new int[n + 1];
    for (int i = 0; i <= n; i++) spf[i] = i;
    for (int i = 2; (long) i * i <= n; i++) {
        if (spf[i] == i) {  // i is prime
            for (int j = i * i; j <= n; j += i) {
                if (spf[j] == j) spf[j] = i;  // i is smallest prime factor of j
            }
        }
    }
    return spf;
}

// Factorize n in O(log n) using SPF sieve
public List<Integer> factorize(int n, int[] spf) {
    List<Integer> factors = new ArrayList<>();
    while (n > 1) {
        factors.add(spf[n]);
        n /= spf[n];
    }
    return factors;
}
```

## LeetCode Problems Using Prime Factorization

```java
// LC #952 — Largest Component Size by Common Factor
// Two numbers in same component if they share a common factor > 1
// Use union-find: for each number, union it with each prime factor
public int largestComponentSize(int[] nums) {
    int max = Arrays.stream(nums).max().getAsInt();
    int[] parent = new int[max + 1];
    for (int i = 0; i <= max; i++) parent[i] = i;

    for (int n : nums) {
        int temp = n;
        for (int f = 2; (long)f*f <= temp; f++) {
            if (temp % f == 0) {
                union(parent, n, f);
                while (temp % f == 0) temp /= f;
            }
        }
        if (temp > 1) union(parent, n, temp);
    }

    Map<Integer, Integer> count = new HashMap<>();
    int result = 0;
    for (int n : nums) {
        int root = find(parent, n);
        count.merge(root, 1, Integer::sum);
        result = Math.max(result, count.get(root));
    }
    return result;
}
int find(int[] p, int x) { return p[x]==x ? x : (p[x]=find(p,p[x])); }
void union(int[] p, int x, int y) { p[find(p,x)] = find(p,y); }

// LC #2507 — Smallest Value After Replacing With Sum of Prime Factors
public int smallestValue(int n) {
    while (true) {
        int sum = 0, temp = n;
        for (int f = 2; (long)f*f <= temp; f++) {
            while (temp % f == 0) { sum += f; temp /= f; }
        }
        if (temp > 1) sum += temp;
        if (sum == n) return n;
        n = sum;
    }
}
```

---

# 5. Modular Arithmetic

## Why Modular Arithmetic?

```
PROBLEM: Answers to combinatorics problems can be astronomically large.
  "Count paths in 100x100 grid" → a number with thousands of digits!
  
SOLUTION: Return answer modulo 10^9 + 7 (= 1,000,000,007).
  This keeps numbers manageable (fits in int/long).

WHY 10^9 + 7?
  1. It's prime (important for modular inverse to exist).
  2. It's just below 2^30 ≈ 10^9, so two values fit in a long (2^63).
     (10^9+7)^2 ≈ 10^18 < 2^63 ≈ 9.2×10^18 → no overflow in long multiplication.
  3. Common alternative: 998244353 (also prime, also used frequently).

BASIC RULES:
  (a + b) % M = ((a % M) + (b % M)) % M
  (a - b) % M = ((a % M) - (b % M) + M) % M  ← +M to handle negative!
  (a * b) % M = ((a % M) * (b % M)) % M
  (a / b) % M = (a * modInverse(b, M)) % M   ← division needs inverse!
```

```java
static final long MOD = 1_000_000_007L;

// Safe addition mod M
long addMod(long a, long b) { return (a + b) % MOD; }

// Safe subtraction mod M (handles negative)
long subMod(long a, long b) { return ((a - b) % MOD + MOD) % MOD; }

// Safe multiplication mod M
long mulMod(long a, long b) { return (a % MOD) * (b % MOD) % MOD; }
```

---

# Part 2 — Modular Arithmetic Deep Dive

# 6. Why Modular Arithmetic in Interviews

```
WHENEVER you see "return answer modulo 10^9 + 7":
  → The actual answer is huge → use mod at EVERY step.
  → Apply mod after each addition and multiplication.
  → Never apply mod to a divisor directly (use modular inverse).

COMMON MISTAKES:
  1. Forgetting mod → overflow (long overflows at ~9.2 * 10^18).
  2. Applying mod at end only → intermediate overflow.
  3. Modding the denominator instead of using modular inverse.
  4. Subtracting without adding M → getting negative results.

OVERFLOW CHECK:
  int  max:  ~2.1 * 10^9  (2^31 - 1)
  long max:  ~9.2 * 10^18 (2^63 - 1)
  
  a * b where a,b < 10^9: product < 10^18 → fits in long ✓
  a * b where a,b < 10^18: MIGHT overflow → use BigInteger or mod first
```

---

# 7. Modular Addition, Subtraction, Multiplication

```java
// Example: Counting paths problem
// dp[i][j] = number of paths to cell (i,j)
public int uniquePaths(int m, int n) {
    long MOD = 1_000_000_007L;
    long[][] dp = new long[m][n];
    Arrays.fill(dp[0], 1);
    for (long[] row : dp) row[0] = 1;

    for (int r = 1; r < m; r++) {
        for (int c = 1; c < n; c++) {
            dp[r][c] = (dp[r-1][c] + dp[r][c-1]) % MOD;  // MOD at each step!
        }
    }
    return (int) dp[m-1][n-1];
}

// CRITICAL: Subtraction with negative handling
long subtract(long a, long b, long mod) {
    return ((a - b) % mod + mod) % mod;  // + mod ensures non-negative
}

// Example where subtraction appears:
// Count of NOT something = Total - Count of SOMETHING
// Count = (Total - Bad + MOD) % MOD
```

---

# 8. Modular Exponentiation (Fast Power)

## The Problem

```
COMPUTE: a^n mod M

NAIVE: multiply a by itself n times → O(n)
If n = 10^18, this is impossibly slow!

FAST POWER (Binary Exponentiation):
  Use the fact that:
    a^n = (a^(n/2))^2       if n is even
    a^n = a * (a^(n-1/2))^2 if n is odd (n-1 is even)

  This halves n at each step → O(log n) multiplications!

Example: 2^10
  10 = 1010 in binary
  2^10 = 2^8 * 2^2
  
  n=10: even → result = 2^5 * 2^5, compute 2^5
  n=5:  odd  → result *= 2^1, compute 2^4
  n=4:  even → result = 2^2 * 2^2, compute 2^2
  n=2:  even → result = 2^1 * 2^1, compute 2^1
  n=1:  odd  → result *= 2^0 = 1

Only 4 multiplications for n=10 instead of 10!
```

```java
// Fast modular exponentiation: a^n mod M in O(log n)
public long fastPow(long base, long exp, long mod) {
    long result = 1;
    base %= mod;

    while (exp > 0) {
        if ((exp & 1) == 1) {        // If current bit of exp is 1
            result = result * base % mod;  // Multiply result by current base
        }
        base = base * base % mod;    // Square the base
        exp >>= 1;                   // Move to next bit
    }

    return result;
}

// Usage:
// 2^10 mod 1e9+7 = fastPow(2, 10, 1e9+7) = 1024
// 3^(10^9) mod 1e9+7 = fastPow(3, 1_000_000_000L, MOD)
```

**Trace of fastPow(2, 10, 1000):**
```
exp=10 (1010b): base=2,  result=1
  bit=0: result stays 1, base=4
exp=5  (101b):  base=4,  result=1
  bit=1: result=4, base=16
exp=2  (10b):   base=16, result=4
  bit=0: result stays 4, base=256
exp=1  (1b):    base=256,result=4
  bit=1: result=4*256=1024, base=65536
exp=0: done
Result = 1024 ✓ (2^10 = 1024)
```

## LeetCode Problems Using Fast Power

```java
// LC #50 — Pow(x, n)
public double myPow(double x, int n) {
    long N = n;
    if (N < 0) { x = 1 / x; N = -N; }
    double result = 1;
    while (N > 0) {
        if ((N & 1) == 1) result *= x;
        x *= x;
        N >>= 1;
    }
    return result;
}

// LC #372 — Super Pow: a^b mod 1337, where b is an array of digits
public int superPow(int a, int[] b) {
    int MOD = 1337;
    int result = 1;
    a %= MOD;
    for (int i = b.length - 1; i >= 0; i--) {
        result = (int)((long)result * fastPow(a, b[i], MOD) % MOD);
        a = (int) fastPow(a, 10, MOD);
    }
    return result;
}
```

---

# 9. Modular Inverse

## The Problem

```
(a / b) mod M ≠ (a mod M) / (b mod M)  ← This is WRONG!

We can't just divide after taking mod. Instead we need the MODULAR INVERSE.

MODULAR INVERSE of b (mod M):
  b^(-1) is a number such that b * b^(-1) ≡ 1 (mod M)
  
  Then: a/b mod M = a * b^(-1) mod M

HOW TO FIND IT:
  Method 1: Fermat's Little Theorem (when M is prime):
    b^(-1) = b^(M-2) mod M   [because b^M ≡ b (mod M), so b^(M-1) ≡ 1]
    Use fastPow(b, M-2, M)
    TIME: O(log M)

  Method 2: Extended Euclidean Algorithm (works even when M is not prime):
    Find x, y such that b*x + M*y = 1
    Then b*x ≡ 1 (mod M), so x is the inverse.
    Works only if GCD(b, M) = 1.

IMPORTANT: Modular inverse exists ONLY when GCD(b, M) = 1.
  If M is prime (like 10^9+7), then all numbers 1 to M-1 have inverses.
```

```java
static final long MOD = 1_000_000_007L;

// Modular inverse using Fermat's Little Theorem (MOD must be prime)
public long modInverse(long a, long mod) {
    return fastPow(a, mod - 2, mod);
}

// Division in modular arithmetic
public long modDivide(long a, long b, long mod) {
    return (a % mod) * modInverse(b, mod) % mod;
}

// Precompute factorials and inverse factorials for combinations
long[] fact, invFact;

public void precompute(int n) {
    fact = new long[n + 1];
    invFact = new long[n + 1];
    fact[0] = 1;
    for (int i = 1; i <= n; i++) fact[i] = fact[i-1] * i % MOD;
    invFact[n] = modInverse(fact[n], MOD);
    for (int i = n-1; i >= 0; i--) invFact[i] = invFact[i+1] * (i+1) % MOD;
}

// C(n, k) = n! / (k! * (n-k)!) mod MOD — O(1) after precomputation
public long comb(int n, int k) {
    if (k < 0 || k > n) return 0;
    return fact[n] * invFact[k] % MOD * invFact[n-k] % MOD;
}
```

---

# 10. Fermat's Little Theorem

```
STATEMENT: If p is prime and GCD(a, p) = 1, then:
  a^(p-1) ≡ 1 (mod p)
  
  Equivalently: a^p ≡ a (mod p)

CONSEQUENCE (modular inverse):
  a^(p-1) ≡ 1 (mod p)
  a * a^(p-2) ≡ 1 (mod p)
  So: a^(-1) ≡ a^(p-2) (mod p)  ← The modular inverse!

EXAMPLE:
  Find inverse of 3 modulo 7 (7 is prime):
  3^(-1) mod 7 = 3^(7-2) mod 7 = 3^5 mod 7
  3^5 = 243. 243 mod 7 = 243 - 34*7 = 243 - 238 = 5.
  Check: 3 * 5 = 15. 15 mod 7 = 1. ✓

WILSON'S THEOREM (less common but useful):
  (p-1)! ≡ -1 (mod p) for prime p.
  
EULER'S THEOREM (generalization):
  a^φ(n) ≡ 1 (mod n) when GCD(a,n)=1
  where φ(n) is Euler's totient function.
  
  φ(n) = n * ∏(1 - 1/p) for all prime p dividing n.
  φ(prime p) = p - 1
  φ(p^k) = p^k - p^(k-1)
```

---

# Part 3 — Combinatorics

# 11. Permutations and Combinations

## Core Formulas

```
PERMUTATION P(n, k):
  Number of ways to ARRANGE k items chosen from n distinct items.
  ORDER MATTERS.
  P(n, k) = n! / (n-k)! = n * (n-1) * ... * (n-k+1)
  
  Example: P(5, 3) = 5*4*3 = 60 ways to arrange 3 from 5 items.

COMBINATION C(n, k):
  Number of ways to CHOOSE k items from n distinct items.
  ORDER DOES NOT MATTER.
  C(n, k) = n! / (k! * (n-k)!)
  
  Example: C(5, 3) = 10 ways to choose 3 from 5 items.

INTUITION:
  P(n,k): first pick k items (n choices, then n-1, ...) = P(n,k)
  C(n,k): P(n,k) but divide by k! arrangements of chosen items.

KEY IDENTITIES:
  C(n, k) = C(n, n-k)           ← choosing k = leaving out n-k
  C(n, 0) = C(n, n) = 1
  C(n, 1) = n
  C(n, k) = C(n-1, k-1) + C(n-1, k)  ← Pascal's rule
  Sum of row n: C(n,0)+C(n,1)+...+C(n,n) = 2^n
```

```java
// C(n, k) without precomputation — O(k) using multiplicative formula
// C(n,k) = n/1 * (n-1)/2 * (n-2)/3 * ... * (n-k+1)/k
public long comb(int n, int k) {
    if (k > n - k) k = n - k;  // C(n,k) = C(n,n-k), use smaller k
    long result = 1;
    for (int i = 0; i < k; i++) {
        result = result * (n - i) / (i + 1);  // Integer division is exact here!
    }
    return result;
}

// C(n, k) mod M — using modular inverse
public long combMod(int n, int k, long mod) {
    if (k > n) return 0;
    long num = 1, den = 1;
    for (int i = 0; i < k; i++) {
        num = num * ((n - i) % mod) % mod;
        den = den * ((i + 1) % mod) % mod;
    }
    return num * fastPow(den, mod - 2, mod) % mod;
}

// Factorial
public long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) result *= i;
    return result;
}

// Factorial mod M
public long factorialMod(int n, long mod) {
    long result = 1;
    for (int i = 2; i <= n; i++) result = result * i % mod;
    return result;
}
```

---

# 12. Pascal's Triangle

```
Pascal's triangle: C(n, k) arranged in rows.

Row 0:           1
Row 1:          1 1
Row 2:         1 2 1
Row 3:        1 3 3 1
Row 4:       1 4 6 4 1
Row 5:      1 5 10 10 5 1

C(n, k) = C(n-1, k-1) + C(n-1, k)

PROPERTIES:
  Row n: coefficients of (a+b)^n  ← Binomial theorem!
  Row n sum: 2^n
  Each row is symmetric
  Diagonal 1: natural numbers 1,2,3,4,...
  Diagonal 2: triangular numbers 1,3,6,10,...

HOW TO BUILD in O(n^2) time and O(n) space:
  Row by row, each value is sum of two values above.
```

```java
// LC #118 — Pascal's Triangle
public List<List<Integer>> generate(int numRows) {
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < numRows; i++) {
        List<Integer> row = new ArrayList<>();
        row.add(1);
        for (int j = 1; j < i; j++) {
            List<Integer> prev = result.get(i-1);
            row.add(prev.get(j-1) + prev.get(j));
        }
        if (i > 0) row.add(1);
        result.add(row);
    }
    return result;
}

// LC #119 — Pascal's Triangle II (just kth row)
public List<Integer> getRow(int rowIndex) {
    List<Integer> row = new ArrayList<>();
    row.add(1);
    for (int i = 1; i <= rowIndex; i++) {
        // Build from right to avoid overwriting values we still need
        for (int j = row.size()-1; j > 0; j--) {
            row.set(j, row.get(j) + row.get(j-1));
        }
        row.add(1);
    }
    return row;
}
```

---

# 13. Counting Paths and Arrangements

## Paths in Grid

```
UNIQUE PATHS in m×n grid (move right or down only):
  At each step: choose right or down.
  Total steps = (m-1) + (n-1) = m+n-2 steps.
  Choose which (m-1) of them are "down": C(m+n-2, m-1).
  
  Example: 3×3 grid → C(4,2) = 6 paths.

WITH RESTRICTIONS:
  If some cells are blocked, use DP instead of formula.
  dp[r][c] = dp[r-1][c] + dp[r][c-1] (if not blocked)

ANAGRAMS / ARRANGEMENTS:
  Arrangements of n items with repetitions:
  Word with n letters where letter i appears k_i times:
  n! / (k_1! * k_2! * ... * k_r!)
  
  Example: "MISSISSIPPI" = 11! / (4! * 4! * 2! * 1!) = 34650 arrangements.

CIRCULAR ARRANGEMENTS:
  n distinct items in a circle: (n-1)!
  (Fix one item, arrange rest: n! / n = (n-1)!)
```

```java
// LC #62 — Unique Paths (combinatorial formula)
public int uniquePaths(int m, int n) {
    // C(m+n-2, m-1)
    int total = m + n - 2;
    int choose = m - 1;
    long result = 1;
    for (int i = 0; i < choose; i++) {
        result = result * (total - i) / (i + 1);
    }
    return (int) result;
}

// Count arrangements of a string with duplicate characters
public long countArrangements(String s) {
    int[] freq = new int[26];
    for (char c : s.toCharArray()) freq[c - 'a']++;
    long result = factorialMod(s.length(), MOD);
    for (int f : freq) result = result * modInverse(factorialMod(f, MOD), MOD) % MOD;
    return result;
}
```

---

# 14. Pigeonhole Principle

```
STATEMENT: If n+1 items are placed in n containers,
           at least one container must contain 2+ items.

APPLICATIONS IN INTERVIEWS:
  1. "Prove a duplicate must exist":
     Array of n+1 numbers from 1..n → at least one repeats.
  
  2. "Find the repeated element":
     Use cycle detection or sum formula instead of brute force.
  
  3. "Any 2 elements in array of size n+1 with values in [1,n]":
     Guaranteed duplicate.

GENERALIZED:
  n*k+1 items in n containers → at least one has k+1 items.
  
INTERVIEW TRICK:
  When asked "find a condition that must be true about large input",
  think pigeonhole: map items to buckets, argue one bucket is full.
```

```java
// LC #41 — First Missing Positive (Pigeonhole thinking)
// n+1 numbers in array of size n → some positive in [1,n] must be missing
public int firstMissingPositive(int[] nums) {
    int n = nums.length;
    // Use array itself as hash table: nums[i] should hold value i+1
    for (int i = 0; i < n; i++) {
        while (nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i]) {
            int temp = nums[nums[i]-1];
            nums[nums[i]-1] = nums[i];
            nums[i] = temp;
        }
    }
    for (int i = 0; i < n; i++) if (nums[i] != i+1) return i+1;
    return n+1;
}

// LC #287 — Find Duplicate Number (Pigeonhole: n+1 numbers in [1,n])
// Floyd's cycle detection (since each value points to next index like linked list)
public int findDuplicate(int[] nums) {
    int slow = nums[0], fast = nums[0];
    do { slow = nums[slow]; fast = nums[nums[fast]]; } while (slow != fast);
    fast = nums[0];
    while (slow != fast) { slow = nums[slow]; fast = nums[fast]; }
    return slow;
}
```

---

# 15. Stars and Bars

```
PROBLEM: How many ways to distribute n identical items into k distinct bins?
  (Bins can be empty; items are identical)

FORMULA: C(n + k - 1, k - 1)

VISUAL:
  n=3 items, k=2 bins:
  Think of n stars (*) and k-1 bars (|) to separate bins.
  ***|   → bin1=3, bin2=0
  **|*   → bin1=2, bin2=1
  *|**   → bin1=1, bin2=2
  |***   → bin1=0, bin2=3
  4 arrangements = C(3+2-1, 2-1) = C(4,1) = 4 ✓

VARIATIONS:
  Each bin must have AT LEAST 1 item: C(n-1, k-1)
    (Give 1 to each bin first, distribute remaining n-k)
  
  Each bin has AT MOST m items: Inclusion-Exclusion.

INTERVIEW USE:
  "How many arrays of k non-negative integers summing to n?"
  → C(n+k-1, k-1)
  
  "How many ways to pick k elements from set of n with repetition?"
  → C(n+k-1, k)
```

```java
// LC #1641 — Count Sorted Vowel Strings
// Strings of length n using vowels in non-decreasing order
// = Stars and Bars: n items (length) into 5 bins (vowels)
// = C(n+5-1, 5-1) = C(n+4, 4)
public int countVowelStrings(int n) {
    // C(n+4, 4) = (n+1)(n+2)(n+3)(n+4) / 24
    return (n+1) * (n+2) * (n+3) * (n+4) / 24;
}

// LC #2338 — Count the Number of Ideal Arrays
// Stars and bars with prime factorization
public int idealArrays(int n, int maxValue) {
    // Each value v has prime factorization p1^a1 * p2^a2 * ...
    // Stars and bars: exponents distributed across n positions
    // For each prime factor with exponent a: C(n+a-1, a) ways
    long MOD = 1_000_000_007L;
    precompute(n + 15);  // Precompute factorials
    long result = 0;
    boolean[] sieve = sieve(maxValue);
    for (int v = 1; v <= maxValue; v++) {
        long ways = 1;
        int temp = v;
        for (int p = 2; p * p <= temp; p++) {
            if (temp % p == 0) {
                int cnt = 0;
                while (temp % p == 0) { cnt++; temp /= p; }
                ways = ways * comb(n + cnt - 1, cnt) % MOD;
            }
        }
        if (temp > 1) ways = ways * comb(n, 1) % MOD;
        result = (result + ways) % MOD;
    }
    return (int) result;
}
```

---

# Part 4 — Common Math Patterns in Interviews

# 16. Pattern 1 — Digit Manipulation

```
KEY OPERATIONS:
  Get last digit:      n % 10
  Remove last digit:   n / 10
  Get digit count:     (int)(Math.log10(n)) + 1
  Reverse a number:    while n>0: reversed = reversed*10 + n%10; n/=10
  Sum of digits:       while n>0: sum += n%10; n/=10
  Digital root:        1 + (n-1) % 9  (if n > 0)

HAPPY NUMBER (LC #202):
  Replace n with sum of squares of its digits.
  If eventually reaches 1 → happy.
  Else → cycles forever (cycle always includes 4).
  
PALINDROME NUMBER (LC #9):
  Reverse the number, compare to original.
  OR: reverse only the second half, compare to first half.
```

```java
// LC #9 — Palindrome Number
public boolean isPalindrome(int x) {
    if (x < 0 || (x % 10 == 0 && x != 0)) return false;
    int reversed = 0;
    while (x > reversed) {
        reversed = reversed * 10 + x % 10;
        x /= 10;
    }
    // For odd digit count: reversed/10 removes middle digit
    return x == reversed || x == reversed / 10;
}

// LC #202 — Happy Number
public boolean isHappy(int n) {
    Set<Integer> seen = new HashSet<>();
    while (n != 1 && !seen.contains(n)) {
        seen.add(n);
        int sum = 0;
        while (n > 0) { int d = n%10; sum += d*d; n /= 10; }
        n = sum;
    }
    return n == 1;
}

// LC #258 — Add Digits (Digital Root)
// Sum digits repeatedly until single digit.
// Formula: 1 + (n-1) % 9  (for n > 0)
public int addDigits(int num) {
    return num == 0 ? 0 : 1 + (num - 1) % 9;
}

// LC #43 — Multiply Strings
public String multiply(String num1, String num2) {
    int m = num1.length(), n = num2.length();
    int[] pos = new int[m + n];
    for (int i = m-1; i >= 0; i--) {
        for (int j = n-1; j >= 0; j--) {
            int mul = (num1.charAt(i)-'0') * (num2.charAt(j)-'0');
            int p1 = i+j, p2 = i+j+1;
            int sum = mul + pos[p2];
            pos[p2] = sum % 10;
            pos[p1] += sum / 10;
        }
    }
    StringBuilder sb = new StringBuilder();
    for (int p : pos) if (!(sb.length()==0 && p==0)) sb.append(p);
    return sb.length() == 0 ? "0" : sb.toString();
}
```

---

# 17. Pattern 2 — Number Base Conversion

```
BINARY, OCTAL, HEX:
  Decimal → Binary:  divide by 2, collect remainders
  Binary → Decimal:  sum bit * 2^position
  
Java built-ins:
  Integer.toBinaryString(n)    // decimal → binary string
  Integer.toOctalString(n)     // decimal → octal string
  Integer.toHexString(n)       // decimal → hex string
  Integer.parseInt("1010", 2)  // binary string → decimal
  Integer.parseInt("ff", 16)   // hex string → decimal
  Integer.bitCount(n)          // count 1-bits

GENERAL BASE CONVERSION:
  n (base 10) → base b: divide n by b, collect remainders
  Result string built from remainders in REVERSE order.
```

```java
// LC #504 — Base 7
public String convertToBase7(int num) {
    if (num == 0) return "0";
    StringBuilder sb = new StringBuilder();
    boolean neg = num < 0;
    num = Math.abs(num);
    while (num > 0) {
        sb.append(num % 7);
        num /= 7;
    }
    if (neg) sb.append('-');
    return sb.reverse().toString();
}

// LC #168 — Excel Sheet Column Title
// 1→A, 2→B, ..., 26→Z, 27→AA, 28→AB
// Like base-26, but 1-indexed (no 0)
public String convertToTitle(int columnNumber) {
    StringBuilder sb = new StringBuilder();
    while (columnNumber > 0) {
        columnNumber--;  // Make 0-indexed
        sb.append((char)('A' + columnNumber % 26));
        columnNumber /= 26;
    }
    return sb.reverse().toString();
}

// LC #171 — Excel Sheet Column Number
public int titleToNumber(String columnTitle) {
    int result = 0;
    for (char c : columnTitle.toCharArray()) {
        result = result * 26 + (c - 'A' + 1);
    }
    return result;
}

// LC #405 — Convert a Number to Hexadecimal
public String toHex(int num) {
    if (num == 0) return "0";
    char[] hex = "0123456789abcdef".toCharArray();
    StringBuilder sb = new StringBuilder();
    // Use unsigned representation (handle negative with bit masking)
    while (num != 0) {
        sb.append(hex[num & 0xF]);  // Last 4 bits
        num >>>= 4;  // Unsigned right shift
    }
    return sb.reverse().toString();
}
```

---

# 18. Pattern 3 — Bit Manipulation as Math

```
KEY BIT TRICKS:
  n & (n-1)   → clear lowest set bit (also: n is power of 2 iff n&(n-1)==0)
  n & (-n)    → isolate lowest set bit
  n | (n-1)   → set all bits below lowest set bit
  n ^ n = 0   → XOR self = 0 (used to find single element)
  n ^ 0 = n   → XOR with 0 = unchanged
  
POWERS OF 2:
  n is power of 2: n > 0 && (n & (n-1)) == 0
  Largest power of 2 ≤ n: Integer.highestOneBit(n)
  
COUNT BITS:
  Integer.bitCount(n)           → Java built-in popcount
  Brian Kernighan: n & (n-1) removes one set bit, count iterations
  
MATH via BIT:
  n * 2 = n << 1
  n / 2 = n >> 1
  n % 2 == n & 1   (check if odd)
  n * (2^k) = n << k
  n / (2^k) = n >> k (for positive n)
```

```java
// LC #231 — Power of Two
public boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n-1)) == 0;
}

// LC #191 — Number of 1 Bits
public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) { n &= (n-1); count++; }  // Clear one bit per iteration
    return count;
}

// LC #338 — Counting Bits
// dp[i] = dp[i & (i-1)] + 1  (remove lowest bit, add 1)
public int[] countBits(int n) {
    int[] dp = new int[n+1];
    for (int i = 1; i <= n; i++) dp[i] = dp[i & (i-1)] + 1;
    return dp;
}

// LC #136 — Single Number (XOR trick)
public int singleNumber(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;
    return result;
}

// LC #137 — Single Number II (appears 3 times, one appears once)
public int singleNumberII(int[] nums) {
    int ones = 0, twos = 0;
    for (int n : nums) {
        ones = (ones ^ n) & ~twos;
        twos = (twos ^ n) & ~ones;
    }
    return ones;
}
```

---

# 19. Pattern 4 — Geometry and Grid Math

```
DISTANCE FORMULAS:
  Euclidean: sqrt((x2-x1)^2 + (y2-y1)^2)  ← use squared to avoid sqrt
  Manhattan: |x2-x1| + |y2-y1|             ← for grid movement
  Chebyshev: max(|x2-x1|, |y2-y1|)         ← 8-directional movement

KEY GEOMETRY TRICKS:
  Area of triangle given 3 points (shoelace formula):
    Area = |x1(y2-y3) + x2(y3-y1) + x3(y1-y2)| / 2
  
  Cross product (to check if 3 points are collinear):
    (b-a) × (c-a) = (bx-ax)(cy-ay) - (by-ay)(cx-ax)
    = 0: collinear, > 0: counter-clockwise, < 0: clockwise
  
  Rotate point (x,y) by 90° clockwise: (y, -x)
  Rotate point (x,y) by 90° counter-clockwise: (-y, x)

CIRCLE:
  Point inside circle: (x-cx)^2 + (y-cy)^2 <= r^2
  Use squared distance to avoid floating point.
```

```java
// LC #1232 — Check If It Is a Straight Line
public boolean checkStraightLine(int[][] coords) {
    int dx = coords[1][0] - coords[0][0];
    int dy = coords[1][1] - coords[0][1];
    for (int i = 2; i < coords.length; i++) {
        int x = coords[i][0] - coords[0][0];
        int y = coords[i][1] - coords[0][1];
        // Cross product = 0 for collinear points
        if (dx * y != dy * x) return false;
    }
    return true;
}

// Manhattan distance for grid problems
int manhattan(int[] a, int[] b) {
    return Math.abs(a[0]-b[0]) + Math.abs(a[1]-b[1]);
}

// LC #593 — Valid Square
// 4 points form a square if: 4 equal non-zero distances + 2 equal diagonals
public boolean validSquare(int[] p1, int[] p2, int[] p3, int[] p4) {
    long[] dists = {
        dist(p1,p2), dist(p1,p3), dist(p1,p4),
        dist(p2,p3), dist(p2,p4), dist(p3,p4)
    };
    Arrays.sort(dists);
    // 4 equal sides + 2 equal diagonals, all non-zero
    return dists[0] > 0
        && dists[0]==dists[1] && dists[1]==dists[2] && dists[2]==dists[3]
        && dists[4]==dists[5];
}
long dist(int[] a, int[] b) {
    long dx=a[0]-b[0], dy=a[1]-b[1]; return dx*dx+dy*dy;
}
```

---

# 20. Pattern 5 — Sequence and Series

```
ARITHMETIC SEQUENCE:
  a, a+d, a+2d, ..., a+(n-1)d
  Sum = n * (first + last) / 2 = n * (2a + (n-1)d) / 2
  
GEOMETRIC SEQUENCE:
  a, ar, ar^2, ..., ar^(n-1)
  Sum = a * (r^n - 1) / (r - 1)

TRIANGULAR NUMBERS: 1, 3, 6, 10, 15, 21, ...
  T(n) = n*(n+1)/2
  Useful for counting pairs, sub-intervals.

CATALAN NUMBERS: 1, 1, 2, 5, 14, 42, 132, 429, ...
  C(n) = C(2n, n) / (n+1)
  Appear in: valid parentheses count, binary tree count,
             polygon triangulation, mountain arrays.

FIBONACCI: 0, 1, 1, 2, 3, 5, 8, 13, 21, ...
  F(n) = F(n-1) + F(n-2)
  F(n) ≈ φ^n / √5 where φ = (1+√5)/2 ≈ 1.618 (golden ratio)

SUM OF INTEGERS 1 to n: n*(n+1)/2
SUM OF SQUARES: n*(n+1)*(2n+1)/6
SUM OF CUBES: (n*(n+1)/2)^2
```

```java
// Check if n is a perfect square
boolean isPerfectSquare(int n) {
    long s = (long) Math.sqrt(n);
    return s * s == n || (s+1)*(s+1) == n;
}

// nth Catalan number mod M
public long catalan(int n) {
    precompute(2*n);
    return fact[2*n] * invFact[n+1] % MOD * invFact[n] % MOD;
}

// LC #343 — Integer Break
// Break n into integers summing to n, maximize product.
// Math insight: use as many 3s as possible; prefer 2 over 1.
public int integerBreak(int n) {
    if (n == 2) return 1;
    if (n == 3) return 2;
    int product = 1;
    while (n > 4) { product *= 3; n -= 3; }
    return product * n;
}

// LC #1012 — Numbers With Repeated Digits
// Count n-digit numbers with at least one repeated digit = n - (count without repeats)
public int numDupDigitsAtMostN(int n) {
    // Count numbers <= n with ALL distinct digits, subtract from n
    List<Integer> digits = new ArrayList<>();
    int temp = n+1;
    while (temp > 0) { digits.add(0, temp%10); temp /= 10; }
    int k = digits.size();

    // Count k-digit numbers with distinct digits
    int count = 0;
    // Numbers with fewer digits than k (all distinct)
    for (int i = 1; i < k; i++) {
        count += 9 * perm(9, i-1);  // First digit: 1-9, rest: permutations
    }

    // Numbers with exactly k digits and <= n
    Set<Integer> seen = new HashSet<>();
    for (int i = 0; i < k; i++) {
        int d = digits.get(i);
        int start = i == 0 ? 1 : 0;
        // Count digits smaller than d that haven't been used
        for (int j = start; j < d; j++) {
            if (!seen.contains(j)) count += perm(10-i-1, k-i-1);
        }
        if (seen.contains(d)) break;
        seen.add(d);
        if (i == k-1) count++;  // n itself has all distinct digits
    }
    return n - count;
}
int perm(int n, int k) {
    int p = 1;
    for (int i = 0; i < k; i++) p *= (n-i);
    return p;
}
```

---

# Part 5 — LeetCode Problems

# 21. Easy Problems

```java
// LC #1 — Two Sum (but using math)
// If sorted: two pointers. If any order: hash map.

// LC #7 — Reverse Integer
public int reverse(int x) {
    long rev = 0;
    while (x != 0) {
        rev = rev * 10 + x % 10;
        x /= 10;
    }
    return (rev > Integer.MAX_VALUE || rev < Integer.MIN_VALUE) ? 0 : (int)rev;
}

// LC #9 — Palindrome Number (shown above)

// LC #136 — Single Number (XOR, shown above)

// LC #168 — Excel Sheet Column Title (shown above)

// LC #172 — Factorial Trailing Zeroes
// Each trailing zero = one factor of 10 = one pair of (2,5).
// 2s always outnumber 5s in n!, so count factors of 5.
public int trailingZeroes(int n) {
    int count = 0;
    while (n > 0) { n /= 5; count += n; }
    return count;
}

// LC #202 — Happy Number (shown above)

// LC #231 — Power of Two (shown above)

// LC #258 — Add Digits (digital root, shown above)

// LC #326 — Power of Three
// 3^19 = 1162261467 is largest power of 3 fitting in int.
// n is power of 3 iff it divides 3^19.
public boolean isPowerOfThree(int n) {
    return n > 0 && 1162261467 % n == 0;
}

// LC #342 — Power of Four
public boolean isPowerOfFour(int n) {
    // n > 0, n is power of 2, single bit at even position
    return n > 0 && (n & (n-1)) == 0 && (n & 0xAAAAAAAA) == 0;
}
```

---

# 22. Medium Problems

```java
// LC #29 — Divide Two Integers (no multiplication/division operators)
public int divide(int dividend, int divisor) {
    if (dividend == Integer.MIN_VALUE && divisor == -1) return Integer.MAX_VALUE;
    long a = Math.abs((long)dividend), b = Math.abs((long)divisor);
    int result = 0;
    for (int i = 31; i >= 0; i--) {
        if ((a >> i) >= b) {
            result += 1 << i;
            a -= b << i;
        }
    }
    return (dividend > 0) == (divisor > 0) ? result : -result;
}

// LC #50 — Pow(x, n) (fast power, shown above)

// LC #166 — Fraction to Recurring Decimal
public String fractionToDecimal(int numerator, int denominator) {
    if (numerator == 0) return "0";
    StringBuilder sb = new StringBuilder();
    if ((numerator < 0) ^ (denominator < 0)) sb.append('-');
    long num = Math.abs((long)numerator), den = Math.abs((long)denominator);
    sb.append(num / den);
    long rem = num % den;
    if (rem == 0) return sb.toString();
    sb.append('.');
    Map<Long, Integer> map = new HashMap<>();
    while (rem != 0) {
        if (map.containsKey(rem)) {
            sb.insert(map.get(rem), '(');
            sb.append(')');
            break;
        }
        map.put(rem, sb.length());
        rem *= 10;
        sb.append(rem / den);
        rem %= den;
    }
    return sb.toString();
}

// LC #343 — Integer Break (shown above)

// LC #365 — Water and Jug Problem (shown above)

// LC #372 — Super Pow (shown above)

// LC #470 — Implement Rand10() Using Rand7()
// Rejection sampling: generate uniform random in [1,40] using two rand7 calls
public int rand10() {
    int row, col, idx;
    do {
        row = rand7(); col = rand7();
        idx = col + (row-1)*7;  // Uniform in [1,49]
    } while (idx > 40);
    return 1 + (idx-1) % 10;
}

// LC #523 — Continuous Subarray Sum (GCD related — multiple of k)
// Prefix sum mod k: if two prefix sums have same mod, subarray sum is multiple of k
public boolean checkSubarraySum(int[] nums, int k) {
    Map<Integer, Integer> modIdx = new HashMap<>();
    modIdx.put(0, -1);
    int sum = 0;
    for (int i = 0; i < nums.length; i++) {
        sum = (sum + nums[i]) % k;
        if (modIdx.containsKey(sum)) {
            if (i - modIdx.get(sum) >= 2) return true;
        } else modIdx.put(sum, i);
    }
    return false;
}

// LC #858 — Mirror Reflection
// Three mirrors. Light reflects. Which receptor does light reach?
// LCM(p,q)/p = number of vertical bounces. LCM(p,q)/q = number of horizontal.
// If bounces is even: receptor 0. Odd and horizontal-even: receptor 2. Else: 1.
public int mirrorReflection(int p, int q) {
    int l = lcm(p, q);
    int m = l / q % 2;  // Number of horizontal walls hit (mod 2)
    int n = l / p % 2;  // Number of vertical walls hit (mod 2)
    if (m == 1 && n == 0) return 0;
    if (m == 1 && n == 1) return 1;
    return 2;
}
int lcm(int a, int b) { return a / gcd(a, b) * b; }
int gcd(int a, int b) { return b == 0 ? a : gcd(b, a % b); }
```

---

# 23. Hard Problems

```java
// LC #149 — Max Points on a Line (shown above with GCD)

// LC #233 — Number of Digit One
// Count total 1s in digits from 1 to n
public int countDigitOne(int n) {
    int count = 0;
    for (long factor = 1; factor <= n; factor *= 10) {
        long divider = factor * 10;
        count += (n / divider) * factor;
        count += Math.min(Math.max(n % divider - factor + 1, 0), factor);
    }
    return count;
}

// LC #264 — Ugly Number II (next n ugly numbers: only 2,3,5 as prime factors)
public int nthUglyNumber(int n) {
    int[] ugly = new int[n];
    ugly[0] = 1;
    int i2=0, i3=0, i5=0;
    for (int i = 1; i < n; i++) {
        int next = Math.min(ugly[i2]*2, Math.min(ugly[i3]*3, ugly[i5]*5));
        ugly[i] = next;
        if (next == ugly[i2]*2) i2++;
        if (next == ugly[i3]*3) i3++;
        if (next == ugly[i5]*5) i5++;
    }
    return ugly[n-1];
}

// LC #780 — Reaching Points
// Can (sx,sy) reach (tx,ty) by operations: (x,y)→(x+y,y) or (x,y)→(x,x+y)?
// Work backwards from (tx,ty):
// If tx > ty: previous state was (tx-ty, ty) = (tx%ty, ty) if ty doesn't divide tx
// If ty > tx: previous state was (tx, ty-tx) = (tx, ty%tx) if tx doesn't divide ty
public boolean reachingPoints(int sx, int sy, int tx, int ty) {
    while (tx >= sx && ty >= sy) {
        if (tx == sx && ty == sy) return true;
        if (tx > ty) {
            if (ty == sy) return (tx - sx) % ty == 0;
            tx %= ty;
        } else {
            if (tx == sx) return (ty - sy) % tx == 0;
            ty %= tx;
        }
    }
    return false;
}

// LC #878 — Nth Magical Number
// A magical number is divisible by a or b. Find nth using binary search.
public int nthMagicalNumber(int n, int a, int b) {
    long MOD = 1_000_000_007L;
    long lcm = (long)a / gcd(a,b) * b;
    long lo = 0, hi = (long)n * Math.min(a,b);
    while (lo < hi) {
        long mid = lo + (hi-lo)/2;
        if (mid/a + mid/b - mid/lcm < n) lo = mid+1;
        else hi = mid;
    }
    return (int)(lo % MOD);
}
```

---

# 24. Complete LeetCode Math Problem List

## 🟢 Easy
| # | Problem | Concept |
|---|---------|---------|
| 7 | Reverse Integer | Digit manipulation |
| 9 | Palindrome Number | Digit manipulation |
| 13 | Roman to Integer | Greedy + lookup |
| 66 | Plus One | Carry propagation |
| 67 | Add Binary | Binary addition |
| 69 | Sqrt(x) | Binary search / Newton |
| 136 | Single Number | XOR trick |
| 168 | Excel Sheet Column Title | Base conversion |
| 171 | Excel Column Number | Base conversion |
| 172 | Factorial Trailing Zeroes | Count factor 5 |
| 202 | Happy Number | Cycle detection |
| 204 | Count Primes | Sieve |
| 231 | Power of Two | Bit trick |
| 258 | Add Digits | Digital root |
| 263 | Ugly Number | Prime factors |
| 292 | Nim Game | Pattern: n%4≠0 |
| 326 | Power of Three | Largest 3^k |
| 342 | Power of Four | Bit pattern |
| 367 | Valid Perfect Square | Binary search |
| 504 | Base 7 | Base conversion |
| 507 | Perfect Number | Factor sum |
| 1137 | Nth Tribonacci | DP |
| 1979 | Find GCD of Array | GCD |

## 🟡 Medium
| # | Problem | Concept |
|---|---------|---------|
| 29 | Divide Two Integers | Bit shift |
| 43 | Multiply Strings | Digit simulation |
| 50 | Pow(x, n) | Fast exponentiation |
| 62 | Unique Paths | Combinations |
| 96 | Unique BSTs | Catalan numbers |
| 118 | Pascal's Triangle | Combinations DP |
| 119 | Pascal's Triangle II | Combinations |
| 166 | Fraction to Recurring Decimal | Remainder map |
| 204 | Count Primes | Sieve |
| 264 | Ugly Number II | Three pointers |
| 279 | Perfect Squares | DP / BFS |
| 343 | Integer Break | Math greedy |
| 365 | Water and Jug Problem | GCD theorem |
| 372 | Super Pow | Fast power |
| 405 | Number to Hex | Bit masking |
| 415 | Add Strings | Digit simulation |
| 470 | Rand10 from Rand7 | Rejection sampling |
| 523 | Continuous Subarray Sum | Prefix mod |
| 528 | Random Pick with Weight | Prefix sum |
| 858 | Mirror Reflection | LCM |
| 878 | Nth Magical Number | Binary search + LCM |
| 952 | Largest Component by Factor | Union-Find + primes |
| 1015 | Smallest Integer Divisible by K | Modular |
| 1641 | Count Sorted Vowel Strings | Stars and bars |
| 2507 | Smallest Value After Sum Primes | Prime factorization |

## 🔴 Hard
| # | Problem | Concept |
|---|---------|---------|
| 149 | Max Points on a Line | GCD for slopes |
| 233 | Number of Digit One | Digit DP math |
| 273 | Integer to English Words | Digit groups |
| 780 | Reaching Points | Reverse + modulo |
| 793 | Preimage Size of Factorial | Binary search + math |
| 818 | Race Car | BFS / DP |
| 829 | Consecutive Numbers Sum | Math formula |
| 866 | Prime Palindrome | Sieve + palindrome |
| 1012 | Numbers With Repeated Digits | Digit DP |
| 1808 | Maximize Number of Nice Divisors | Fast power |
| 2338 | Count Ideal Arrays | Stars + bars + prime |

---

# Part 6 — Interview Mastery

# 25. How to Recognize Math Problems

```
KEYWORD → LIKELY MATH CONCEPT
────────────────────────────────────────────────────────────
"GCD" / "common factor" / "coprime"     → Euclidean algorithm
"prime" / "factor"                      → Sieve / trial division
"modulo" / "mod 10^9+7" / "large number"→ Modular arithmetic
"power" / "exponentiation" / "a^b"      → Fast exponentiation
"combinations" / "choose" / "C(n,k)"   → Combinatorics + Pascal
"paths in grid"                          → C(m+n-2, m-1) or DP
"trailing zeros"                         → Count factors of 5
"palindrome number"                      → Reverse half
"digit sum" / "repeated sum"            → Digital root formula
"power of 2/3/4"                        → Bit tricks
"base conversion"                        → Divide and collect remainders
"n choose k mod prime"                  → Factorial + modular inverse
"distribute n into k bins"              → Stars and bars: C(n+k-1,k-1)
"XOR trick" / "find single element"    → XOR properties
"number of divisors"                    → Prime factorization
"LCM" / "synchronized cycles"          → LCM = a*b/GCD

COMMON MATH IDENTITIES TO MEMORIZE:
  n*(n+1)/2 = sum 1..n
  n*(n+1)*(2n+1)/6 = sum of squares 1..n^2
  (n*(n+1)/2)^2 = sum of cubes 1..n^3
  GCD(a,b) * LCM(a,b) = a * b
  Fermat: a^(p-1) ≡ 1 (mod p) for prime p
  Catalan: C(2n,n)/(n+1)
  Trailing zeros in n!: floor(n/5) + floor(n/25) + floor(n/125) + ...
```

---

# 26. Math Interview Cheat Sheet

## ⚡ Must-Know Implementations

```java
// ============================================================
// 1. GCD (EUCLIDEAN)
// ============================================================
int gcd(int a, int b) { return b == 0 ? a : gcd(b, a % b); }
long lcm(long a, long b) { return a / gcd((int)a, (int)b) * b; }

// ============================================================
// 2. CHECK PRIME
// ============================================================
boolean isPrime(int n) {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;
    for (int i = 3; (long)i*i <= n; i += 2) if (n%i==0) return false;
    return true;
}

// ============================================================
// 3. SIEVE OF ERATOSTHENES
// ============================================================
boolean[] sieve(int n) {
    boolean[] p = new boolean[n+1];
    Arrays.fill(p, true);
    p[0]=p[1]=false;
    for (int i=2; (long)i*i<=n; i++) if (p[i]) for (int j=i*i; j<=n; j+=i) p[j]=false;
    return p;
}

// ============================================================
// 4. FAST MODULAR EXPONENTIATION
// ============================================================
long fastPow(long base, long exp, long mod) {
    long result = 1; base %= mod;
    while (exp > 0) {
        if ((exp&1)==1) result=result*base%mod;
        base=base*base%mod; exp>>=1;
    }
    return result;
}

// ============================================================
// 5. MODULAR INVERSE (Fermat, prime mod only)
// ============================================================
long modInverse(long a, long mod) { return fastPow(a, mod-2, mod); }

// ============================================================
// 6. PRECOMPUTE FACTORIALS + INVERSE FACTORIALS
// ============================================================
long MOD = 1_000_000_007L;
long[] fact, inv;
void precompute(int n) {
    fact = new long[n+1]; inv = new long[n+1];
    fact[0] = 1;
    for (int i=1;i<=n;i++) fact[i]=fact[i-1]*i%MOD;
    inv[n] = modInverse(fact[n], MOD);
    for (int i=n-1;i>=0;i--) inv[i]=inv[i+1]*(i+1)%MOD;
}
long comb(int n, int k) {
    if (k<0||k>n) return 0;
    return fact[n]*inv[k]%MOD*inv[n-k]%MOD;
}

// ============================================================
// 7. PRIME FACTORIZATION
// ============================================================
Map<Integer,Integer> primeFactors(int n) {
    Map<Integer,Integer> f = new TreeMap<>();
    for (int i=2;(long)i*i<=n;i++) while(n%i==0){f.merge(i,1,Integer::sum);n/=i;}
    if (n>1) f.merge(n,1,Integer::sum);
    return f;
}

// ============================================================
// 8. COMMON FORMULAS
// ============================================================
long sumTo(long n)       { return n*(n+1)/2; }        // 1+2+...+n
long triangular(long n)  { return n*(n+1)/2; }
boolean isPerfSq(long n) { long s=(long)Math.sqrt(n); return s*s==n||(s+1)*(s+1)==n; }
boolean isPow2(int n)    { return n>0 && (n&(n-1))==0; }
int digits(int n)        { return n==0?1:(int)Math.log10(n)+1; }
int digitSum(int n)      { int s=0; while(n>0){s+=n%10;n/=10;} return s; }
int digitalRoot(int n)   { return n==0?0:1+(n-1)%9; }
int trailingZeros(int n) { int c=0; while(n>=5){n/=5;c+=n;} return c; }
```

## ⚡ Modular Arithmetic Rules (Never Forget)

```java
// ALWAYS do this:
long ans = 0;
ans = (ans + x) % MOD;          // Addition
ans = (ans - x + MOD) % MOD;    // Subtraction — add MOD before %!
ans = ans * x % MOD;             // Multiplication — use long to avoid overflow
ans = ans * modInverse(x, MOD) % MOD;  // Division — use inverse, never ans/x%MOD

// NEVER do this:
ans = (ans - x) % MOD;    // ❌ Can be negative!
ans = ans / x % MOD;      // ❌ Wrong! Division doesn't distribute over mod.
ans = (int)(a*b) % MOD;   // ❌ int overflow before %!
```

## ⚡ The 5 Golden Rules of Math Problems

```
1. GCD: Always use Euclidean algorithm. O(log(min(a,b))).
   gcd(a, b) = gcd(b, a%b). Never implement without recursion/iteration.

2. PRIME CHECK: Only check up to √n. Skip even numbers after 2.
   For many primes up to N: use Sieve of Eratosthenes, NOT repeated isPrime().

3. MODULAR ARITHMETIC:
   Add MOD before subtracting (to avoid negative).
   Use long for multiplication before taking mod.
   Division = multiplication by modular inverse.
   Precompute factorials + inverse factorials for combination queries.

4. FAST POWER: Binary exponentiation for a^n in O(log n).
   Needed whenever you see "compute X^Y mod M" with large Y.

5. COMBINATORICS:
   Precompute fact[] and invFact[] once, then answer C(n,k) in O(1).
   C(n,k) = 0 when k<0 or k>n.
   Pascal's rule: C(n,k) = C(n-1,k-1) + C(n-1,k).
   Stars and bars: n into k bins = C(n+k-1, k-1).
```

---

> **Your 3-Week Learning Path:**
>
> Week 1 — Number Theory:
> GCD/LCM, primality, Sieve, prime factorization.
> Solve: LC #204, #1979, #365, #149, #172, #263, #264
>
> Week 2 — Modular Arithmetic + Fast Power:
> Implement fast power, modular inverse, precomputed C(n,k).
> Solve: LC #50, #372, #96 (Catalan), #62 (combinations), #1641
>
> Week 3 — Combinatorics + Patterns:
> Pascal's, stars and bars, digit manipulation, base conversion.
> Solve: LC #118, #119, #343, #279, #233, #780, #858
>
> The secret: Math problems are pattern recognition.
> Memorize 8 implementations (GCD, sieve, fast power, modular inverse,
> factorial precomputation, C(n,k), prime factorization, digital root).
> When you see a math problem, ask: which of these 8 does it need?

---

*This guide covers 70+ LeetCode math problems across 6 patterns — from basic digit manipulation to modular inverse and combinatorial counting.*
