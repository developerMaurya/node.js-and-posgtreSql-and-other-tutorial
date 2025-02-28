# Traffic Management in Node.js

A comprehensive guide to implementing traffic management strategies in Node.js applications.

## 1. Rate Limiting

### Rate Limiter Implementation
```javascript
class RateLimiter {
  constructor(options = {}) {
    this.options = {
      windowMs: options.windowMs || 60000,
      maxRequests: options.maxRequests || 100,
      strategy: options.strategy || 'sliding-window',
      ...options
    };
    
    this.store = new Map();
  }

  async isAllowed(key) {
    const now = Date.now();
    const windowStart = now - this.options.windowMs;
    
    // Clean old requests
    this.cleanOldRequests(key, windowStart);
    
    // Get current requests
    const requests = this.store.get(key) || [];
    
    // Check limit
    if (requests.length >= this.options.maxRequests) {
      return false;
    }
    
    // Add new request
    requests.push(now);
    this.store.set(key, requests);
    
    return true;
  }

  cleanOldRequests(key, windowStart) {
    const requests = this.store.get(key);
    if (!requests) return;
    
    const validRequests = requests.filter(timestamp => 
      timestamp > windowStart
    );
    
    if (validRequests.length === 0) {
      this.store.delete(key);
    } else {
      this.store.set(key, validRequests);
    }
  }

  getMiddleware() {
    return async (req, res, next) => {
      const key = this.getKey(req);
      const allowed = await this.isAllowed(key);
      
      if (!allowed) {
        return res.status(429).json({
          error: 'Too Many Requests',
          retryAfter: this.getRetryAfter(key)
        });
      }
      
      next();
    };
  }

  getKey(req) {
    return req.ip;
  }

  getRetryAfter(key) {
    const requests = this.store.get(key) || [];
    if (requests.length === 0) return 0;
    
    const oldestRequest = requests[0];
    return Math.max(0, this.options.windowMs - (Date.now() - oldestRequest));
  }
}
```

## 2. Circuit Breaker

### Circuit Breaker Implementation
```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.options = {
      failureThreshold: options.failureThreshold || 5,
      resetTimeout: options.resetTimeout || 60000,
      halfOpenTimeout: options.halfOpenTimeout || 30000,
      ...options
    };
    
    this.state = 'closed';
    this.failures = 0;
    this.lastFailureTime = null;
    this.executionTimeout = options.executionTimeout || 10000;
  }

  async execute(command) {
    if (!this.canExecute()) {
      throw new Error('Circuit is open');
    }
    
    try {
      const result = await this.executeWithTimeout(command);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  canExecute() {
    if (this.state === 'closed') {
      return true;
    }
    
    if (this.state === 'open') {
      const now = Date.now();
      if (now - this.lastFailureTime >= this.options.resetTimeout) {
        this.state = 'half-open';
        return true;
      }
      return false;
    }
    
    // Half-open state
    return true;
  }

  async executeWithTimeout(command) {
    return Promise.race([
      command(),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Timeout')), this.executionTimeout)
      )
    ]);
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'closed';
  }

  onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();
    
    if (this.failures >= this.options.failureThreshold) {
      this.state = 'open';
    }
  }
}
```

## 3. Load Shedding

### Load Shedder Implementation
```javascript
class LoadShedder {
  constructor(options = {}) {
    this.options = {
      cpuThreshold: options.cpuThreshold || 80,
      memoryThreshold: options.memoryThreshold || 85,
      requestQueueSize: options.requestQueueSize || 1000,
      ...options
    };
    
    this.requestQueue = [];
    this.metrics = new MetricsCollector();
  }

  async handle(request) {
    if (this.shouldShed()) {
      throw new Error('Service overloaded');
    }
    
    if (this.requestQueue.length >= this.options.requestQueueSize) {
      throw new Error('Request queue full');
    }
    
    return new Promise((resolve, reject) => {
      this.requestQueue.push({
        request,
        resolve,
        reject,
        timestamp: Date.now()
      });
    });
  }

  shouldShed() {
    const metrics = this.metrics.getCurrentMetrics();
    
    return metrics.cpu > this.options.cpuThreshold ||
           metrics.memory > this.options.memoryThreshold;
  }

  async processQueue() {
    while (this.requestQueue.length > 0) {
      const { request, resolve, reject, timestamp } = this.requestQueue.shift();
      
      // Check if request is too old
      if (Date.now() - timestamp > 5000) {
        reject(new Error('Request timeout'));
        continue;
      }
      
      try {
        const result = await this.processRequest(request);
        resolve(result);
      } catch (error) {
        reject(error);
      }
    }
  }

  async processRequest(request) {
    // Implement request processing
  }
}
```

