# Worker Threads in Node.js

A comprehensive guide to using Worker Threads for CPU-intensive tasks and parallel processing in Node.js applications.

## Basic Worker Thread Usage

### Simple Worker Thread
```javascript
// main.js
const { Worker } = require('worker_threads');

function runWorker(workerData) {
    return new Promise((resolve, reject) => {
        const worker = new Worker('./worker.js', { workerData });
        
        worker.on('message', resolve);
        worker.on('error', reject);
        worker.on('exit', (code) => {
            if (code !== 0) {
                reject(new Error(`Worker stopped with exit code ${code}`));
            }
        });
    });
}

// Usage
async function main() {
    try {
        const result = await runWorker({ number: 50 });
        console.log('Result:', result);
    } catch (err) {
        console.error(err);
    }
}

main();

// worker.js
const { parentPort, workerData } = require('worker_threads');

function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

const result = fibonacci(workerData.number);
parentPort.postMessage(result);
```

## Worker Pool Implementation

### Thread Pool Manager
```javascript
const { Worker } = require('worker_threads');
const os = require('os');

class WorkerPool {
    constructor(workerScript, numWorkers = os.cpus().length) {
        this.workerScript = workerScript;
        this.numWorkers = numWorkers;
        this.workers = [];
        this.freeWorkers = [];
        this.tasks = [];
        
        this.init();
    }

    init() {
        // Create worker threads
        for (let i = 0; i < this.numWorkers; i++) {
            const worker = new Worker(this.workerScript);
            this.workers.push(worker);
            this.freeWorkers.push(worker);
            
            worker.on('message', (result) => {
                this.handleTaskCompletion(worker, null, result);
            });
            
            worker.on('error', (error) => {
                this.handleTaskCompletion(worker, error, null);
            });
        }
    }

    handleTaskCompletion(worker, error, result) {
        const { resolve, reject } = this.tasks.shift();
        
        // Add worker back to free pool
        this.freeWorkers.push(worker);
        
        // Handle next task if any
        if (this.tasks.length > 0) {
            this.runTask(this.tasks[0]);
        }
        
        // Resolve or reject the task promise
        if (error) {
            reject(error);
        } else {
            resolve(result);
        }
    }

    runTask(task) {
        const worker = this.freeWorkers.pop();
        worker.postMessage(task.data);
    }

    executeTask(data) {
        return new Promise((resolve, reject) => {
            const task = { data, resolve, reject };
            this.tasks.push(task);
            
            if (this.freeWorkers.length > 0) {
                this.runTask(task);
            }
        });
    }

    async close() {
        await Promise.all(
            this.workers.map(worker => worker.terminate())
        );
    }
}

// Usage
const pool = new WorkerPool('./worker.js');

async function processArray(array) {
    const results = await Promise.all(
        array.map(item => pool.executeTask(item))
    );
    return results;
}
```

## Data Transfer and Shared Memory

### SharedArrayBuffer Usage
```javascript
// main.js
const { Worker } = require('worker_threads');

// Create shared buffer
const sharedBuffer = new SharedArrayBuffer(4 * 1024); // 4KB
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker('./worker.js', {
    workerData: { sharedBuffer }
});

// Write to shared memory
sharedArray[0] = 123;

worker.on('message', (msg) => {
    console.log('Worker modified value:', sharedArray[0]);
});

// worker.js
const { parentPort, workerData } = require('worker_threads');
const { sharedBuffer } = workerData;

const sharedArray = new Int32Array(sharedBuffer);
console.log('Initial value:', sharedArray[0]);

// Modify shared memory
Atomics.add(sharedArray, 0, 1);
parentPort.postMessage('done');
```

### Transferable Objects
```javascript
// main.js
const { Worker } = require('worker_threads');

const worker = new Worker('./worker.js');

// Create transferable object
const arrayBuffer = new ArrayBuffer(1024);
const array = new Uint8Array(arrayBuffer);
array[0] = 123;

// Transfer ownership to worker
worker.postMessage({ arrayBuffer }, [arrayBuffer]);

// worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', ({ arrayBuffer }) => {
    const array = new Uint8Array(arrayBuffer);
    console.log('Received value:', array[0]);
});
```

