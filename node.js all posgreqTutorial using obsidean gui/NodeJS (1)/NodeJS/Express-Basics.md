# Express.js Basics

A comprehensive guide to getting started with Express.js, a fast, unopinionated web framework for Node.js.

## Installation and Setup

### Basic Setup
```javascript
// Initialize a new project
// $ npm init -y
// $ npm install express

// Basic Express Application
const express = require('express');
const app = express();
const port = 3000;

// Basic middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Start server
app.listen(port, () => {
    console.log(`Server running at http://localhost:${port}`);
});
```

## Basic Routing

### Route Methods
```javascript
// GET request
app.get('/', (req, res) => {
    res.send('Hello World!');
});

// POST request
app.post('/api/users', (req, res) => {
    const user = req.body;
    // Process user data
    res.json({ message: 'User created', user });
});

// PUT request
app.put('/api/users/:id', (req, res) => {
    const { id } = req.params;
    const userData = req.body;
    res.json({ message: `User ${id} updated`, data: userData });
});

// DELETE request
app.delete('/api/users/:id', (req, res) => {
    const { id } = req.params;
    res.json({ message: `User ${id} deleted` });
});
```

### Route Parameters
```javascript
// URL parameters
app.get('/users/:id', (req, res) => {
    const userId = req.params.id;
    res.json({ userId });
});

// Multiple parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
    const { userId, postId } = req.params;
    res.json({ userId, postId });
});

// Query strings
app.get('/search', (req, res) => {
    const { q, page = 1 } = req.query;
    res.json({ query: q, page });
});
```

## Middleware

### Custom Middleware
```javascript
// Logger middleware
const logger = (req, res, next) => {
    console.log(`${req.method} ${req.url} at ${new Date()}`);
    next();
};

// Error handling middleware
const errorHandler = (err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ error: 'Something went wrong!' });
};

// Using middleware
app.use(logger);
app.use(errorHandler);

// Route-specific middleware
const authenticate = (req, res, next) => {
    const authToken = req.headers.authorization;
    if (!authToken) {
        return res.status(401).json({ error: 'Authentication required' });
    }
    next();
};

app.get('/protected', authenticate, (req, res) => {
    res.json({ message: 'Protected route accessed' });
});
```

## Request and Response

### Request Object
```javascript
app.post('/api/data', (req, res) => {
    // Request body
    console.log(req.body);
    
    // Request headers
    console.log(req.headers);
    
    // Request query parameters
    console.log(req.query);
    
    // Request parameters
    console.log(req.params);
    
    // Request cookies
    console.log(req.cookies);
    
    res.json({ received: true });
});
```

### Response Object
```javascript
app.get('/api/responses', (req, res) => {
    // Send JSON
    res.json({ message: 'JSON response' });
    
    // Send plain text
    res.send('Plain text response');
    
    // Send status
    res.sendStatus(200);
    
    // Set status and send
    res.status(201).json({ created: true });
    
    // Redirect
    res.redirect('/new-location');
    
    // Send file
    res.sendFile('/path/to/file.pdf');
});
```

## Static Files

### Serving Static Files
```javascript
// Serve static files from 'public' directory
app.use(express.static('public'));

// Serve static files with virtual path prefix
app.use('/static', express.static('public'));

// Multiple static directories
app.use(express.static('public'));
app.use(express.static('files'));
```

## Template Engines

### EJS Setup
```javascript
// $ npm install ejs

// Set EJS as template engine
app.set('view engine', 'ejs');
app.set('views', './views');

// Render template
app.get('/', (req, res) => {
    res.render('index', {
        title: 'Home Page',
        message: 'Welcome to Express'
    });
});
```

## Error Handling

### Basic Error Handling
```javascript
// Synchronous error handling
app.get('/error', (req, res) => {
    throw new Error('Something went wrong');
});

// Asynchronous error handling
app.get('/async-error', async (req, res, next) => {
    try {
        // Async operation that might fail
        await someAsyncOperation();
    } catch (error) {
        next(error);
    }
});

// Error handling middleware
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({
        error: {
            message: err.message,
            status: 500
        }
    });
});
```

## Basic Security

### Security Middleware
```javascript
const helmet = require('helmet');
const cors = require('cors');

// Add basic security headers
app.use(helmet());

// Enable CORS
app.use(cors());

// Rate limiting
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // limit each IP to 100 requests per windowMs
});
app.use(limiter);
```

## Related Topics
- [[Express-Routing]] - Advanced routing concepts
- [[Middleware-Deep-Dive]] - Detailed middleware usage
- [[Express-Security]] - Comprehensive security measures
- [[Express-Performance]] - Performance optimization

## Practice Projects
1. Build a basic REST API
2. Create a file upload server
3. Implement user authentication
4. Build a blog with templates

## Resources
- [Express.js Documentation](https://expressjs.com/)
- [Express.js Guide](https://expressjs.com/en/guide/routing.html)
- [[Learning-Resources#Express|Express.js Learning Resources]]

## Tags
#express #nodejs #web-framework #routing #middleware
