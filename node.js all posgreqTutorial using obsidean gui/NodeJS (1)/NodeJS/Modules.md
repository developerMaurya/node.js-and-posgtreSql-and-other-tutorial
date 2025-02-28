# Node.js Modules

Modules are the building blocks of Node.js applications. They help organize code into reusable components.

## Types of Modules

### 1. Core Modules
Built-in modules that come with Node.js:
- `fs` - File System operations
- `http` - HTTP server and client
- `path` - Path manipulation
- `os` - Operating system information
- `events` - Event handling

### 2. Local Modules
Your own files:
```javascript
// math.js
exports.add = (a, b) => a + b;
exports.subtract = (a, b) => a - b;

// main.js
const math = require('./math');
console.log(math.add(5, 3)); // 8
```

### 3. Third-party Modules
Installed via npm:
```javascript
const express = require('express');
const app = express();
```

## Module Patterns

### CommonJS Pattern
```javascript
// Exporting
module.exports = {
    function1,
    function2
};

// Importing
const module = require('./module');
```

### ES Modules (ESM)
```javascript
// Exporting
export const function1 = () => {};
export default class MainClass {};

// Importing
import { function1 } from './module';
import MainClass from './module';
```

## Best Practices
- One responsibility per module
- Clear and descriptive naming
- Proper error handling
- Documentation for public APIs

## Related Topics
- [[npm]] - Package management
- [[NodeJS-Fundamentals]] - Core concepts
- [[Project-Structure]] - Organizing modules

Tags: #nodejs #modules #javascript #commonjs #esm
