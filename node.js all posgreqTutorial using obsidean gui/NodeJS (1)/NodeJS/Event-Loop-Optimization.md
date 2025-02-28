# Event Loop Optimization in Node.js

A comprehensive guide to understanding and optimizing the Node.js event loop for maximum performance.

## Understanding the Event Loop

### Event Loop Phases
```javascript
// The event loop phases in order:
// 1. timers: setTimeout(), setInterval()
// 2. pending callbacks: I/O callbacks
// 3. idle, prepare: internal use
// 4. poll: retrieve new I/O events
// 5. check: setImmediate() callbacks
// 6. close callbacks: socket.on('close', ...)
```

### Monitoring Event Loop Lag
```javascript
const toobusy = require('toobusy-js');
const express = require('express');
const app = express();

// Set maximum lag threshold (in ms)
toobusy.maxLag(70);

// Monitor event loop lag
app.use((req, res, next) => {
    if (toobusy()) {
        res.status(503).send("Server is too busy right now");
        return;
    }
    next();
});

// Get current lag
app.get('/health', (req, res) => {
    res.json({
        lag: toobusy.lag(),
        status: toobusy() ? 'busy' : 'normal'
    });
});

process.on('SIGINT', () => {
    toobusy.shutdown();
    process.exit();
});
```

## CPU-Intensive Operations

### Moving CPU Work to Worker Threads
```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    // Main thread code
    const worker = new Worker(__filename);
    
    app.get('/compute', (req, res) => {
        worker.postMessage({ number: req.query.n });
        
        worker.once('message', result => {
            res.json({ result });
        });
    });
} else {
    // Worker thread code
    parentPort.on('message', data => {
        const result = fibonacci(data.number);
        parentPort.postMessage(result);
    });
}

function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
```

### Using setImmediate for Breaking Up Work
```javascript
function processLargeArray(array, callback) {
    const results = [];
    let index = 0;

    function next() {
        if (index === array.length) {
            callback(results);
            return;
        }

        // Process chunk of work
        for (let i = 0; i < 1000 && index < array.length; i++) {
            results.push(heavyOperation(array[index++]));
        }

        // Schedule next chunk
        setImmediate(next);
    }

    next();
}

// Usage
const largeArray = Array(1000000).fill().map((_, i) => i);
processLargeArray(largeArray, results => {
    console.log('Processing complete');
});
```

## I/O Operations

### Optimizing Database Queries
```javascript
const mongoose = require('mongoose');
const { promisify } = require('util');

// Connection pooling
mongoose.connect('mongodb://localhost/test', {
    poolSize: 10,
    bufferMaxEntries: 0,
    useNewUrlParser: true,
    useUnifiedTopology: true
});

// Batch operations
async function batchProcess(items, batchSize = 1000) {
    const operations = [];
    
    for (let i = 0; i < items.length; i += batchSize) {
        const batch = items.slice(i, i + batchSize);
        operations.push(
            Model.insertMany(batch, { ordered: false })
        );
    }
    
    return Promise.all(operations);
}

// Streaming large datasets
function streamLargeDataset() {
    return new Promise((resolve, reject) => {
        const results = [];
        const stream = Model.find().cursor();
        
        stream.on('data', doc => {
            results.push(doc);
            
            // Process in chunks to avoid memory issues
            if (results.length >= 1000) {
                stream.pause();
                processChunk(results.splice(0)).then(() => {
                    stream.resume();
                });
            }
        });
        
        stream.on('end', () => {
            processChunk(results).then(resolve);
        });
        
        stream.on('error', reject);
    });
}
```

### File System Operations
```javascript
const fs = require('fs');
const { pipeline } = require('stream');
const { promisify } = require('util');

// Stream large files
function copyLargeFile(source, destination) {
    return new Promise((resolve, reject) => {
        const readStream = fs.createReadStream(source);
        const writeStream = fs.createWriteStream(destination);
        
        pipeline(
            readStream,
            writeStream,
            (err) => {
                if (err) reject(err);
                else resolve();
            }
        );
    });
}

// Batch file operations
async function processManyFiles(files) {
    const batchSize = 100;
    const results = [];
    
    for (let i = 0; i < files.length; i += batchSize) {
        const batch = files.slice(i, i + batchSize);
        const batchPromises = batch.map(file => 
            processFile(file).catch(err => ({ file, error: err }))
        );
        
        const batchResults = await Promise.all(batchPromises);
        results.push(...batchResults);
        
        // Give event loop a chance to handle other events
        await new Promise(resolve => setImmediate(resolve));
    }
    
    return results;
}
```

