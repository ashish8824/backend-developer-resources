# Java Threading & Multithreading — Complete Guide

---

## Table of Contents

1. [What is a Thread?](#1-what-is-a-thread)
2. [Process vs Thread](#2-process-vs-thread)
3. [Thread Lifecycle](#3-thread-lifecycle)
4. [Creating Threads](#4-creating-threads)
   - [Extending Thread Class](#41-extending-thread-class)
   - [Implementing Runnable Interface](#42-implementing-runnable-interface)
   - [Using Lambda (Java 8+)](#43-using-lambda-java-8)
   - [Using Callable & Future](#44-using-callable--future)
5. [Thread Methods](#5-thread-methods)
6. [Thread Synchronization](#6-thread-synchronization)
   - [The Race Condition Problem](#61-the-race-condition-problem)
   - [synchronized Keyword](#62-synchronized-keyword)
   - [synchronized Block](#63-synchronized-block)
7. [Inter-Thread Communication](#7-inter-thread-communication)
8. [Thread Priorities](#8-thread-priorities)
9. [Daemon Threads](#9-daemon-threads)
10. [Deadlock](#10-deadlock)
11. [volatile Keyword](#11-volatile-keyword)
12. [ThreadLocal](#12-threadlocal)
13. [Executor Framework](#13-executor-framework)
    - [ExecutorService](#131-executorservice)
    - [ScheduledExecutorService](#132-scheduledexecutorservice)
14. [Concurrent Collections](#14-concurrent-collections)
15. [Locks (java.util.concurrent.locks)](#15-locks-javautilconcurrentlocks)
16. [CountDownLatch, CyclicBarrier, Semaphore](#16-countdownlatch-cyclicbarrier-semaphore)
17. [CompletableFuture (Java 8+)](#17-completablefuture-java-8)
18. [Best Practices & Common Pitfalls](#18-best-practices--common-pitfalls)

---

## 1. What is a Thread?

A **thread** is the smallest unit of execution within a process. Think of it as a lightweight sub-process that shares the same memory space as other threads in the same process.

- A Java program always starts with **one thread** — the **main thread**.
- You can create additional threads to do work **concurrently**.

```
Program starts
     |
  main thread
     |
     |-------> Thread-1 (downloads a file)
     |-------> Thread-2 (updates UI)
     |-------> Thread-3 (sends analytics)
     |
  continues executing
```

---

## 2. Process vs Thread

| Feature        | Process                          | Thread                              |
|----------------|----------------------------------|-------------------------------------|
| Memory         | Own separate memory space        | Shares memory with other threads    |
| Communication  | IPC (pipes, sockets, etc.)       | Shared memory — direct & fast       |
| Creation cost  | Heavy (expensive)                | Lightweight (cheap)                 |
| Crash impact   | Isolated                         | One thread can crash the whole app  |
| Example        | Chrome tab, JVM instance         | Thread inside a Java app            |

---

## 3. Thread Lifecycle

A thread goes through these **states** during its lifetime:

```
NEW ──► RUNNABLE ──► RUNNING ──► TERMINATED
                        │
                        ▼
                    BLOCKED / WAITING / TIMED_WAITING
                        │
                        └──► back to RUNNABLE
```

| State            | Description                                                       |
|------------------|-------------------------------------------------------------------|
| `NEW`            | Thread created but `start()` not called yet                       |
| `RUNNABLE`       | Ready to run (waiting for CPU time)                               |
| `RUNNING`        | Currently executing on CPU                                        |
| `BLOCKED`        | Waiting for a monitor lock (e.g., trying to enter `synchronized`) |
| `WAITING`        | Waiting indefinitely — until notified (`wait()`, `join()`)        |
| `TIMED_WAITING`  | Waiting for a specific time (`sleep()`, `wait(timeout)`)          |
| `TERMINATED`     | Finished execution                                                |

```java
Thread t = new Thread(() -> System.out.println("Hello"));
System.out.println(t.getState()); // NEW

t.start();
System.out.println(t.getState()); // RUNNABLE or TERMINATED (very fast)
```

---

## 4. Creating Threads

### 4.1 Extending Thread Class

```java
class MyThread extends Thread {
    private String name;

    public MyThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println(name + " → Count: " + i);
            try {
                Thread.sleep(500); // pause 500ms
            } catch (InterruptedException e) {
                System.out.println(name + " was interrupted!");
            }
        }
    }
}

public class Main {
    public static void main(String[] args) {
        MyThread t1 = new MyThread("Thread-A");
        MyThread t2 = new MyThread("Thread-B");

        t1.start(); // ✅ Call start(), NOT run()
        t2.start(); // Both run concurrently
    }
}
```

**Output (order may vary):**
```
Thread-A → Count: 1
Thread-B → Count: 1
Thread-A → Count: 2
Thread-B → Count: 2
...
```

> ⚠️ **Calling `run()` directly does NOT create a new thread.** It just executes the method on the current thread.

---

### 4.2 Implementing Runnable Interface

This is **preferred** over extending Thread because:
- Java supports only single inheritance — extending Thread blocks other inheritance
- Separates task logic from thread management

```java
class DownloadTask implements Runnable {
    private String url;

    public DownloadTask(String url) {
        this.url = url;
    }

    @Override
    public void run() {
        System.out.println("Downloading: " + url + " on " + Thread.currentThread().getName());
        try {
            Thread.sleep(1000); // simulate download
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("Done: " + url);
    }
}

public class Main {
    public static void main(String[] args) {
        Runnable task1 = new DownloadTask("https://example.com/file1.zip");
        Runnable task2 = new DownloadTask("https://example.com/file2.zip");

        Thread t1 = new Thread(task1, "Downloader-1");
        Thread t2 = new Thread(task2, "Downloader-2");

        t1.start();
        t2.start();
    }
}
```

---

### 4.3 Using Lambda (Java 8+)

For short tasks, lambdas are concise and clean:

```java
public class Main {
    public static void main(String[] args) {
        // Lambda implements Runnable (functional interface)
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 3; i++) {
                System.out.println("Lambda Thread: " + i);
            }
        });

        Thread t2 = new Thread(() -> System.out.println("Quick task!"));

        t1.start();
        t2.start();
    }
}
```

---

### 4.4 Using Callable & Future

`Runnable.run()` returns **void**. If you need a **return value** from a thread, use `Callable<T>`:

```java
import java.util.concurrent.*;

public class Main {
    public static void main(String[] args) throws Exception {
        // Callable returns a result
        Callable<Integer> sumTask = () -> {
            int sum = 0;
            for (int i = 1; i <= 100; i++) sum += i;
            Thread.sleep(1000); // simulate work
            return sum;
        };

        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Integer> future = executor.submit(sumTask);

        System.out.println("Task submitted. Doing other work...");

        // future.get() blocks until result is ready
        Integer result = future.get();
        System.out.println("Sum = " + result); // Sum = 5050

        executor.shutdown();
    }
}
```

---

## 5. Thread Methods

```java
public class ThreadMethodsDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("Thread name: " + Thread.currentThread().getName());
            System.out.println("Thread ID: " + Thread.currentThread().getId());
            System.out.println("Is daemon: " + Thread.currentThread().isDaemon());
            System.out.println("Priority: " + Thread.currentThread().getPriority());
        }, "MyWorker");

        t.setPriority(Thread.MAX_PRIORITY); // 1–10, default is 5
        t.setDaemon(false);
        t.start();

        t.join(); // main thread waits for t to finish
        System.out.println("Worker thread finished. State: " + t.getState()); // TERMINATED
    }
}
```

| Method                  | Description                                           |
|-------------------------|-------------------------------------------------------|
| `start()`               | Start the thread (JVM calls `run()`)                  |
| `run()`                 | Entry point of thread logic (don't call directly)     |
| `sleep(ms)`             | Pause current thread for given milliseconds           |
| `join()`                | Wait for this thread to complete                      |
| `join(ms)`              | Wait at most `ms` milliseconds                        |
| `interrupt()`           | Set interrupt flag (thread must check/handle it)      |
| `isInterrupted()`       | Check if thread was interrupted                       |
| `isAlive()`             | Returns true if thread has started and not terminated |
| `getName()` / `setName()` | Get or set the thread name                          |
| `getPriority()` / `setPriority()` | Get or set priority (1–10)               |
| `getState()`            | Returns current `Thread.State` enum                   |
| `currentThread()`       | Static — returns reference to currently running thread|
| `yield()`               | Hint to scheduler to give up CPU (rarely used)        |

---

## 6. Thread Synchronization

### 6.1 The Race Condition Problem

When multiple threads access **shared mutable data** without coordination, you get **race conditions** — unpredictable, incorrect results.

```java
class Counter {
    int count = 0;

    void increment() {
        count++; // NOT atomic! 3 steps: read → add 1 → write
    }
}

public class RaceConditionDemo {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // Expected: 20000, Actual: random number < 20000 ❌
        System.out.println("Count: " + counter.count);
    }
}
```

---

### 6.2 synchronized Keyword

`synchronized` ensures **only one thread** executes the method at a time.

```java
class Counter {
    int count = 0;

    synchronized void increment() {  // ✅ Thread-safe
        count++;
    }

    synchronized int getCount() {
        return count;
    }
}

public class SyncDemo {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("Count: " + counter.getCount()); // Always 20000 ✅
    }
}
```

---

### 6.3 synchronized Block

Use a `synchronized` block to lock only a specific section (better performance):

```java
class BankAccount {
    private double balance;
    private final Object lock = new Object(); // explicit lock object

    public BankAccount(double initialBalance) {
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        synchronized (lock) { // only this block is locked, not the whole method
            System.out.println(Thread.currentThread().getName() + " depositing: " + amount);
            balance += amount;
        }
    }

    public void withdraw(double amount) {
        synchronized (lock) {
            if (balance >= amount) {
                System.out.println(Thread.currentThread().getName() + " withdrawing: " + amount);
                balance -= amount;
            } else {
                System.out.println("Insufficient funds!");
            }
        }
    }

    public double getBalance() {
        synchronized (lock) {
            return balance;
        }
    }
}

public class BankDemo {
    public static void main(String[] args) throws InterruptedException {
        BankAccount account = new BankAccount(1000.0);

        Thread t1 = new Thread(() -> account.deposit(500), "Depositor");
        Thread t2 = new Thread(() -> account.withdraw(300), "Withdrawer");

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("Final Balance: " + account.getBalance());
    }
}
```

---

## 7. Inter-Thread Communication

Threads can **communicate** using `wait()`, `notify()`, and `notifyAll()`.

Classic example — **Producer-Consumer**:

```java
import java.util.LinkedList;
import java.util.Queue;

class SharedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int CAPACITY = 5;

    public synchronized void produce(int item) throws InterruptedException {
        while (queue.size() == CAPACITY) {
            System.out.println("Buffer full! Producer waiting...");
            wait(); // release lock and wait
        }
        queue.add(item);
        System.out.println("Produced: " + item + " | Buffer size: " + queue.size());
        notifyAll(); // wake up waiting consumers
    }

    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) {
            System.out.println("Buffer empty! Consumer waiting...");
            wait(); // release lock and wait
        }
        int item = queue.poll();
        System.out.println("Consumed: " + item + " | Buffer size: " + queue.size());
        notifyAll(); // wake up waiting producers
        return item;
    }
}

public class ProducerConsumerDemo {
    public static void main(String[] args) {
        SharedBuffer buffer = new SharedBuffer();

        Thread producer = new Thread(() -> {
            for (int i = 1; i <= 10; i++) {
                try {
                    buffer.produce(i);
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Producer");

        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    buffer.consume();
                    Thread.sleep(300); // consumer is slower
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Consumer");

        producer.start();
        consumer.start();
    }
}
```

> **Key Rule:** `wait()`, `notify()`, and `notifyAll()` must be called from within a `synchronized` block/method, otherwise you get `IllegalMonitorStateException`.

---

## 8. Thread Priorities

Java threads have priority values from **1 (MIN) to 10 (MAX)**, default is **5 (NORM)**.

```java
public class PriorityDemo {
    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName()
                    + " [P:" + Thread.currentThread().getPriority() + "] → " + i);
            }
        };

        Thread low = new Thread(task, "LOW");
        Thread mid = new Thread(task, "MID");
        Thread high = new Thread(task, "HIGH");

        low.setPriority(Thread.MIN_PRIORITY);   // 1
        mid.setPriority(Thread.NORM_PRIORITY);  // 5
        high.setPriority(Thread.MAX_PRIORITY);  // 10

        low.start();
        mid.start();
        high.start();
    }
}
```

> ⚠️ **Priority is a hint to the OS scheduler, NOT a guarantee.** The JVM does not guarantee execution order based solely on priority.

---

## 9. Daemon Threads

**Daemon threads** are background threads that the JVM kills automatically when all **user threads** finish.

Use cases: garbage collector, log writers, heartbeat monitors.

```java
public class DaemonDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread daemon = new Thread(() -> {
            while (true) { // infinite loop!
                System.out.println("Daemon thread running...");
                try { Thread.sleep(500); } catch (InterruptedException e) { break; }
            }
        });

        daemon.setDaemon(true); // Must set BEFORE start()
        daemon.start();

        Thread.sleep(2000); // main thread runs for 2 seconds
        System.out.println("Main thread ending. Daemon will be killed automatically.");
        // JVM exits here, killing the daemon thread
    }
}
```

---

## 10. Deadlock

**Deadlock** happens when two (or more) threads are **waiting for each other** to release locks — forever.

```java
public class DeadlockDemo {
    static final Object LOCK_A = new Object();
    static final Object LOCK_B = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (LOCK_A) {
                System.out.println("T1: Holding A, waiting for B...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (LOCK_B) { // ← waits for LOCK_B held by T2
                    System.out.println("T1: Got both locks!");
                }
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            synchronized (LOCK_B) {
                System.out.println("T2: Holding B, waiting for A...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (LOCK_A) { // ← waits for LOCK_A held by T1
                    System.out.println("T2: Got both locks!");
                }
            }
        }, "Thread-2");

        t1.start();
        t2.start();
        // DEADLOCK — neither thread will ever proceed ❌
    }
}
```

**Prevention Strategies:**

```java
// ✅ Fix: Always acquire locks in the SAME ORDER
Thread t1 = new Thread(() -> {
    synchronized (LOCK_A) {       // Both threads: A first
        synchronized (LOCK_B) {   //               then B
            System.out.println("T1: Done!");
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (LOCK_A) {       // Same order ✅
        synchronized (LOCK_B) {
            System.out.println("T2: Done!");
        }
    }
});
```

---

## 11. volatile Keyword

`volatile` ensures a variable's value is always **read from main memory** (not from CPU cache), making changes visible across threads.

```java
public class VolatileDemo {
    // Without volatile, thread may never see updated value due to CPU caching
    private static volatile boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            System.out.println("Worker started.");
            while (running) {
                // keep running
            }
            System.out.println("Worker stopped.");
        });

        worker.start();
        Thread.sleep(1000);

        running = false; // visible to worker thread immediately because of volatile
        System.out.println("Stop signal sent.");
    }
}
```

> `volatile` guarantees **visibility** but NOT **atomicity**. Use `AtomicInteger` etc. for compound operations like `count++`.

---

## 12. ThreadLocal

`ThreadLocal` gives each thread its **own isolated copy** of a variable — no sharing, no synchronization needed.

```java
public class ThreadLocalDemo {
    // Each thread gets its own copy of this variable
    private static final ThreadLocal<String> userContext = new ThreadLocal<>();

    static void processRequest(String user) {
        userContext.set(user); // stored per-thread
        System.out.println(Thread.currentThread().getName() + " processing: " + userContext.get());
        try { Thread.sleep(500); } catch (InterruptedException e) {}
        System.out.println(Thread.currentThread().getName() + " done for: " + userContext.get());
        userContext.remove(); // ✅ Always clean up to prevent memory leaks
    }

    public static void main(String[] args) {
        new Thread(() -> processRequest("Alice"), "Thread-Alice").start();
        new Thread(() -> processRequest("Bob"), "Thread-Bob").start();
        new Thread(() -> processRequest("Charlie"), "Thread-Charlie").start();
    }
}
```

**Output:**
```
Thread-Alice processing: Alice
Thread-Bob processing: Bob
Thread-Charlie processing: Charlie
Thread-Alice done for: Alice     (no cross-contamination!)
Thread-Bob done for: Bob
Thread-Charlie done for: Charlie
```

---

## 13. Executor Framework

Instead of manually managing threads, use the **Executor Framework** (`java.util.concurrent`). It provides a **thread pool** — reusing threads instead of creating/destroying them.

### 13.1 ExecutorService

```java
import java.util.concurrent.*;

public class ExecutorDemo {
    public static void main(String[] args) throws InterruptedException {
        // Fixed thread pool: max 3 threads at a time
        ExecutorService executor = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 8; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " started by " + Thread.currentThread().getName());
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
                System.out.println("Task " + taskId + " finished.");
            });
        }

        executor.shutdown(); // stop accepting new tasks
        executor.awaitTermination(30, TimeUnit.SECONDS); // wait for all to finish
        System.out.println("All tasks done.");
    }
}
```

**Common Executors:**

| Factory Method                      | Description                                        |
|-------------------------------------|----------------------------------------------------|
| `newFixedThreadPool(n)`             | Pool of `n` threads                                |
| `newCachedThreadPool()`             | Grows as needed, reuses idle threads               |
| `newSingleThreadExecutor()`         | One thread, tasks execute sequentially             |
| `newScheduledThreadPool(n)`         | For delayed/periodic tasks                         |
| `newWorkStealingPool()`             | Fork/Join based, uses all available processors     |

---

### 13.2 ScheduledExecutorService

```java
import java.util.concurrent.*;

public class ScheduledDemo {
    public static void main(String[] args) throws InterruptedException {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

        // Run once after 2 seconds delay
        scheduler.schedule(() -> System.out.println("Delayed task ran!"), 2, TimeUnit.SECONDS);

        // Run every 1 second starting after 0 seconds
        scheduler.scheduleAtFixedRate(() ->
            System.out.println("Heartbeat: " + System.currentTimeMillis()),
            0, 1, TimeUnit.SECONDS);

        Thread.sleep(5000);
        scheduler.shutdown();
    }
}
```

---

## 14. Concurrent Collections

Regular collections (`ArrayList`, `HashMap`) are **NOT thread-safe**. Use these alternatives:

| Regular              | Thread-Safe Equivalent              | Notes                                   |
|----------------------|-------------------------------------|-----------------------------------------|
| `ArrayList`          | `CopyOnWriteArrayList`              | Reads fast, writes make a copy          |
| `HashMap`            | `ConcurrentHashMap`                 | Fine-grained locking per segment        |
| `LinkedList` (queue) | `LinkedBlockingQueue`               | Blocks on empty get / full put          |
| `HashSet`            | `CopyOnWriteArraySet`               | Backed by CopyOnWriteArrayList          |
| `PriorityQueue`      | `PriorityBlockingQueue`             | Thread-safe priority queue              |

```java
import java.util.concurrent.*;

public class ConcurrentCollectionDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        Thread writer1 = new Thread(() -> {
            for (int i = 0; i < 5; i++) map.put("Key-" + i, i);
        });

        Thread writer2 = new Thread(() -> {
            for (int i = 5; i < 10; i++) map.put("Key-" + i, i);
        });

        writer1.start();
        writer2.start();
        writer1.join();
        writer2.join();

        System.out.println("Map size: " + map.size()); // 10 ✅
        map.forEach((k, v) -> System.out.println(k + " = " + v));
    }
}
```

---

## 15. Locks (java.util.concurrent.locks)

`ReentrantLock` gives more control than `synchronized`:

```java
import java.util.concurrent.locks.*;

class SafeCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock(); // acquire lock
        try {
            count++;
        } finally {
            lock.unlock(); // ALWAYS unlock in finally!
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}

public class LockDemo {
    public static void main(String[] args) throws InterruptedException {
        SafeCounter counter = new SafeCounter();
        ExecutorService executor = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 10000; i++) {
            executor.submit(counter::increment);
        }

        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("Count: " + counter.getCount()); // Always 10000 ✅
    }
}
```

**ReentrantReadWriteLock** — allows **multiple readers** or **one writer** at a time:

```java
import java.util.concurrent.locks.*;

class Cache {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private String data = "initial";

    public String read() {
        rwLock.readLock().lock(); // Multiple readers can hold this simultaneously
        try {
            return data;
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void write(String newData) {
        rwLock.writeLock().lock(); // Only one writer, blocks all readers
        try {
            data = newData;
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

---

## 16. CountDownLatch, CyclicBarrier, Semaphore

### CountDownLatch — Wait until N events complete

```java
import java.util.concurrent.*;

public class LatchDemo {
    public static void main(String[] args) throws InterruptedException {
        int workers = 3;
        CountDownLatch latch = new CountDownLatch(workers);

        for (int i = 1; i <= workers; i++) {
            final int id = i;
            new Thread(() -> {
                System.out.println("Worker " + id + " doing setup...");
                try { Thread.sleep((long)(Math.random() * 2000)); } catch (InterruptedException e) {}
                System.out.println("Worker " + id + " ready!");
                latch.countDown(); // decrement counter
            }).start();
        }

        System.out.println("Main thread waiting for all workers...");
        latch.await(); // blocks until count reaches 0
        System.out.println("All workers ready! Starting application.");
    }
}
```

---

### CyclicBarrier — All threads wait at a checkpoint

```java
import java.util.concurrent.*;

public class BarrierDemo {
    public static void main(String[] args) {
        int parties = 3;
        CyclicBarrier barrier = new CyclicBarrier(parties, () ->
            System.out.println("=== All threads reached barrier! Proceeding... ===\n"));

        for (int i = 1; i <= parties; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    System.out.println("Phase 1: Thread " + id + " working...");
                    Thread.sleep((long)(Math.random() * 1000));
                    System.out.println("Thread " + id + " reached barrier.");
                    barrier.await(); // wait for everyone

                    System.out.println("Phase 2: Thread " + id + " continuing...");
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

---

### Semaphore — Limit concurrent access

```java
import java.util.concurrent.*;

public class SemaphoreDemo {
    // Only 2 threads can use the "database connection" at once
    static Semaphore semaphore = new Semaphore(2);

    static void accessDatabase(int threadId) throws InterruptedException {
        System.out.println("Thread " + threadId + " waiting for connection...");
        semaphore.acquire(); // blocks if no permits available
        System.out.println("Thread " + threadId + " got connection!");
        Thread.sleep(2000); // simulate DB work
        System.out.println("Thread " + threadId + " releasing connection.");
        semaphore.release();
    }

    public static void main(String[] args) {
        for (int i = 1; i <= 5; i++) {
            final int id = i;
            new Thread(() -> {
                try { accessDatabase(id); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }).start();
        }
    }
}
```

---

## 17. CompletableFuture (Java 8+)

`CompletableFuture` enables **asynchronous, non-blocking** programming with chaining:

```java
import java.util.concurrent.*;

public class CompletableFutureDemo {
    public static void main(String[] args) throws Exception {

        // 1. Run async task
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("Fetching user data...");
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            return "UserData{name=Alice, id=42}";
        });

        // 2. Chain transformations
        CompletableFuture<String> processed = future
            .thenApply(data -> {
                System.out.println("Processing: " + data);
                return data.toUpperCase(); // transform result
            })
            .thenApply(data -> "RESULT: " + data);

        // 3. Handle errors
        CompletableFuture<String> safe = processed
            .exceptionally(ex -> "Error: " + ex.getMessage());

        // 4. Get final result
        System.out.println(safe.get());

        // ─────────────────────────────────────────
        // Combine two futures
        CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> "Alice");
        CompletableFuture<Integer> ageFuture = CompletableFuture.supplyAsync(() -> 30);

        CompletableFuture<String> combined = userFuture.thenCombine(ageFuture,
            (name, age) -> name + " is " + age + " years old");

        System.out.println(combined.get()); // Alice is 30 years old

        // ─────────────────────────────────────────
        // Wait for all
        CompletableFuture<Void> all = CompletableFuture.allOf(userFuture, ageFuture);
        all.join(); // block until all complete
        System.out.println("All futures done.");
    }
}
```

---

## 18. Best Practices & Common Pitfalls

### ✅ Best Practices

```java
// 1. Use ExecutorService instead of creating raw threads
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// 2. Always shut down executors
executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);

// 3. Always release locks in finally
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // ← never forget this
}

// 4. Use AtomicInteger for simple counters (no need for synchronized)
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // atomic, thread-safe

// 5. Catch InterruptedException and restore interrupt flag
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // ← restore the flag
    return; // exit the task
}

// 6. Prefer Concurrent collections over synchronized wrappers
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>(); // ✅
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>()); // ❌ less efficient
```

### ❌ Common Pitfalls

```java
// ❌ Calling run() instead of start()
thread.run();   // NOT a new thread, runs on current thread!
thread.start(); // ✅ This creates a new thread

// ❌ Synchronizing on non-final objects
Object lock = new Object();
synchronized (lock) { // ❌ if 'lock' is reassigned, threads use different monitors!
}

// ❌ Ignoring InterruptedException
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // ❌ silently swallowing it
}

// ❌ Not cleaning up ThreadLocal
threadLocal.set(value);
// ... forget to call threadLocal.remove() → memory leak in thread pools!
```

### Summary Table

| Use Case                               | Recommended Tool                  |
|----------------------------------------|-----------------------------------|
| Simple background task                 | `Thread` + `Runnable`             |
| Task with return value                 | `Callable` + `Future`             |
| Managing many tasks                    | `ExecutorService`                 |
| Async chaining / pipelines             | `CompletableFuture`               |
| Periodic tasks                         | `ScheduledExecutorService`        |
| Simple thread-safe counter             | `AtomicInteger`                   |
| Thread-safe map                        | `ConcurrentHashMap`               |
| Wait for N tasks to complete           | `CountDownLatch`                  |
| Sync N threads at a checkpoint         | `CyclicBarrier`                   |
| Limit concurrent access to a resource | `Semaphore`                       |
| Thread-specific data                   | `ThreadLocal`                     |
| Advanced locking control               | `ReentrantLock` / `ReadWriteLock` |

---

*Happy Coding! Threading mastery comes with practice — start simple and build up.* 🧵
