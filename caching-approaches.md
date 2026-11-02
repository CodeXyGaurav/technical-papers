# Caching Approaches: Fixing Performance and Scaling Issues

## Introduction

I recently joined a new project that is facing performance and scaling issues. After some analysis, the team found that the same data is fetched from the database again and again, and response times grow as the number of users increases. My team lead asked me to investigate different caching approaches. This paper explains what caching is, compares the most common caching strategies with code samples, and lists the problems we need to watch for. All code samples are written in JavaScript.

## What Is Caching?

A cache is a fast, temporary storage layer that keeps a copy of data that is expensive to fetch or compute. When the data is found in the cache, it is called a cache hit. When it is not found, it is called a cache miss, and the application must go to the original source, such as a database or an external API.

Caching helps us in three ways:

1. It reduces response time, because memory is much faster than disk or network calls.
2. It reduces the load on the database, so the system can serve more users.
3. It lowers cost, because fewer expensive queries and API calls are made.

## Where Can We Cache?

Caching can happen at several layers. A large system usually uses more than one.

* Browser cache: Stores static files such as images, CSS and JavaScript on the user's device.
* CDN cache: Stores content on servers that are close to the user.
* Application cache: Stores data in the memory of the application process.
* Distributed cache: Stores data in a shared service such as Redis or Memcached, so that many servers can use the same cache.
* Database cache: Stores query results or frequently used pages inside the database engine.

## Caching Strategies

The strategies below describe how the application, the cache and the database work together. For the samples, I use a simple cache class with a time-to-live (TTL) and a fake database.

```javascript
class SimpleCache {
  constructor() {
    this.store = new Map();
  }

  get(key) {
    const entry = this.store.get(key);
    if (!entry) return undefined;

    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value;
  }

  set(key, value, ttlMs = 60000) {
    this.store.set(key, { value, expiresAt: Date.now() + ttlMs });
  }

  delete(key) {
    this.store.delete(key);
  }
}

const database = {
  users: { 1: { id: 1, name: "Asha" } },

  async getUser(id) {
    console.log("Reading from the database");
    return this.users[id];
  },

  async updateUser(id, data) {
    console.log("Writing to the database");
    this.users[id] = { ...this.users[id], ...data };
    return this.users[id];
  },
};
```

### 1. Cache-Aside (Lazy Loading)

The application is responsible for everything. It checks the cache first. On a miss, it reads from the database and then stores the result in the cache. On a write, it updates the database and removes the old cache entry.

```javascript
const cache = new SimpleCache();

async function getUser(id) {
  const cached = cache.get(`user:${id}`);
  if (cached) return cached; // Cache hit

  const user = await database.getUser(id); // Cache miss
  cache.set(`user:${id}`, user);
  return user;
}

async function updateUser(id, data) {
  const user = await database.updateUser(id, data);
  cache.delete(`user:${id}`); // Invalidate the old entry
  return user;
}
```

* Good for: Read-heavy data such as user profiles and product pages.
* Advantages: It is simple, and only data that is really requested is cached.
* Disadvantages: The first request is always slow, and the data can become stale if invalidation is missed.

### 2. Read-Through

The application only talks to the cache. If the data is missing, the cache itself loads it from the database. The loading logic lives in one place instead of being spread across the application.

```javascript
class ReadThroughCache {
  constructor(loader, ttlMs = 60000) {
    this.cache = new SimpleCache();
    this.loader = loader;
    this.ttlMs = ttlMs;
  }

  async get(key) {
    const cached = this.cache.get(key);
    if (cached) return cached;

    const value = await this.loader(key); // The cache loads the data
    this.cache.set(key, value, this.ttlMs);
    return value;
  }
}

const userCache = new ReadThroughCache((id) => database.getUser(id));
userCache.get(1);
```

* Good for: Systems where many parts of the code read the same data.
* Advantages: The application code is cleaner.
* Disadvantages: The first request is still slow, and the cache needs to know how to reach the database.

### 3. Write-Through

Every write goes to the cache and to the database at the same time. The cache is always up to date.

```javascript
async function updateUserWriteThrough(id, data) {
  const user = await database.updateUser(id, data);
  cache.set(`user:${id}`, user); // Cache and database stay in sync
  return user;
}
```

* Good for: Data that must stay consistent, such as account balances or inventory counts.
* Advantages: Reads are fast and the data is fresh.
* Disadvantages: Writes are slower because two places must be updated, and we may cache data that is never read.

### 4. Write-Back (Write-Behind)

The application writes only to the cache, and the cache updates the database later in the background, often in batches.

