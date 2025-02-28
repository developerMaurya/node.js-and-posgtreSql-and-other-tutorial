# Express.js Middleware Guide

Middleware functions are the backbone of Express.js applications. This guide covers everything you need to know about creating and using middleware.

## Understanding Middleware

### Basic Middleware Structure
```javascript
const express = require('express');
const app = express();

// Basic middleware template
const basicMiddleware = (req, res, next) => {
    // 1. Modify request or response objects
    req.customData = 'some data';
    
    // 2. Execute any code
    console.log('Middleware executed');
    
    // 3. Call next middleware
    next();
};

// Using middleware
app.use(basicMiddleware);
```

## Built-in Middleware

### 1. Static Files
```javascript
// Serve static files from 'public' directory
app.use(express.static('public'));

// Multiple static directories
app.use(express.static('public'));
app.use(express.static('uploads'));

// Virtual path prefix
app.use('/static', express.static('public'));
```

### 2. Request Parsing
```javascript
// Parse JSON payloads
app.use(express.json());

// Parse URL-encoded bodies
app.use(express.urlencoded({ extended: true }));

// Parse raw bodies
app.use(express.raw());

// Parse text bodies
app.use(express.text());
```

## Custom Middleware Examples

### 1. Request Logger
```javascript
const requestLogger = (req, res, next) => {
    const start = Date.now();
    
    // Log when response finishes
    res.on('finish', () => {
        const duration = Date.now() - start;
        console.log(`${req.method} ${req.url} - ${res.statusCode} - ${duration}ms`);
    });
    
    next();
};

app.use(requestLogger);
```

### 2. Authentication Middleware
```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
    try {
        const token = req.header('Authorization')?.replace('Bearer ', '');
        
        if (!token) {
            throw new Error('No token provided');
        }
        
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({
            error: 'Please authenticate'
        });
    }
};

// Protected route example
app.get('/protected', authMiddleware, (req, res) => {
    res.json({ user: req.user });
});
```

### 3. Error Handler
```javascript
const errorHandler = (err, req, res, next) => {
    // Log error
    console.error(err.stack);
    
    // Default error response
    const statusCode = err.statusCode || 500;
    const message = err.message || 'Internal Server Error';
    
    res.status(statusCode).json({
        error: {
            message,
            ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
        }
    });
};

// Use at the end of middleware chain
app.use(errorHandler);
```

### 4. Request Validator
```javascript
const validateUser = (req, res, next) => {
    const { email, password } = req.body;
    
    const errors = [];
    
    if (!email) {
        errors.push('Email is required');
    } else if (!/\S+@\S+\.\S+/.test(email)) {
        errors.push('Email is invalid');
    }
    
    if (!password) {
        errors.push('Password is required');
    } else if (password.length < 6) {
        errors.push('Password must be at least 6 characters');
    }
    
    if (errors.length > 0) {
        return res.status(400).json({ errors });
    }
    
    next();
};

// Use in route
app.post('/users', validateUser, createUser);
```

## Route-Specific Middleware

### 1. Route Middleware
```javascript
const router = express.Router();

// Middleware specific to this router
router.use((req, res, next) => {
    console.log('Router Specific Middleware');
    next();
});

// Apply middleware to specific route
router.get('/users/:id', checkUserExists, getUser);

// Chain multiple middleware
router.post('/posts',
    checkAuth,
    validatePost,
    uploadImage,
    createPost
);
```

### 2. Conditional Middleware
```javascript
const checkRole = (role) => {
    return (req, res, next) => {
        if (req.user && req.user.role === role) {
            next();
        } else {
            res.status(403).json({
                error: 'Access denied'
            });
        }
    };
};

// Use in routes
app.get('/admin', checkRole('admin'), adminDashboard);
app.get('/manager', checkRole('manager'), managerDashboard);
```

## Advanced Middleware Patterns

### 1. Middleware Factory
```javascript
const rateLimit = (options = {}) => {
    const {
        windowMs = 60 * 1000, // 1 minute
        max = 100, // max requests per window
        message = 'Too many requests'
    } = options;
    
    const requests = new Map();
    
    return (req, res, next) => {
        const key = req.ip;
        const now = Date.now();
        
        // Clean old entries
        if (requests.has(key)) {
            const { count, start } = requests.get(key);
            if (now - start > windowMs) {
                requests.delete(key);
            }
        }
        
        // Check rate limit
        if (requests.has(key)) {
            const { count, start } = requests.get(key);
            if (count >= max) {
                return res.status(429).json({ error: message });
            }
            requests.set(key, { count: count + 1, start });
        } else {
            requests.set(key, { count: 1, start: now });
        }
        
        next();
    };
};

// Use rate limiter
app.use('/api', rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100
}));
```

### 2. Async Middleware Wrapper
```javascript
const asyncHandler = (fn) => {
    return (req, res, next) => {
        Promise.resolve(fn(req, res, next)).catch(next);
    };
};

// Use with async route handlers
app.get('/users', asyncHandler(async (req, res) => {
    const users = await User.find();
    res.json(users);
}));
```

## Best Practices

1. Keep middleware functions focused and single-purpose
2. Use early returns to avoid deep nesting
3. Always call next() or send a response
4. Handle errors properly
5. Use middleware composition for complex logic
6. Order middleware properly (more generic first)

## Related Topics
- [[Express-Framework]] - Express.js basics
- [[Express-Routing]] - Routing in Express.js
- [[Express-Security]] - Security middleware
- [[API-Authentication]] - Authentication strategies

Tags: #nodejs #express #middleware #backend #web-development
