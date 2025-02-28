# Error Handling and Debugging in Node.js

A comprehensive guide to error handling, debugging, logging, and monitoring in Node.js applications.

## 1. Error Handling Patterns

### Custom Error Classes
```javascript
class AppError extends Error {
    constructor(message, statusCode = 500, isOperational = true) {
        super(message);
        this.statusCode = statusCode;
        this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
        this.isOperational = isOperational;

        Error.captureStackTrace(this, this.constructor);
    }
}

class ValidationError extends AppError {
    constructor(message) {
        super(message, 400);
        this.name = 'ValidationError';
    }
}

class DatabaseError extends AppError {
    constructor(message) {
        super(message, 500, false);
        this.name = 'DatabaseError';
    }
}

class NotFoundError extends AppError {
    constructor(resource = 'Resource') {
        super(`${resource} not found`, 404);
        this.name = 'NotFoundError';
    }
}

class AuthenticationError extends AppError {
    constructor(message = 'Authentication failed') {
        super(message, 401);
        this.name = 'AuthenticationError';
    }
}
```

### Global Error Handler
```javascript
const errorHandler = (err, req, res, next) => {
    err.statusCode = err.statusCode || 500;
    err.status = err.status || 'error';

    if (process.env.NODE_ENV === 'development') {
        sendErrorDev(err, res);
    } else {
        sendErrorProd(err, res);
    }
};

const sendErrorDev = (err, res) => {
    res.status(err.statusCode).json({
        status: err.status,
        error: err,
        message: err.message,
        stack: err.stack
    });
};

const sendErrorProd = (err, res) => {
    // Operational, trusted error: send message to client
    if (err.isOperational) {
        res.status(err.statusCode).json({
            status: err.status,
            message: err.message
        });
    } 
    // Programming or other unknown error: don't leak error details
    else {
        console.error('ERROR 💥', err);
        res.status(500).json({
            status: 'error',
            message: 'Something went wrong'
        });
    }
};

// Express error handling middleware
app.use(errorHandler);
```

### Async Error Handler
```javascript
const catchAsync = (fn) => {
    return (req, res, next) => {
        fn(req, res, next).catch(next);
    };
};

// Usage in routes
app.get('/api/users/:id', catchAsync(async (req, res) => {
    const user = await User.findById(req.params.id);
    if (!user) {
        throw new NotFoundError('User');
    }
    res.json(user);
}));
```

### Process Level Error Handling
```javascript
process.on('uncaughtException', (err) => {
    console.error('UNCAUGHT EXCEPTION! 💥 Shutting down...');
    console.error(err.name, err.message, err.stack);
    process.exit(1);
});

process.on('unhandledRejection', (err) => {
    console.error('UNHANDLED REJECTION! 💥 Shutting down...');
    console.error(err.name, err.message);
    server.close(() => {
        process.exit(1);
    });
});

process.on('SIGTERM', () => {
    console.log('👋 SIGTERM RECEIVED. Shutting down gracefully');
    server.close(() => {
        console.log('💥 Process terminated!');
    });
});
```

## 2. Debugging Techniques

### Debug Module Integration
```javascript
const debug = require('debug');

// Create namespaced debuggers
const debugHttp = debug('app:http');
const debugDb = debug('app:db');
const debugAuth = debug('app:auth');

class UserService {
    async getUser(id) {
        debugHttp(`Fetching user with id: ${id}`);
        
        try {
            debugDb('Querying database...');
            const user = await User.findById(id);
            
            if (!user) {
                debugHttp(`User ${id} not found`);
                throw new NotFoundError('User');
            }
            
            debugHttp(`User ${id} found`);
            return user;
        } catch (error) {
            debugDb(`Database error: ${error.message}`);
            throw error;
        }
    }

    async authenticate(email, password) {
        debugAuth(`Attempting authentication for ${email}`);
        
        try {
            const user = await User.findOne({ email });
            if (!user) {
                debugAuth(`Authentication failed: user not found`);
                throw new AuthenticationError();
            }

            const isValid = await user.comparePassword(password);
            if (!isValid) {
                debugAuth(`Authentication failed: invalid password`);
                throw new AuthenticationError();
            }

            debugAuth(`Authentication successful for ${email}`);
            return user;
        } catch (error) {
            debugAuth(`Authentication error: ${error.message}`);
            throw error;
        }
    }
}
```

