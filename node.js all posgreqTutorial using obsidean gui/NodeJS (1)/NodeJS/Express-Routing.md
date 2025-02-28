# Express.js Routing Guide

A comprehensive guide to routing in Express.js applications, covering basic to advanced routing concepts.

## Basic Routing

### 1. Route Methods
```javascript
const express = require('express');
const app = express();

// Basic route methods
app.get('/', (req, res) => {
    res.send('Hello World!');
});

app.post('/submit', (req, res) => {
    res.json({ message: 'Data received' });
});

app.put('/update/:id', (req, res) => {
    res.json({ message: `Updated ${req.params.id}` });
});

app.delete('/delete/:id', (req, res) => {
    res.json({ message: `Deleted ${req.params.id}` });
});

// Handle all HTTP methods
app.all('/api/*', (req, res, next) => {
    console.log('Logging for all API routes');
    next();
});
```

### 2. Route Parameters
```javascript
// Named parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
    const { userId, postId } = req.params;
    res.json({ userId, postId });
});

// Optional parameters
app.get('/users/:userId?', (req, res) => {
    if (req.params.userId) {
        res.json({ user: `User ${req.params.userId}` });
    } else {
        res.json({ users: 'All users' });
    }
});

// Regular expression in routes
app.get(/.*fly$/, (req, res) => {
    res.send('URL ends with "fly"');
});
```

## Express Router

### 1. Basic Router Setup
```javascript
const express = require('express');
const router = express.Router();

// Router middleware
router.use((req, res, next) => {
    console.log('Time:', Date.now());
    next();
});

// Router routes
router.get('/', (req, res) => {
    res.send('Users home page');
});

router.get('/:id', (req, res) => {
    res.send(`User ${req.params.id}`);
});

// Mount router
app.use('/users', router);
```

### 2. Modular Routing
```javascript
// users.routes.js
const express = require('express');
const router = express.Router();

router.get('/', userController.getUsers);
router.post('/', userController.createUser);
router.get('/:id', userController.getUser);
router.put('/:id', userController.updateUser);
router.delete('/:id', userController.deleteUser);

module.exports = router;

// posts.routes.js
const express = require('express');
const router = express.Router();

router.get('/', postController.getPosts);
router.post('/', postController.createPost);
router.get('/:id', postController.getPost);
router.put('/:id', postController.updatePost);
router.delete('/:id', postController.deletePost);

module.exports = router;

// app.js
const usersRouter = require('./routes/users.routes');
const postsRouter = require('./routes/posts.routes');

app.use('/api/users', usersRouter);
app.use('/api/posts', postsRouter);
```

## Advanced Routing

### 1. Nested Routes
```javascript
// Create nested router
const userPostsRouter = express.Router({ mergeParams: true });

// Mount nested router
userRouter.use('/:userId/posts', userPostsRouter);

// Define nested routes
userPostsRouter.get('/', (req, res) => {
    const { userId } = req.params;
    res.json({ message: `All posts for user ${userId}` });
});

userPostsRouter.post('/', (req, res) => {
    const { userId } = req.params;
    res.json({ message: `Created post for user ${userId}` });
});
```

### 2. Route Handlers
```javascript
// Multiple callback functions
app.get('/example', 
    (req, res, next) => {
        console.log('First handler');
        next();
    },
    (req, res) => {
        res.send('Second handler');
    }
);

// Array of middleware
const validate = (req, res, next) => {
    if (!req.body.name) {
        return res.status(400).json({ error: 'Name required' });
    }
    next();
};

const log = (req, res, next) => {
    console.log('Request logged');
    next();
};

app.post('/example', [validate, log], (req, res) => {
    res.json({ message: 'Success' });
});
```

### 3. Route Chaining
```javascript
router.route('/users/:id')
    .all((req, res, next) => {
        // runs for all HTTP methods
        console.log('Processing user request');
        next();
    })
    .get((req, res) => {
        res.json({ message: 'Get user' });
    })
    .put((req, res) => {
        res.json({ message: 'Update user' });
    })
    .delete((req, res) => {
        res.json({ message: 'Delete user' });
    });
```

## Response Methods

### 1. Common Response Methods
```javascript
// Send various responses
app.get('/responses', (req, res) => {
    // Send string
    res.send('Hello World');
    
    // Send JSON
    res.json({ message: 'Hello World' });
    
    // Send file
    res.sendFile('/path/to/file.pdf');
    
    // Send status
    res.sendStatus(200);
    
    // Chain methods
    res.status(201).json({ message: 'Created' });
    
    // Redirect
    res.redirect('/new-page');
    
    // Render template
    res.render('index', { title: 'Home' });
});
```

### 2. Custom Response Headers
```javascript
app.get('/custom-headers', (req, res) => {
    res
        .set('Content-Type', 'application/json')
        .set('X-Custom-Header', 'Custom Value')
        .json({ message: 'Response with custom headers' });
});
```

## Error Handling

### 1. Route Error Handler
```javascript
// Async route handler
const asyncHandler = (fn) => (req, res, next) =>
    Promise.resolve(fn(req, res, next)).catch(next);

// Use with async routes
app.get('/users', asyncHandler(async (req, res) => {
    const users = await User.find();
    res.json(users);
}));

// 404 handler
app.use((req, res, next) => {
    res.status(404).json({
        error: 'Not Found'
    });
});

// Error handler
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({
        error: 'Something broke!'
    });
});
```

## Best Practices

1. Organize routes by resource
2. Use descriptive route names
3. Version your API routes
4. Handle all errors appropriately
5. Use proper HTTP methods
6. Keep routes simple and focused
7. Use middleware for common functionality
8. Document your routes

## Related Topics
- [[Express-Framework]] - Express.js basics
- [[Express-Middleware]] - Middleware in Express.js
- [[REST-API-Design]] - RESTful API design
- [[API-Documentation]] - API documentation

Tags: #nodejs #express #routing #backend #web-development
