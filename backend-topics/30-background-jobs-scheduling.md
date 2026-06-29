# Background Jobs & Task Scheduling

## What are Background Jobs?

Background jobs are tasks that run asynchronously, outside the request-response cycle. Instead of making the user wait for a time-consuming operation, the task is queued and processed independently.

```
Without Background Jobs:
User uploads 1000 product CSV -> Server processes all 1000 rows synchronously
User waits 60 seconds -> Timeout or very poor UX

With Background Jobs:
User uploads 1000 product CSV -> Server queues the job -> Response: "Processing started"
                              -> Background worker processes CSV asynchronously
                              -> User notified when done (email / WebSocket)
Total user wait: <1 second
```

## Types of Background Processing

### 1. Immediate Background Jobs (Async Tasks)

Tasks triggered by a user action, processed in the background without making the user wait.

```
Examples:
- Send welcome email after signup
- Resize uploaded image to multiple dimensions
- Generate PDF invoice after order placed
- Send push notification
- Update search index after content change
```

### 2. Scheduled Jobs (Cron Jobs)

Tasks that run at specific times or intervals, independent of user actions.

```
Examples:
- Daily: send digest emails at 8 AM
- Hourly: sync data with external API
- Every 5 minutes: check for expired sessions and clean up
- Weekly: generate analytics reports
- Monthly: charge subscription billing
- Midnight: archive old records to cold storage
```

### 3. Delayed Jobs

Tasks that should run after a specific delay.

```
Examples:
- Send follow-up email 3 days after signup
- Cancel an unpaid order after 30 minutes
- Send reminder 1 hour before appointment
- Retry failed payment after 24 hours
```

## Spring Boot: @Async (Simple Background Tasks)

For simple fire-and-forget tasks within the same JVM.

```java
// Enable async
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);       // always-on threads
        executor.setMaxPoolSize(20);       // burst capacity
        executor.setQueueCapacity(100);    // queue before creating new threads
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// Async service method
@Service
public class EmailService {

    @Async
    public CompletableFuture<Void> sendWelcomeEmail(String email, String name) {
        // runs in separate thread from the pool
        log.info("Sending welcome email to {}", email);
        emailProvider.send(email, "Welcome!", buildWelcomeTemplate(name));
        return CompletableFuture.completedFuture(null);
    }

    @Async
    public CompletableFuture<String> generateReport(Long reportId) {
        // Long-running async task returning a result
        String result = reportGenerator.generate(reportId);
        return CompletableFuture.completedFuture(result);
    }
}

// Usage — non-blocking, returns immediately
@PostMapping("/register")
public ResponseEntity<?> register(@RequestBody RegisterRequest req) {
    User user = userService.createUser(req);
    emailService.sendWelcomeEmail(user.getEmail(), user.getName());  // async, non-blocking
    return ResponseEntity.ok("Registration successful");
}
```

**Limitations of @Async:**
- Task is lost if the server restarts (no persistence)
- No retry on failure
- Can't be scheduled or delayed
- For simple one-off tasks only

## Spring Boot: @Scheduled (Cron Jobs)

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {}

@Component
public class ScheduledTasks {

    // Fixed rate: every 5 minutes regardless of execution time
    @Scheduled(fixedRate = 5 * 60 * 1000)
    public void refreshCache() {
        log.info("Refreshing cache...");
        cacheService.refreshHotProducts();
    }

    // Fixed delay: 10 minutes AFTER previous execution completes
    @Scheduled(fixedDelay = 10 * 60 * 1000)
    public void processOutboxEvents() {
        outboxService.publishPendingEvents();
    }

    // Cron expression
    @Scheduled(cron = "0 0 8 * * *")          // 8:00 AM every day
    public void sendDailyDigest() {
        log.info("Sending daily digest emails...");
        emailService.sendDailyDigest();
    }

    @Scheduled(cron = "0 0 1 * * MON")        // 1:00 AM every Monday
    public void generateWeeklyReport() {
        reportService.generateWeeklyReport();
    }

    @Scheduled(cron = "0 0 2 1 * *")          // 2:00 AM on 1st of every month
    public void monthlyBilling() {
        billingService.processMonthlySubscriptions();
    }

    @Scheduled(cron = "0 */30 * * * *")       // every 30 minutes
    public void cleanupExpiredSessions() {
        sessionService.deleteExpiredSessions();
    }
}
```

### Cron Expression Reference

```
@Scheduled(cron = "second minute hour day month weekday")

Second:   0-59
Minute:   0-59
Hour:     0-23
Day:      1-31 (or ?)
Month:    1-12 (or JAN-DEC)
Weekday:  0-7 (0/7=SUN, or MON-SUN) (or ?)

