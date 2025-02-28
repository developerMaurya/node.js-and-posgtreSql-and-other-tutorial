# Core Modules in Node.js

Essential built-in modules that come with Node.js installation.

## File System (fs)

### Basic File Operations
```javascript
const fs = require('fs');

// Synchronous operations
try {
    // Read file
    const content = fs.readFileSync('file.txt', 'utf8');
    console.log('File content:', content);

    // Write file
    fs.writeFileSync('output.txt', 'Hello, Node.js!');

    // Append to file
    fs.appendFileSync('output.txt', '\nNew line');

    // Check if file exists
    const exists = fs.existsSync('file.txt');
} catch (err) {
    console.error('Error:', err);
}

// Asynchronous operations with promises
const fsPromises = fs.promises;

async function handleFiles() {
    try {
        // Read file
        const content = await fsPromises.readFile('file.txt', 'utf8');
        
        // Write file
        await fsPromises.writeFile('output.txt', 'Hello, Node.js!');
        
        // Append to file
        await fsPromises.appendFile('output.txt', '\nNew line');
        
        // Get file stats
        const stats = await fsPromises.stat('file.txt');
        console.log('File size:', stats.size);
    } catch (err) {
        console.error('Error:', err);
    }
}
```

### Directory Operations
```javascript
// Create directory
fs.mkdirSync('new-folder');

// Read directory
const files = fs.readdirSync('.');
console.log('Files:', files);

// Remove directory
fs.rmdirSync('empty-folder');

// Recursive directory removal
fs.rmSync('folder', { recursive: true, force: true });
```

## Path Module

### Path Operations
```javascript
const path = require('path');

// Join paths
const fullPath = path.join(__dirname, 'folder', 'file.txt');

// Resolve path
const resolvedPath = path.resolve('folder', 'file.txt');

// Get path components
console.log(path.basename(fullPath));  // file.txt
console.log(path.dirname(fullPath));   // directory path
console.log(path.extname(fullPath));   // .txt

// Parse path
const pathInfo = path.parse(fullPath);
console.log(pathInfo);
// {
//   root: 'C:\\',
//   dir: 'C:\\path\\to',
//   base: 'file.txt',
//   ext: '.txt',
//   name: 'file'
// }
```

## HTTP Module

### Creating HTTP Server
```javascript
const http = require('http');

// Create server
const server = http.createServer((req, res) => {
    // Set response header
    res.setHeader('Content-Type', 'application/json');
    
    // Handle different routes
    if (req.url === '/') {
        res.end(JSON.stringify({ message: 'Hello, World!' }));
    } else if (req.url === '/api') {
        res.end(JSON.stringify({ data: 'API endpoint' }));
    } else {
        res.statusCode = 404;
        res.end(JSON.stringify({ error: 'Not found' }));
    }
});

// Start server
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

### Making HTTP Requests
```javascript
// GET request
http.get('http://api.example.com', (res) => {
    let data = '';
    
    res.on('data', (chunk) => {
        data += chunk;
    });
    
    res.on('end', () => {
        console.log(JSON.parse(data));
    });
}).on('error', (err) => {
    console.error('Error:', err);
});

// POST request
const options = {
    hostname: 'api.example.com',
    port: 80,
    path: '/data',
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    }
};

const req = http.request(options, (res) => {
    let data = '';
    
    res.on('data', (chunk) => {
        data += chunk;
    });
    
    res.on('end', () => {
        console.log(JSON.parse(data));
    });
});

req.on('error', (err) => {
    console.error('Error:', err);
});

req.write(JSON.stringify({ key: 'value' }));
req.end();
```

## Events Module

### Event Emitter
```javascript
const EventEmitter = require('events');

// Create custom event emitter
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();

// Add event listener
myEmitter.on('event', (data) => {
    console.log('Event occurred:', data);
});

// Emit event
myEmitter.emit('event', { message: 'Hello!' });

// One-time event listener
myEmitter.once('oneTime', () => {
    console.log('This will only run once');
});

// Remove event listener
const listener = (data) => {
    console.log('Event data:', data);
};
myEmitter.on('event', listener);
myEmitter.removeListener('event', listener);
```

## OS Module

### System Information
```javascript
const os = require('os');

// System information
console.log('CPU Architecture:', os.arch());
console.log('CPUs:', os.cpus());
console.log('Total Memory:', os.totalmem());
console.log('Free Memory:', os.freemem());
console.log('OS Platform:', os.platform());
console.log('OS Release:', os.release());

// Network interfaces
console.log('Network Interfaces:', os.networkInterfaces());

// User information
console.log('Home Directory:', os.homedir());
console.log('Current User:', os.userInfo());
```

## Crypto Module

### Cryptographic Operations
```javascript
const crypto = require('crypto');

// Hash data
const hash = crypto.createHash('sha256');
hash.update('Hello, World!');
console.log('Hash:', hash.digest('hex'));

// Generate random bytes
const randomBytes = crypto.randomBytes(16);
console.log('Random Bytes:', randomBytes.toString('hex'));

// Encrypt data
const algorithm = 'aes-256-cbc';
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

function encrypt(text) {
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
}

function decrypt(encrypted) {
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
}

const text = 'Secret message';
const encrypted = encrypt(text);
const decrypted = decrypt(encrypted);

console.log('Original:', text);
console.log('Encrypted:', encrypted);
console.log('Decrypted:', decrypted);
```

## Util Module

### Utility Functions
```javascript
const util = require('util');

// Promisify callback-based function
const sleep = util.promisify(setTimeout);

async function delayedOperation() {
    console.log('Starting');
    await sleep(2000);
    console.log('After 2 seconds');
}

// Format strings
const formatted = util.format('Hello, %s!', 'World');
console.log(formatted);

// Type checking
console.log(util.types.isDate(new Date()));
console.log(util.types.isPromise(Promise.resolve()));

// Deprecation warning
util.deprecate(() => {
    console.log('Old function');
}, 'This function is deprecated')();
```

## Related Topics
- [[File-System-Operations]] - Detailed file system operations
- [[HTTP-Server]] - Building HTTP servers
- [[Event-Driven-Programming]] - Event-driven architecture
- [[Crypto-Operations]] - Cryptography in Node.js
- [[System-Integration]] - OS and system integration

## Practice Projects
1. Build a file manager
2. Create a simple web server
3. Implement an event-based logger
4. Build a crypto tool
5. Create a system monitor

## Resources
- [Node.js Documentation](https://nodejs.org/api/)
- [[Learning-Resources#CoreModules|Core Modules Resources]]

## Tags
#nodejs #core-modules #fs #http #events #crypto #util
