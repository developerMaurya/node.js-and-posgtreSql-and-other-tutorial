# Performance Monitoring and Optimization in Node.js

A comprehensive guide to monitoring, profiling, and optimizing Node.js applications for maximum performance and reliability.

## 1. Application Performance Monitoring (APM)

### Custom APM Implementation
```javascript
const EventEmitter = require('events');
const os = require('os');
const v8 = require('v8');

class PerformanceMonitor extends EventEmitter {
    constructor(options = {}) {
        super();
        this.interval = options.interval || 5000;
        this.metrics = {
            cpu: [],
            memory: [],
            eventLoop: [],
            handles: []
        };
        this.maxDataPoints = options.maxDataPoints || 100;
    }

    start() {
        this.timer = setInterval(() => {
            this.collectMetrics();
        }, this.interval);
    }

    stop() {
        clearInterval(this.timer);
    }

    collectMetrics() {
        const metrics = {
            timestamp: Date.now(),
            cpu: this.getCPUMetrics(),
            memory: this.getMemoryMetrics(),
            eventLoop: this.getEventLoopMetrics(),
            handles: this.getHandleMetrics()
        };

        Object.keys(this.metrics).forEach(key => {
            this.metrics[key].push(metrics[key]);
            if (this.metrics[key].length > this.maxDataPoints) {
                this.metrics[key].shift();
            }
        });

        this.emit('metrics', metrics);
    }

    getCPUMetrics() {
        const cpus = os.cpus();
        const totalUsage = cpus.reduce((acc, cpu) => {
            const total = Object.values(cpu.times).reduce((a, b) => a + b);
            const idle = cpu.times.idle;
            return acc + ((total - idle) / total);
        }, 0);

        return {
            usage: (totalUsage / cpus.length) * 100,
            loadAvg: os.loadavg()
        };
    }

    getMemoryMetrics() {
        const heapStats = v8.getHeapStatistics();
        const memoryUsage = process.memoryUsage();

        return {
            heapTotal: memoryUsage.heapTotal,
            heapUsed: memoryUsage.heapUsed,
            rss: memoryUsage.rss,
            external: memoryUsage.external,
            heapSizeLimit: heapStats.heap_size_limit,
            totalPhysicalSize: heapStats.total_physical_size,
            totalAvailable: heapStats.total_available_size
        };
    }

    getEventLoopMetrics() {
        // Requires installation of perf_hooks in Node.js
        const { performance } = require('perf_hooks');
        return {
            latency: performance.nodeTiming.duration,
            timestamp: performance.now()
        };
    }

    getHandleMetrics() {
        return {
            active: process._getActiveHandles().length,
            requests: process._getActiveRequests().length
        };
    }

    getMetricsSummary() {
        const latest = {
            cpu: this.metrics.cpu[this.metrics.cpu.length - 1],
            memory: this.metrics.memory[this.metrics.memory.length - 1],
            eventLoop: this.metrics.eventLoop[this.metrics.eventLoop.length - 1],
            handles: this.metrics.handles[this.metrics.handles.length - 1]
        };

        return {
            cpu: {
                current: latest.cpu.usage.toFixed(2) + '%',
                loadAverage: latest.cpu.loadAvg.map(load => load.toFixed(2))
            },
            memory: {
                heapUsed: (latest.memory.heapUsed / 1024 / 1024).toFixed(2) + 'MB',
                rss: (latest.memory.rss / 1024 / 1024).toFixed(2) + 'MB',
                heapTotal: (latest.memory.heapTotal / 1024 / 1024).toFixed(2) + 'MB'
            },
            eventLoop: {
                latency: latest.eventLoop.latency.toFixed(2) + 'ms'
            },
            handles: {
                active: latest.handles.active,
                requests: latest.handles.requests
            }
        };
    }
}

// Usage
const monitor = new PerformanceMonitor({ interval: 5000 });

monitor.on('metrics', (metrics) => {
    console.log('Performance Metrics:', monitor.getMetricsSummary());
});

monitor.start();
```

## 2. Memory and CPU Profiling

### Memory Leak Detection
```javascript
const heapdump = require('heapdump');
const path = require('path');

class MemoryProfiler {
    constructor(options = {}) {
        this.thresholdMB = options.thresholdMB || 1024; // 1GB
        this.heapDumpPath = options.heapDumpPath || './heapdumps';
        this.interval = options.interval || 30000; // 30 seconds
    }

    start() {
        this.timer = setInterval(() => {
            this.checkMemoryUsage();
        }, this.interval);
    }

    stop() {
        clearInterval(this.timer);
    }

    checkMemoryUsage() {
        const memoryUsage = process.memoryUsage();
        const heapUsedMB = memoryUsage.heapUsed / 1024 / 1024;

        if (heapUsedMB > this.thresholdMB) {
            this.generateHeapDump();
        }
    }

    generateHeapDump() {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = path.join(
            this.heapDumpPath,
            `heapdump-${timestamp}.heapsnapshot`
        );

        heapdump.writeSnapshot(filename, (err) => {
            if (err) {
                console.error('Failed to generate heap dump:', err);
            } else {
                console.log(`Heap dump written to ${filename}`);
            }
        });
    }
}

// Usage
const memoryProfiler = new MemoryProfiler({
    thresholdMB: 1024,
    interval: 30000
});

memoryProfiler.start();
```

