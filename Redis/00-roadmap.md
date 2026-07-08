# Redis Complete Masterclass (for Node.js Developers)

A from-scratch-to-production Redis course, written for a Node.js/Express backend developer. Every module is explained in simple language, with real, fully-commented code examples.

## How to use this course

Go through the modules **in order** — each one builds on the previous. Do the homework at the end of each module before moving on; it's short by design, just enough to confirm the concept actually stuck.

## Modules

| # | Module | Level |
|---|---|---|
| 1 | [What is Redis?](./01-what-is-redis.md) | Beginner |
| 2 | [Why Redis Exists](./02-why-redis-exists.md) | Beginner |
| 3 | [Installation & Setup](./03-installation-setup.md) | Beginner |
| 4 | [Redis Data Types](./04-redis-data-types.md) | Beginner |
| 5 | [Redis Commands](./05-redis-commands.md) | Beginner |
| 6 | [How Redis Works Internally](./06-how-redis-works-internally.md) | Intermediate |
| 7 | [Memory Management](./07-memory-management.md) | Intermediate |
| 8 | [Persistence (RDB & AOF)](./08-persistence.md) | Intermediate |
| 9 | [Redis with Node.js](./09-redis-with-nodejs.md) | Intermediate |
| 10 | [Caching](./10-caching.md) | Intermediate |
| 11 | [Session Storage](./11-session-storage.md) | Intermediate |
| 12 | [Pub/Sub](./12-pubsub.md) | Intermediate |
| 13 | [Rate Limiting](./13-rate-limiting.md) | Advanced |
| 14 | [Distributed Locks](./14-distributed-locks.md) | Advanced |
| 15 | [Queues](./15-queues.md) | Advanced |
| 16 | [Redis Cluster & Replication](./16-cluster-and-replication.md) | Advanced |
| 17 | [Production Best Practices](./17-production-best-practices.md) | Advanced |
| 18 | [Interview Questions](./18-interview-questions.md) | Advanced |

## Stack used in examples

- **Node.js** + **Express.js**
- **ioredis** as the Redis client
- **express-session** + **connect-redis** for sessions (Module 11)
- **BullMQ** for production-grade queues (Module 15)
