# Process Management in Node.js

A comprehensive guide to managing Node.js processes in production environments.

## 1. Process Manager Setup

### PM2 Advanced Configuration
```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'api-service',
    script: 'dist/server.js',
    instances: 'max',
    exec_mode: 'cluster',
    watch: false,
    max_memory_restart: '1G',
    
    // Advanced Settings
    node_args: '--max-old-space-size=8192',
    kill_timeout: 3000,
    wait_ready: true,
    listen_timeout: 10000,
    
    // Environment Variables
    env: {
      NODE_ENV: 'development',
      PORT: 3000
    },
    env_production: {
      NODE_ENV: 'production',
      PORT: 80
    },
    
    // Log Configuration
    error_file: 'logs/error.log',
    out_file: 'logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    
    // Metrics
    instance_var: 'INSTANCE_ID',
    increment_var: 'PORT',
    
    // Restart Behavior
    min_uptime: '60s',
    max_restarts: 10,
    restart_delay: 4000,
    
    // Cluster Management
    automation: true,
    autorestart: true,
    exp_backoff_restart_delay: 100,
    
    // Resource Management
    shutdown_with_message: true,
    treekill: false
  }]
};
```

## 2. Process Monitoring

### Custom Process Monitor
```javascript
class ProcessMonitor {
  constructor(options = {}) {
    this.metrics = {
      cpu: [],
      memory: [],
      eventLoop: [],
      handles: []
    };
    
    this.options = {
      sampleInterval: options.sampleInterval || 1000,
      maxSamples: options.maxSamples || 60,
      alertThresholds: {
        cpu: options.cpuThreshold || 80,
        memory: options.memoryThreshold || 85,
        eventLoop: options.eventLoopThreshold || 1000
      }
    };
    
    this.alerts = new Set();
  }

  start() {
    this.interval = setInterval(() => this.collect(), this.options.sampleInterval);
    this.setupEventListeners();
  }

  async collect() {
    const metrics = await this.gatherMetrics();
    this.updateMetrics(metrics);
    this.checkThresholds(metrics);
  }

  async gatherMetrics() {
    const used = process.memoryUsage();
    const cpuUsage = process.cpuUsage();
    
    return {
      timestamp: Date.now(),
      cpu: {
        user: cpuUsage.user,
        system: cpuUsage.system
      },
      memory: {
        heapUsed: used.heapUsed,
        heapTotal: used.heapTotal,
        external: used.external,
        rss: used.rss
      },
      eventLoop: await this.getEventLoopLag(),
      handles: process._getActiveHandles().length
    };
  }

  async getEventLoopLag() {
    const start = Date.now();
    return new Promise(resolve => {
      setImmediate(() => {
        resolve(Date.now() - start);
      });
    });
  }

  updateMetrics(metrics) {
    for (const [key, value] of Object.entries(metrics)) {
      if (Array.isArray(this.metrics[key])) {
        this.metrics[key].push(value);
        if (this.metrics[key].length > this.options.maxSamples) {
          this.metrics[key].shift();
        }
      }
    }
  }

  checkThresholds(metrics) {
    const memoryUsage = (metrics.memory.heapUsed / metrics.memory.heapTotal) * 100;
    
    if (memoryUsage > this.options.alertThresholds.memory) {
      this.alert('memory', `Memory usage high: ${memoryUsage.toFixed(2)}%`);
    }

    if (metrics.eventLoop > this.options.alertThresholds.eventLoop) {
      this.alert('eventLoop', `Event loop lag high: ${metrics.eventLoop}ms`);
    }
  }

  alert(type, message) {
    const alert = { type, message, timestamp: Date.now() };
    this.alerts.add(alert);
    this.emit('alert', alert);
  }

  setupEventListeners() {
    process.on('uncaughtException', (error) => {
      this.alert('uncaughtException', error.message);
      this.cleanup();
      process.exit(1);
    });

    process.on('unhandledRejection', (reason) => {
      this.alert('unhandledRejection', reason);
    });

    process.on('SIGTERM', () => {
      this.cleanup();
      process.exit(0);
    });
  }

  cleanup() {
    clearInterval(this.interval);
    // Additional cleanup tasks
  }

  getMetrics() {
    return {
      metrics: this.metrics,
      alerts: Array.from(this.alerts)
    };
  }
}
```

## 3. Worker Thread Management

