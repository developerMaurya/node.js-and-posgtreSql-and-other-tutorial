# Node.js Clustering

A comprehensive guide to implementing and managing clustering in Node.js applications for improved performance and scalability.

## Overview

Clustering in Node.js allows you to create child processes (workers) that share server ports, enabling your application to handle more load by utilizing multiple CPU cores.

## Basic Clustering Implementation

### Simple Cluster Setup
```javascript
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
        // Replace the dead worker
        cluster.fork();
    });
} else {
    // Workers can share any TCP connection
    http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Hello from Worker ${process.pid}\n`);
    }).listen(8000);

    console.log(`Worker ${process.pid} started`);
}
```

## Advanced Clustering

### Express.js with Clustering
```javascript
// cluster-app.js
const cluster = require('cluster');
const express = require('express');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
    console.log(`Master ${process.pid} is running`);

    // Keep track of workers
    const workers = new Set();

    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        const worker = cluster.fork();
        workers.add(worker);
    }

    // Worker management
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        workers.delete(worker);

        if (signal !== 'SIGTERM') {
            const newWorker = cluster.fork();
            workers.add(newWorker);
        }
    });

    // Graceful shutdown
    process.on('SIGTERM', () => {
        console.log('Master received SIGTERM. Shutting down...');
        
        for (const worker of workers) {
            worker.send('shutdown');
        }
    });
} else {
    const app = express();

    // Middleware
    app.use(express.json());

    // Routes
    app.get('/', (req, res) => {
        res.send(`Hello from Worker ${process.pid}`);
    });

    // CPU-intensive task
    app.get('/compute', (req, res) => {
        const result = fibonacci(parseInt(req.query.n) || 40);
        res.json({ pid: process.pid, result });
    });

    // Graceful shutdown handling
    process.on('message', (msg) => {
        if (msg === 'shutdown') {
            console.log(`Worker ${process.pid} shutting down...`);
            server.close(() => {
                process.exit(0);
            });
        }
    });

    const server = app.listen(3000, () => {
        console.log(`Worker ${process.pid} started`);
    });
}

// CPU-intensive function
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
```

## Inter-Process Communication (IPC)

### Message Passing
```javascript
// Master process
if (cluster.isMaster) {
    const workers = new Map();

    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        const worker = cluster.fork();
        workers.set(worker.id, worker);

        // Handle messages from worker
        worker.on('message', (msg) => {
            console.log(`Master received: ${JSON.stringify(msg)} from worker ${worker.id}`);
            
            // Broadcast to other workers
            for (const [id, w] of workers) {
                if (id !== worker.id) {
                    w.send({ type: 'broadcast', data: msg });
                }
            }
        });
    }
} else {
    // Worker process
    process.on('message', (msg) => {
        console.log(`Worker ${process.pid} received: ${JSON.stringify(msg)}`);
    });

    // Send message to master
    process.send({ type: 'status', pid: process.pid, status: 'ready' });
}
```

## State Management

### Using Redis for Shared State
```javascript
const Redis = require('ioredis');
const express = require('express');
const cluster = require('cluster');

if (cluster.isMaster) {
    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
} else {
    const app = express();
    const redis = new Redis({
        host: 'localhost',
        port: 6379
    });

    // Share state across workers
    app.get('/increment', async (req, res) => {
        const value = await redis.incr('counter');
        res.json({
            worker: process.pid,
            counter: value
        });
    });

    app.get('/counter', async (req, res) => {
        const value = await redis.get('counter');
        res.json({
            worker: process.pid,
            counter: value
        });
    });

    app.listen(3000);
}
```

## Load Balancing Strategies

### Custom Load Balancing
```javascript
if (cluster.isMaster) {
    const workers = new Map();
    let currentWorker = 0;

    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        const worker = cluster.fork();
        workers.set(worker.id, {
            instance: worker,
            load: 0
        });
    }

    // Custom load balancing
    function getNextWorker() {
        const workerIds = Array.from(workers.keys());
        const worker = workers.get(workerIds[currentWorker]);
        currentWorker = (currentWorker + 1) % workerIds.length;
        return worker;
    }

    // Track worker load
    cluster.on('message', (worker, message) => {
        if (message.type === 'load_update') {
            const workerData = workers.get(worker.id);
            if (workerData) {
                workerData.load = message.load;
            }
        }
    });
}
```

## Monitoring and Debugging

### Cluster Metrics
```javascript
// metrics.js
const promClient = require('prom-client');
const register = new promClient.Registry();

// Cluster metrics
const activeWorkers = new promClient.Gauge({
    name: 'node_cluster_workers_active',
    help: 'Number of active workers'
});

const workerRestarts = new promClient.Counter({
    name: 'node_cluster_worker_restarts_total',
    help: 'Number of times workers have been restarted'
});

register.registerMetric(activeWorkers);
register.registerMetric(workerRestarts);

if (cluster.isMaster) {
    // Update metrics
    setInterval(() => {
        activeWorkers.set(Object.keys(cluster.workers).length);
    }, 5000);

    cluster.on('exit', () => {
        workerRestarts.inc();
    });

    // Metrics endpoint
    http.createServer(async (req, res) => {
        if (req.url === '/metrics') {
            res.setHeader('Content-Type', register.contentType);
            res.end(await register.metrics());
        }
    }).listen(9100);
}
```

## Best Practices

### 1. Zero-Downtime Reload
```javascript
if (cluster.isMaster) {
    const workers = new Set();

    // Fork initial workers
    for (let i = 0; i < numCPUs; i++) {
        workers.add(cluster.fork());
    }

    // Zero-downtime reload
    function reload() {
        const oldWorkers = new Set(workers);
        
        // Fork new workers
        for (let i = 0; i < numCPUs; i++) {
            const newWorker = cluster.fork();
            workers.add(newWorker);

            // When new worker is ready, kill an old worker
            newWorker.on('listening', () => {
                const oldWorker = oldWorkers.values().next().value;
                if (oldWorker) {
                    oldWorkers.delete(oldWorker);
                    oldWorker.disconnect();
                }
            });
        }
    }

    // Handle reload signal
    process.on('SIGHUP', reload);
}
```

### 2. Error Handling
```javascript
if (cluster.isMaster) {
    // Handle worker errors
    cluster.on('exit', (worker, code, signal) => {
        console.error(`Worker ${worker.process.pid} died. Code: ${code}, Signal: ${signal}`);
        
        if (signal !== 'SIGTERM') {
            console.log('Starting new worker...');
            cluster.fork();
        }
    });

    // Handle uncaught exceptions in master
    process.on('uncaughtException', (err) => {
        console.error('Uncaught exception in master:', err);
        
        // Gracefully shutdown
        for (const id in cluster.workers) {
            cluster.workers[id].send('shutdown');
        }
        
        process.exit(1);
    });
}
```

## Related Topics
- [[Load-Balancing]] - Load balancing strategies
- [[Process-Management]] - Node.js process management
- [[Performance-Optimization]] - Performance optimization techniques
- [[Monitoring]] - Application monitoring

## Practice Projects
1. Build a cluster-enabled REST API
2. Implement zero-downtime deployments
3. Create a worker pool system
4. Build a real-time monitoring dashboard

## Resources
- [Node.js Cluster Documentation](https://nodejs.org/api/cluster.html)
- [PM2 Process Manager](https://pm2.keymetrics.io/)
- [[Learning-Resources#Clustering|Clustering Resources]]

## Tags
#clustering #scalability #performance #nodejs #ipc
