# System Architecture in Node.js

A comprehensive guide to implementing robust system architectures in Node.js applications.

## 1. System Architecture Foundations

### Core Architecture Components
```javascript
// System Configuration Management
class SystemConfig {
  constructor(options = {}) {
    this.options = {
      environment: options.environment || 'development',
      logLevel: options.logLevel || 'info',
      ...options
    };
    
    this.configurations = new Map();
    this.loadConfigurations();
  }

  loadConfigurations() {
    // Load environment-specific configurations
    const envConfig = require(`./config/${this.options.environment}`);
    this.configurations.set('database', envConfig.database);
    this.configurations.set('services', envConfig.services);
    this.configurations.set('cache', envConfig.cache);
  }

  get(key) {
    if (!this.configurations.has(key)) {
      throw new Error(`Configuration not found: ${key}`);
    }
    return this.configurations.get(key);
  }

  set(key, value) {
    this.configurations.set(key, value);
  }
}

// System Health Monitoring
class HealthMonitor {
  constructor(subsystems) {
    this.subsystems = subsystems;
    this.healthChecks = new Map();
    this.setupHealthChecks();
  }

  setupHealthChecks() {
    for (const [name, subsystem] of Object.entries(this.subsystems)) {
      this.healthChecks.set(name, async () => {
        try {
          await subsystem.checkHealth();
          return { status: 'healthy' };
        } catch (error) {
          return {
            status: 'unhealthy',
            error: error.message
          };
        }
      });
    }
  }

  async checkHealth() {
    const results = {};
    
    for (const [name, check] of this.healthChecks) {
      results[name] = await check();
    }
    
    return {
      timestamp: new Date(),
      status: this.getOverallStatus(results),
      subsystems: results
    };
  }

  getOverallStatus(results) {
    return Object.values(results).every(
      r => r.status === 'healthy'
    ) ? 'healthy' : 'unhealthy';
  }
}

// System Metrics Collection
class MetricsCollector {
  constructor(options = {}) {
    this.options = {
      interval: options.interval || 5000,
      retention: options.retention || 3600000,
      ...options
    };
    
    this.metrics = new Map();
    this.startCollection();
  }

  startCollection() {
    setInterval(() => {
      this.collectMetrics();
    }, this.options.interval);
  }

  async collectMetrics() {
    const metrics = {
      timestamp: Date.now(),
      system: await this.collectSystemMetrics(),
      process: await this.collectProcessMetrics(),
      custom: await this.collectCustomMetrics()
    };
    
    this.storeMetrics(metrics);
  }

  async collectSystemMetrics() {
    return {
      memory: process.memoryUsage(),
      cpu: process.cpuUsage(),
      uptime: process.uptime()
    };
  }

  async collectProcessMetrics() {
    return {
      activeHandles: process._getActiveHandles().length,
      activeRequests: process._getActiveRequests().length,
      eventLoopLag: await this.measureEventLoopLag()
    };
  }

  async measureEventLoopLag() {
    const start = Date.now();
    return new Promise(resolve => {
      setImmediate(() => {
        resolve(Date.now() - start);
      });
    });
  }

  storeMetrics(metrics) {
    const key = metrics.timestamp;
    this.metrics.set(key, metrics);
    
    // Cleanup old metrics
    const cutoff = Date.now() - this.options.retention;
    for (const [timestamp] of this.metrics) {
      if (timestamp < cutoff) {
        this.metrics.delete(timestamp);
      }
    }
  }

  getMetrics(duration) {
    const cutoff = Date.now() - (duration || this.options.retention);
    return Array.from(this.metrics.values())
      .filter(m => m.timestamp >= cutoff);
  }
}
```

## 2. System Integration Patterns

