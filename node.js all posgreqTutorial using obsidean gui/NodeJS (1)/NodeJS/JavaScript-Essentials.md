# JavaScript Essentials for Node.js

Core JavaScript concepts and features essential for Node.js development.

## Callbacks & Callback Hell

### Basic Callbacks
```javascript
// Simple callback example
function fetchData(callback) {
    setTimeout(() => {
        const data = { id: 1, name: 'Example' };
        callback(null, data);
    }, 1000);
}

fetchData((error, data) => {
    if (error) {
        console.error('Error:', error);
        return;
    }
    console.log('Data:', data);
});
```

### Callback Hell Problem
```javascript
// Callback Hell Example
getUserData(userId, (error, user) => {
    if (error) {
        handleError(error);
        return;
    }
    
    getOrders(user.id, (error, orders) => {
        if (error) {
            handleError(error);
            return;
        }
        
        getOrderDetails(orders[0].id, (error, details) => {
            if (error) {
                handleError(error);
                return;
            }
            
            processOrder(details, (error, result) => {
                if (error) {
                    handleError(error);
                    return;
                }
                console.log('Result:', result);
            });
        });
    });
});
```

## Promises

### Creating and Using Promises
```javascript
// Creating a Promise
function fetchUserData(userId) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            const user = { id: userId, name: 'John Doe' };
            if (user) {
                resolve(user);
            } else {
                reject(new Error('User not found'));
            }
        }, 1000);
    });
}

// Using Promises
fetchUserData(123)
    .then(user => {
        console.log('User:', user);
        return getUserOrders(user.id);
    })
    .then(orders => {
        console.log('Orders:', orders);
    })
    .catch(error => {
        console.error('Error:', error);
    })
    .finally(() => {
        console.log('Operation completed');
    });

// Promise.all
Promise.all([
    fetchUserData(1),
    fetchUserData(2),
    fetchUserData(3)
])
    .then(users => {
        console.log('All users:', users);
    })
    .catch(error => {
        console.error('Error fetching users:', error);
    });

// Promise.race
Promise.race([
    fetchUserData(1),
    fetchUserData(2)
])
    .then(firstUser => {
        console.log('First user to resolve:', firstUser);
    })
    .catch(error => {
        console.error('Error:', error);
    });
```

## Async/Await

### Basic Usage
```javascript
async function getUserDetails(userId) {
    try {
        const user = await fetchUserData(userId);
        const orders = await getUserOrders(user.id);
        const details = await getOrderDetails(orders[0].id);
        return details;
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}

// Using async function
getUserDetails(123)
    .then(details => {
        console.log('Details:', details);
    })
    .catch(error => {
        console.error('Error:', error);
    });
```

### Parallel Operations
```javascript
async function getMultipleUsers() {
    try {
        const userIds = [1, 2, 3];
        const userPromises = userIds.map(id => fetchUserData(id));
        const users = await Promise.all(userPromises);
        return users;
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}

// Error handling with async/await
async function handleAsyncOperations() {
    try {
        const [users, orders, products] = await Promise.all([
            getUsers(),
            getOrders(),
            getProducts()
        ]);
        
        return { users, orders, products };
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}
```

## Error Handling

### Try-Catch Blocks
```javascript
// Basic error handling
try {
    const result = JSON.parse('invalid json');
} catch (error) {
    console.error('Parsing error:', error);
}

// Async error handling
async function handleAsyncErrors() {
    try {
        const data = await fetchData();
        return processData(data);
    } catch (error) {
        if (error instanceof NetworkError) {
            console.error('Network error:', error);
        } else if (error instanceof ValidationError) {
            console.error('Validation error:', error);
        } else {
            console.error('Unknown error:', error);
        }
        throw error;
    }
}
```

### Custom Error Classes
```javascript
class ValidationError extends Error {
    constructor(message) {
        super(message);
        this.name = 'ValidationError';
    }
}

class DatabaseError extends Error {
    constructor(message) {
        super(message);
        this.name = 'DatabaseError';
    }
}

// Using custom errors
function validateUser(user) {
    if (!user.name) {
        throw new ValidationError('Name is required');
    }
    if (!user.email) {
        throw new ValidationError('Email is required');
    }
}
```

## ES6+ Features

### Destructuring
```javascript
// Object destructuring
const user = { name: 'John', age: 30, email: 'john@example.com' };
const { name, age, email } = user;

// Array destructuring
const numbers = [1, 2, 3, 4, 5];
const [first, second, ...rest] = numbers;

// Nested destructuring
const data = {
    user: {
        details: {
            firstName: 'John',
            lastName: 'Doe'
        }
    }
};
const { user: { details: { firstName, lastName } } } = data;
```

### Spread and Rest Operators
```javascript
// Spread operator
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2];

// Object spread
const defaultConfig = { port: 3000, host: 'localhost' };
const userConfig = { port: 8080 };
const config = { ...defaultConfig, ...userConfig };

// Rest parameters
function sum(...numbers) {
    return numbers.reduce((total, num) => total + num, 0);
}
```

### Template Literals
```javascript
const name = 'John';
const age = 30;

// Basic template literal
const greeting = `Hello, ${name}! You are ${age} years old.`;

// Multi-line strings
const html = `
    <div>
        <h1>Welcome, ${name}</h1>
        <p>Age: ${age}</p>
    </div>
`;

// Tagged templates
function highlight(strings, ...values) {
    return strings.reduce((result, str, i) => 
        `${result}${str}${values[i] ? `<strong>${values[i]}</strong>` : ''}`, '');
}

const highlighted = highlight`Hello, ${name}! You are ${age} years old.`;
```

## Arrow Functions

### Basic Syntax
```javascript
// Regular function
function add(a, b) {
    return a + b;
}

// Arrow function
const addArrow = (a, b) => a + b;

// Arrow function with block
const calculate = (a, b) => {
    const result = a * b;
    return result;
};

// Arrow function in callbacks
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(num => num * 2);
const filtered = numbers.filter(num => num > 2);
```

## Related Topics
- [[Async-Programming]] - Advanced asynchronous patterns
- [[Error-Handling-Advanced]] - Comprehensive error handling
- [[ES6-Features]] - Modern JavaScript features
- [[Functional-Programming]] - Functional programming concepts

## Practice Projects
1. Build an async data fetcher
2. Create a promise-based API wrapper
3. Implement error handling system
4. Build a functional programming utility library

## Resources
- [MDN JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [JavaScript.info](https://javascript.info/)
- [[Learning-Resources#JavaScript|JavaScript Learning Resources]]

## Tags
#javascript #async #promises #es6 #error-handling
