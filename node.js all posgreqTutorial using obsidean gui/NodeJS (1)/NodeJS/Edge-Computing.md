# Edge Computing with Node.js

A comprehensive guide to implementing edge computing solutions using Node.js.

## 1. Edge Computing Fundamentals

### Core Concepts
- Edge computing brings computation closer to data sources
- Reduces latency and bandwidth usage
- Improves response times and data privacy
- Enables real-time processing
- Supports IoT and distributed systems

### Edge Computing vs Cloud Computing
- Location: Edge (near data source) vs Cloud (centralized)
- Latency: Low vs Higher
- Bandwidth: Optimized vs Heavy usage
- Processing: Real-time vs Batch
- Scale: Distributed vs Centralized

## 2. Edge Server Implementation

### Basic Edge Server
```javascript
const express = require('express');
const compression = require('compression');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

class EdgeServer {
  constructor(options = {}) {
    this.app = express();
    this.port = options.port || 3000;
    this.setupMiddleware();
    this.setupRoutes();
    this.setupErrorHandling();
  }

  setupMiddleware() {
    // Security
    this.app.use(helmet());
    
    // Compression
    this.app.use(compression());
    
    // Rate Limiting
    this.app.use(rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100 // limit each IP to 100 requests per windowMs
    }));
    
    // Request Parsing
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
  }

  setupRoutes() {
    // Health Check
    this.app.get('/health', (req, res) => {
      res.status(200).json({ status: 'healthy' });
    });
    
    // Edge-specific routes
    this.app.use('/edge', this.createEdgeRouter());
  }

  createEdgeRouter() {
    const router = express.Router();
    
    router.post('/process', async (req, res) => {
      try {
        const result = await this.processAtEdge(req.body);
        res.json(result);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    return router;
  }

  async processAtEdge(data) {
    // Implement edge processing logic
    return {
      processed: true,
      timestamp: Date.now(),
      result: data
    };
  }

  setupErrorHandling() {
    // 404 Handler
    this.app.use((req, res) => {
      res.status(404).json({ error: 'Not Found' });
    });
    
    // Error Handler
    this.app.use((err, req, res, next) => {
      console.error(err.stack);
      res.status(500).json({ error: 'Internal Server Error' });
    });
  }

  start() {
    return new Promise((resolve) => {
      this.server = this.app.listen(this.port, () => {
        console.log(`Edge server running on port ${this.port}`);
        resolve(this.server);
      });
    });
  }

  stop() {
    return new Promise((resolve) => {
      if (this.server) {
        this.server.close(resolve);
      } else {
        resolve();
      }
    });
  }
}
```

## 3. Data Processing at the Edge

### Edge Cache Implementation
```javascript
const LRU = require('lru-cache');

class EdgeCache {
  constructor(options = {}) {
    this.cache = new LRU({
      max: options.maxSize || 1000,
      maxAge: options.maxAge || 1000 * 60 * 60, // 1 hour
      updateAgeOnGet: true
    });
    
    this.metrics = {
      hits: 0,
      misses: 0,
      sets: 0
    };
  }

  async get(key) {
    const value = this.cache.get(key);
    if (value !== undefined) {
      this.metrics.hits++;
      return value;
    }
    this.metrics.misses++;
    return null;
  }

  async set(key, value, ttl) {
    this.cache.set(key, value, ttl);
    this.metrics.sets++;
  }

  async delete(key) {
    return this.cache.del(key);
  }

  async clear() {
    return this.cache.reset();
  }

  getMetrics() {
    return {
      ...this.metrics,
      size: this.cache.length,
      itemCount: this.cache.itemCount
    };
  }
}
```

