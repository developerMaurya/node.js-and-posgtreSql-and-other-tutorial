# Security in Node.js

Essential security practices for Node.js applications.

## Key Security Concepts

### 1. Input Validation
```javascript
const { body, validationResult } = require('express-validator');

app.post('/user',
    body('email').isEmail(),
    body('password').isLength({ min: 8 }),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        // Process valid input
    }
);
```

### 2. Authentication & Authorization
```javascript
const jwt = require('jsonwebtoken');

// Create JWT
const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET);

// Verify JWT
const authenticateToken = (req, res, next) => {
    const token = req.headers['authorization'];
    
    if (!token) return res.sendStatus(401);
    
    jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
        if (err) return res.sendStatus(403);
        req.user = user;
        next();
    });
};
```

## Common Security Issues

### 1. Cross-Site Scripting (XSS)
Prevention:
```javascript
const helmet = require('helmet');
app.use(helmet()); // Adds various HTTP headers

// Sanitize user input
const sanitizeHtml = require('sanitize-html');
const clean = sanitizeHtml(dirtyInput);
```

### 2. SQL Injection
Prevention:
```javascript
// Use parameterized queries
const { rows } = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
);
```

### 3. CSRF Protection
```javascript
const csrf = require('csurf');
app.use(csrf({ cookie: true }));

app.get('/form', (req, res) => {
    res.render('form', { csrfToken: req.csrfToken() });
});
```

## Best Practices
1. Use HTTPS
2. Implement rate limiting
3. Keep dependencies updated
4. Use security headers
5. Implement proper logging
6. Use environment variables for secrets

## Security Headers
```javascript
// Using helmet
app.use(helmet({
    contentSecurityPolicy: true,
    xssFilter: true,
    noSniff: true,
    hsts: true
}));
```

## Related Topics
- [[Authentication]] - User authentication methods
- [[Error-Handling-Advanced]] - Secure error handling
- [[Database-Integration]] - Database security
- [[Express-Framework]] - Express.js security

Tags: #nodejs #security #authentication #authorization
