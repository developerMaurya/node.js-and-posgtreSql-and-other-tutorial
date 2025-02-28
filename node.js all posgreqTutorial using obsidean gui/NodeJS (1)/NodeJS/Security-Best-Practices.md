# Security Best Practices in Node.js

A comprehensive guide to implementing security best practices in Node.js applications, covering common vulnerabilities, prevention strategies, and security implementations.

## 1. Input Validation and Sanitization

### Request Validation
```javascript
const express = require('express');
const { body, validationResult } = require('express-validator');

const app = express();
app.use(express.json());

// Validation middleware
const validateUser = [
    body('email').isEmail().normalizeEmail(),
    body('password')
        .isLength({ min: 8 })
        .matches(/^(?=.*\d)(?=.*[a-z])(?=.*[A-Z])(?=.*[^a-zA-Z0-9]).{8,}$/),
    body('name').trim().escape()
];

app.post('/api/users', validateUser, (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        return res.status(400).json({ errors: errors.array() });
    }
    // Process valid request
});
```

### SQL Injection Prevention
```javascript
const mysql = require('mysql2/promise');
const pool = mysql.createPool({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME
});

async function getUserById(id) {
    try {
        // Use parameterized queries
        const [rows] = await pool.execute(
            'SELECT * FROM users WHERE id = ?',
            [id]
        );
        return rows[0];
    } catch (error) {
        console.error('Database error:', error);
        throw new Error('Database error');
    }
}
```

### NoSQL Injection Prevention
```javascript
const mongoose = require('mongoose');
const { escape } = require('mongo-sanitize');

const UserSchema = new mongoose.Schema({
    username: String,
    email: String
});

async function findUser(query) {
    // Sanitize query parameters
    const sanitizedQuery = escape(query);
    return await User.findOne(sanitizedQuery);
}
```

## 2. Authentication and Authorization

### JWT Implementation
```javascript
const jwt = require('jsonwebtoken');

class AuthService {
    static generateToken(user) {
        return jwt.sign(
            {
                id: user.id,
                role: user.role
            },
            process.env.JWT_SECRET,
            {
                expiresIn: '1h',
                algorithm: 'HS256'
            }
        );
    }

    static verifyToken(token) {
        try {
            return jwt.verify(token, process.env.JWT_SECRET);
        } catch (error) {
            throw new Error('Invalid token');
        }
    }
}

// Middleware to protect routes
const authenticate = (req, res, next) => {
    try {
        const token = req.headers.authorization?.split(' ')[1];
        if (!token) {
            return res.status(401).json({ message: 'No token provided' });
        }

        const decoded = AuthService.verifyToken(token);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ message: 'Invalid token' });
    }
};
```

### Role-Based Access Control (RBAC)
```javascript
const ROLES = {
    ADMIN: 'admin',
    USER: 'user',
    GUEST: 'guest'
};

const PERMISSIONS = {
    CREATE_USER: 'create:user',
    DELETE_USER: 'delete:user',
    VIEW_USERS: 'view:users'
};

const rolePermissions = {
    [ROLES.ADMIN]: [
        PERMISSIONS.CREATE_USER,
        PERMISSIONS.DELETE_USER,
        PERMISSIONS.VIEW_USERS
    ],
    [ROLES.USER]: [PERMISSIONS.VIEW_USERS],
    [ROLES.GUEST]: []
};

const checkPermission = (permission) => {
    return (req, res, next) => {
        const userRole = req.user.role;
        if (!rolePermissions[userRole]?.includes(permission)) {
            return res.status(403).json({
                message: 'Insufficient permissions'
            });
        }
        next();
    };
};

// Usage in routes
app.delete('/api/users/:id',
    authenticate,
    checkPermission(PERMISSIONS.DELETE_USER),
    async (req, res) => {
        // Handle delete user
    }
);
```

## 3. Session Management

