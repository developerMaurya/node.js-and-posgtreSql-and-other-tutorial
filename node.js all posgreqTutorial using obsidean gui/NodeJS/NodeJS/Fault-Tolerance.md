# Fault Tolerance Patterns in Node.js

A comprehensive guide to implementing fault tolerance patterns in Node.js applications.

## 1. Retry Pattern

### Exponential Backoff Implementation
```javascript
class RetryManager {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.baseDelay = options.baseDelay || 1000;
    this.maxDelay = options.maxDelay || 30000;
    this.exponential = options.exponential || 2;
  }

  async execute(operation) {
    let lastError;
    let retryCount = 0;

    while (retryCount <= this.maxRetries) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        retryCount++;

        if (retryCount <= this.maxRetries) {
          const delay = this.calculateDelay(retryCount);
          console.log(`Retry ${retryCount} after ${delay}ms`);
          await this.sleep(delay);
        }
      }
    }

    throw new Error(`Operation failed after ${retryCount} retries: ${lastError.message}`);
  }

  calculateDelay(retryCount) {
    const delay = Math.min(
      this.maxDelay,
      this.baseDelay * Math.pow(this.exponential, retryCount - 1)
    );
    return delay + Math.random() * delay * 0.1; // Add jitter
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage example
const retryManager = new RetryManager({
  maxRetries: 3,
  baseDelay: 1000,
  maxDelay: 30000
});

async function makeHttpRequest() {
  return retryManager.execute(async () => {
    const response = await fetch('https://api.example.com/data');
    if (!response.ok) throw new Error(`HTTP error: ${response.status}`);
    return response.json();
  });
}
```

## 2. Bulkhead Pattern

### Service Isolation
```javascript
class BulkheadManager {
  constructor(options = {}) {
    this.pools = new Map();
    this.defaultOptions = {
      maxConcurrent: 10,
      queueSize: 100,
      timeout: 5000
    };
  }

  createPool(serviceName, options = {}) {
    const poolOptions = { ...this.defaultOptions, ...options };
    const pool = {
      executing: 0,
      queue: [],
      ...poolOptions
    };
    this.pools.set(serviceName, pool);
  }

  async execute(serviceName, operation) {
    const pool = this.pools.get(serviceName);
    if (!pool) {
      throw new Error(`No pool found for service: ${serviceName}`);
    }

    if (pool.executing >= pool.maxConcurrent) {
      if (pool.queue.length >= pool.queueSize) {
        throw new Error('Service queue full');
      }
      
      await new Promise((resolve, reject) => {
        const timeoutId = setTimeout(() => {
          const index = pool.queue.indexOf(queueItem);
          if (index !== -1) {
            pool.queue.splice(index, 1);
            reject(new Error('Queue timeout'));
          }
        }, pool.timeout);

        const queueItem = { resolve, reject, timeoutId };
        pool.queue.push(queueItem);
      });
    }

    pool.executing++;
    try {
      return await operation();
    } finally {
      pool.executing--;
      if (pool.queue.length > 0) {
        const { resolve, reject, timeoutId } = pool.queue.shift();
        clearTimeout(timeoutId);
        resolve();
      }
    }
  }
}

// Usage example
const bulkhead = new BulkheadManager();

bulkhead.createPool('database', {
  maxConcurrent: 5,
  queueSize: 50,
  timeout: 10000
});

bulkhead.createPool('externalApi', {
  maxConcurrent: 3,
  queueSize: 20,
  timeout: 5000
});

async function makeIsolatedRequest() {
  return bulkhead.execute('externalApi', async () => {
    const response = await fetch('https://api.example.com/data');
    return response.json();
  });
}
```

## 3. Fallback Pattern