Examples:
"0 0 * * * *"       -> every hour at :00
"0 */15 * * * *"    -> every 15 minutes
"0 0 9-17 * * MON-FRI" -> every hour 9AM-5PM weekdays
"0 0 0 * * *"       -> midnight every day
"0 0 0 1 1 *"       -> midnight New Year's Day
```

## Distributed Scheduling (Avoid Duplicate Execution)

Problem: if 3 instances of your app run, `@Scheduled` fires on ALL 3 instances — the job runs 3 times.

### Solution 1: ShedLock (Most Popular)

ShedLock uses a DB/Redis lock to ensure only one node runs a scheduled task at a time.

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-redis-spring</artifactId>
    <version>5.10.0</version>
</dependency>
```

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulingConfig {

    @Bean
    public LockProvider lockProvider(RedisConnectionFactory connectionFactory) {
        return new RedisLockProvider(connectionFactory, "myapp");
    }
}

@Component
public class ScheduledTasks {

    @Scheduled(cron = "0 0 8 * * *")
    @SchedulerLock(
        name = "sendDailyDigest",
        lockAtLeastFor = "5m",   // hold lock for minimum 5 min (prevent double-run on short tasks)
        lockAtMostFor = "15m"    // release lock after 15 min even if node dies
    )
    public void sendDailyDigest() {
        // Only ONE node executes this — ShedLock ensures it
        emailService.sendDailyDigest();
    }
}
```

### Solution 2: Dedicated Worker Service

Only one service/pod is responsible for scheduling — no distributed lock needed.

```yaml
# Deploy scheduling separately
# scheduler-service: replicas: 1  (always single instance)
# api-service: replicas: 5       (horizontally scaled)
```

## Job Queue with Redis (Bull/BullMQ — Node.js)

For Node.js, BullMQ is the standard solution for background job processing with Redis.

### Setup

```bash
npm install bullmq ioredis
```

### Producer (Add Jobs to Queue)

```javascript
const { Queue } = require('bullmq');
const { Redis } = require('ioredis');

const connection = new Redis({ host: 'localhost', port: 6379, maxRetriesPerRequest: null });

// Create queues
const emailQueue    = new Queue('emails', { connection });
const reportQueue   = new Queue('reports', { connection });
const imageQueue    = new Queue('images', { connection });

// Add immediate job
async function queueWelcomeEmail(userId, email) {
    await emailQueue.add('welcome', { userId, email }, {
        attempts: 3,           // retry up to 3 times on failure
        backoff: {
            type: 'exponential',
            delay: 2000        // 2s, 4s, 8s
        }
    });
}

// Add delayed job (runs after delay)
async function queueFollowUpEmail(userId, email) {
    await emailQueue.add('followup', { userId, email }, {
        delay: 3 * 24 * 60 * 60 * 1000  // 3 days
    });
}

// Add scheduled (repeating) job
async function scheduleReport() {
    await reportQueue.add('weekly', {}, {
        repeat: { cron: '0 0 9 * * MON' }  // 9 AM every Monday
    });
}

// Add job with priority (lower number = higher priority)
async function queueUrgentEmail(userId, email) {
    await emailQueue.add('urgent', { userId, email }, {
        priority: 1  // highest priority
    });
}
```

### Worker (Process Jobs)

```javascript
const { Worker } = require('bullmq');

const emailWorker = new Worker('emails', async (job) => {
    const { userId, email } = job.data;

    switch (job.name) {
        case 'welcome':
            await sendWelcomeEmail(email);
            break;
        case 'followup':
            await sendFollowUpEmail(email);
            break;
        case 'urgent':
            await sendUrgentEmail(email);
            break;
    }

    // Return value is saved as job result
    return { sent: true, timestamp: new Date().toISOString() };

}, {
    connection,
    concurrency: 5,    // process 5 jobs simultaneously
    limiter: {
        max: 100,      // max 100 jobs per duration
        duration: 1000 // per second
    }
});

// Event handlers
emailWorker.on('completed', (job, result) => {
    console.log(`Job ${job.id} completed:`, result);
});

emailWorker.on('failed', (job, error) => {
    console.error(`Job ${job.id} failed:`, error.message);
    // After all retries exhausted, job moves to failed list
});

emailWorker.on('progress', (job, progress) => {
    console.log(`Job ${job.id} is ${progress}% done`);
});
```

### Reporting Progress

```javascript
const imageWorker = new Worker('images', async (job) => {
    const images = job.data.images;
    const total = images.length;

    for (let i = 0; i < images.length; i++) {
        await resizeImage(images[i]);
        await job.updateProgress(Math.round(((i + 1) / total) * 100));
    }

    return { processed: total };
}, { connection });
```

### Job Dashboard (Bull Board)

```javascript
const { createBullBoard } = require('@bull-board/api');
const { BullMQAdapter } = require('@bull-board/api/bullMQAdapter');
const { ExpressAdapter } = require('@bull-board/express');

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
    queues: [
        new BullMQAdapter(emailQueue),
        new BullMQAdapter(reportQueue),
        new BullMQAdapter(imageQueue)
    ],
    serverAdapter
});