### Performance Debugging
```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

class PerformanceDebugger {
    constructor() {
        this.marks = new Map();
        this.setupObserver();
    }

    setupObserver() {
        const obs = new PerformanceObserver((list) => {
            const entries = list.getEntries();
            entries.forEach((entry) => {
                console.log(`${entry.name}: ${entry.duration}ms`);
            });
        });
        obs.observe({ entryTypes: ['measure'], buffered: true });
    }

    startMeasure(name) {
        const markName = `${name}_start`;
        performance.mark(markName);
        this.marks.set(name, markName);
    }

    endMeasure(name) {
        const startMark = this.marks.get(name);
        if (!startMark) return;

        const endMark = `${name}_end`;
        performance.mark(endMark);
        performance.measure(name, startMark, endMark);
        this.marks.delete(name);
    }
}

// Usage
const perfDebugger = new PerformanceDebugger();

async function complexOperation() {
    perfDebugger.startMeasure('complexOp');
    
    // Perform operation
    await someComplexTask();
    
    perfDebugger.endMeasure('complexOp');
}
```

### Memory Leak Detection
```javascript
const v8 = require('v8');
const fs = require('fs');

class MemoryDebugger {
    constructor(options = {}) {
        this.heapDumpPath = options.heapDumpPath || './heapdumps';
        this.thresholdMB = options.thresholdMB || 1024; // 1GB
        this.checkInterval = options.checkInterval || 30000; // 30 seconds
    }

    start() {
        this.interval = setInterval(() => {
            this.checkMemoryUsage();
        }, this.checkInterval);
    }

    stop() {
        if (this.interval) {
            clearInterval(this.interval);
        }
    }

    checkMemoryUsage() {
        const used = process.memoryUsage();
        const heapUsedMB = used.heapUsed / 1024 / 1024;

        console.log(`Memory Usage:
            - Heap Used: ${heapUsedMB.toFixed(2)} MB
            - Heap Total: ${(used.heapTotal / 1024 / 1024).toFixed(2)} MB
            - RSS: ${(used.rss / 1024 / 1024).toFixed(2)} MB
            - External: ${(used.external / 1024 / 1024).toFixed(2)} MB`);

        if (heapUsedMB > this.thresholdMB) {
            this.generateHeapDump();
        }
    }

    generateHeapDump() {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = `${this.heapDumpPath}/heapdump-${timestamp}.heapsnapshot`;

        const heapSnapshot = v8.getHeapSnapshot();
        fs.writeFileSync(filename, JSON.stringify(heapSnapshot));
        console.log(`Heap dump written to ${filename}`);
    }
}
```

## 3. Logging Strategies

### Advanced Logger Implementation
```javascript
const winston = require('winston');
const { format } = winston;

class Logger {
    constructor(options = {}) {
        this.options = {
            level: options.level || 'info',
            service: options.service || 'app',
            ...options
        };

        this.logger = winston.createLogger({
            level: this.options.level,
            format: format.combine(
                format.timestamp(),
                format.errors({ stack: true }),
                format.metadata(),
                format.json()
            ),
            defaultMeta: { service: this.options.service },
            transports: this.setupTransports()
        });
    }

    setupTransports() {
        const transports = [
            new winston.transports.File({
                filename: 'logs/error.log',
                level: 'error',
                maxsize: 5242880, // 5MB
                maxFiles: 5
            }),
            new winston.transports.File({
                filename: 'logs/combined.log',
                maxsize: 5242880,
                maxFiles: 5
            })
        ];

        if (process.env.NODE_ENV !== 'production') {
            transports.push(
                new winston.transports.Console({
                    format: format.combine(
                        format.colorize(),
                        format.simple()
                    )
                })
            );
        }

        return transports;
    }

    log(level, message, meta = {}) {
        this.logger.log(level, message, meta);
    }

    error(message, meta = {}) {
        this.logger.error(message, meta);
    }

    warn(message, meta = {}) {
        this.logger.warn(message, meta);
    }

    info(message, meta = {}) {
        this.logger.info(message, meta);
    }

    debug(message, meta = {}) {
        this.logger.debug(message, meta);
    }

    // Request logging middleware
    requestLogger() {
        return (req, res, next) => {
            const start = Date.now();

            res.on('finish', () => {
                const duration = Date.now() - start;
                this.info('HTTP Request', {
                    method: req.method,
                    url: req.url,
                    status: res.statusCode,
                    duration,
                    ip: req.ip,
                    userAgent: req.get('user-agent')
                });
            });

            next();
        };
    }

    // Error logging middleware
    errorLogger() {
        return (err, req, res, next) => {
            this.error('Request Error', {
                error: err.message,
                stack: err.stack,
                method: req.method,
                url: req.url,
                body: req.body
            });
            next(err);
        };
    }
}
```

