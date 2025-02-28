# Node.js Event Loop

The event loop is the heart of Node.js's non-blocking I/O operations despite JavaScript being single-threaded.

## Event Loop Phases
1. **Timers** - Execute `setTimeout` and `setInterval` callbacks
2. **Pending callbacks** - Execute I/O callbacks deferred to the next loop iteration
3. **Idle, prepare** - Internal use only
4. **Poll** - Retrieve new I/O events
5. **Check** - Execute `setImmediate` callbacks
6. **Close callbacks** - Execute close event callbacks

## Example
```javascript
console.log('1');

setTimeout(() => {
  console.log('2');
}, 0);

Promise.resolve().then(() => {
  console.log('3');
});

console.log('4');

// Output:
// 1
// 4
// 3
// 2
```

## Related Topics
- [[NodeJS-Fundamentals]] - Basic Node.js concepts
- [[Async-Programming]] - How to work with asynchronous code
- [[Event-Emitters]] - Node.js event system

## Common Pitfalls
- Blocking the event loop with CPU-intensive tasks
- Not understanding the execution order of different types of callbacks
- Mishandling error events

Tags: #nodejs #event-loop #async #javascript