### Message-Based Integration
```javascript
// Message Broker Integration
class MessageBroker {
  constructor(options = {}) {
    this.options = {
      retryAttempts: options.retryAttempts || 3,
      retryDelay: options.retryDelay || 1000,
      ...options
    };
    
    this.channels = new Map();
    this.handlers = new Map();
  }

  async publish(channel, message) {
    if (!this.channels.has(channel)) {
      await this.createChannel(channel);
    }
    
    return this.channels.get(channel).publish(
      Buffer.from(JSON.stringify(message))
    );
  }

  async subscribe(channel, handler) {
    if (!this.channels.has(channel)) {
      await this.createChannel(channel);
    }
    
    this.handlers.set(channel, handler);
    await this.channels.get(channel).consume(
      async (message) => {
        try {
          const content = JSON.parse(message.content.toString());
          await handler(content);
          message.ack();
        } catch (error) {
          await this.handleError(channel, message, error);
        }
      }
    );
  }

  async handleError(channel, message, error) {
    const retryCount = (message.properties.headers.retryCount || 0) + 1;
    
    if (retryCount <= this.options.retryAttempts) {
      // Retry with delay
      setTimeout(() => {
        this.channels.get(channel).publish(
          message.content,
          {
            headers: { retryCount }
          }
        );
      }, this.options.retryDelay * retryCount);
      
      message.ack();
    } else {
      // Move to dead letter queue
      await this.publishToDLQ(channel, message);
      message.ack();
    }
  }

  async createChannel(name) {
    const channel = await this.connection.createChannel();
    await channel.assertQueue(name, { durable: true });
    this.channels.set(name, channel);
  }

  async publishToDLQ(originalChannel, message) {
    const dlqChannel = `${originalChannel}.dlq`;
    if (!this.channels.has(dlqChannel)) {
      await this.createChannel(dlqChannel);
    }
    
    return this.channels.get(dlqChannel).publish(
      message.content,
      {
        headers: {
          originalChannel,
          error: 'Max retry attempts exceeded'
        }
      }
    );
  }
}

// Event Bus Implementation
class EventBus {
  constructor() {
    this.subscribers = new Map();
  }

  subscribe(eventType, handler) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, new Set());
    }
    
    this.subscribers.get(eventType).add(handler);
    
    return () => {
      this.subscribers.get(eventType).delete(handler);
    };
  }

  async publish(event) {
    const eventType = event.constructor.name;
    const handlers = this.subscribers.get(eventType) || new Set();
    
    const promises = Array.from(handlers).map(handler =>
      handler(event).catch(error => {
        console.error(
          `Error handling event ${eventType}:`,
          error
        );
      })
    );
    
    await Promise.all(promises);
  }
}

```

## 3. Scalability Patterns

### Load Distribution
```javascript
// Load Balancer Implementation
class LoadBalancer {
  constructor(options = {}) {
    this.options = {
      strategy: options.strategy || 'round-robin',
      healthCheck: options.healthCheck || (() => true),
      ...options
    };
    
    this.servers = new Map();
    this.currentIndex = 0;
  }

  addServer(server) {
    this.servers.set(server.id, {
      ...server,
      healthy: true,
      load: 0
    });
  }

  removeServer(serverId) {
    this.servers.delete(serverId);
  }

  async getServer() {
    const healthyServers = Array.from(this.servers.values())
      .filter(s => s.healthy);
    
    if (healthyServers.length === 0) {
      throw new Error('No healthy servers available');
    }
    
    switch (this.options.strategy) {
      case 'round-robin':
        return this.roundRobin(healthyServers);
      case 'least-connections':
        return this.leastConnections(healthyServers);
      case 'random':
        return this.random(healthyServers);
      default:
        return this.roundRobin(healthyServers);
    }
  }

  roundRobin(servers) {
    const server = servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % servers.length;
    return server;
  }

  leastConnections(servers) {
    return servers.reduce((min, server) => 
      server.load < min.load ? server : min
    );
  }

  random(servers) {
    const index = Math.floor(Math.random() * servers.length);
    return servers[index];
  }

  async checkHealth() {
    for (const [id, server] of this.servers) {
      try {
        const healthy = await this.options.healthCheck(server);
        this.servers.set(id, { ...server, healthy });
      } catch (error) {
        this.servers.set(id, { ...server, healthy: false });
      }
    }
  }
}

// Rate Limiter Implementation
class RateLimiter {
  constructor(options = {}) {
    this.options = {
      window: options.window || 60000,
      limit: options.limit || 100,
      ...options
    };
    
    this.requests = new Map();
  }

  async checkLimit(key) {
    const now = Date.now();
    const windowStart = now - this.options.window;
    
    // Clean old requests
    this.cleanup(key, windowStart);
    
    // Get current requests in window
    const requests = this.requests.get(key) || [];
    
    if (requests.length >= this.options.limit) {
      return {
        allowed: false,
        reset: requests[0] + this.options.window
      };
    }
    
    // Add new request
    requests.push(now);
    this.requests.set(key, requests);
    
    return {
      allowed: true,
      remaining: this.options.limit - requests.length
    };
  }

  cleanup(key, windowStart) {
    const requests = this.requests.get(key);
    if (!requests) return;
    
    const valid = requests.filter(time => time > windowStart);
    if (valid.length > 0) {
      this.requests.set(key, valid);
    } else {
      this.requests.delete(key);
    }
  }
}

```

## 4. Resilience Patterns