## Worker Communication Patterns

### Message Channel
```javascript
// main.js
const { Worker, MessageChannel } = require('worker_threads');

// Create workers
const worker1 = new Worker('./worker1.js');
const worker2 = new Worker('./worker2.js');

// Create message channel
const { port1, port2 } = new MessageChannel();

// Send ports to workers
worker1.postMessage({ port: port1 }, [port1]);
worker2.postMessage({ port: port2 }, [port2]);

// worker1.js
const { parentPort } = require('worker_threads');

parentPort.once('message', ({ port }) => {
    port.postMessage('Hello from Worker 1');
    port.on('message', (msg) => {
        console.log('Worker 1 received:', msg);
    });
});

// worker2.js
const { parentPort } = require('worker_threads');

parentPort.once('message', ({ port }) => {
    port.on('message', (msg) => {
        console.log('Worker 2 received:', msg);
        port.postMessage('Hello back from Worker 2');
    });
});
```

## Error Handling and Monitoring

### Worker Error Handling
```javascript
class ResilientWorker {
    constructor(scriptPath, options = {}) {
        this.scriptPath = scriptPath;
        this.options = options;
        this.maxRetries = options.maxRetries || 3;
        this.retryDelay = options.retryDelay || 1000;
        this.worker = null;
        this.retryCount = 0;
    }

    async start() {
        return new Promise((resolve, reject) => {
            this.worker = new Worker(this.scriptPath, this.options);

            this.worker.on('online', () => {
                console.log('Worker started');
                this.retryCount = 0;
                resolve();
            });

            this.worker.on('error', async (error) => {
                console.error('Worker error:', error);
                if (this.retryCount < this.maxRetries) {
                    await this.restart();
                } else {
                    reject(error);
                }
            });

            this.worker.on('exit', (code) => {
                if (code !== 0) {
                    console.error(`Worker stopped with exit code ${code}`);
                }
            });
        });
    }

    async restart() {
        this.retryCount++;
        console.log(`Restarting worker (attempt ${this.retryCount})`);
        await new Promise(resolve => setTimeout(resolve, this.retryDelay));
        return this.start();
    }

    async execute(data) {
        return new Promise((resolve, reject) => {
            this.worker.once('message', resolve);
            this.worker.once('error', reject);
            this.worker.postMessage(data);
        });
    }

    terminate() {
        if (this.worker) {
            return this.worker.terminate();
        }
    }
}
```

### Worker Monitoring
```javascript
class WorkerMonitor {
    constructor() {
        this.metrics = new Map();
    }

    trackWorker(worker, id) {
        const metrics = {
            startTime: Date.now(),
            messageCount: 0,
            errorCount: 0,
            lastActive: Date.now()
        };
        
        this.metrics.set(id, metrics);

        worker.on('message', () => {
            metrics.messageCount++;
            metrics.lastActive = Date.now();
        });

        worker.on('error', () => {
            metrics.errorCount++;
        });

        worker.on('exit', () => {
            metrics.endTime = Date.now();
        });
    }

    getMetrics(id) {
        const metrics = this.metrics.get(id);
        if (!metrics) return null;

        const now = Date.now();
        return {
            uptime: now - metrics.startTime,
            messageCount: metrics.messageCount,
            errorCount: metrics.errorCount,
            lastActiveSecs: (now - metrics.lastActive) / 1000
        };
    }

    getAllMetrics() {
        const result = {};
        for (const [id, metrics] of this.metrics) {
            result[id] = this.getMetrics(id);
        }
        return result;
    }
}
```

## Performance Optimization

