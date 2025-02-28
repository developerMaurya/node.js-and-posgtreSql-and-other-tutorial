# Basic Debugging in Node.js

Essential debugging techniques and tools for Node.js applications.

## Console Methods

### Basic Console Usage
```javascript
// Basic logging
console.log('Basic message');
console.info('Info message');
console.warn('Warning message');
console.error('Error message');

// Styled console output
console.log('\x1b[36m%s\x1b[0m', 'Cyan colored text');
console.log('\x1b[33m%s\x1b[0m', 'Yellow colored text');

// Object logging
const user = { name: 'John', age: 30 };
console.log('User:', user);

// Table format
const users = [
    { name: 'John', age: 30 },
    { name: 'Jane', age: 25 }
];
console.table(users);

// Time tracking
console.time('operation');
// ... some operations
console.timeEnd('operation');

// Stack trace
console.trace('Trace message');
```

### Advanced Console Features
```javascript
// Group related logs
console.group('User Operations');
console.log('Fetching user...');
console.log('Processing data...');
console.log('Update complete');
console.groupEnd();

// Conditional logging
console.assert(1 === 2, 'This will show an error');

// Count occurrences
console.count('counter');
console.count('counter');
console.countReset('counter');
```

## Node.js Debugger

### Using Built-in Debugger
```javascript
// Add debugger statement
function calculateTotal(items) {
    debugger; // Execution will pause here
    return items.reduce((total, item) => total + item.price, 0);
}

// Run with node inspect
// $ node inspect app.js
```

### Debug Commands
```bash
# Basic debug commands
cont, c    - Continue execution
step, s    - Step next
next, n    - Step over
out, o     - Step out
pause      - Pause running code

# Breakpoint commands
setBreakpoint(), sb()    - Set breakpoint on current line
setBreakpoint(line), sb(line)
clearBreakpoint(), cb()
```

## VS Code Debugging

### Launch Configuration
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Debug Program",
            "program": "${workspaceFolder}/app.js",
            "skipFiles": [
                "<node_internals>/**"
            ]
        },
        {
            "type": "node",
            "request": "attach",
            "name": "Attach to Process",
            "port": 9229
        }
    ]
}
```

### Debug Scripts
```json
// package.json
{
    "scripts": {
        "start": "node app.js",
        "debug": "node --inspect app.js",
        "debug-brk": "node --inspect-brk app.js"
    }
}
```

## Basic Logging

### Winston Logger Setup
```javascript
const winston = require('winston');

// Create logger
const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' })
    ]
});

// Add console transport in development
if (process.env.NODE_ENV !== 'production') {
    logger.add(new winston.transports.Console({
        format: winston.format.simple()
    }));
}

// Usage
logger.info('Application started');
logger.error('Error occurred', { error: 'Details here' });
```

### Debug Module
```javascript
const debug = require('debug');

// Create debug loggers
const dbDebug = debug('app:db');
const authDebug = debug('app:auth');

// Usage
dbDebug('Connected to database');
authDebug('User authenticated');

// Run with DEBUG environment variable
// $ DEBUG=app:* node app.js
// $ DEBUG=app:db node app.js
```

## Error Tracking

### Basic Error Handler
```javascript
process.on('uncaughtException', (error) => {
    console.error('Uncaught Exception:', error);
    // Log to file or error tracking service
    process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
    // Log to file or error tracking service
});
```

### Custom Error Handler
```javascript
class ErrorHandler {
    constructor() {
        this.handleError = this.handleError.bind(this);
    }

    handleError(error) {
        console.error('Error occurred:', error);
        
        // Log error details
        this.logError(error);
        
        // Determine if server should crash
        if (!this.isTrustedError(error)) {
            process.exit(1);
        }
    }

    logError(error) {
        // Log to file system
        const timestamp = new Date().toISOString();
        const errorLog = `${timestamp}: ${error.stack}\n`;
        
        // Could write to file or send to logging service
        console.error(errorLog);
    }

    isTrustedError(error) {
        // Determine if error is operational
        return error.isOperational;
    }
}

// Usage
const errorHandler = new ErrorHandler();
process.on('uncaughtException', errorHandler.handleError);
process.on('unhandledRejection', errorHandler.handleError);
```

## Performance Monitoring

### Basic Performance Hooks
```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

// Create performance observer
const obs = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    entries.forEach((entry) => {
        console.log(`${entry.name}: ${entry.duration}`);
    });
});
obs.observe({ entryTypes: ['measure'] });

// Measure performance
performance.mark('A');
// ... some operations
performance.mark('B');
performance.measure('A to B', 'A', 'B');
```

## Related Topics
- [[Error-Handling-Advanced]] - Advanced error handling
- [[Logging-Strategies]] - Comprehensive logging
- [[Performance-Monitoring]] - Advanced monitoring
- [[Testing]] - Testing strategies

## Practice Projects
1. Create a custom logger
2. Build an error tracking system
3. Implement performance monitoring
4. Create a debugging tool

## Resources
- [Node.js Debugging Guide](https://nodejs.org/en/docs/guides/debugging-getting-started/)
- [VS Code Debugging](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)
- [[Learning-Resources#Debugging|Debugging Resources]]

## Tags
#nodejs #debugging #logging #error-handling #performance
