# Load Balancing in Node.js

A comprehensive guide to implementing and managing load balancing in Node.js applications.

## Overview

Load balancing is a critical technique for distributing incoming network traffic across multiple servers to ensure high availability, optimal resource utilization, and improved application performance.

## Load Balancing Strategies

### 1. Round Robin
```javascript
// Using Node.js cluster module for round-robin load balancing
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
    console.log(`Master ${process.pid} is running`);

    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }

    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        // Replace dead worker
        cluster.fork();
    });
} else {
    // Workers share the TCP connection
    http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Worker ${process.pid} handled the request\n`);
    }).listen(8000);

    console.log(`Worker ${process.pid} started`);
}
```

### 2. Least Connections
```javascript
// Using PM2 for advanced load balancing
// ecosystem.config.js
module.exports = {
    apps: [{
        name: 'app',
        script: 'app.js',
        instances: 'max',
        exec_mode: 'cluster',
        wait_ready: true,
        listen_timeout: 3000,
        kill_timeout: 5000
    }]
};

// app.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send(`Handled by process ${process.pid}`);
});

app.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
    process.send('ready'); // PM2 ready signal
});
```

### 3. IP Hash
```javascript
// Using NGINX for IP hash load balancing
// nginx.conf
http {
    upstream node_backend {
        ip_hash;
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
        server 127.0.0.1:3003;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://node_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

## Implementation with Popular Tools

### 1. HAProxy Implementation
```bash
# haproxy.cfg
global
    log /dev/log local0
    maxconn 4096

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend http-in
    bind *:80
    default_backend node_servers

backend node_servers
    balance roundrobin
    option httpchk GET /health
    server node1 127.0.0.1:3001 check
    server node2 127.0.0.1:3002 check
    server node3 127.0.0.1:3003 check
```

### 2. Docker Swarm Setup
```yaml
# docker-compose.yml
version: '3'
services:
  web:
    image: your-node-app
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    ports:
      - "80:3000"
    networks:
      - webnet

networks:
  webnet:
```

## Health Checks

### Implementation
```javascript
// health-check.js
const express = require('express');
const os = require('os');

const app = express();

// Basic health check endpoint
app.get('/health', (req, res) => {
    const healthcheck = {
        uptime: process.uptime(),
        message: 'OK',
        timestamp: Date.now(),
        cpuLoad: os.loadavg(),
        memoryUsage: process.memoryUsage(),
        pid: process.pid
    };
    
    try {
        res.send(healthcheck);
    } catch (e) {
        healthcheck.message = e;
        res.status(503).send();
    }
});

// Detailed health check
app.get('/health/detailed', async (req, res) => {
    try {
        const dbHealth = await checkDatabaseConnection();
        const cacheHealth = await checkCacheConnection();
        const diskSpace = await checkDiskSpace();
        
        res.json({
            status: 'UP',
            components: {
                database: dbHealth,
                cache: cacheHealth,
                diskSpace: diskSpace
            }
        });
    } catch (error) {
        res.status(503).json({
            status: 'DOWN',
            error: error.message
        });
    }
});
```

## Session Management

### Sticky Sessions
```javascript
// Using express-session with Redis for session management
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis')(session);
const Redis = require('ioredis');

const app = express();
const redis = new Redis({
    host: 'redis-server',
    port: 6379
});

app.use(session({
    store: new RedisStore({ client: redis }),
    secret: 'your-secret-key',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: process.env.NODE_ENV === 'production',
        maxAge: 1000 * 60 * 60 * 24 // 24 hours
    }
}));
```

## Monitoring and Logging

### Prometheus Integration
```javascript
const express = require('express');
const promClient = require('prom-client');
const app = express();

// Create a Registry
const register = new promClient.Registry();

// Add default metrics
promClient.collectDefaultMetrics({ register });

// Create custom metrics
const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route', 'status'],
    buckets: [0.1, 0.5, 1, 2, 5]
});
register.registerMetric(httpRequestDuration);

// Metrics endpoint
app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
});

// Request duration middleware
app.use((req, res, next) => {
    const end = httpRequestDuration.startTimer();
    res.on('finish', () => {
        end({
            method: req.method,
            route: req.route?.path || req.path,
            status: res.statusCode
        });
    });
    next();
});
```

## Best Practices

1. **Graceful Shutdown**
```javascript
// Implementing graceful shutdown
process.on('SIGTERM', () => {
    console.log('Received SIGTERM. Starting graceful shutdown...');
    
    // Stop accepting new connections
    server.close(() => {
        console.log('Server closed');
        
        // Close database connections
        mongoose.connection.close(false, () => {
            console.log('Database connection closed');
            process.exit(0);
        });
    });
    
    // Force shutdown after timeout
    setTimeout(() => {
        console.error('Could not close connections in time, forcefully shutting down');
        process.exit(1);
    }, 10000);
});
```

2. **Circuit Breaker Pattern**
```javascript
const CircuitBreaker = require('opossum');

const breaker = new CircuitBreaker(async function() {
    // Your external service call here
    return await axios.get('http://api.example.com/data');
}, {
    timeout: 3000, // If our function takes longer than 3 seconds, trigger a failure
    errorThresholdPercentage: 50, // When 50% of requests fail, trip the circuit
    resetTimeout: 30000 // After 30 seconds, try again
});

breaker.fallback(() => ({ data: 'fallback data' }));

breaker.on('success', (result) => console.log('Success:', result));
breaker.on('timeout', () => console.log('Timeout'));
breaker.on('reject', () => console.log('Circuit breaker is open'));
```

## Related Topics
- [[Clustering]] - Node.js clustering implementation
- [[Performance-Monitoring]] - Detailed monitoring setup
- [[High-Availability]] - High availability patterns
- [[Docker-Deployment]] - Container deployment

## Practice Projects
1. Build a load-balanced Node.js application using HAProxy
2. Implement session management with Redis
3. Create a monitoring dashboard with Prometheus and Grafana
4. Deploy a load-balanced application using Docker Swarm

## Resources
- [Node.js Cluster Documentation](https://nodejs.org/api/cluster.html)
- [HAProxy Documentation](http://www.haproxy.org/)
- [NGINX Load Balancing Guide](https://nginx.org/en/docs/http/load_balancing.html)
- [[Learning-Resources#LoadBalancing|Load Balancing Resources]]

## Tags
#load-balancing #high-availability #scaling #performance #nodejs