### CPU Profiling
```javascript
const { PerformanceObserver, performance } = require('perf_hooks');
const fs = require('fs');

class CPUProfiler {
    constructor() {
        this.profiles = new Map();
        this.setupPerformanceObserver();
    }

    setupPerformanceObserver() {
        const obs = new PerformanceObserver((list) => {
            const entries = list.getEntries();
            entries.forEach((entry) => {
                if (this.profiles.has(entry.name)) {
                    const profile = this.profiles.get(entry.name);
                    profile.duration = entry.duration;
                    profile.endTime = entry.startTime + entry.duration;
                }
            });
        });

        obs.observe({ entryTypes: ['measure'], buffered: true });
    }

    startProfiling(name) {
        this.profiles.set(name, {
            startTime: performance.now(),
            startMark: `${name}_start`,
            endMark: `${name}_end`
        });

        performance.mark(`${name}_start`);
    }

    stopProfiling(name) {
        if (!this.profiles.has(name)) {
            return null;
        }

        performance.mark(`${name}_end`);
        performance.measure(name,
            `${name}_start`,
            `${name}_end`
        );

        const profile = this.profiles.get(name);
        this.profiles.delete(name);

        return {
            name,
            duration: profile.duration,
            startTime: profile.startTime,
            endTime: profile.endTime
        };
    }

    saveProfile(name, profile) {
        const filename = `cpu-profile-${name}-${Date.now()}.json`;
        fs.writeFileSync(filename, JSON.stringify(profile, null, 2));
        return filename;
    }
}

// Usage
const profiler = new CPUProfiler();

function complexOperation() {
    profiler.startProfiling('complexOp');

    // Simulate complex operation
    let result = 0;
    for (let i = 0; i < 1000000; i++) {
        result += Math.sqrt(i);
    }

    const profile = profiler.stopProfiling('complexOp');
    console.log('Operation Profile:', profile);
}
```

## 3. Performance Optimization

### Route Performance Monitoring
```javascript
const express = require('express');
const { performance } = require('perf_hooks');

function routePerformanceMiddleware() {
    return (req, res, next) => {
        const start = performance.now();
        const url = req.url;

        // Add response hook
        res.on('finish', () => {
            const duration = performance.now() - start;
            const status = res.statusCode;

            // Log performance metrics
            console.log({
                type: 'route_performance',
                url,
                method: req.method,
                status,
                duration: `${duration.toFixed(2)}ms`
            });

            // Emit metrics for monitoring
            if (global.performanceMonitor) {
                global.performanceMonitor.emit('routeMetric', {
                    url,
                    method: req.method,
                    status,
                    duration
                });
            }
        });

        next();
    };
}

// Usage
const app = express();
app.use(routePerformanceMiddleware());
```

### Database Query Optimization
```javascript
class QueryOptimizer {
    constructor(pool) {
        this.pool = pool;
        this.queryStats = new Map();
    }

    async executeQuery(sql, params = []) {
        const start = performance.now();
        const queryKey = this.generateQueryKey(sql);

        try {
            const result = await this.pool.query(sql, params);
            this.recordQueryStats(queryKey, performance.now() - start);
            return result;
        } catch (error) {
            this.recordQueryError(queryKey, error);
            throw error;
        }
    }

    generateQueryKey(sql) {
        // Normalize SQL by removing variable values
        return sql.replace(/\d+|'[^']*'/g, '?');
    }

    recordQueryStats(queryKey, duration) {
        if (!this.queryStats.has(queryKey)) {
            this.queryStats.set(queryKey, {
                count: 0,
                totalDuration: 0,
                avgDuration: 0,
                slowQueries: 0
            });
        }

        const stats = this.queryStats.get(queryKey);
        stats.count++;
        stats.totalDuration += duration;
        stats.avgDuration = stats.totalDuration / stats.count;

        if (duration > 100) { // Threshold for slow queries (100ms)
            stats.slowQueries++;
        }
    }

    recordQueryError(queryKey, error) {
        console.error('Query Error:', {
            query: queryKey,
            error: error.message,
            timestamp: new Date()
        });
    }

    getQueryStats() {
        const stats = {};
        for (const [query, metrics] of this.queryStats) {
            stats[query] = {
                count: metrics.count,
                avgDuration: `${metrics.avgDuration.toFixed(2)}ms`,
                slowQueries: metrics.slowQueries
            };
        }
        return stats;
    }

    getSlowestQueries(limit = 10) {
        return Array.from(this.queryStats.entries())
            .sort((a, b) => b[1].avgDuration - a[1].avgDuration)
            .slice(0, limit)
            .map(([query, metrics]) => ({
                query,
                avgDuration: `${metrics.avgDuration.toFixed(2)}ms`,
                count: metrics.count,
                slowQueries: metrics.slowQueries
            }));
    }
}
```