## 4. Monitoring and Alerting

### Application Monitor
```javascript
const EventEmitter = require('events');
const os = require('os');

class AppMonitor extends EventEmitter {
    constructor(options = {}) {
        super();
        this.thresholds = {
            cpu: options.cpuThreshold || 80,
            memory: options.memoryThreshold || 80,
            requests: options.requestThreshold || 1000
        };
        
        this.metrics = {
            requests: 0,
            errors: 0,
            avgResponseTime: 0
        };

        this.startMonitoring();
    }

    startMonitoring() {
        // Monitor system metrics
        setInterval(() => {
            this.checkSystemMetrics();
        }, 5000);

        // Reset request counts every minute
        setInterval(() => {
            this.metrics.requests = 0;
            this.metrics.errors = 0;
        }, 60000);
    }

    checkSystemMetrics() {
        const cpuUsage = this.getCPUUsage();
        const memoryUsage = this.getMemoryUsage();

        if (cpuUsage > this.thresholds.cpu) {
            this.emit('alert', {
                type: 'cpu',
                message: `High CPU usage: ${cpuUsage}%`,
                value: cpuUsage
            });
        }

        if (memoryUsage > this.thresholds.memory) {
            this.emit('alert', {
                type: 'memory',
                message: `High memory usage: ${memoryUsage}%`,
                value: memoryUsage
            });
        }
    }

    getCPUUsage() {
        const cpus = os.cpus();
        const usage = cpus.reduce((acc, cpu) => {
            const total = Object.values(cpu.times).reduce((a, b) => a + b);
            const idle = cpu.times.idle;
            return acc + ((total - idle) / total) * 100;
        }, 0);

        return usage / cpus.length;
    }

    getMemoryUsage() {
        const used = process.memoryUsage().heapUsed;
        const total = os.totalmem();
        return (used / total) * 100;
    }

    // Express middleware for request monitoring
    requestMonitor() {
        return (req, res, next) => {
            const start = Date.now();
            this.metrics.requests++;

            res.on('finish', () => {
                const duration = Date.now() - start;
                
                // Update average response time
                this.metrics.avgResponseTime = 
                    (this.metrics.avgResponseTime + duration) / 2;

                if (res.statusCode >= 400) {
                    this.metrics.errors++;
                }

                // Check request threshold
                if (this.metrics.requests > this.thresholds.requests) {
                    this.emit('alert', {
                        type: 'requests',
                        message: `High request count: ${this.metrics.requests}`,
                        value: this.metrics.requests
                    });
                }
            });

            next();
        };
    }

    getMetrics() {
        return {
            system: {
                cpu: this.getCPUUsage(),
                memory: this.getMemoryUsage(),
                uptime: process.uptime()
            },
            application: {
                requests: this.metrics.requests,
                errors: this.metrics.errors,
                avgResponseTime: this.metrics.avgResponseTime
            }
        };
    }
}

// Usage
const monitor = new AppMonitor();

monitor.on('alert', (alert) => {
    console.log(`⚠️ Alert: ${alert.message}`);
    // Send alert to monitoring service
    // notifyTeam(alert);
});

// Express middleware
app.use(monitor.requestMonitor());

// Metrics endpoint
app.get('/metrics', (req, res) => {
    res.json(monitor.getMetrics());
});
```

## Related Topics
- [[Logging-Best-Practices]] - Advanced logging techniques
- [[Performance-Monitoring]] - Application performance monitoring
- [[Error-Handling-Advanced]] - Error handling patterns
- [[Debugging-Tools]] - Debugging tools and techniques

## Practice Projects
1. Build a centralized logging system
2. Create a real-time monitoring dashboard
3. Implement a comprehensive error handling system
4. Develop a performance profiling tool

## Resources
- [Node.js Debugging Guide](https://nodejs.org/en/docs/guides/debugging-getting-started/)
- [Winston Documentation](https://github.com/winstonjs/winston)
- [[Learning-Resources#Debugging|Debugging Resources]]

## Tags
#error-handling #debugging #logging #monitoring #nodejs