app.use('/admin/queues', serverAdapter.getRouter());
// Visit http://localhost:3000/admin/queues for visual dashboard
```

## Spring Boot Job Queue with Redis (Lettuce + Custom)

```java
@Service
public class JobQueueService {

    @Autowired
    private StringRedisTemplate redis;

    private static final String EMAIL_QUEUE = "queue:emails";

    // Enqueue job
    public void enqueueEmailJob(EmailJobData data) {
        String payload = objectMapper.writeValueAsString(data);
        redis.opsForList().rightPush(EMAIL_QUEUE, payload);
    }

    // Worker - processes jobs
    @Async
    public void processEmailQueue() {
        while (true) {
            // BLPOP: blocking pop - waits up to 30s for a job
            List<String> result = redis.opsForList()
                    .leftPop(EMAIL_QUEUE, Duration.ofSeconds(30));

            if (result != null && !result.isEmpty()) {
                EmailJobData job = objectMapper.readValue(result.get(1), EmailJobData.class);
                try {
                    emailService.send(job);
                } catch (Exception e) {
                    // Retry: push back to queue or dead letter
                    redis.opsForList().rightPush("queue:emails:failed", result.get(1));
                }
            }
        }
    }
}
```

## Job Patterns

### Job Chaining (Pipeline)

```javascript
// Each step runs after the previous completes
const flow = new FlowProducer({ connection });

await flow.add({
    name: 'process-order',
    queueName: 'orders',
    data: { orderId: 'ORD-123' },
    children: [
        { name: 'charge-payment', queueName: 'payments', data: { orderId: 'ORD-123' } },
        { name: 'reserve-inventory', queueName: 'inventory', data: { orderId: 'ORD-123' } },
        { name: 'send-confirmation', queueName: 'emails', data: { orderId: 'ORD-123' } }
    ]
});
// Parent job waits for all children to complete
```

### Dead Letter Queue

```javascript
// After max retries, failed jobs move to a 'failed' list automatically in BullMQ
// Query failed jobs:
const failedJobs = await emailQueue.getFailed();
for (const job of failedJobs) {
    console.log(job.id, job.failedReason, job.attemptsMade);
    await job.retry();  // manually retry
}
```

## Interview Questions

**Q1: Why use background jobs instead of processing synchronously in the request?**

Synchronous processing blocks the response until completion. Long tasks (sending emails, processing files, generating reports) make users wait seconds or minutes — poor UX and timeout risk. Background jobs let the server respond instantly ("job queued"), process the task asynchronously, and notify the user on completion. Scales better too — workers can be scaled independently from API servers.

**Q2: What is the problem with @Scheduled in a multi-instance deployment and how do you fix it?**

`@Scheduled` fires on every instance — if you have 5 pods, the job runs 5 times simultaneously. This causes duplicate emails sent, reports generated multiple times, and data inconsistency. Fix with ShedLock (acquires a distributed lock before running, only one instance proceeds) or dedicate a single scheduling service (scaled to 1 replica).

**Q3: What is the difference between fixedRate and fixedDelay in @Scheduled?**

`fixedRate` triggers every N milliseconds from the start of the previous execution — if the job takes 2s and rate is 5s, next run starts 3s after the previous completes. `fixedDelay` waits N milliseconds after the previous execution COMPLETES before starting the next — guarantees N ms gap between runs. Use `fixedDelay` for jobs that shouldn't overlap if they run longer than expected.

**Q4: How do you handle retries and failures in a job queue?**

Configure max attempts and backoff strategy (exponential backoff: retry after 2s, 4s, 8s, 16s). After all retries exhausted, move the job to a dead-letter queue (failed list). Monitor the DLQ, alert on failures, allow manual replay. BullMQ handles this automatically; for custom Redis queues, implement retry logic in the worker with a counter tracked on the job.

**Q5: How do you implement a delayed job (run job after N hours)?**

BullMQ/Bull: add job with `{ delay: N_milliseconds }`. The queue holds it until the delay expires, then makes it available for workers. In Redis directly: use a Sorted Set with the execution timestamp as the score. A scheduler polls for jobs with score <= now() and enqueues them. In Spring: `@Scheduled` with initial delay, or use a proper job scheduler like Quartz for complex scheduling needs.
