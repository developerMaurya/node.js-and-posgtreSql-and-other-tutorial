# Node.js Fundamentals

Node.js is a JavaScript runtime built on Chrome's V8 JavaScript engine.

## Key Concepts
- [[V8-Engine]] - The JavaScript engine that powers Node.js
- [[Event-Loop]] - How Node.js handles asynchronous operations
- [[Global-Objects]] - Built-in objects available in Node.js

## Core Features
- Single-threaded event loop model
- Non-blocking I/O operations
- Event-driven programming

## Basic Usage
```javascript
// Hello World example
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
});

server.listen(3000);
```

## Related Topics
- [[Modules]] - How to organize code in Node.js
- [[npm]] - Managing packages and dependencies
- [[Event-Emitters]] - Understanding the event system

Tags: #nodejs #fundamentals #javascript #backend
