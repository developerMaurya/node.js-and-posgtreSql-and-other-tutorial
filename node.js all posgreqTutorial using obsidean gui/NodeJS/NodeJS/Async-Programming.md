# Asynchronous Programming in Node.js

Understanding asynchronous programming is crucial for Node.js development.

## Core Concepts

### 1. Callbacks
The traditional way of handling async operations:
```javascript
fs.readFile('file.txt', (err, data) => {
    if (err) throw err;
    console.log(data);
});
```

### 2. Promises
Modern way to handle async operations:
```javascript
const readFilePromise = new Promise((resolve, reject) => {
    fs.readFile('file.txt', (err, data) => {
        if (err) reject(err);
        resolve(data);
    });
});

readFilePromise
    .then(data => console.log(data))
    .catch(err => console.error(err));
```

### 3. Async/Await
The most recent and readable way:
```javascript
async function readFile() {
    try {
        const data = await fs.promises.readFile('file.txt');
        console.log(data);
    } catch (err) {
        console.error(err);
    }
}
```

## Common Async Operations
- File operations
- Database queries
- HTTP requests
- Timer functions

## Error Handling
```javascript
// With Promises
promise
    .then(success)
    .catch(error)
    .finally(cleanup);

// With Async/Await
try {
    await asyncOperation();
} catch (error) {
    handleError(error);
} finally {
    cleanup();
}
```

## Best Practices
1. Avoid callback hell
2. Always handle errors
3. Use try/catch with async/await
4. Consider Promise.all() for parallel operations

[[Async-Programming-Deep-Dive]]

## Related Topics
- [[Event-Loop]] - How async operations work
- [[Error-Handling-Advanced]] - Managing errors
- [[Performance]] - Optimizing async operations

Tags: #nodejs #async #promises #javascript
