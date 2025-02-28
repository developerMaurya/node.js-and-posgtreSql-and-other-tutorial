# Child Processes in Node.js

A comprehensive guide to managing child processes in Node.js for running external commands, scripts, and parallel processing.

## Basic Child Process Methods

### spawn() - For Long-Running Processes
```javascript
const { spawn } = require('child_process');

// Run a long-running process
const child = spawn('python', ['script.py'], {
    stdio: 'pipe',
    shell: true
});

// Handle data events
child.stdout.on('data', (data) => {
    console.log(`stdout: ${data}`);
});

child.stderr.on('data', (data) => {
    console.error(`stderr: ${data}`);
});

child.on('close', (code) => {
    console.log(`Child process exited with code ${code}`);
});

// Send data to child process
child.stdin.write('some input\n');
child.stdin.end();
```

### exec() - For Command Line Operations
```javascript
const { exec } = require('child_process');

// Execute command
exec('ls -la', (error, stdout, stderr) => {
    if (error) {
        console.error(`Error: ${error}`);
        return;
    }
    if (stderr) {
        console.error(`stderr: ${stderr}`);
        return;
    }
    console.log(`stdout: ${stdout}`);
});

// With Promise wrapper
function execPromise(command) {
    return new Promise((resolve, reject) => {
        exec(command, (error, stdout, stderr) => {
            if (error) {
                reject(error);
                return;
            }
            resolve(stdout.trim());
        });
    });
}

// Usage
async function runCommand() {
    try {
        const result = await execPromise('git status');
        console.log(result);
    } catch (error) {
        console.error('Failed to execute command:', error);
    }
}
```

### execFile() - For Executable Files
```javascript
const { execFile } = require('child_process');

execFile('node', ['--version'], (error, stdout, stderr) => {
    if (error) {
        console.error('Error:', error);
        return;
    }
    console.log('Node version:', stdout);
});
```

### fork() - For Node.js Scripts
```javascript
// parent.js
const { fork } = require('child_process');

const child = fork('child.js');

// Send message to child
child.send({ hello: 'world' });

// Receive messages from child
child.on('message', (message) => {
    console.log('Received from child:', message);
});

// Handle child exit
child.on('exit', (code, signal) => {
    console.log('Child process exited with code', code);
});

// child.js
process.on('message', (message) => {
    console.log('Received from parent:', message);
    process.send({ received: true });
});
```

## Process Pool Management

### Process Pool Implementation
```javascript
class ProcessPool {
    constructor(scriptPath, poolSize = 4) {
        this.scriptPath = scriptPath;
        this.poolSize = poolSize;
        this.workers = new Map();
        this.taskQueue = [];
        this.initialize();
    }

    initialize() {
        for (let i = 0; i < this.poolSize; i++) {
            this.addWorker();
        }
    }

    addWorker() {
        const worker = fork(this.scriptPath);
        const id = Date.now() + Math.random();
        
        worker.on('message', (message) => {
            if (message.type === 'result') {
                this.handleTaskCompletion(id, null, message.data);
            }
        });

        worker.on('error', (error) => {
            this.handleTaskCompletion(id, error);
        });

        worker.on('exit', (code) => {
            if (code !== 0) {
                console.log(`Worker ${id} died. Spawning new worker...`);
                this.workers.delete(id);
                this.addWorker();
            }
        });

        this.workers.set(id, {
            process: worker,
            busy: false
        });
    }

    handleTaskCompletion(workerId, error, result) {
        const worker = this.workers.get(workerId);
        if (!worker) return;

        const task = worker.currentTask;
        if (!task) return;

        worker.busy = false;
        worker.currentTask = null;

        if (error) {
            task.reject(error);
        } else {
            task.resolve(result);
        }

        this.processNextTask();
    }

    async executeTask(data) {
        return new Promise((resolve, reject) => {
            const task = { data, resolve, reject };
            this.taskQueue.push(task);
            this.processNextTask();
        });
    }

    processNextTask() {
        if (this.taskQueue.length === 0) return;

        const availableWorker = Array.from(this.workers.entries())
            .find(([_, worker]) => !worker.busy);

        if (!availableWorker) return;

        const [workerId, worker] = availableWorker;
        const task = this.taskQueue.shift();

        worker.busy = true;
        worker.currentTask = task;
        worker.process.send({ type: 'task', data: task.data });
    }

    shutdown() {
        for (const [_, worker] of this.workers) {
            worker.process.kill();
        }
        this.workers.clear();
    }
}

// Usage
const pool = new ProcessPool('./worker.js', 4);

async function processData(data) {
    try {
        const result = await pool.executeTask(data);
        console.log('Task completed:', result);
    } catch (error) {
        console.error('Task failed:', error);
    }
}
```

