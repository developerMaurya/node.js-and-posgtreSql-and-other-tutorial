# JavaScript Fundamentals for Node.js

A strong foundation in JavaScript is essential for Node.js development. This guide covers the key JavaScript concepts you need to know.

## Variables and Data Types

### 1. Variable Declarations
```javascript
// var (function-scoped) - avoid using
var x = 1;

// let (block-scoped) - preferred for mutable variables
let y = 2;

// const (block-scoped) - preferred for immutable references
const z = 3;
```

### 2. Data Types
```javascript
// Primitive Types
const string = 'Hello';
const number = 42;
const boolean = true;
const nullValue = null;
const undefinedValue = undefined;
const symbol = Symbol('description');
const bigInt = 9007199254740991n;

// Reference Types
const object = { key: 'value' };
const array = [1, 2, 3];
const date = new Date();
const regex = /pattern/;
```

## Functions

### 1. Function Declarations
```javascript
// Regular function
function add(a, b) {
    return a + b;
}

// Function expression
const subtract = function(a, b) {
    return a - b;
};

// Arrow function
const multiply = (a, b) => a * b;

// IIFE (Immediately Invoked Function Expression)
(function() {
    console.log('Executed immediately');
})();
```

### 2. Advanced Function Concepts
```javascript
// Default parameters
function greet(name = 'Guest') {
    return `Hello, ${name}!`;
}

// Rest parameters
function sum(...numbers) {
    return numbers.reduce((total, num) => total + num, 0);
}

// Closure
function counter() {
    let count = 0;
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count
    };
}
```

## Objects and Prototypes

### 1. Object Creation and Manipulation
```javascript
// Object literal
const person = {
    name: 'John',
    age: 30,
    greet() {
        return `Hello, I'm ${this.name}`;
    }
};

// Constructor function
function Person(name, age) {
    this.name = name;
    this.age = age;
}

Person.prototype.greet = function() {
    return `Hello, I'm ${this.name}`;
};

// Class syntax (ES6+)
class User {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }

    greet() {
        return `Hello, I'm ${this.name}`;
    }
}
```

### 2. Object Methods and Properties
```javascript
const obj = { a: 1, b: 2 };

// Object operations
Object.keys(obj);                // ['a', 'b']
Object.values(obj);              // [1, 2]
Object.entries(obj);             // [['a', 1], ['b', 2]]
Object.freeze(obj);              // Make object immutable
Object.assign({}, obj, { c: 3 }); // Merge objects
```

## ES6+ Features

### 1. Destructuring
```javascript
// Array destructuring
const [first, second, ...rest] = [1, 2, 3, 4, 5];

// Object destructuring
const { name, age, address: { city } = {} } = person;

// Parameter destructuring
function processUser({ name, age }) {
    console.log(name, age);
}
```

### 2. Template Literals
```javascript
const name = 'John';
const greeting = `Hello, ${name}!
This is a multiline
string.`;
```

### 3. Spread Operator
```javascript
// Array spread
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];

// Object spread
const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 };
```

## Promises and Async Programming

### 1. Promises
```javascript
function fetchData() {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            const success = true;
            if (success) {
                resolve('Data fetched');
            } else {
                reject(new Error('Failed to fetch data'));
            }
        }, 1000);
    });
}

// Promise chaining
fetchData()
    .then(data => processData(data))
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

### 2. Async/Await
```javascript
async function getData() {
    try {
        const data = await fetchData();
        const processed = await processData(data);
        return processed;
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}
```

## Error Handling

### 1. Try-Catch
```javascript
try {
    // Code that might throw an error
    throw new Error('Something went wrong');
} catch (error) {
    console.error('Error caught:', error.message);
} finally {
    // Always executed
    console.log('Cleanup code');
}
```

### 2. Custom Errors
```javascript
class ValidationError extends Error {
    constructor(message) {
        super(message);
        this.name = 'ValidationError';
    }
}

try {
    throw new ValidationError('Invalid input');
} catch (error) {
    if (error instanceof ValidationError) {
        console.log('Validation failed:', error.message);
    } else {
        console.log('Unknown error:', error);
    }
}
```

## Related Topics
- [[Async-Programming]] - Deep dive into asynchronous JavaScript
- [[Error-Handling-Advanced]] - Advanced error handling patterns
- [[ES6-Features]] - Modern JavaScript features
- [[JavaScript-Best-Practices]] - Coding standards and patterns

Tags: #javascript #fundamentals #programming #es6 #async