## 4. Traffic Shaping

### Traffic Shaper Implementation
```javascript
class TrafficShaper {
  constructor(options = {}) {
    this.options = {
      tokensPerSecond: options.tokensPerSecond || 100,
      burstSize: options.burstSize || 10,
      ...options
    };
    
    this.tokens = this.options.burstSize;
    this.lastRefill = Date.now();
  }

  async acquire(tokens = 1) {
    this.refillTokens();
    
    if (this.tokens < tokens) {
      return false;
    }
    
    this.tokens -= tokens;
    return true;
  }

  refillTokens() {
    const now = Date.now();
    const timePassed = (now - this.lastRefill) / 1000;
    const newTokens = timePassed * this.options.tokensPerSecond;
    
    this.tokens = Math.min(
      this.options.burstSize,
      this.tokens + newTokens
    );
    
    this.lastRefill = now;
  }

  getMiddleware() {
    return async (req, res, next) => {
      const required = this.getRequiredTokens(req);
      const allowed = await this.acquire(required);
      
      if (!allowed) {
        return res.status(429).json({
          error: 'Rate limit exceeded'
        });
      }
      
      next();
    };
  }

  getRequiredTokens(req) {
    // Implement custom token calculation based on request
    return 1;
  }
}
```

## 5. Request Prioritization

### Priority Queue Implementation
```javascript
class PriorityQueue {
  constructor(options = {}) {
    this.options = {
      maxSize: options.maxSize || 1000,
      defaultPriority: options.defaultPriority || 5,
      ...options
    };
    
    this.queues = new Map();
    for (let i = 1; i <= 10; i++) {
      this.queues.set(i, []);
    }
  }

  async enqueue(request, priority = this.options.defaultPriority) {
    const queue = this.queues.get(priority);
    if (!queue) {
      throw new Error('Invalid priority');
    }
    
    if (this.getTotalSize() >= this.options.maxSize) {
      throw new Error('Queue is full');
    }
    
    return new Promise((resolve, reject) => {
      queue.push({
        request,
        resolve,
        reject,
        timestamp: Date.now()
      });
    });
  }

  async processQueues() {
    for (let priority = 1; priority <= 10; priority++) {
      const queue = this.queues.get(priority);
      
      while (queue.length > 0) {
        const { request, resolve, reject, timestamp } = queue.shift();
        
        // Check if request is too old
        if (Date.now() - timestamp > 5000) {
          reject(new Error('Request timeout'));
          continue;
        }
        
        try {
          const result = await this.processRequest(request);
          resolve(result);
        } catch (error) {
          reject(error);
        }
      }
    }
  }

  getTotalSize() {
    return Array.from(this.queues.values())
      .reduce((total, queue) => total + queue.length, 0);
  }

  async processRequest(request) {
    // Implement request processing
  }
}
```

## Related Topics
- [[Load-Balancing]] - Advanced Load Balancing
- [[High-Availability]] - High Availability
- [[Global-Scaling]] - Global Scale Architecture
- [[Multi-Region]] - Multi-Region Deployment

## Practice Projects
1. Build rate limiting system
2. Implement circuit breaker
3. Create load shedding system
4. Design traffic shaping

## Resources
- [Traffic Management Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/category/resiliency)
- [[Learning-Resources#TrafficManagement|Traffic Management Resources]]

## Tags
#traffic-management #nodejs #rate-limiting #circuit-breaker #load-shedding
