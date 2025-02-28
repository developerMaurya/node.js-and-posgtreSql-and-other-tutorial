# Security Architecture in Node.js

A comprehensive guide to implementing secure architecture in Node.js applications.

## 1. Authentication System

### JWT Authentication Implementation
```javascript
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const crypto = require('crypto');

class AuthenticationSystem {
  constructor(options = {}) {
    this.options = {
      jwtSecret: options.jwtSecret || crypto.randomBytes(32).toString('hex'),
      jwtExpiry: options.jwtExpiry || '1h',
      refreshTokenExpiry: options.refreshTokenExpiry || '7d',
      saltRounds: options.saltRounds || 10
    };
    
    this.refreshTokens = new Map();
  }

  async hashPassword(password) {
    return bcrypt.hash(password, this.options.saltRounds);
  }

  async verifyPassword(password, hash) {
    return bcrypt.compare(password, hash);
  }

  async generateTokens(userId, roles = []) {
    const accessToken = jwt.sign(
      { userId, roles },
      this.options.jwtSecret,
      { expiresIn: this.options.jwtExpiry }
    );
    
    const refreshToken = crypto.randomBytes(40).toString('hex');
    const refreshTokenExpiry = Date.now() + ms(this.options.refreshTokenExpiry);
    
    this.refreshTokens.set(refreshToken, {
      userId,
      roles,
      expiry: refreshTokenExpiry
    });
    
    return {
      accessToken,
      refreshToken,
      expiresIn: ms(this.options.jwtExpiry)
    };
  }

  async verifyToken(token) {
    try {
      return jwt.verify(token, this.options.jwtSecret);
    } catch (error) {
      throw new Error('Invalid token');
    }
  }

  async refreshAccessToken(refreshToken) {
    const tokenData = this.refreshTokens.get(refreshToken);
    
    if (!tokenData || Date.now() > tokenData.expiry) {
      throw new Error('Invalid or expired refresh token');
    }
    
    const { userId, roles } = tokenData;
    return this.generateTokens(userId, roles);
  }

  async revokeRefreshToken(refreshToken) {
    this.refreshTokens.delete(refreshToken);
  }

  async revokeAllUserTokens(userId) {
    for (const [token, data] of this.refreshTokens.entries()) {
      if (data.userId === userId) {
        this.refreshTokens.delete(token);
      }
    }
  }
}
```

## 2. Authorization System

### RBAC Implementation
```javascript
class RBACSystem {
  constructor() {
    this.roles = new Map();
    this.permissions = new Map();
    this.roleHierarchy = new Map();
  }

  addRole(role, parentRole = null) {
    if (!this.roles.has(role)) {
      this.roles.set(role, new Set());
    }
    
    if (parentRole) {
      if (!this.roleHierarchy.has(role)) {
        this.roleHierarchy.set(role, new Set());
      }
      this.roleHierarchy.get(role).add(parentRole);
    }
  }

  addPermission(permission, roles) {
    this.permissions.set(permission, new Set(roles));
  }

  hasPermission(role, permission) {
    if (!this.roles.has(role)) {
      return false;
    }
    
    const permissionRoles = this.permissions.get(permission);
    if (!permissionRoles) {
      return false;
    }
    
    // Check direct permission
    if (permissionRoles.has(role)) {
      return true;
    }
    
    // Check inherited permissions
    const parentRoles = this.getParentRoles(role);
    return [...parentRoles].some(parentRole => 
      permissionRoles.has(parentRole)
    );
  }

  getParentRoles(role, visited = new Set()) {
    const parentRoles = new Set();
    
    if (visited.has(role)) {
      return parentRoles;
    }
    
    visited.add(role);
    
    const directParents = this.roleHierarchy.get(role);
    if (directParents) {
      for (const parent of directParents) {
        parentRoles.add(parent);
        const grandParents = this.getParentRoles(parent, visited);
        for (const grandParent of grandParents) {
          parentRoles.add(grandParent);
        }
      }
    }
    
    return parentRoles;
  }
}
```

## 3. Encryption System

### Data Encryption Implementation
```javascript
class EncryptionSystem {
  constructor(options = {}) {
    this.options = {
      algorithm: options.algorithm || 'aes-256-gcm',
      keyLength: options.keyLength || 32,
      ivLength: options.ivLength || 16,
      saltLength: options.saltLength || 16
    };
  }

  async generateKey(password, salt) {
    return new Promise((resolve, reject) => {
      crypto.pbkdf2(
        password,
        salt,
        100000,
        this.options.keyLength,
        'sha512',
        (err, key) => {
          if (err) reject(err);
          else resolve(key);
        }
      );
    });
  }

  async encrypt(data, key) {
    const iv = crypto.randomBytes(this.options.ivLength);
    const cipher = crypto.createCipheriv(
      this.options.algorithm,
      key,
      iv
    );
    
    let encrypted = cipher.update(
      typeof data === 'string' ? data : JSON.stringify(data),
      'utf8',
      'hex'
    );
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }

  async decrypt(encryptedData, key) {
    const decipher = crypto.createDecipheriv(
      this.options.algorithm,
      key,
      Buffer.from(encryptedData.iv, 'hex')
    );
    
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    try {
      return JSON.parse(decrypted);
    } catch {
      return decrypted;
    }
  }
}
```