### Thread Pool Implementation
```javascript
const { Worker } = require('worker_threads');
const path = require('path');

class ThreadPool {
  constructor(options = {}) {
    this.size = options.size || require('os').cpus().length;
    this.taskQueue = [];
    this.workers = [];
    this.activeWorkers = new Map();
    
    this.initialize();
  }

  initialize() {
    for (let i = 0; i < this.size; i++) {
      const worker = new Worker(path.join(__dirname, 'worker.js'));
      
      worker.on('message', (result) => {
        this.handleTaskCompletion(worker, result);
      });
      
      worker.on('error', (error) => {
        this.handleWorkerError(worker, error);
      });
      
      worker.on('exit', (code) => {
        this.handleWorkerExit(worker, code);
      });
      
      this.workers.push(worker);
    }
  }

  async executeTask(task) {
    return new Promise((resolve, reject) => {
      const taskWrapper = {
        id: Date.now() + Math.random(),
        task,
        resolve,
        reject
      };

      const availableWorker = this.getAvailableWorker();
      
      if (availableWorker) {
        this.assignTask(availableWorker, taskWrapper);
      } else {
        this.taskQueue.push(taskWrapper);
      }
    });
  }

  getAvailableWorker() {
    return this.workers.find(worker => !this.activeWorkers.has(worker));
  }

  assignTask(worker, taskWrapper) {
    this.activeWorkers.set(worker, taskWrapper);
    worker.postMessage({ id: taskWrapper.id, task: taskWrapper.task });
  }

  handleTaskCompletion(worker, result) {
    const taskWrapper = this.activeWorkers.get(worker);
    if (taskWrapper) {
      taskWrapper.resolve(result);
      this.activeWorkers.delete(worker);
      this.processNextTask(worker);
    }
  }

  handleWorkerError(worker, error) {
    const taskWrapper = this.activeWorkers.get(worker);
    if (taskWrapper) {
      taskWrapper.reject(error);
      this.activeWorkers.delete(worker);
      this.processNextTask(worker);
    }
  }

  handleWorkerExit(worker, code) {
    const index = this.workers.indexOf(worker);
    if (index !== -1) {
      this.workers.splice(index, 1);
      
      if (code !== 0) {
        const newWorker = new Worker(path.join(__dirname, 'worker.js'));
        this.workers.push(newWorker);
      }
    }
  }

  processNextTask(worker) {
    if (this.taskQueue.length > 0) {
      const nextTask = this.taskQueue.shift();
      this.assignTask(worker, nextTask);
    }
  }

  async shutdown() {
    for (const worker of this.workers) {
      worker.terminate();
    }
    this.workers = [];
    this.activeWorkers.clear();
  }
}

// Worker implementation (worker.js)
const { parentPort } = require('worker_threads');

parentPort.on('message', async ({ id, task }) => {
  try {
    const result = await executeTask(task);
    parentPort.postMessage({ id, result });
  } catch (error) {
    parentPort.postMessage({ id, error: error.message });
  }
});

function executeTask(task) {
  // Task execution logic
  return task();
}
```

## 4. Process Communication

### IPC Implementation
```javascript
const { fork } = require('child_process');

class ProcessManager {
  constructor() {
    this.workers = new Map();
    this.messageHandlers = new Map();
  }

  startWorker(id, scriptPath, env = {}) {
    const worker = fork(scriptPath, [], {
      env: { ...process.env, ...env },
      silent: true
    });

    worker.on('message', (message) => {
      this.handleWorkerMessage(id, message);
    });

    worker.on('error', (error) => {
      this.handleWorkerError(id, error);
    });

    worker.on('exit', (code) => {
      this.handleWorkerExit(id, code);
    });

    this.workers.set(id, worker);
    return worker;
  }

  sendToWorker(workerId, message) {
    const worker = this.workers.get(workerId);
    if (worker) {
      worker.send(message);
    }
  }

  broadcastToWorkers(message) {
    for (const worker of this.workers.values()) {
      worker.send(message);
    }
  }

  registerMessageHandler(type, handler) {
    this.messageHandlers.set(type, handler);
  }

  handleWorkerMessage(workerId, message) {
    const handler = this.messageHandlers.get(message.type);
    if (handler) {
      handler(workerId, message.data);
    }
  }

  handleWorkerError(workerId, error) {
    console.error(`Worker ${workerId} error:`, error);
    this.restartWorker(workerId);
  }

  handleWorkerExit(workerId, code) {
    console.log(`Worker ${workerId} exited with code ${code}`);
    this.workers.delete(workerId);
    
    if (code !== 0) {
      this.restartWorker(workerId);
    }
  }

  async restartWorker(workerId) {
    const worker = this.workers.get(workerId);
    if (worker) {
      worker.kill();
      this.workers.delete(workerId);
    }
    
    // Wait before restarting
    await new Promise(resolve => setTimeout(resolve, 1000));
    this.startWorker(workerId, worker.spawnfile);
  }

  shutdown() {
    for (const worker of this.workers.values()) {
      worker.kill();
    }
    this.workers.clear();
  }
}
```

## 5. Resource Management

### Memory Management
```javascript
class MemoryManager {
  constructor(options = {}) {
    this.options = {
      heapThreshold: options.heapThreshold || 0.8,
      gcInterval: options.gcInterval || 30000,
      ...options
    };
    
    this.metrics = {
      lastGC: 0,
      gcCount: 0,
      maxHeapUsed: 0
    };
  }

  start() {
    this.interval = setInterval(() => this.check(), this.options.gcInterval);
  }

  async check() {
    const memoryUsage = process.memoryUsage();
    const heapUsed = memoryUsage.heapUsed / memoryUsage.heapTotal;

    this.metrics.maxHeapUsed = Math.max(this.metrics.maxHeapUsed, heapUsed);

    if (heapUsed > this.options.heapThreshold) {
      await this.forceGC();
    }
  }

  async forceGC() {
    if (global.gc) {
      global.gc();
      this.metrics.lastGC = Date.now();
      this.metrics.gcCount++;
    }
  }

  getMetrics() {
    return {
      ...this.metrics,
      currentMemory: process.memoryUsage()
    };
  }

  stop() {
    clearInterval(this.interval);
  }
}
```

## Related Topics
- [[High-Availability]] - High availability implementation
- [[Fault-Tolerance]] - Fault tolerance patterns
- [[Monitoring]] - System monitoring
- [[Performance-Optimization]] - Performance optimization

## Practice Projects
1. Build a custom process manager
2. Implement worker thread pool
3. Create IPC system
4. Design memory management system

## Resources
- [Node.js Process Documentation](https://nodejs.org/api/process.html)
- [PM2 Documentation](https://pm2.keymetrics.io/)
- [[Learning-Resources#ProcessManagement|Process Management Resources]]

## Tags
#process-management #nodejs #threads #ipc #resource-management