```javascript
const pendingWrites = new Map();

function updateUserWriteBack(id, data) {
  const current = cache.get(`user:${id}`) || {};
  const updated = { ...current, ...data };
  cache.set(`user:${id}`, updated);
  pendingWrites.set(id, updated); // Saved to the database later
  return updated;
}

// Flush the queued writes every five seconds
setInterval(async () => {
  for (const [id, user] of pendingWrites) {
    await database.updateUser(id, user);
    pendingWrites.delete(id);
  }
}, 5000);
```

* Good for: Write-heavy workloads such as logs, analytics counters and activity feeds.
* Advantages: Writes are very fast, and many updates can be combined into one database call.
* Disadvantages: If the cache crashes before the flush, the data is lost, so it is risky for important records.

### 5. Write-Around

Writes go straight to the database and skip the cache. The cache is filled only when the data is read later.

```javascript
async function createLog(entry) {
  return database.updateUser(entry.id, entry); // Cache is not touched
}
```

* Good for: Data that is written often but rarely read, such as logs.
* Advantages: The cache is not filled with data that nobody reads.
* Disadvantages: The first read after a write is a cache miss.

## Quick Comparison

1. Cache-aside: Read-heavy systems, simple to build, risk of stale data.
2. Read-through: Same as cache-aside, but the cache loads the data.
3. Write-through: Strong consistency, slower writes.
4. Write-back: Fastest writes, risk of data loss.
5. Write-around: Good for rarely read data, slow first read.

In real systems, we can mix strategies. For example, user profiles can use cache-aside, inventory can use write-through, and analytics counters can use write-back.

## Eviction Policies

A cache has limited memory, so it must remove old entries. Common policies are:

* LRU (Least Recently Used): Removes the entry that has not been used for the longest time.
* LFU (Least Frequently Used): Removes the entry that is used the least number of times.
* FIFO (First In, First Out): Removes the oldest entry.
* TTL (Time To Live): Removes an entry after a fixed time.

A small LRU cache can be written with a JavaScript Map, because a Map remembers the insertion order.

```javascript
class LRUCache {
  constructor(limit) {
    this.limit = limit;
    this.map = new Map();
  }

  get(key) {
    if (!this.map.has(key)) return undefined;

    const value = this.map.get(key);
    this.map.delete(key);
    this.map.set(key, value); // Move to the most recent position
    return value;
  }

  set(key, value) {
    if (this.map.has(key)) this.map.delete(key);
    this.map.set(key, value);

    if (this.map.size > this.limit) {
      const oldestKey = this.map.keys().next().value;
      this.map.delete(oldestKey); // Remove the least recently used entry
    }
  }
}
```

## Common Problems to Watch For

1. Stale data: The cache shows old data after the database has changed. Use TTL and invalidate entries on every write.
2. Cache stampede: A popular key expires and many requests hit the database at the same time. We can add a small random value to the TTL or let only one request reload the key.
3. Cache penetration: Requests for keys that do not exist always miss the cache. We can cache the empty result for a short time.
4. Inconsistency between servers: In-process caches on different servers can hold different values. A shared cache such as Redis avoids this problem.
5. Wrong data in the cache: Caching data that changes every second, or data that is different for every user, gives little benefit.

## My Recommendation for Our Project

1. Measure first. Find the slowest and most repeated queries before adding any cache.
2. Start with cache-aside using Redis for read-heavy data, with a sensible TTL.
3. Use write-through only for the few records that must always be consistent.
4. Add a cache hit ratio metric, so that we know whether the cache is really helping.
5. Add caching step by step and test after each step.

## Conclusion

Caching is one of the most effective ways to improve performance and scaling, but it adds complexity, because we must keep the cached data correct. There is no single best strategy. The right choice depends on how often the data is read, how often it changes, and how much stale data the business can accept. For our project, I suggest starting with cache-aside and adding other strategies only where the measurements show a need.

## References

* [Top Caching Strategies - ByteByteGo](https://blog.bytebytego.com/p/top-caching-strategies)
* [Caching Strategies: Cache-Aside, Write-Through, and Write-Behind - Codemia](https://codemia.io/courses/system_design_fundamentals/caching_strategies)
* [The Latency vs Complexity Tradeoffs with 6 Caching Strategies - ScyllaDB](https://resources.scylladb.com/books/the-latency-vs-complexity-tradeoffs-with-6-caching-strategies)
* [Caching Strategies to Know - DragonflyDB](https://www.dragonflydb.io/guides/caching-strategies-to-know)
* [How Does Caching Work and What Are Common Caching Strategies in System Design - DesignGurus](https://www.designgurus.io/answers/detail/how-does-caching-work-and-what-are-common-caching-strategies-in-system-design)
* [Technical Communication Instructions - Mountblue Life Skills Track](https://github.com/mountblue/life-skills-track/blob/main/technical-communication.md)