### Memory Optimization
```javascript
class MemoryOptimizer {
    constructor(options = {}) {
        this.maxCacheSize = options.maxCacheSize || 1000;
        this.maxCacheAge = options.maxCacheAge || 3600000; // 1 hour
        this.cache = new Map();
        this.lastCleanup = Date.now();
    }

    set(key, value) {
        this.cleanup();
        
        if (this.cache.size >= this.maxCacheSize) {
            const oldestKey = Array.from(this.cache.keys())[0];
            this.cache.delete(oldestKey);
        }

        this.cache.set(key, {
            value,
            timestamp: Date.now()
        });
    }

    get(key) {
        const item = this.cache.get(key);
        if (!item) return null;

        if (Date.now() - item.timestamp > this.maxCacheAge) {
            this.cache.delete(key);
            return null;
        }

        return item.value;
    }

    cleanup() {
        const now = Date.now();
        if (now - this.lastCleanup < 60000) return; // Clean up max once per minute

        for (const [key, item] of this.cache) {
            if (now - item.timestamp > this.maxCacheAge) {
                this.cache.delete(key);
            }
        }

        this.lastCleanup = now;
    }

    getStats() {
        return {
            size: this.cache.size,
            maxSize: this.maxCacheSize,
            ageLimit: this.maxCacheAge,
            lastCleanup: new Date(this.lastCleanup)
        };
    }
}
```

## 4. Performance Testing

### Load Testing Setup
```javascript
const autocannon = require('autocannon');
const { promisify } = require('util');

class LoadTester {
    constructor(options = {}) {
        this.defaults = {
            url: 'http://localhost:3000',
            connections: 10,
            pipelining: 1,
            duration: 10
        };
        this.options = { ...this.defaults, ...options };
    }

    async runTest(endpoint, customOptions = {}) {
        const testOptions = {
            ...this.options,
            ...customOptions,
            url: `${this.options.url}${endpoint}`
        };

        const result = await promisify(autocannon)(testOptions);
        return this.analyzeResults(result);
    }

    analyzeResults(results) {
        return {
            summary: {
                url: results.url,
                duration: `${results.duration}s`,
                connections: results.connections,
                pipelining: results.pipelining
            },
            metrics: {
                requests: {
                    average: results.requests.average,
                    mean: results.requests.mean,
                    stddev: results.requests.stddev,
                    min: results.requests.min,
                    max: results.requests.max,
                    total: results.requests.total,
                    sent: results.requests.sent
                },
                latency: {
                    average: `${results.latency.average}ms`,
                    mean: `${results.latency.mean}ms`,
                    stddev: `${results.latency.stddev}ms`,
                    min: `${results.latency.min}ms`,
                    max: `${results.latency.max}ms`,
                    p99: `${results.latency.p99}ms`
                },
                throughput: {
                    average: `${(results.throughput.average / 1024 / 1024).toFixed(2)}MB/s`,
                    mean: `${(results.throughput.mean / 1024 / 1024).toFixed(2)}MB/s`,
                    stddev: `${(results.throughput.stddev / 1024 / 1024).toFixed(2)}MB/s`,
                    min: `${(results.throughput.min / 1024 / 1024).toFixed(2)}MB/s`,
                    max: `${(results.throughput.max / 1024 / 1024).toFixed(2)}MB/s`
                },
                errors: results.errors,
                timeouts: results.timeouts,
                mismatches: results.mismatches,
                non2xx: results.non2xx,
                resets: results.resets
            }
        };
    }
}

// Usage
async function runLoadTests() {
    const tester = new LoadTester({
        url: 'http://localhost:3000',
        connections: 100,
        duration: 30
    });

    console.log('Running load tests...');

    // Test different endpoints
    const endpoints = [
        { path: '/api/users', method: 'GET' },
        { path: '/api/products', method: 'GET' },
        { path: '/api/orders', method: 'POST', headers: { 'content-type': 'application/json' } }
    ];

    for (const endpoint of endpoints) {
        console.log(`Testing ${endpoint.method} ${endpoint.path}...`);
        const results = await tester.runTest(endpoint.path, {
            method: endpoint.method,
            headers: endpoint.headers
        });
        console.log('Results:', JSON.stringify(results, null, 2));
    }
}
```

## Related Topics
- [[Memory-Management]] - Detailed memory management strategies
- [[CPU-Profiling]] - Advanced CPU profiling techniques
- [[Database-Optimization]] - Database performance optimization
- [[Load-Testing]] - Comprehensive load testing strategies

## Practice Projects
1. Build a real-time performance monitoring dashboard
2. Create a memory leak detection system
3. Implement a query performance analyzer
4. Develop a load testing suite

## Resources
- [Node.js Performance Documentation](https://nodejs.org/en/docs/guides/diagnostics-flamegraph/)
- [V8 Profiler Documentation](https://v8.dev/docs/profile)
- [[Learning-Resources#Performance|Performance Resources]]

## Tags
#performance #monitoring #profiling #optimization #nodejs