### Worker Resource Management
```javascript
class OptimizedWorkerPool {
    constructor(options = {}) {
        this.minWorkers = options.minWorkers || 2;
        this.maxWorkers = options.maxWorkers || os.cpus().length;
        this.maxIdleTime = options.maxIdleTime || 60000; // 1 minute
        this.workers = new Map();
        this.taskQueue = [];
    }

    async scaleWorkers() {
        const currentLoad = this.getPoolLoad();
        
        if (currentLoad > 0.8 && this.workers.size < this.maxWorkers) {
            // Scale up
            await this.addWorker();
        } else if (currentLoad < 0.2 && this.workers.size > this.minWorkers) {
            // Scale down
            this.removeIdleWorker();
        }
    }

    getPoolLoad() {
        const busyWorkers = Array.from(this.workers.values())
            .filter(w => w.isBusy).length;
        return busyWorkers / this.workers.size;
    }

    async addWorker() {
        const worker = new ResilientWorker('./worker.js');
        await worker.start();
        this.workers.set(worker.id, worker);
    }

    removeIdleWorker() {
        const now = Date.now();
        for (const [id, worker] of this.workers) {
            if (!worker.isBusy && 
                (now - worker.lastActive) > this.maxIdleTime) {
                worker.terminate();
                this.workers.delete(id);
                break;
            }
        }
    }
}
```

### Task Prioritization
```javascript
class PriorityWorkerPool {
    constructor() {
        this.highPriorityQueue = [];
        this.lowPriorityQueue = [];
        this.workers = new Map();
    }

    async addTask(task, priority = 'low') {
        const queue = priority === 'high' ? 
            this.highPriorityQueue : 
            this.lowPriorityQueue;
            
        return new Promise((resolve, reject) => {
            queue.push({
                task,
                resolve,
                reject,
                timestamp: Date.now()
            });
            this.processNextTask();
        });
    }

    async processNextTask() {
        if (!this.hasFreeWorker()) return;

        const task = this.highPriorityQueue.shift() || 
                    this.lowPriorityQueue.shift();
                    
        if (!task) return;

        const worker = this.getIdleWorker();
        try {
            const result = await worker.execute(task.task);
            task.resolve(result);
        } catch (error) {
            task.reject(error);
        }
    }
}
```

## Best Practices

### 1. Resource Management
```javascript
class WorkerManager {
    constructor() {
        this.maxMemoryUsage = 1024 * 1024 * 1024; // 1GB
        this.checkInterval = 5000; // 5 seconds
    }

    startMonitoring(worker) {
        setInterval(() => {
            const usage = process.memoryUsage();
            if (usage.heapUsed > this.maxMemoryUsage) {
                console.warn('High memory usage detected');
                this.handleHighMemory(worker);
            }
        }, this.checkInterval);
    }

    handleHighMemory(worker) {
        // Implement memory management strategy
        // e.g., restart worker, clear caches, etc.
    }
}
```

### 2. Graceful Shutdown
```javascript
class GracefulWorker {
    constructor() {
        this.shutdownPromise = null;
        this.isShuttingDown = false;
        
        process.on('SIGTERM', () => this.shutdown());
        process.on('SIGINT', () => this.shutdown());
    }

    async shutdown() {
        if (this.isShuttingDown) return this.shutdownPromise;
        
        this.isShuttingDown = true;
        this.shutdownPromise = new Promise(async (resolve) => {
            console.log('Worker shutting down...');
            
            // Complete current task if any
            if (this.currentTask) {
                await this.currentTask;
            }
            
            // Cleanup resources
            await this.cleanup();
            
            resolve();
        });
        
        return this.shutdownPromise;
    }
}
```

## Related Topics
- [[Process-Management]] - Node.js process management
- [[Performance-Optimization]] - Performance optimization techniques
- [[CPU-Intensive-Tasks]] - Handling CPU-intensive operations
- [[Parallel-Processing]] - Parallel processing patterns

## Practice Projects
1. Build a worker pool manager
2. Implement a task scheduler with priorities
3. Create a worker monitoring system
4. Develop a parallel processing pipeline

## Resources
- [Node.js Worker Threads Documentation](https://nodejs.org/api/worker_threads.html)
- [Node.js Threading Model](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/)
- [[Learning-Resources#WorkerThreads|Worker Threads Resources]]

## Tags
#worker-threads #parallel-processing #performance #nodejs #concurrency
