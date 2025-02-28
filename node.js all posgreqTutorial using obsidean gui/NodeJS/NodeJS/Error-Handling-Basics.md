# Error Handling Basics in Node.js

Proper error handling is crucial for building robust Node.js applications. This guide covers different types of errors and how to handle them effectively.

## Types of Errors

### 1. Built-in Error Types
```javascript
// Standard JavaScript errors
const standardErrors = {
    Error: new Error('Generic error'),
    SyntaxError: new SyntaxError('Invalid syntax'),
    ReferenceError: new ReferenceError('Variable not defined'),
    TypeError: new TypeError('Invalid type'),
    RangeError: new RangeError('Value out of range'),
    URIError: new URIError('Invalid URI')
};

// Node.js specific errors
const systemErrors = {
    ENOENT: 'No such file or directory',
    EACCES: 'Permission denied',
    EADDRINUSE: 'Address already in use',
    ECONNREFUSED: 'Connection refused'
};
```

### 2. Custom Errors
```javascript
class ValidationError extends Error {
    constructor(message) {
        super(message);
        this.name = 'ValidationError';
        this.code = 'VALIDATION_ERROR';
        
        // Capture stack trace
        Error.captureStackTrace(this, ValidationError);
    }
}

class DatabaseError extends Error {
    constructor(message, query) {
        super(message);
        this.name = 'DatabaseError';
        this.query = query;
        this.timestamp = new Date();
    }
}
```

## Synchronous Error Handling

### 1. Try-Catch Blocks
```javascript
function divide(a, b) {
    try {
        if (b === 0) {
            throw new Error('Division by zero');
        }
        return a / b;
    } catch (error) {
        console.error('Error in division:', error.message);
        throw error; // Re-throw if you can't handle it here
    } finally {
        // Cleanup code (always executed)
        console.log('Division operation attempted');
    }
}

// Usage
try {
    const result = divide(10, 0);
} catch (error) {
    console.error('Could not perform division:', error);
}
```

### 2. Error Propagation
```javascript
function validateUser(user) {
    if (!user.name) {
        throw new ValidationError('Name is required');
    }
    if (!user.email) {
        throw new ValidationError('Email is required');
    }
}

function createUser(userData) {
    try {
        validateUser(userData);
        // Process user creation
    } catch (error) {
        if (error instanceof ValidationError) {
            console.error('Validation failed:', error.message);
        } else {
            console.error('Unknown error:', error);
            throw error; // Re-throw unknown errors
        }
    }
}
```

## Asynchronous Error Handling

### 1. Promises
```javascript
function fetchData() {
    return new Promise((resolve, reject) => {
        // Simulate API call
        setTimeout(() => {
            const success = Math.random() > 0.5;
            if (success) {
                resolve({ data: 'Success' });
            } else {
                reject(new Error('Failed to fetch data'));
            }
        }, 1000);
    });
}

// Using .then/.catch
fetchData()
    .then(result => {
        console.log('Data:', result);
    })
    .catch(error => {
        console.error('Error:', error);
    })
    .finally(() => {
        console.log('Operation completed');
    });

// Using async/await
async function getData() {
    try {
        const result = await fetchData();
        console.log('Data:', result);
    } catch (error) {
        console.error('Error:', error);
    }
}
```

### 2. Event Emitters
```javascript
const EventEmitter = require('events');

class DatabaseConnection extends EventEmitter {
    connect() {
        // Simulate database connection
        setTimeout(() => {
            if (Math.random() > 0.5) {
                this.emit('connected');
            } else {
                this.emit('error', new DatabaseError('Connection failed'));
            }
        }, 1000);
    }
}

const db = new DatabaseConnection();

db.on('connected', () => {
    console.log('Successfully connected to database');
});

db.on('error', (error) => {
    console.error('Database error:', error);
});
```

## Error Handling Patterns

### 1. Error-First Callback Pattern
```javascript
function readConfig(path, callback) {
    fs.readFile(path, 'utf8', (error, data) => {
        if (error) {
            return callback(error);
        }
        try {
            const config = JSON.parse(data);
            callback(null, config);
        } catch (parseError) {
            callback(parseError);
        }
    });
}

// Usage
readConfig('config.json', (error, config) => {
    if (error) {
        console.error('Error reading config:', error);
        return;
    }
    console.log('Config loaded:', config);
});
```

### 2. Middleware Error Handling (Express.js)
```javascript
const express = require('express');
const app = express();

// Regular middleware
app.use((req, res, next) => {
    // Simulate an error
    if (req.path === '/error') {
        next(new Error('Simulated error'));
    }
    next();
});

// Error handling middleware
app.use((error, req, res, next) => {
    console.error('Error occurred:', error);
    res.status(500).json({
        error: {
            message: error.message,
            stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
        }
    });
});
```

## Best Practices

### 1. Operational vs Programmer Errors
```javascript
// Operational errors (can be handled)
function handleOperationalError(error) {
    if (error.code === 'ENOENT') {
        // Handle missing file
        return createDefaultConfig();
    }
    if (error.code === 'ECONNREFUSED') {
        // Handle connection failure
        return reconnectWithRetry();
    }
}

// Programmer errors (should crash)
function validateInput(data) {
    if (!data) {
        // This is a programmer error
        throw new TypeError('Data is required');
    }
}
```

### 2. Error Recovery Strategies
```javascript
class RetryOperation {
    constructor(operation, maxRetries = 3) {
        this.operation = operation;
        this.maxRetries = maxRetries;
    }

    async execute() {
        let lastError;
        
        for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
            try {
                return await this.operation();
            } catch (error) {
                lastError = error;
                console.log(`Attempt ${attempt} failed:`, error);
                
                if (attempt < this.maxRetries) {
                    await this.delay(attempt * 1000); // Exponential backoff
                }
            }
        }
        
        throw new Error(`Operation failed after ${this.maxRetries} attempts: ${lastError}`);
    }

    delay(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Usage
const retryOperation = new RetryOperation(async () => {
    // Risky operation
    return await fetchData();
});

retryOperation.execute()
    .then(result => console.log('Success:', result))
    .catch(error => console.error('Final error:', error));
```

### 3. Logging and Monitoring
```javascript
class ErrorLogger {
    static log(error, context = {}) {
        const errorInfo = {
            message: error.message,
            stack: error.stack,
            code: error.code,
            timestamp: new Date().toISOString(),
            context
        };

        // Log to console
        console.error('Error occurred:', errorInfo);

        // Could also log to file or external service
        // this.logToFile(errorInfo);
        // this.sendToErrorTracking(errorInfo);
    }

    static logToFile(errorInfo) {
        // Implementation for file logging
    }

    static sendToErrorTracking(errorInfo) {
        // Implementation for error tracking service
    }
}
```

## Related Topics
- [[Async-Programming]] - Asynchronous error handling
- [[Express-Framework]] - Express.js error handling
- [[Logging]] - Error logging strategies
- [[Testing]] - Error testing strategies

Tags: #nodejs #error-handling #exceptions #async #best-practices