## Memory Management

### Memory Leak Detection
```javascript
const heapdump = require('heapdump');
const memwatch = require('@airbnb/node-memwatch');

// Generate heap snapshot
process.on('SIGUSR2', () => {
    heapdump.writeSnapshot(`./heap-${Date.now()}.heapsnapshot`);
});

// Monitor memory leaks
let hd = new memwatch.HeapDiff();

// Check memory difference after operations
function checkMemory() {
    const diff = hd.end();
    console.log('Memory change:', diff.change);
    console.log('Memory by module:', diff.groups);
    
    // Start new comparison
    hd = new memwatch.HeapDiff();
}

// Monitor garbage collection
memwatch.on('stats', stats => {
    console.log('GC Stats:', stats);
});

memwatch.on('leak', info => {
    console.log('Memory leak detected:', info);
});
```

### Memory-Efficient Data Structures
```javascript
const LRU = require('lru-cache');

// Use LRU cache for memory-bounded caching
const cache = new LRU({
    max: 500,    // Maximum items
    maxAge: 1000 * 60 * 60, // Items live for 1 hour
    updateAgeOnGet: true,    // Update item age on access
    length: (item) => {
        // Calculate item size
        return JSON.stringify(item).length;
    }
});

// Use WeakMap for object references
const weakMap = new WeakMap();

function processObject(obj) {
    if (weakMap.has(obj)) {
        return weakMap.get(obj);
    }
    
    const result = expensiveOperation(obj);
    weakMap.set(obj, result);
    return result;
}
```

## Performance Monitoring

### Custom Metrics
```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

// Create performance marks
function measureOperation() {
    performance.mark('operation-start');
    
    // Perform operation
    someOperation();
    
    performance.mark('operation-end');
    performance.measure('operation', 'operation-start', 'operation-end');
}

// Monitor performance
const obs = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    entries.forEach((entry) => {
        console.log(`${entry.name}: ${entry.duration}`);
    });
});

obs.observe({ entryTypes: ['measure'], buffered: true });
```

### Event Loop Metrics
```javascript
const express = require('express');
const app = express();

// Track event loop lag
let lastLoop = Date.now();
const interval = setInterval(() => {
    const now = Date.now();
    const lag = now - lastLoop - 1000;
    lastLoop = now;
    
    if (lag > 100) {
        console.warn(`Event loop lag detected: ${lag}ms`);
    }
}, 1000);

// Track operation duration
app.use((req, res, next) => {
    const start = process.hrtime();
    
    res.on('finish', () => {
        const [seconds, nanoseconds] = process.hrtime(start);
        const duration = seconds * 1000 + nanoseconds / 1000000;
        
        console.log(`${req.method} ${req.url}: ${duration}ms`);
    });
    
    next();
});
```

## Best Practices

### 1. Async Operations
```javascript
// Bad - Blocking operation
function processData(data) {
    for (let item of data) {
        heavyOperation(item);
    }
}

// Good - Non-blocking with async iteration
async function processData(data) {
    for (let item of data) {
        await new Promise(resolve => setImmediate(resolve));
        heavyOperation(item);
    }
}

// Better - Parallel processing with concurrency control
async function processData(data, concurrency = 3) {
    const chunks = chunk(data, concurrency);
    
    for (const batch of chunks) {
        await Promise.all(
            batch.map(item => heavyOperation(item))
        );
    }
}
```

### 2. Error Handling
```javascript
// Handle uncaught exceptions
process.on('uncaughtException', (err) => {
    console.error('Uncaught Exception:', err);
    // Perform cleanup
    process.exit(1);
});

// Handle unhandled rejections
process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
});

// Handle graceful shutdown
process.on('SIGTERM', async () => {
    console.log('Received SIGTERM. Performing graceful shutdown...');
    
    // Close server
    await new Promise(resolve => server.close(resolve));
    
    // Close database connections
    await mongoose.connection.close();
    
    process.exit(0);
});
```

## Related Topics
- [[Performance-Monitoring]] - Detailed monitoring setup
- [[Worker-Threads]] - CPU-intensive operations
- [[Memory-Management]] - Memory optimization
- [[Debugging]] - Advanced debugging techniques

## Practice Projects
1. Build a real-time performance monitoring system
2. Implement a worker pool for CPU-intensive tasks
3. Create a memory-efficient caching system
4. Develop a load testing framework

## Resources
- [Node.js Event Loop Documentation](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Node.js Performance Guide](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/)
- [[Learning-Resources#EventLoop|Event Loop Resources]]

## Tags
#event-loop #performance #optimization #nodejs #async
