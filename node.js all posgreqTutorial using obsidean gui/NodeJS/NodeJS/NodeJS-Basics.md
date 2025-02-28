# Node.js Basics

An introduction to Node.js, its core concepts, and fundamental principles.

## What is Node.js?

Node.js is a JavaScript runtime built on Chrome's V8 JavaScript engine. It allows you to:
- Run JavaScript outside the browser
- Build server-side applications
- Create command-line tools
- Develop desktop applications

### Key Features
- Non-blocking I/O
- Event-driven architecture
- Single-threaded event loop
- NPM (Node Package Manager)
- Cross-platform compatibility

## How Node.js Works

### Event Loop
```javascript
// Example of event loop in action
console.log('Starting');

setTimeout(() => {
    console.log('2 seconds passed');
}, 2000);

setImmediate(() => {
    console.log('Immediate execution');
});

process.nextTick(() => {
    console.log('Next tick execution');
});

console.log('Ending');

// Output:
// Starting
// Ending
// Next tick execution
// Immediate execution
// 2 seconds passed
```

### Single-Threaded Nature
```javascript
// Single thread handling multiple operations
const fs = require('fs');

// Non-blocking file read
fs.readFile('file.txt', 'utf8', (err, data) => {
    if (err) {
        console.error('Error reading file:', err);
        return;
    }
    console.log('File content:', data);
});

console.log('Continue executing while file is being read');
```

## Installation & Setup

### Direct Installation
1. Download from [nodejs.org](https://nodejs.org)
2. Follow installation wizard
3. Verify installation:
```bash
node --version
npm --version
```

### Using Node Version Manager (nvm)
```bash
# Install nvm (Windows)
nvm install latest
nvm use <version>

# Install specific version
nvm install 14.17.0
nvm use 14.17.0
```

## Running JavaScript in Node.js

### REPL (Read-Eval-Print Loop)
```bash
# Start REPL
node

# Basic operations
> 2 + 2
4
> const name = 'Node.js'
undefined
> console.log(`Hello, ${name}!`)
Hello, Node.js!
```

### Running Files
```bash
# Create a file
echo "console.log('Hello, Node.js!')" > hello.js

# Run the file
node hello.js
```

## Node.js Execution Model

### Global Objects
```javascript
// Available globally
console.log(__dirname);  // Current directory
console.log(__filename); // Current file
console.log(process);    // Process information
console.log(module);     // Current module
console.log(require);    // Module loader
```

### Module System
```javascript
// math.js
module.exports = {
    add: (a, b) => a + b,
    subtract: (a, b) => a - b
};

// main.js
const math = require('./math');
console.log(math.add(5, 3));      // 8
console.log(math.subtract(5, 3)); // 2
```

### Process Events
```javascript
// Handle process events
process.on('exit', (code) => {
    console.log(`Process exiting with code: ${code}`);
});

process.on('uncaughtException', (err) => {
    console.error('Uncaught exception:', err);
    process.exit(1);
});

// Environment variables
console.log(process.env.NODE_ENV);
```

## Best Practices

1. Error Handling
```javascript
// Always handle errors in callbacks
fs.readFile('file.txt', (err, data) => {
    if (err) {
        console.error('Error:', err);
        return;
    }
    // Process data
});

// Use try-catch for synchronous code
try {
    const data = fs.readFileSync('file.txt');
    // Process data
} catch (err) {
    console.error('Error:', err);
}
```

2. Asynchronous Programming
```javascript
// Use async/await for cleaner code
async function readFileAsync(path) {
    try {
        const data = await fs.promises.readFile(path, 'utf8');
        return data;
    } catch (err) {
        console.error('Error reading file:', err);
        throw err;
    }
}
```

3. Environment Configuration
```javascript
// Use environment variables for configuration
require('dotenv').config();

const config = {
    port: process.env.PORT || 3000,
    env: process.env.NODE_ENV || 'development',
    dbUrl: process.env.DATABASE_URL
};
```

## Common Pitfalls

1. Blocking Operations
```javascript
// Bad: Blocking operation
const data = fs.readFileSync('large-file.txt');

// Good: Non-blocking operation
fs.readFile('large-file.txt', (err, data) => {
    // Handle file data
});
```

2. Callback Hell
```javascript
// Bad: Callback hell
fs.readFile('file1.txt', (err, data1) => {
    fs.readFile('file2.txt', (err, data2) => {
        fs.readFile('file3.txt', (err, data3) => {
            // Nested callbacks
        });
    });
});

// Good: Using async/await
async function readFiles() {
    const data1 = await fs.promises.readFile('file1.txt');
    const data2 = await fs.promises.readFile('file2.txt');
    const data3 = await fs.promises.readFile('file3.txt');
    return [data1, data2, data3];
}
```

## Related Topics
- [[Event-Loop]] - Detailed explanation of the event loop
- [[Async-Programming]] - Asynchronous programming patterns
- [[Error-Handling-Advanced]] - Error handling strategies
- [[Node-Process]] - Node.js process and environment
- [[Debugging-Basics]] - Basic debugging techniques

## Practice Projects
1. Create a simple file reader
2. Build a basic HTTP server
3. Implement a command-line tool
4. Create a simple event emitter

## Resources
- [Official Node.js Documentation](https://nodejs.org/docs)
- [Node.js Design Patterns](https://www.nodejsdesignpatterns.com/)
- [[Learning-Resources#NodeJS|Node.js Learning Resources]]

## Tags
#nodejs #basics #javascript #runtime #event-loop