### Secure Session Configuration
```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);
const redis = require('redis');

const redisClient = redis.createClient({
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT,
    password: process.env.REDIS_PASSWORD
});

app.use(session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET,
    name: 'sessionId', // Don't use default connect.sid
    cookie: {
        secure: process.env.NODE_ENV === 'production',
        httpOnly: true,
        maxAge: 1000 * 60 * 60 * 24, // 24 hours
        sameSite: 'strict'
    },
    resave: false,
    saveUninitialized: false
}));
```

## 4. Security Headers

### Helmet Configuration
```javascript
const helmet = require('helmet');

app.use(helmet()); // Basic configuration

// Custom configuration
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            scriptSrc: ["'self'", "'unsafe-inline'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            imgSrc: ["'self'", "data:", "https:"],
            connectSrc: ["'self'", "https://api.example.com"],
            fontSrc: ["'self'", "https://fonts.gstatic.com"],
            objectSrc: ["'none'"],
            mediaSrc: ["'self'"],
            frameSrc: ["'none'"]
        }
    },
    crossOriginEmbedderPolicy: true,
    crossOriginOpenerPolicy: true,
    crossOriginResourcePolicy: { policy: "same-site" },
    dnsPrefetchControl: true,
    frameguard: { action: "deny" },
    hidePoweredBy: true,
    hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
    },
    ieNoOpen: true,
    noSniff: true,
    referrerPolicy: { policy: "strict-origin-when-cross-origin" },
    xssFilter: true
}));
```

## 5. Rate Limiting

### Rate Limiter Implementation
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Create Redis client
const redisClient = redis.createClient({
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT,
    password: process.env.REDIS_PASSWORD
});

// Configure rate limiter
const limiter = rateLimit({
    store: new RedisStore({
        client: redisClient,
        prefix: 'rate_limit:'
    }),
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // Limit each IP to 100 requests per windowMs
    message: {
        error: 'Too many requests, please try again later.'
    },
    standardHeaders: true,
    legacyHeaders: false
});

// Apply rate limiting to all routes
app.use(limiter);

// Apply specific limits to authentication routes
const authLimiter = rateLimit({
    store: new RedisStore({
        client: redisClient,
        prefix: 'auth_limit:'
    }),
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 5, // 5 attempts per hour
    message: {
        error: 'Too many login attempts, please try again later.'
    }
});

app.use('/api/auth', authLimiter);
```

## 6. File Upload Security

### Secure File Upload Implementation
```javascript
const multer = require('multer');
const path = require('path');
const crypto = require('crypto');

// Configure storage
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        // Generate random filename
        crypto.randomBytes(16, (err, raw) => {
            if (err) return cb(err);

            cb(null, raw.toString('hex') + path.extname(file.originalname));
        });
    }
});

// Configure upload limits and file filtering
const upload = multer({
    storage: storage,
    limits: {
        fileSize: 5 * 1024 * 1024, // 5MB
        files: 1
    },
    fileFilter: (req, file, cb) => {
        // Allowed file types
        const allowedTypes = /jpeg|jpg|png|pdf/;
        const extname = allowedTypes.test(
            path.extname(file.originalname).toLowerCase()
        );
        const mimetype = allowedTypes.test(file.mimetype);

        if (extname && mimetype) {
            return cb(null, true);
        }
        cb(new Error('Invalid file type'));
    }
});

// Route handler
app.post('/api/upload',
    authenticate,
    upload.single('file'),
    async (req, res) => {
        try {
            if (!req.file) {
                return res.status(400).json({
                    message: 'No file uploaded'
                });
            }

            // Scan file for viruses (implement with appropriate library)
            await scanFile(req.file.path);

            // Process file
            const fileUrl = await processAndStoreFile(req.file);

            res.json({ url: fileUrl });
        } catch (error) {
            res.status(500).json({
                message: 'File upload failed'
            });
        }
    }
);
```

## 7. Error Handling and Logging

### Secure Error Handler
```javascript
class AppError extends Error {
    constructor(message, statusCode) {
        super(message);
        this.statusCode = statusCode;
        this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
        this.isOperational = true;

        Error.captureStackTrace(this, this.constructor);
    }
}