## Inter-Process Communication (IPC)

### Advanced Message Passing
```javascript
// parent.js
const { fork } = require('child_process');
const children = new Map();

// Create multiple child processes
for (let i = 0; i < 3; i++) {
    const child = fork('./child.js');
    children.set(child.pid, child);

    // Handle messages from child
    child.on('message', (message) => {
        if (message.type === 'broadcast') {
            // Broadcast to other children
            for (const [pid, proc] of children) {
                if (pid !== child.pid) {
                    proc.send({
                        type: 'broadcast',
                        from: child.pid,
                        data: message.data
                    });
                }
            }
        }
    });
}

// child.js
process.on('message', (message) => {
    if (message.type === 'broadcast') {
        console.log(`Process ${process.pid} received broadcast from ${message.from}`);
    }
});

// Broadcast message to other processes
process.send({
    type: 'broadcast',
    data: `Hello from ${process.pid}`
});
```

### Shared State Management
```javascript
// shared-state.js
const { EventEmitter } = require('events');
const state = new EventEmitter();
let data = {};

state.setItem = (key, value) => {
    data[key] = value;
    state.emit('change', { key, value });
};

state.getItem = (key) => data[key];

module.exports = state;

// parent.js
const { fork } = require('child_process');
const children = [];

for (let i = 0; i < 2; i++) {
    const child = fork('./worker.js');
    children.push(child);

    child.on('message', (message) => {
        if (message.type === 'state_change') {
            // Propagate state change to other children
            children.forEach(otherChild => {
                if (otherChild.pid !== child.pid) {
                    otherChild.send({
                        type: 'state_update',
                        data: message.data
                    });
                }
            });
        }
    });
}

// worker.js
const state = require('./shared-state');

process.on('message', (message) => {
    if (message.type === 'state_update') {
        state.setItem(message.data.key, message.data.value);
    }
});

state.on('change', (change) => {
    process.send({
        type: 'state_change',
        data: change
    });
});
```

## Process Monitoring and Management

### Health Check System
```javascript
class ProcessMonitor {
    constructor() {
        this.processes = new Map();
        this.healthChecks = new Map();
    }

    addProcess(child, options = {}) {
        const pid = child.pid;
        this.processes.set(pid, {
            process: child,
            startTime: Date.now(),
            restarts: 0,
            lastActive: Date.now()
        });

        // Setup health check interval
        if (options.healthCheck) {
            const interval = setInterval(() => {
                this.checkProcessHealth(pid);
            }, options.healthCheckInterval || 5000);
            
            this.healthChecks.set(pid, interval);
        }

        // Monitor process events
        child.on('message', () => {
            const proc = this.processes.get(pid);
            if (proc) {
                proc.lastActive = Date.now();
            }
        });

        child.on('exit', (code) => {
            if (code !== 0 && options.autoRestart) {
                this.restartProcess(pid);
            }
        });
    }

    checkProcessHealth(pid) {
        const proc = this.processes.get(pid);
        if (!proc) return;

        const inactiveDuration = Date.now() - proc.lastActive;
        if (inactiveDuration > 30000) { // 30 seconds threshold
            console.warn(`Process ${pid} appears to be unresponsive`);
            this.restartProcess(pid);
        }
    }

    async restartProcess(pid) {
        const proc = this.processes.get(pid);
        if (!proc) return;

        // Kill existing process
        proc.process.kill();

        // Clear health check interval
        clearInterval(this.healthChecks.get(pid));
        this.healthChecks.delete(pid);

        // Start new process
        const newChild = fork(proc.process.spawnfile);
        proc.process = newChild;
        proc.restarts++;
        proc.lastActive = Date.now();

        console.log(`Process ${pid} restarted. Total restarts: ${proc.restarts}`);
    }

    getProcessStats() {
        const stats = {};
        for (const [pid, proc] of this.processes) {
            stats[pid] = {
                uptime: Date.now() - proc.startTime,
                restarts: proc.restarts,
                lastActive: proc.lastActive
            };
        }
        return stats;
    }
}
```

