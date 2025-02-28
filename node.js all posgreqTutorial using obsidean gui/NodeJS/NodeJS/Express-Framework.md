# Express.js Framework

Express is a fast, unopinionated web framework for Node.js.

## Core Concepts

### 1. Basic Setup
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Hello World!');
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});
```

### 2. Middleware
Functions that have access to request and response objects:
```javascript
// Custom middleware
app.use((req, res, next) => {
    console.log(`${req.method} ${req.url}`);
    next();
});

// Built-in middleware
app.use(express.json()); // Parse JSON bodies
app.use(express.static('public')); // Serve static files
```

### 3. Routing
```javascript
// Basic routing
app.get('/users', getUsers);
app.post('/users', createUser);

// Router module
const router = express.Router();
router.get('/', getUsers);
router.post('/', createUser);
app.use('/users', router);
```

### 4. Request Handling
```javascript
app.post('/api/users', (req, res) => {
    const { name, email } = req.body;
    const queryParam = req.query.filter;
    const urlParam = req.params.id;
    
    res.status(201).json({
        message: 'User created',
        user: { name, email }
    });
});
```

## Best Practices
1. Use appropriate HTTP methods
2. Implement proper error handling
3. Structure routes logically
4. Use environment variables
5. Implement input validation

## Common Middleware
- `body-parser` - Parse request bodies
- `cors` - Handle CORS
- `helmet` - Security headers
- `morgan` - HTTP request logger

## Related Topics
- [[REST-API]] - RESTful service design
- [[Authentication]] - User authentication
- [[Database-Integration]] - Working with databases
- [[Security]] - Security best practices

Tags: #nodejs #express #web-framework #backend
