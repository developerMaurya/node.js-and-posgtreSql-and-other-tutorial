# Events and EventEmitter in Node.js

Node.js uses an event-driven architecture where certain types of objects (emitters) emit named events that cause listeners to be called. Understanding events is crucial for Node.js development.

## EventEmitter Basics

### 1. Creating an EventEmitter
```javascript
const EventEmitter = require('events');

// Create a custom event emitter
class MyEmitter extends EventEmitter {
    constructor() {
        super();
    }
}

// Or use directly
const myEmitter = new EventEmitter();
```

### 2. Basic Event Handling
```javascript
const myEmitter = new EventEmitter();

// Register event listener
myEmitter.on('event', (arg1, arg2) => {
    console.log('Event occurred:', arg1, arg2);
});

// Emit event
myEmitter.emit('event', 'Hello', 'World');
```

## Event Listener Methods

### 1. Different Ways to Listen
```javascript
const myEmitter = new EventEmitter();

// Listen for all occurrences
myEmitter.on('event', callback);

// Listen only once
myEmitter.once('event', callback);

// Prepend listener (will be called first)
myEmitter.prependListener('event', callback);

// Prepend once listener
myEmitter.prependOnceListener('event', callback);
```

### 2. Removing Listeners
```javascript
function handleEvent(arg) {
    console.log('Event handled:', arg);
}

// Add listener
myEmitter.on('event', handleEvent);

// Remove specific listener
myEmitter.removeListener('event', handleEvent);
// or
myEmitter.off('event', handleEvent);

// Remove all listeners for specific event
myEmitter.removeAllListeners('event');

// Remove all listeners for all events
myEmitter.removeAllListeners();
```

## Custom EventEmitter Class

### 1. Basic Implementation
```javascript
const EventEmitter = require('events');

class DatabaseEmitter extends EventEmitter {
    constructor() {
        super();
        this.connected = false;
    }

    connect() {
        // Simulate database connection
        setTimeout(() => {
            this.connected = true;
            this.emit('connected', { timestamp: Date.now() });
        }, 1000);
    }

    query(sql) {
        if (!this.connected) {
            this.emit('error', new Error('Not connected'));
            return;
        }
        
        // Simulate query
        setTimeout(() => {
            this.emit('result', { 
                sql,
                rows: [],
                timestamp: Date.now()
            });
        }, 500);
    }
}

// Usage
const db = new DatabaseEmitter();

db.on('connected', (info) => {
    console.log('Database connected at:', info.timestamp);
    db.query('SELECT * FROM users');
});

db.on('result', (data) => {
    console.log('Query result:', data);
});

db.on('error', (error) => {
    console.error('Database error:', error);
});

db.connect();
```

### 2. Real-World Example: File Watcher
```javascript
class FileWatcher extends EventEmitter {
    constructor(directory) {
        super();
        this.directory = directory;
        this.watcher = null;
    }

    start() {
        const fs = require('fs');
        
        try {
            this.watcher = fs.watch(this.directory, (eventType, filename) => {
                if (filename) {
                    this.emit('fileChanged', {
                        type: eventType,
                        filename,
                        timestamp: Date.now()
                    });
                }
            });

            this.emit('watching', this.directory);
        } catch (error) {
            this.emit('error', error);
        }
    }

    stop() {
        if (this.watcher) {
            this.watcher.close();
            this.emit('stopped');
        }
    }
}

// Usage
const watcher = new FileWatcher('./myDirectory');

watcher.on('watching', (dir) => {
    console.log(`Watching directory: ${dir}`);
});

watcher.on('fileChanged', (info) => {
    console.log(`File ${info.filename} ${info.type} at ${info.timestamp}`);
});

watcher.on('stopped', () => {
    console.log('Watcher stopped');
});

watcher.on('error', (error) => {
    console.error('Watcher error:', error);
});

watcher.start();
```

## Error Handling

### 1. Error Events
```javascript
const myEmitter = new EventEmitter();

// Always handle 'error' events
myEmitter.on('error', (error) => {
    console.error('An error occurred:', error);
});

// If no error listener is registered, Node.js will crash
myEmitter.emit('error', new Error('Something went wrong'));
```

### 2. Async Error Handling
```javascript
class AsyncOperationEmitter extends EventEmitter {
    async performOperation() {
        try {
            await this.riskyOperation();
            this.emit('success');
        } catch (error) {
            this.emit('error', error);
        }
    }

    async riskyOperation() {
        // Simulate async operation that might fail
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                Math.random() > 0.5 
                    ? resolve()
                    : reject(new Error('Operation failed'));
            }, 1000);
        });
    }
}
```

## Best Practices

### 1. Memory Management
```javascript
// Set maximum listeners (default is 10)
myEmitter.setMaxListeners(20);

// Get current maximum listeners
console.log(myEmitter.getMaxListeners());

// Check number of listeners
console.log(myEmitter.listenerCount('event'));

// Get array of listeners
console.log(myEmitter.listeners('event'));
```

### 2. Event Names
```javascript
// Use constants for event names
const EVENTS = {
    CONNECT: 'connect',
    DISCONNECT: 'disconnect',
    DATA: 'data',
    ERROR: 'error'
};

class Connection extends EventEmitter {
    connect() {
        // ... connection logic
        this.emit(EVENTS.CONNECT);
    }

    disconnect() {
        // ... disconnection logic
        this.emit(EVENTS.DISCONNECT);
    }
}
```

### 3. Cleanup
```javascript
class CleanupExample extends EventEmitter {
    constructor() {
        super();
        this.setup();
    }

    setup() {
        // Attach listeners
        this.boundHandler = this.handleEvent.bind(this);
        process.on('SIGINT', this.boundHandler);
    }

    cleanup() {
        // Remove listeners to prevent memory leaks
        process.removeListener('SIGINT', this.boundHandler);
        this.removeAllListeners();
    }

    handleEvent() {
        console.log('Handling event');
    }
}
```

## Debugging Events

### 1. Monitoring Events
```javascript
// List all event names
console.log(myEmitter.eventNames());

// Debug event emissions
myEmitter.on('newListener', (event, listener) => {
    console.log(`New listener added for event: ${event}`);
});

myEmitter.on('removeListener', (event, listener) => {
    console.log(`Listener removed for event: ${event}`);
});
```

### 2. Event Tracking
```javascript
class TrackedEmitter extends EventEmitter {
    emit(event, ...args) {
        console.log(`Event '${event}' emitted with args:`, args);
        return super.emit(event, ...args);
    }
}
```

## Related Topics
- [[Async-Programming]] - Asynchronous operations
- [[Error-Handling-Advanced]] - Error management
- [[Streams-and-Buffers]] - Stream events
- [[Design-Patterns]] - Event-driven patterns

Tags: #nodejs #events #eventemitter #async #patterns