## 4. Security Middleware

### Security Middleware Implementation
```javascript
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const cors = require('cors');

class SecurityMiddleware {
  constructor(options = {}) {
    this.options = {
      rateLimitWindowMs: options.rateLimitWindowMs || 15 * 60 * 1000,
      rateLimitMax: options.rateLimitMax || 100,
      corsOrigin: options.corsOrigin || '*',
      ...options
    };
  }

  getMiddleware() {
    return [
      // Basic Security Headers
      helmet(),
      
      // Rate Limiting
      rateLimit({
        windowMs: this.options.rateLimitWindowMs,
        max: this.options.rateLimitMax
      }),
      
      // CORS
      cors({
        origin: this.options.corsOrigin,
        methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
        allowedHeaders: ['Content-Type', 'Authorization']
      }),
      
      // Request Validation
      this.validateRequest.bind(this),
      
      // Response Security
      this.secureResponse.bind(this)
    ];
  }

  validateRequest(req, res, next) {
    // Content-Type validation
    if (req.method !== 'GET' && !req.is('application/json')) {
      return res.status(415).json({
        error: 'Unsupported Media Type'
      });
    }
    
    // Content length validation
    const contentLength = parseInt(req.headers['content-length'] || 0);
    if (contentLength > this.options.maxContentLength) {
      return res.status(413).json({
        error: 'Payload Too Large'
      });
    }
    
    next();
  }

  secureResponse(req, res, next) {
    // Remove sensitive headers
    const originalSend = res.send;
    res.send = function (body) {
      // Remove internal headers
      res.removeHeader('X-Powered-By');
      
      // Add security headers
      res.setHeader('X-Content-Type-Options', 'nosniff');
      res.setHeader('X-Frame-Options', 'DENY');
      res.setHeader('X-XSS-Protection', '1; mode=block');
      
      return originalSend.call(this, body);
    };
    
    next();
  }
}
```

## 5. Audit System

### Audit Logger Implementation
```javascript
class AuditLogger {
  constructor(options = {}) {
    this.options = {
      logLevel: options.logLevel || 'info',
      logFormat: options.logFormat || 'json',
      storage: options.storage || 'file',
      retention: options.retention || '30d',
      ...options
    };
    
    this.setupStorage();
  }

  setupStorage() {
    switch (this.options.storage) {
      case 'file':
        // Setup file storage
        break;
      case 'database':
        // Setup database storage
        break;
      case 'external':
        // Setup external service
        break;
    }
  }

  async log(event) {
    const auditEntry = {
      timestamp: new Date().toISOString(),
      event: event.type,
      user: event.user,
      action: event.action,
      resource: event.resource,
      status: event.status,
      metadata: event.metadata,
      ip: event.ip,
      userAgent: event.userAgent
    };
    
    await this.store(auditEntry);
  }

  async store(entry) {
    switch (this.options.storage) {
      case 'file':
        await this.storeToFile(entry);
        break;
      case 'database':
        await this.storeToDatabase(entry);
        break;
      case 'external':
        await this.storeToExternalService(entry);
        break;
    }
  }

  async query(filters) {
    // Implement query logic based on storage type
  }

  async cleanup() {
    // Implement cleanup logic based on retention policy
  }
}
```

## 6. Security Testing

### Security Test Implementation
```javascript
const supertest = require('supertest');
const { expect } = require('chai');

class SecurityTester {
  constructor(app) {
    this.request = supertest(app);
  }

  async testAuthentication() {
    // Test authentication endpoints
    const tests = [
      {
        name: 'Should reject invalid credentials',
        endpoint: '/auth/login',
        method: 'post',
        data: { username: 'test', password: 'wrong' },
        expectedStatus: 401
      },
      {
        name: 'Should reject invalid tokens',
        endpoint: '/protected',
        method: 'get',
        headers: { Authorization: 'Bearer invalid' },
        expectedStatus: 401
      }
    ];
    
    for (const test of tests) {
      const response = await this.request[test.method](test.endpoint)
        .send(test.data)
        .set(test.headers || {});
      
      expect(response.status).to.equal(test.expectedStatus);
    }
  }

  async testAuthorization() {
    // Test authorization rules
  }

  async testInputValidation() {
    // Test input validation
  }

  async testRateLimiting() {
    // Test rate limiting
  }

  async testSecurityHeaders() {
    // Test security headers
  }

  async runAllTests() {
    await this.testAuthentication();
    await this.testAuthorization();
    await this.testInputValidation();
    await this.testRateLimiting();
    await this.testSecurityHeaders();
  }
}
```

## Related Topics
- [[Zero-Trust]] - Zero Trust Architecture
- [[Compliance-Guide]] - Compliance Implementation
- [[Penetration-Testing]] - Security Testing Guide
- [[Audit-Logging]] - Audit System

## Practice Projects
1. Build authentication system
2. Implement RBAC
3. Create encryption system
4. Design audit logger

## Resources
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [OWASP Node.js Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html)
- [[Learning-Resources#Security|Security Resources]]

## Tags
#security #nodejs #authentication #authorization #encryption #audit