### Resource Usage Monitoring
```javascript
class ResourceMonitor {
    constructor(process) {
        this.process = process;
        this.stats = {
            cpu: [],
            memory: []
        };
        this.maxSamples = 60; // Keep last 60 samples
    }

    startMonitoring(interval = 1000) {
        setInterval(() => {
            this.collectMetrics();
        }, interval);
    }

    collectMetrics() {
        const usage = process.cpuUsage();
        const memory = process.memoryUsage();

        this.stats.cpu.push({
            timestamp: Date.now(),
            usage: {
                user: usage.user,
                system: usage.system
            }
        });

        this.stats.memory.push({
            timestamp: Date.now(),
            usage: {
                rss: memory.rss,
                heapTotal: memory.heapTotal,
                heapUsed: memory.heapUsed,
                external: memory.external
            }
        });

        // Keep only last maxSamples
        if (this.stats.cpu.length > this.maxSamples) {
            this.stats.cpu.shift();
        }
        if (this.stats.memory.length > this.maxSamples) {
            this.stats.memory.shift();
        }
    }

    getMetrics() {
        return {
            cpu: this.calculateCPUMetrics(),
            memory: this.calculateMemoryMetrics()
        };
    }

    calculateCPUMetrics() {
        if (this.stats.cpu.length < 2) return null;

        const latest = this.stats.cpu[this.stats.cpu.length - 1];
        const previous = this.stats.cpu[this.stats.cpu.length - 2];

        const userDiff = latest.usage.user - previous.usage.user;
        const systemDiff = latest.usage.system - previous.usage.system;
        const timeDiff = latest.timestamp - previous.timestamp;

        return {
            user: userDiff / timeDiff,
            system: systemDiff / timeDiff,
            total: (userDiff + systemDiff) / timeDiff
        };
    }

    calculateMemoryMetrics() {
        if (this.stats.memory.length === 0) return null;

        const latest = this.stats.memory[this.stats.memory.length - 1];
        return latest.usage;
    }
}
```

## Best Practices

### 1. Error Handling
```javascript
class ProcessManager {
    handleProcessError(process, error) {
        console.error(`Process ${process.pid} error:`, error);
        
        // Log error details
        this.logError({
            pid: process.pid,
            error: error.message,
            stack: error.stack,
            timestamp: new Date()
        });

        // Determine if process should be restarted
        if (this.shouldRestart(process)) {
            this.restartProcess(process);
        }
    }

    shouldRestart(process) {
        const stats = this.getProcessStats(process.pid);
        return stats.restarts < 5 && // Max restart attempts
               Date.now() - stats.startTime > 60000; // Minimum uptime
    }

    logError(errorDetails) {
        // Implement error logging logic
        // e.g., write to file, send to monitoring service
    }
}
```

### 2. Graceful Shutdown
```javascript
class GracefulShutdown {
    constructor(processes) {
        this.processes = processes;
        this.isShuttingDown = false;
        
        process.on('SIGTERM', () => this.shutdown());
        process.on('SIGINT', () => this.shutdown());
    }

    async shutdown() {
        if (this.isShuttingDown) return;
        this.isShuttingDown = true;

        console.log('Initiating graceful shutdown...');

        // Stop accepting new tasks
        this.stopAcceptingTasks();

        // Wait for ongoing tasks to complete
        await this.waitForTasks();

        // Terminate child processes
        await this.terminateProcesses();

        // Cleanup resources
        await this.cleanup();

        process.exit(0);
    }

    async terminateProcesses() {
        const shutdownPromises = [];

        for (const [pid, process] of this.processes) {
            shutdownPromises.push(
                new Promise((resolve) => {
                    process.on('exit', resolve);
                    process.send({ type: 'shutdown' });

                    // Force kill after timeout
                    setTimeout(() => {
                        process.kill('SIGKILL');
                    }, 5000);
                })
            );
        }

        await Promise.all(shutdownPromises);
    }
}
```

## Related Topics
- [[Process-Management]] - Process management strategies
- [[IPC-Patterns]] - Inter-process communication patterns
- [[Performance-Monitoring]] - Monitoring and metrics
- [[Error-Handling-Advanced]] - Error handling strategies

## Practice Projects
1. Build a process pool manager
2. Create a process monitoring dashboard
3. Implement a distributed task processor
4. Develop a process health check system

## Resources
- [Node.js Child Process Documentation](https://nodejs.org/api/child_process.html)
- [Process Management Best Practices](https://nodejs.org/en/docs/guides/process-management/)
- [[Learning-Resources#ChildProcesses|Child Processes Resources]]

## Tags
#child-process #process-management #ipc #monitoring #nodejs