// Global error handler
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
        // Production error response
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
                message: 'Something went wrong'
            });
        }
    }
};

app.use(errorHandler);
```

### Secure Logging
```javascript
const winston = require('winston');
const { format } = winston;

// Create logger
const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: format.combine(
        format.timestamp(),
        format.errors({ stack: true }),
        format.splat(),
        format.json()
    ),
    defaultMeta: { service: 'user-service' },
    transports: [
        new winston.transports.File({
            filename: 'logs/error.log',
            level: 'error',
            maxsize: 5242880, // 5MB
            maxFiles: 5
        }),
        new winston.transports.File({
            filename: 'logs/combined.log',
            maxsize: 5242880,
            maxFiles: 5
        })
    ]
});

// Add console transport in development
if (process.env.NODE_ENV !== 'production') {
    logger.add(new winston.transports.Console({
        format: format.combine(
            format.colorize(),
            format.simple()
        )
    }));
}

// Sanitize sensitive data
const sanitizeData = (data) => {
    const sensitiveFields = ['password', 'token', 'credit_card'];
    const sanitized = { ...data };

    sensitiveFields.forEach(field => {
        if (sanitized[field]) {
            sanitized[field] = '[REDACTED]';
        }
    });

    return sanitized;
};

// Logging middleware
app.use((req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info({
            method: req.method,
            url: req.url,
            status: res.statusCode,
            duration,
            ip: req.ip,
            user: req.user?.id,
            body: sanitizeData(req.body)
        });
    });

    next();
});
```

## 8. Security Monitoring and Auditing

### Security Event Monitoring
```javascript
const EventEmitter = require('events');
const securityEvents = new EventEmitter();

class SecurityMonitor {
    constructor() {
        this.failedLogins = new Map();
        this.suspiciousIPs = new Set();

        this.setupEventHandlers();
    }

    setupEventHandlers() {
        securityEvents.on('failedLogin', ({ ip, username }) => {
            this.trackFailedLogin(ip);
        });

        securityEvents.on('suspiciousActivity', ({ ip, activity }) => {
            this.trackSuspiciousActivity(ip, activity);
        });
    }

    trackFailedLogin(ip) {
        const attempts = (this.failedLogins.get(ip) || 0) + 1;
        this.failedLogins.set(ip, attempts);

        if (attempts >= 5) {
            this.blockIP(ip);
        }
    }

    trackSuspiciousActivity(ip, activity) {
        logger.warn({
            message: 'Suspicious activity detected',
            ip,
            activity
        });

        this.suspiciousIPs.add(ip);
    }

    blockIP(ip) {
        // Implement IP blocking logic
        logger.error({
            message: 'IP blocked due to multiple failed attempts',
            ip
        });
    }

    generateSecurityReport() {
        return {
            blockedIPs: Array.from(this.suspiciousIPs),
            failedLoginAttempts: Array.from(this.failedLogins.entries()),
            timestamp: new Date()
        };
    }
}

const securityMonitor = new SecurityMonitor();

// Usage in authentication routes
app.post('/api/login', async (req, res) => {
    try {
        const { username, password } = req.body;
        const user = await authenticate(username, password);

        if (!user) {
            securityEvents.emit('failedLogin', {
                ip: req.ip,
                username
            });
            return res.status(401).json({
                message: 'Invalid credentials'
            });
        }

        // Continue with successful login
    } catch (error) {
        logger.error('Login error:', error);
        res.status(500).json({
            message: 'Login failed'
        });
    }
});
```

## Related Topics
- [[API-Authentication]] - Detailed API authentication strategies
- [[Data-Encryption]] - Data encryption implementations
- [[Secure-Communication]] - Secure communication protocols
- [[Vulnerability-Testing]] - Security testing methodologies

## Practice Projects
1. Build a secure authentication system
2. Implement RBAC with JWT
3. Create a secure file upload service
4. Develop a security monitoring dashboard

## Resources
- [OWASP Node.js Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [[Learning-Resources#Security|Security Resources]]

## Tags
#security #authentication #authorization #nodejs #best-practices
