# REST API Design in Node.js

A comprehensive guide to designing and implementing RESTful APIs using Node.js and Express.

## API Design Principles

### 1. RESTful Endpoints
```javascript
// Example Express Router setup with RESTful endpoints
const express = require('express');
const router = express.Router();

// GET /api/users - List all users
router.get('/users', listUsers);

// GET /api/users/:id - Get single user
router.get('/users/:id', getUser);

// POST /api/users - Create user
router.post('/users', createUser);

// PUT /api/users/:id - Update user
router.put('/users/:id', updateUser);

// DELETE /api/users/:id - Delete user
router.delete('/users/:id', deleteUser);
```

### 2. Resource Naming Conventions
```javascript
// Good Examples
router.get('/articles');                    // List articles
router.get('/articles/:id');                // Get article
router.get('/articles/:id/comments');       // List article comments
router.post('/articles/:id/comments');      // Create comment
router.get('/users/:userId/articles');      // List user's articles

// Bad Examples (avoid)
router.get('/getArticles');                 // Don't use verbs
router.get('/articles/getComments/:id');    // Don't mix verbs and nouns
router.get('/articles_list');               // Don't use underscores
```

## Request Handling

### 1. Request Validation
```javascript
const { body, param, query, validationResult } = require('express-validator');

// Validation middleware
const validateUser = [
    body('email').isEmail().normalizeEmail(),
    body('password').isLength({ min: 6 }),
    body('name').trim().notEmpty(),
    
    // Custom validation
    (req, res, next) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        next();
    }
];

// Usage in route
router.post('/users', validateUser, async (req, res) => {
    try {
        const user = await createUser(req.body);
        res.status(201).json(user);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

### 2. Query Parameters
```javascript
// GET /api/articles?page=1&limit=10&sort=date
router.get('/articles', async (req, res) => {
    try {
        const {
            page = 1,
            limit = 10,
            sort = 'createdAt',
            order = 'desc',
            search
        } = req.query;

        const query = {};
        if (search) {
            query.title = new RegExp(search, 'i');
        }

        const articles = await Article
            .find(query)
            .sort({ [sort]: order })
            .skip((page - 1) * limit)
            .limit(Number(limit));

        const total = await Article.countDocuments(query);

        res.json({
            articles,
            pagination: {
                page: Number(page),
                limit: Number(limit),
                total,
                pages: Math.ceil(total / limit)
            }
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

## Response Handling

### 1. Standard Response Format
```javascript
// Response helper
class ApiResponse {
    static success(res, data, message = 'Success', code = 200) {
        return res.status(code).json({
            success: true,
            message,
            data
        });
    }

    static error(res, message = 'Error', code = 500, errors = null) {
        return res.status(code).json({
            success: false,
            message,
            errors
        });
    }
}

// Usage
router.get('/users/:id', async (req, res) => {
    try {
        const user = await User.findById(req.params.id);
        if (!user) {
            return ApiResponse.error(res, 'User not found', 404);
        }
        return ApiResponse.success(res, user);
    } catch (error) {
        return ApiResponse.error(res, error.message);
    }
});
```

### 2. HTTP Status Codes
```javascript
const HttpStatus = {
    OK: 200,
    CREATED: 201,
    NO_CONTENT: 204,
    BAD_REQUEST: 400,
    UNAUTHORIZED: 401,
    FORBIDDEN: 403,
    NOT_FOUND: 404,
    CONFLICT: 409,
    INTERNAL_SERVER: 500
};

// Example usage
router.post('/articles', async (req, res) => {
    try {
        const article = await Article.create(req.body);
        return ApiResponse.success(res, article, 'Article created', HttpStatus.CREATED);
    } catch (error) {
        if (error.code === 11000) { // Duplicate key error
            return ApiResponse.error(res, 'Article already exists', HttpStatus.CONFLICT);
        }
        return ApiResponse.error(res, error.message, HttpStatus.INTERNAL_SERVER);
    }
});
```

## Middleware Implementation

### 1. Authentication Middleware
```javascript
const jwt = require('jsonwebtoken');

const auth = async (req, res, next) => {
    try {
        const token = req.header('Authorization').replace('Bearer ', '');
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        
        const user = await User.findOne({
            _id: decoded.id,
            'tokens.token': token
        });

        if (!user) {
            throw new Error();
        }

        req.token = token;
        req.user = user;
        next();
    } catch (error) {
        res.status(401).json({
            error: 'Please authenticate'
        });
    }
};

// Usage
router.get('/profile', auth, async (req, res) => {
    res.json(req.user);
});
```

### 2. Error Handling Middleware
```javascript
// Custom error class
class ApiError extends Error {
    constructor(message, statusCode) {
        super(message);
        this.statusCode = statusCode;
        this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
        this.isOperational = true;

        Error.captureStackTrace(this, this.constructor);
    }
}

// Error handling middleware
const errorHandler = (err, req, res, next) => {
    err.statusCode = err.statusCode || 500;
    err.status = err.status || 'error';

    if (process.env.NODE_ENV === 'development') {
        res.status(err.statusCode).json({
            status: err.status,
            error: err,
            message: err.message,
            stack: err.stack
        });
    } else {
        // Production mode
        if (err.isOperational) {
            res.status(err.statusCode).json({
                status: err.status,
                message: err.message
            });
        } else {
            // Programming or unknown errors
            console.error('ERROR 💥', err);
            res.status(500).json({
                status: 'error',
                message: 'Something went wrong!'
            });
        }
    }
};

// Usage
app.use(errorHandler);
```

## API Documentation

### 1. Swagger/OpenAPI Integration
```javascript
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const options = {
    definition: {
        openapi: '3.0.0',
        info: {
            title: 'My API',
            version: '1.0.0',
            description: 'API documentation'
        },
        servers: [
            {
                url: 'http://localhost:3000/api'
            }
        ]
    },
    apis: ['./routes/*.js']
};

const specs = swaggerJsdoc(options);
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(specs));

// Route documentation example
/**
 * @swagger
 * /users:
 *   get:
 *     summary: Returns list of users
 *     responses:
 *       200:
 *         description: List of users
 */
router.get('/users', listUsers);
```

## API Versioning

### 1. URL Versioning
```javascript
// api/v1/users
const v1Router = express.Router();
app.use('/api/v1', v1Router);

// api/v2/users
const v2Router = express.Router();
app.use('/api/v2', v2Router);

// Version-specific implementations
v1Router.get('/users', listUsersV1);
v2Router.get('/users', listUsersV2);
```

### 2. Header Versioning
```javascript
const versionCheck = (req, res, next) => {
    const version = req.headers['accept-version'];
    req.apiVersion = version || '1.0.0';
    next();
};

router.get('/users', versionCheck, (req, res) => {
    switch (req.apiVersion) {
        case '2.0.0':
            return listUsersV2(req, res);
        default:
            return listUsersV1(req, res);
    }
});
```

## Security Best Practices

### 1. Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: 'Too many requests from this IP, please try again later'
});

// Apply to all routes
app.use(limiter);

// Apply to specific routes
app.use('/api/', limiter);
```

### 2. Security Headers
```javascript
const helmet = require('helmet');

// Basic security headers
app.use(helmet());

// Custom security headers
app.use((req, res, next) => {
    res.setHeader('Content-Security-Policy', "default-src 'self'");
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('X-XSS-Protection', '1; mode=block');
    next();
});
```

## Testing

### 1. API Testing with Jest and Supertest
```javascript
const request = require('supertest');
const app = require('../app');

describe('User API', () => {
    it('should create a new user', async () => {
        const res = await request(app)
            .post('/api/users')
            .send({
                name: 'Test User',
                email: 'test@example.com',
                password: 'password123'
            });

        expect(res.statusCode).toBe(201);
        expect(res.body).toHaveProperty('id');
        expect(res.body.name).toBe('Test User');
    });

    it('should get user profile', async () => {
        const token = 'valid-token'; // Setup test token
        
        const res = await request(app)
            .get('/api/users/profile')
            .set('Authorization', `Bearer ${token}`);

        expect(res.statusCode).toBe(200);
        expect(res.body).toHaveProperty('email');
    });
});
```

## Related Topics
- [[API-Authentication]] - Authentication strategies
- [[API-Security]] - Security best practices
- [[Express-Framework]] - Express.js basics
- [[Testing]] - API testing strategies

Tags: #nodejs #api #rest #express #backend
