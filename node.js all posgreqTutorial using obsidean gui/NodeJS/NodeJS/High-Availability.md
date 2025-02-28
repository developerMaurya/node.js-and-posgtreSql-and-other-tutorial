# High Availability in Node.js

A comprehensive guide to implementing high availability in Node.js applications.

## 1. Process Management

### PM2 Configuration
```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'server.js',
    instances: 'max',
    exec_mode: 'cluster',
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'development'
    },
    env_production: {
      NODE_ENV: 'production'
    },
    error_file: 'logs/err.log',
    out_file: 'logs/out.log',
    merge_logs: true,
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    
    // Load Balancing
    instance_var: 'INSTANCE_ID',
    increment_var: 'PORT',
    
    // Health Check
    exp_backoff_restart_delay: 100,
    
    // Graceful Shutdown
    kill_timeout: 5000,
    wait_ready: true,
    listen_timeout: 3000
  }]
};
```

### Graceful Shutdown
```javascript
// server.js
const express = require('express');
const app = express();

let server;
let shutdownInProgress = false;

// Graceful shutdown handler
async function gracefulShutdown(signal) {
  console.log(`${signal} received. Starting graceful shutdown...`);
  shutdownInProgress = true;

  // Stop accepting new requests
  server.close(async () => {
    console.log('Server closed. Cleaning up...');
    
    try {
      // Close database connections
      await db.disconnect();
      
      // Close message queue connections
      await messageQueue.close();
      
      // Clear caches
      await cache.clear();
      
      console.log('Cleanup completed. Exiting...');
      process.exit(0);
    } catch (error) {
      console.error('Error during cleanup:', error);
      process.exit(1);
    }
  });

  // Force shutdown after timeout
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

// Health check middleware
app.use('/health', (req, res) => {
  if (shutdownInProgress) {
    res.status(503).json({ status: 'shutting_down' });
    return;
  }
  res.json({ status: 'healthy' });
});

// Signal handlers
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

server = app.listen(3000);
```

## 2. Load Balancing

### NGINX Configuration
```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream myapp {
        least_conn;  # Load balancing method
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
        server 127.0.0.1:3003;
        server 127.0.0.1:3004;
        
        keepalive 32;  # Keep-alive connections
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://myapp;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            
            # Health checks
            health_check interval=5s fails=3 passes=2;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
    }
}
```

## 3. Session Management

### Redis Session Store
```javascript
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const Redis = require('ioredis');

const app = express();

// Redis client setup with high availability
const redisClient = new Redis.Cluster([{
  host: 'redis-1',
  port: 6379
}, {
  host: 'redis-2',
  port: 6379
}], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
    tls: true
  },
  scaleReads: 'all',  // Read from all replicas
  maxRedirections: 16,
  retryDelayOnFailover: 100
});

// Session configuration
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  rolling: true,
  cookie: {
    secure: true,
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));
```

## 4. Database High Availability

### MongoDB Replica Set
```javascript
const mongoose = require('mongoose');

const options = {
  replicaSet: 'rs0',
  readPreference: 'secondaryPreferred',
  retryWrites: true,
  w: 'majority',
  wtimeout: 5000,
  maxPoolSize: 10,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  keepAlive: true,
  keepAliveInitialDelay: 300000
};

mongoose.connect('mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp', options)
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('MongoDB connection error:', err));

// Connection events
mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});

mongoose.connection.on('reconnected', () => {
  console.log('MongoDB reconnected');
});
```

### PostgreSQL High Availability
```javascript
const { Pool } = require('pg');

const masterPool = new Pool({
  user: 'dbuser',
  host: 'master.example.com',
  database: 'myapp',
  password: 'password',
  port: 5432,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

const replicaPool = new Pool({
  user: 'dbuser',
  host: 'replica.example.com',
  database: 'myapp',
  password: 'password',
  port: 5432,
  max: 20,
});

// Query router
async function executeQuery(query, params, queryType = 'read') {
  const pool = queryType === 'read' ? replicaPool : masterPool;
  
  try {
    const client = await pool.connect();
    try {
      const result = await client.query(query, params);
      return result.rows;
    } finally {
      client.release();
    }
  } catch (error) {
    console.error('Database error:', error);
    throw error;
  }
}
```

## 5. Caching Strategy

### Multi-Layer Caching
```javascript
const NodeCache = require('node-cache');
const Redis = require('ioredis');

class CacheManager {
  constructor() {
    // In-memory cache
    this.localCache = new NodeCache({
      stdTTL: 60,  // 1 minute
      checkperiod: 120,
      useClones: false
    });

    // Distributed cache
    this.redisCache = new Redis.Cluster([
      {
        host: 'redis-1',
        port: 6379
      },
      {
        host: 'redis-2',
        port: 6379
      }
    ]);
  }

  async get(key) {
    // Try local cache first
    const localValue = this.localCache.get(key);
    if (localValue) {
      return localValue;
    }

    // Try Redis cache
    try {
      const redisValue = await this.redisCache.get(key);
      if (redisValue) {
        // Update local cache
        this.localCache.set(key, redisValue);
        return redisValue;
      }
    } catch (error) {
      console.error('Redis cache error:', error);
    }

    return null;
  }

  async set(key, value, ttl = 3600) {
    try {
      // Set in Redis
      await this.redisCache.set(key, value, 'EX', ttl);
      
      // Set in local cache with shorter TTL
      this.localCache.set(key, value, Math.min(ttl, 60));
      
      return true;
    } catch (error) {
      console.error('Cache set error:', error);
      return false;
    }
  }

  async invalidate(key) {
    try {
      await this.redisCache.del(key);
      this.localCache.del(key);
    } catch (error) {
      console.error('Cache invalidation error:', error);
    }
  }
}
```

## 6. Circuit Breaker Pattern

### Service Circuit Breaker
```javascript
class CircuitBreaker {
  constructor(service, options = {}) {
    this.service = service;
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.state = 'CLOSED';
    this.failures = 0;
    this.lastFailureTime = null;
    this.monitors = new Set();
  }

  async execute(request) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
        this.state = 'HALF-OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const response = await this.service.execute(request);
      this.onSuccess();
      return response;
    } catch (error) {
      this.onFailure(error);
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
    this.notifyMonitors();
  }

  onFailure(error) {
    this.failures++;
    this.lastFailureTime = Date.now();

    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
    }

    this.notifyMonitors();
  }

  addMonitor(monitor) {
    this.monitors.add(monitor);
  }

  notifyMonitors() {
    for (const monitor of this.monitors) {
      monitor.update(this.state, this.failures);
    }
  }
}

// Usage example
const serviceBreaker = new CircuitBreaker(externalService, {
  failureThreshold: 3,
  resetTimeout: 30000
});

serviceBreaker.addMonitor({
  update: (state, failures) => {
    console.log(`Circuit Breaker State: ${state}, Failures: ${failures}`);
    // Send metrics to monitoring system
  }
});
```

## Related Topics
- [[Load-Balancing]] - Advanced load balancing
- [[Process-Management]] - Process management guide
- [[Fault-Tolerance]] - Fault tolerance patterns
- [[Monitoring]] - System monitoring

## Practice Projects
1. Implement high availability cluster
2. Build resilient caching system
3. Create fault-tolerant database setup
4. Design circuit breaker implementation

## Resources
- [Node.js Clustering Guide](https://nodejs.org/api/cluster.html)
- [PM2 Documentation](https://pm2.keymetrics.io/)
- [[Learning-Resources#HighAvailability|High Availability Resources]]

## Tags
#high-availability #nodejs #clustering #load-balancing #fault-tolerance