### Service Fallback Implementation
```javascript
class FallbackManager {
  constructor() {
    this.fallbacks = new Map();
    this.metrics = new Map();
  }

  register(serviceName, primaryFn, fallbackFn, options = {}) {
    this.fallbacks.set(serviceName, {
      primary: primaryFn,
      fallback: fallbackFn,
      threshold: options.threshold || 0.5,
      windowSize: options.windowSize || 100,
      minimumAttempts: options.minimumAttempts || 10
    });
    
    this.metrics.set(serviceName, {
      attempts: 0,
      failures: 0,
      window: []
    });
  }

  async execute(serviceName, ...args) {
    const service = this.fallbacks.get(serviceName);
    if (!service) {
      throw new Error(`No service registered: ${serviceName}`);
    }

    const metrics = this.metrics.get(serviceName);
    const shouldUseFallback = this.shouldFallback(metrics, service);

    try {
      if (shouldUseFallback) {
        return await service.fallback(...args);
      }
      
      const result = await service.primary(...args);
      this.recordSuccess(metrics);
      return result;
    } catch (error) {
      this.recordFailure(metrics);
      
      if (!shouldUseFallback) {
        try {
          return await service.fallback(...args);
        } catch (fallbackError) {
          throw new Error(`Both primary and fallback failed: ${error.message}, ${fallbackError.message}`);
        }
      }
      
      throw error;
    }
  }

  shouldFallback(metrics, service) {
    if (metrics.attempts < service.minimumAttempts) {
      return false;
    }

    this.updateWindow(metrics);
    const failureRate = metrics.window.filter(x => !x).length / metrics.window.length;
    return failureRate >= service.threshold;
  }

  updateWindow(metrics) {
    while (metrics.window.length >= service.windowSize) {
      metrics.window.shift();
    }
  }

  recordSuccess(metrics) {
    metrics.attempts++;
    metrics.window.push(true);
  }

  recordFailure(metrics) {
    metrics.attempts++;
    metrics.failures++;
    metrics.window.push(false);
  }
}

// Usage example
const fallbackManager = new FallbackManager();

fallbackManager.register(
  'userService',
  async (userId) => {
    // Primary service call
    const response = await fetch(`https://api.example.com/users/${userId}`);
    return response.json();
  },
  async (userId) => {
    // Fallback service call
    return { id: userId, name: 'Unknown', cached: true };
  },
  {
    threshold: 0.3,
    windowSize: 50,
    minimumAttempts: 5
  }
);
```

## 4. Rate Limiting

### Token Bucket Implementation
```javascript
class TokenBucket {
  constructor(options = {}) {
    this.capacity = options.capacity || 60;
    this.fillPerSecond = options.fillPerSecond || 10;
    this.tokens = this.capacity;
    this.lastFill = Date.now();
  }

  async consume(tokens = 1) {
    this.refill();

    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }

    return false;
  }

  refill() {
    const now = Date.now();
    const deltaSeconds = (now - this.lastFill) / 1000;
    this.tokens = Math.min(
      this.capacity,
      this.tokens + deltaSeconds * this.fillPerSecond
    );
    this.lastFill = now;
  }
}

// Distributed rate limiter with Redis
class DistributedRateLimiter {
  constructor(redis, options = {}) {
    this.redis = redis;
    this.window = options.window || 60;
    this.limit = options.limit || 100;
  }

  async consume(key, tokens = 1) {
    const now = Date.now();
    const windowKey = `ratelimit:${key}:${Math.floor(now / (this.window * 1000))}`;

    const multi = this.redis.multi();
    multi.incrby(windowKey, tokens);
    multi.expire(windowKey, this.window);

    const [count] = await multi.exec();
    return count <= this.limit;
  }
}
```

## 5. Dead Letter Queue

### Message Recovery Implementation
```javascript
class DeadLetterQueue {
  constructor(options = {}) {
    this.mainQueue = options.mainQueue;
    this.dlqQueue = options.dlqQueue;
    this.maxRetries = options.maxRetries || 3;
    this.retryDelay = options.retryDelay || 1000;
  }

  async processMessage(message) {
    try {
      await this.mainQueue.process(message);
    } catch (error) {
      await this.handleFailure(message, error);
    }
  }

  async handleFailure(message, error) {
    const retryCount = (message.retryCount || 0) + 1;
    
    if (retryCount <= this.maxRetries) {
      message.retryCount = retryCount;
      message.lastError = error.message;
      message.nextRetry = Date.now() + (this.retryDelay * Math.pow(2, retryCount - 1));

      await this.mainQueue.delay(message, message.nextRetry);
    } else {
      await this.moveToDeadLetter(message, error);
    }
  }

  async moveToDeadLetter(message, error) {
    const dlqMessage = {
      original: message,
      error: error.message,
      timestamp: Date.now(),
      retryCount: message.retryCount
    };

    await this.dlqQueue.add(dlqMessage);
  }

  async retryDeadLetter(messageId) {
    const message = await this.dlqQueue.get(messageId);
    if (message) {
      message.original.retryCount = 0;
      await this.mainQueue.add(message.original);
      await this.dlqQueue.remove(messageId);
    }
  }
}
```

## Related Topics
- [[High-Availability]] - High availability implementation
- [[Circuit-Breaker]] - Circuit breaker pattern
- [[Error-Handling-Advanced]] - Error handling strategies
- [[Monitoring]] - System monitoring

## Practice Projects
1. Implement distributed rate limiter
2. Build message recovery system
3. Create service fallback system
4. Design bulkhead isolation

## Resources
- [Node.js Error Handling](https://nodejs.org/api/errors.html)
- [Resilience Patterns](https://docs.microsoft.com/azure/architecture/patterns/category/resiliency)
- [[Learning-Resources#FaultTolerance|Fault Tolerance Resources]]

## Tags
#fault-tolerance #nodejs #resilience #error-handling #patterns
