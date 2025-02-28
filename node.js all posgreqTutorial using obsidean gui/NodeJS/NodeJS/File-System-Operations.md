# File System Operations in Node.js

Node.js provides powerful file system operations through the `fs` module. This guide covers both synchronous and asynchronous file operations.

## Basic File Operations

### 1. Reading Files
```javascript
const fs = require('fs');
const fsPromises = require('fs').promises;

// Asynchronous with promises (recommended)
async function readFileExample() {
    try {
        const data = await fsPromises.readFile('file.txt', 'utf8');
        console.log(data);
    } catch (error) {
        console.error('Error reading file:', error);
    }
}

// Asynchronous with callbacks
fs.readFile('file.txt', 'utf8', (err, data) => {
    if (err) {
        console.error('Error:', err);
        return;
    }
    console.log(data);
});

// Synchronous (blocking) - use with caution
try {
    const data = fs.readFileSync('file.txt', 'utf8');
    console.log(data);
} catch (error) {
    console.error('Error:', error);
}
```

### 2. Writing Files
```javascript
const content = 'Hello, Node.js!';

// Asynchronous with promises
async function writeFileExample() {
    try {
        await fsPromises.writeFile('output.txt', content);
        console.log('File written successfully');
    } catch (error) {
        console.error('Error writing file:', error);
    }
}

// Asynchronous with callbacks
fs.writeFile('output.txt', content, (err) => {
    if (err) {
        console.error('Error:', err);
        return;
    }
    console.log('File written successfully');
});

// Synchronous
try {
    fs.writeFileSync('output.txt', content);
    console.log('File written successfully');
} catch (error) {
    console.error('Error:', error);
}
```

## Directory Operations

### 1. Creating Directories
```javascript
// Create directory
async function createDirectoryExample() {
    try {
        await fsPromises.mkdir('new-directory');
        console.log('Directory created');
    } catch (error) {
        if (error.code === 'EEXIST') {
            console.log('Directory already exists');
        } else {
            console.error('Error:', error);
        }
    }
}

// Create nested directories
async function createNestedDirectories() {
    try {
        await fsPromises.mkdir('parent/child/grandchild', { recursive: true });
        console.log('Nested directories created');
    } catch (error) {
        console.error('Error:', error);
    }
}
```

### 2. Reading Directories
```javascript
async function readDirectoryExample() {
    try {
        const files = await fsPromises.readdir('directory-path');
        
        // Get detailed file information
        const fileStats = await Promise.all(
            files.map(async file => {
                const stats = await fsPromises.stat(`directory-path/${file}`);
                return {
                    name: file,
                    isDirectory: stats.isDirectory(),
                    size: stats.size,
                    created: stats.birthtime,
                    modified: stats.mtime
                };
            })
        );
        
        console.log(fileStats);
    } catch (error) {
        console.error('Error:', error);
    }
}
```

## File Streams

### 1. Reading Streams
```javascript
const readStream = fs.createReadStream('large-file.txt', {
    encoding: 'utf8',
    highWaterMark: 64 * 1024 // 64KB chunks
});

readStream.on('data', (chunk) => {
    console.log('Received chunk:', chunk.length);
});

readStream.on('end', () => {
    console.log('Finished reading file');
});

readStream.on('error', (error) => {
    console.error('Error:', error);
});
```

### 2. Writing Streams
```javascript
const writeStream = fs.createWriteStream('output.txt');

writeStream.write('Hello, ');
writeStream.write('World!');
writeStream.end();

writeStream.on('finish', () => {
    console.log('Finished writing');
});

writeStream.on('error', (error) => {
    console.error('Error:', error);
});
```

### 3. Piping Streams
```javascript
const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');

// Pipe data from read stream to write stream
readStream.pipe(writeStream);

// Handle events
readStream.on('error', console.error);
writeStream.on('error', console.error);
writeStream.on('finish', () => console.log('Copy completed'));
```

## File Watching

### 1. Watch File Changes
```javascript
// Watch a file for changes
fs.watch('file.txt', (eventType, filename) => {
    console.log(`Event: ${eventType}`);
    if (filename) {
        console.log(`File changed: ${filename}`);
    }
});

// Watch directory for changes
fs.watch('directory', { recursive: true }, (eventType, filename) => {
    console.log(`Event: ${eventType}`);
    if (filename) {
        console.log(`Changed: ${filename}`);
    }
});
```

## Advanced Operations

### 1. File Information
```javascript
async function getFileInfo(path) {
    try {
        const stats = await fsPromises.stat(path);
        return {
            size: stats.size,
            created: stats.birthtime,
            modified: stats.mtime,
            isFile: stats.isFile(),
            isDirectory: stats.isDirectory(),
            permissions: stats.mode
        };
    } catch (error) {
        console.error('Error:', error);
    }
}
```

### 2. File Operations with Temporary Files
```javascript
const os = require('os');
const path = require('path');

async function workWithTempFile() {
    const tempPath = path.join(os.tmpdir(), 'temp-' + Date.now());
    
    try {
        // Write to temp file
        await fsPromises.writeFile(tempPath, 'Temporary data');
        
        // Process the file
        const data = await fsPromises.readFile(tempPath, 'utf8');
        console.log('Temp file data:', data);
        
        // Clean up
        await fsPromises.unlink(tempPath);
    } catch (error) {
        console.error('Error:', error);
    }
}
```

### 3. Recursive Directory Operations
```javascript
async function recursivelyProcessDirectory(dirPath) {
    try {
        const entries = await fsPromises.readdir(dirPath, { withFileTypes: true });
        
        for (const entry of entries) {
            const fullPath = path.join(dirPath, entry.name);
            
            if (entry.isDirectory()) {
                await recursivelyProcessDirectory(fullPath);
            } else {
                await processFile(fullPath);
            }
        }
    } catch (error) {
        console.error('Error:', error);
    }
}

async function processFile(filePath) {
    // Process individual file
    console.log(`Processing: ${filePath}`);
}
```

## Best Practices

1. **Error Handling**
   - Always use try-catch with async operations
   - Handle specific error codes appropriately
   - Clean up resources in finally blocks

2. **Performance**
   - Use streams for large files
   - Avoid synchronous operations in production
   - Consider memory usage with large directories

3. **Security**
   - Validate file paths
   - Check file permissions
   - Handle sensitive data appropriately

## Related Topics
- [[Streams-and-Buffers]] - Detailed stream operations
- [[Error-Handling-Advanced]] - Error management
- [[Security]] - Security considerations
- [[Performance-Optimization]] - Performance tips

Tags: #nodejs #filesystem #streams #async #files