### Fault Tolerance
```javascript
// Circuit Breaker Implementation
class CircuitBreaker {
  constructor(options = {}) {
    this.options = {
      failureThreshold: options.failureThreshold || 5,
      resetTimeout: options.resetTimeout || 60000,
      ...options
    };
    
    this.state = 'closed';
    this.failures = 0;
    this.lastFailureTime = null;
  }

  async execute(command) {
    if (this.state === 'open') {
      if (this.shouldReset()) {
        this.halfOpen();
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    try {
      const result = await command();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  shouldReset() {
    return Date.now() - this.lastFailureTime > this.options.resetTimeout;
  }

  halfOpen() {
    this.state = 'half-open';
    this.failures = 0;
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

// Bulkhead Implementation
class Bulkhead {
  constructor(options = {}) {
    this.options = {
      maxConcurrent: options.maxConcurrent || 10,
      maxQueued: options.maxQueued || 20,
      ...options
    };
    
    this.executing = 0;
    this.queue = [];
  }

  async execute(command) {
    if (this.executing >= this.options.maxConcurrent) {
      return this.enqueue(command);
    }
    
    return this.runCommand(command);
  }

  async enqueue(command) {
    if (this.queue.length >= this.options.maxQueued) {
      throw new Error('Bulkhead queue is full');
    }
    
    return new Promise((resolve, reject) => {
      this.queue.push({ command, resolve, reject });
    });
  }

  async runCommand(command) {
    this.executing++;
    
    try {
      const result = await command();
      return result;
    } finally {
      this.executing--;
      this.processQueue();
    }
  }

  processQueue() {
    if (this.queue.length === 0 || 
        this.executing >= this.options.maxConcurrent) {
      return;
    }
    
    const { command, resolve, reject } = this.queue.shift();
    this.runCommand(command).then(resolve).catch(reject);
  }
}

// Retry Implementation
class Retry {
  constructor(options = {}) {
    this.options = {
      maxAttempts: options.maxAttempts || 3,
      delay: options.delay || 1000,
      backoff: options.backoff || 2,
      ...options
    };
  }

  async execute(command) {
    let lastError;
    let attempt = 0;
    
    while (attempt < this.options.maxAttempts) {
      try {
        return await command();
      } catch (error) {
        lastError = error;
        attempt++;
        
        if (attempt < this.options.maxAttempts) {
          await this.delay(attempt);
        }
      }
    }
    
    throw lastError;
  }

  async delay(attempt) {
    const ms = this.options.delay * 
      Math.pow(this.options.backoff, attempt - 1);
    
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

```

## 5. System Resources Management

### Resource Pool Implementation
```javascript
// Connection Pool
class ConnectionPool {
  constructor(options = {}) {
    this.options = {
      min: options.min || 2,
      max: options.max || 10,
      idleTimeout: options.idleTimeout || 30000,
      ...options
    };
    
    this.resources = new Map();
    this.waiting = [];
    this.initialize();
  }

  async initialize() {
    for (let i = 0; i < this.options.min; i++) {
      await this.createResource();
    }
  }

  async acquire() {
    const available = Array.from(this.resources.values())
      .find(r => !r.inUse);
    
    if (available) {
      return this.checkOut(available);
    }
    
    if (this.resources.size < this.options.max) {
      const resource = await this.createResource();
      return this.checkOut(resource);
    }
    
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        const index = this.waiting
          .findIndex(w => w.resolve === resolve);
        if (index !== -1) {
          this.waiting.splice(index, 1);
          reject(new Error('Connection timeout'));
        }
      }, this.options.acquireTimeout);
      
      this.waiting.push({ resolve, reject, timeout });
    });
  }

  async release(resource) {
    resource.inUse = false;
    resource.lastUsed = Date.now();
    
    if (this.waiting.length > 0) {
      const { resolve, timeout } = this.waiting.shift();
      clearTimeout(timeout);
      resolve(this.checkOut(resource));
    }
  }

  async createResource() {
    const resource = {
      id: crypto.randomUUID(),
      connection: await this.options.create(),
      inUse: false,
      lastUsed: Date.now()
    };
    
    this.resources.set(resource.id, resource);
    return resource;
  }

  checkOut(resource) {
    resource.inUse = true;
    resource.lastUsed = Date.now();
    return resource.connection;
  }

  async cleanup() {
    const now = Date.now();
    const minResources = this.options.min;
    
    for (const [id, resource] of this.resources) {
      if (!resource.inUse && 
          this.resources.size > minResources &&
          now - resource.lastUsed > this.options.idleTimeout) {
        await this.destroyResource(id);
      }
    }
  }

  async destroyResource(id) {
    const resource = this.resources.get(id);
    if (resource) {
      await this.options.destroy(resource.connection);
      this.resources.delete(id);
    }
  }
}

## Related Topics
- [[High-Availability]] - High Availability Implementation
- [[Scalability]] - Scalability Patterns
- [[Monitoring]] - System Monitoring
- [[Performance]] - Performance Optimization

## Practice Projects
1. Build distributed system architecture
2. Implement message-based integration
3. Create scalable load balancer
4. Design resilient system components

## Resources
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [[Learning-Resources#Architecture|Architecture Resources]]

## Tags
#system-architecture #nodejs #scalability #resilience #integration
