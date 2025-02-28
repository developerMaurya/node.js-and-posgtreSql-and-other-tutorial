# Streams and Buffers in Node.js

Streams and buffers are fundamental concepts in Node.js for handling data flow and memory efficiency.

## Buffers

### 1. Buffer Basics
```javascript
// Creating buffers
const buf1 = Buffer.alloc(10);  // Creates a buffer of 10 bytes filled with zeros
const buf2 = Buffer.from('Hello, Node.js');  // Creates a buffer from string
const buf3 = Buffer.from([1, 2, 3, 4]);  // Creates a buffer from array

// Writing to buffers
buf1.write('Hello');
console.log(buf1.toString());  // 'Hello'

// Buffer operations
console.log(buf1.length);  // Buffer size
console.log(buf1[0]);  // Access individual bytes
```

### 2. Buffer Manipulation
```javascript
// Copying buffers
const source = Buffer.from('Hello');
const target = Buffer.alloc(5);
source.copy(target);

// Slicing buffers
const slice = source.slice(0, 2);  // Creates a new buffer with first 2 bytes

// Concatenating buffers
const combined = Buffer.concat([buf1, buf2]);

// Compare buffers
const isEqual = buf1.equals(buf2);
```

## Streams

### 1. Types of Streams
```javascript
const fs = require('fs');

// Readable Stream
const readStream = fs.createReadStream('input.txt', {
    encoding: 'utf8',
    highWaterMark: 64 * 1024 // 64KB chunks
});

// Writable Stream
const writeStream = fs.createWriteStream('output.txt');

// Duplex Stream (both readable and writable)
const { Duplex } = require('stream');
const duplexStream = new Duplex({
    read(size) {
        // Implementation for reading
    },
    write(chunk, encoding, callback) {
        // Implementation for writing
        callback();
    }
});

// Transform Stream (transforms data while reading/writing)
const { Transform } = require('stream');
const transformStream = new Transform({
    transform(chunk, encoding, callback) {
        const upperChunk = chunk.toString().toUpperCase();
        callback(null, upperChunk);
    }
});
```

### 2. Stream Events
```javascript
// Readable stream events
readStream.on('data', (chunk) => {
    console.log('Received chunk:', chunk.length);
});

readStream.on('end', () => {
    console.log('No more data');
});

readStream.on('error', (error) => {
    console.error('Error:', error);
});

// Writable stream events
writeStream.on('finish', () => {
    console.log('Writing completed');
});

writeStream.on('error', (error) => {
    console.error('Error:', error);
});
```

### 3. Piping Streams
```javascript
// Basic piping
readStream.pipe(writeStream);

// Chaining pipes with transform
readStream
    .pipe(transformStream)
    .pipe(writeStream);

// Error handling with pipeline
const { pipeline } = require('stream/promises');

async function processFile() {
    try {
        await pipeline(
            readStream,
            transformStream,
            writeStream
        );
        console.log('Pipeline succeeded');
    } catch (error) {
        console.error('Pipeline failed:', error);
    }
}
```

## Advanced Stream Operations

### 1. Custom Streams
```javascript
const { Readable, Writable } = require('stream');

// Custom Readable Stream
class CustomReadable extends Readable {
    constructor(data) {
        super();
        this.data = data;
        this.index = 0;
    }

    _read(size) {
        if (this.index < this.data.length) {
            const chunk = this.data.slice(this.index, this.index + size);
            this.push(chunk);
            this.index += size;
        } else {
            this.push(null); // End of stream
        }
    }
}

// Custom Writable Stream
class CustomWritable extends Writable {
    constructor(options) {
        super(options);
        this.data = [];
    }

    _write(chunk, encoding, callback) {
        // Process the chunk
        this.data.push(chunk);
        callback();
    }
}

// Usage
const customReadable = new CustomReadable('Hello, World!');
const customWritable = new CustomWritable();

customReadable.pipe(customWritable);
```

### 2. Stream Utilities
```javascript
const { finished } = require('stream/promises');
const { compose } = require('stream');

// Using finished utility
async function streamComplete(stream) {
    try {
        await finished(stream);
        console.log('Stream completed');
    } catch (error) {
        console.error('Stream failed:', error);
    }
}

// Composing transforms
const uppercase = new Transform({
    transform(chunk, encoding, callback) {
        callback(null, chunk.toString().toUpperCase());
    }
});

const reverse = new Transform({
    transform(chunk, encoding, callback) {
        callback(null, chunk.toString().split('').reverse().join(''));
    }
});

const composed = compose(uppercase, reverse);
```

### 3. Memory Efficiency
```javascript
// Processing large files efficiently
async function processLargeFile(inputPath, outputPath) {
    const readStream = fs.createReadStream(inputPath, {
        highWaterMark: 64 * 1024 // 64KB chunks
    });

    const writeStream = fs.createWriteStream(outputPath);

    const transformStream = new Transform({
        transform(chunk, encoding, callback) {
            // Process chunk without loading entire file into memory
            const processed = processChunk(chunk);
            callback(null, processed);
        }
    });

    try {
        await pipeline(
            readStream,
            transformStream,
            writeStream
        );
        console.log('File processed successfully');
    } catch (error) {
        console.error('Error processing file:', error);
    }
}
```

## Best Practices

### 1. Memory Management
```javascript
// Handle backpressure
writeStream.on('drain', () => {
    // Resume sending data
    readStream.resume();
});

// Monitor memory usage
const used = process.memoryUsage();
console.log(`Memory usage: ${Math.round(used.heapUsed / 1024 / 1024)}MB`);
```

### 2. Error Handling
```javascript
function handleStreamError(stream, errorCallback) {
    stream.on('error', (error) => {
        console.error('Stream error:', error);
        if (errorCallback) errorCallback(error);
    });
}

// Clean up resources
function cleanupStream(stream) {
    if (stream.destroy) stream.destroy();
}
```

### 3. Performance Optimization
```javascript
// Optimize chunk size
const readStream = fs.createReadStream('file.txt', {
    highWaterMark: 256 * 1024 // 256KB chunks
});

// Use cork/uncork for batching writes
writeStream.cork();
writeStream.write('Some data');
writeStream.write('More data');
process.nextTick(() => writeStream.uncork());
```

## Related Topics
- [[File-System-Operations]] - File system streams
- [[Performance-Optimization]] - Stream performance
- [[Error-Handling-Advanced]] - Error management
- [[Memory-Management]] - Memory considerations

Tags: #nodejs #streams #buffers #performance #memory