### Edge Data Processor
```javascript
class EdgeDataProcessor {
  constructor(options = {}) {
    this.cache = new EdgeCache(options.cache);
    this.processors = new Map();
    this.setupDefaultProcessors();
  }

  setupDefaultProcessors() {
    // Image processing
    this.registerProcessor('image', async (data) => {
      // Implement image processing logic
      return data;
    });
    
    // Time series data
    this.registerProcessor('timeseries', async (data) => {
      // Implement time series processing
      return data;
    });
    
    // Sensor data
    this.registerProcessor('sensor', async (data) => {
      // Implement sensor data processing
      return data;
    });
  }

  registerProcessor(type, processor) {
    this.processors.set(type, processor);
  }

  async process(type, data) {
    const processor = this.processors.get(type);
    if (!processor) {
      throw new Error(`No processor found for type: ${type}`);
    }
    
    // Check cache first
    const cacheKey = this.getCacheKey(type, data);
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return cached;
    }
    
    // Process data
    const result = await processor(data);
    
    // Cache result
    await this.cache.set(cacheKey, result);
    
    return result;
  }

  getCacheKey(type, data) {
    return `${type}:${JSON.stringify(data)}`;
  }
}
```

## 4. Edge Network Management

### Service Discovery
```javascript
const dns = require('dns').promises;
const os = require('os');

class EdgeServiceDiscovery {
  constructor(options = {}) {
    this.services = new Map();
    this.interval = options.interval || 30000;
    this.hostname = os.hostname();
  }

  register(service) {
    this.services.set(service.id, {
      ...service,
      hostname: this.hostname,
      lastSeen: Date.now()
    });
  }

  async discover(serviceType) {
    try {
      const records = await dns.resolveSrv(`${serviceType}.service.local`);
      return records.map(record => ({
        host: record.name,
        port: record.port,
        priority: record.priority,
        weight: record.weight
      }));
    } catch (error) {
      console.error('Service discovery error:', error);
      return [];
    }
  }

  async heartbeat() {
    for (const [id, service] of this.services) {
      if (Date.now() - service.lastSeen > this.interval * 2) {
        this.services.delete(id);
      }
    }
  }

  start() {
    this.heartbeatInterval = setInterval(() => this.heartbeat(), this.interval);
  }

  stop() {
    clearInterval(this.heartbeatInterval);
  }
}
```

## 5. Edge Security

### Security Implementation
```javascript
const crypto = require('crypto');
const jwt = require('jsonwebtoken');

class EdgeSecurity {
  constructor(options = {}) {
    this.secret = options.secret || crypto.randomBytes(32).toString('hex');
    this.algorithm = options.algorithm || 'HS256';
  }

  async generateToken(payload, expiresIn = '1h') {
    return jwt.sign(payload, this.secret, { 
      algorithm: this.algorithm,
      expiresIn 
    });
  }

  async verifyToken(token) {
    try {
      return jwt.verify(token, this.secret, { 
        algorithms: [this.algorithm] 
      });
    } catch (error) {
      throw new Error('Invalid token');
    }
  }

  async encrypt(data) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', Buffer.from(this.secret, 'hex'), iv);
    
    let encrypted = cipher.update(JSON.stringify(data), 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }

  async decrypt(data) {
    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      Buffer.from(this.secret, 'hex'),
      Buffer.from(data.iv, 'hex')
    );
    
    decipher.setAuthTag(Buffer.from(data.authTag, 'hex'));
    
    let decrypted = decipher.update(data.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return JSON.parse(decrypted);
  }
}
```

## 6. Edge Deployment

### Deployment Configuration
```javascript
module.exports = {
  edge: {
    regions: [
      {
        name: 'us-east',
        instances: 3,
        config: {
          port: 3000,
          cache: {
            maxSize: 1000,
            maxAge: 3600000
          }
        }
      },
      {
        name: 'eu-west',
        instances: 2,
        config: {
          port: 3000,
          cache: {
            maxSize: 1000,
            maxAge: 3600000
          }
        }
      }
    ],
    routing: {
      strategy: 'latency',
      fallback: 'random'
    },
    security: {
      algorithm: 'HS256',
      tokenExpiry: '1h'
    }
  }
};
```

## Related Topics
- [[Cloud-Deployment]] - Cloud deployment strategies
- [[High-Availability]] - High availability implementation
- [[Security-Architecture]] - Security architecture
- [[Performance-Optimization]] - Performance optimization

## Practice Projects
1. Build an edge computing platform
2. Implement edge caching system
3. Create edge security system
4. Design service discovery

## Resources
- [Edge Computing Documentation](https://nodejs.org/api/worker_threads.html)
- [[Learning-Resources#EdgeComputing|Edge Computing Resources]]

## Tags
#edge-computing #nodejs #distributed-systems #iot #real-time
