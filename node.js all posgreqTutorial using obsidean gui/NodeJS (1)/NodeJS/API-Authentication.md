# API Authentication with JWT

A comprehensive guide to implementing JSON Web Token (JWT) authentication in Node.js REST APIs.

## JWT Basics

### 1. JWT Structure
```javascript
// Example JWT
const jwt = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c';

// Parts:
// Header: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
// Payload: eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ
// Signature: SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 2. JWT Workflow
```mermaid
graph TD
    A[Client] -->|Login Request| B[Server]
    B -->|Validate Credentials| C{Valid?}
    C -->|Yes| D[Generate JWT]
    D --> E[Send JWT to Client]
    C -->|No| F[Send Error]
    E --> G[Client Stores JWT]
    G -->|API Requests| H[Server]
    H -->|Verify JWT| I{Valid?}
    I -->|Yes| J[Process Request]
    I -->|No| K[Send Error]
```

## Implementation

### 1. JWT Setup
```javascript
const jwt = require('jsonwebtoken');
require('dotenv').config();

// Configuration
const JWT_CONFIG = {
    secret: process.env.JWT_SECRET || 'your-secret-key',
    expiresIn: process.env.JWT_EXPIRES || '1h',
    algorithm: 'HS256'
};
```

### 2. Token Generation
```javascript
const generateToken = (payload) => {
    return jwt.sign(payload, JWT_CONFIG.secret, {
        expiresIn: JWT_CONFIG.expiresIn,
        algorithm: JWT_CONFIG.algorithm
    });
};

// Example usage
const user = {
    id: 123,
    email: 'user@example.com',
    role: 'user'
};

const token = generateToken(user);
```

### 3. Token Verification
```javascript
const verifyToken = (token) => {
    try {
        return jwt.verify(token, JWT_CONFIG.secret);
    } catch (error) {
        throw new Error('Invalid token');
    }
};

// Example usage
const decoded = verifyToken(token);
console.log(decoded); // { id: 123, email: 'user@example.com', role: 'user', iat: 1234567890, exp: 1234567890 }
```

## Authentication Middleware

### 1. Auth Middleware
```javascript
const authMiddleware = async (req, res, next) => {
    try {
        // Get token from header
        const token = req.header('Authorization')?.replace('Bearer ', '');
        
        if (!token) {
            throw new Error('No token provided');
        }

        // Verify token
        const decoded = verifyToken(token);

        // Attach user to request
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({
            error: 'Please authenticate'
        });
    }
};

// Usage in routes
app.get('/protected', authMiddleware, (req, res) => {
    res.json({ user: req.user });
});
```

### 2. Role-Based Access
```javascript
const roleMiddleware = (roles) => {
    return (req, res, next) => {
        if (!roles.includes(req.user.role)) {
            return res.status(403).json({
                error: 'Access denied'
            });
        }
        next();
    };
};

// Usage
app.get('/admin', 
    authMiddleware,
    roleMiddleware(['admin', 'super-admin']),
    (req, res) => {
        res.json({ message: 'Welcome admin' });
    }
);
```

## Security Best Practices

### 1. Secure Token Storage
```javascript
// Client-side storage options
const storageOptions = {
    // Best: HTTP-only, Secure cookies
    cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict'
    },

    // Alternative: Local storage (less secure)
    localStorage: {
        // Implement refresh token rotation
    }
};
```

### 2. Token Rotation
```javascript
const refreshToken = (req, res) => {
    try {
        const oldToken = req.header('Authorization')?.replace('Bearer ', '');
        const decoded = verifyToken(oldToken);

        // Generate new token
        const newToken = generateToken({
            id: decoded.id,
            email: decoded.email,
            role: decoded.role
        });

        res.json({ token: newToken });
    } catch (error) {
        res.status(401).json({
            error: 'Invalid token'
        });
    }
};

// Usage
app.post('/refresh-token', refreshToken);
```

## Testing Authentication

### 1. Unit Tests
```javascript
const { generateToken, verifyToken } = require('./auth');

describe('JWT Authentication', () => {
    test('should generate valid token', () => {
        const token = generateToken({ id: 1 });
        expect(token).toBeDefined();
    });

    test('should verify valid token', () => {
        const token = generateToken({ id: 1 });
        const decoded = verifyToken(token);
        expect(decoded.id).toBe(1);
    });
});
```

## Related Topics
- [[REST-API-Design]] - REST API design patterns
- [[Express-Security]] - Security best practices
- [[Express-Middleware]] - Middleware implementation
- [[Testing]] - API testing strategies

Tags: #nodejs #api #authentication #jwt #security
