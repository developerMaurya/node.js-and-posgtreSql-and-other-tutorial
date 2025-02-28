# Fastify Middleware Guide: Beginner to Expert

## 1. Basic Middleware

### 1.1 Simple Request Hook

```javascript
// Basic middleware using hooks
const fastify = require('fastify')();

fastify.addHook('onRequest', async (request, reply) => {
  request.log.info('Request received');
});

fastify.get('/', async (request, reply) => {
  return { hello: 'world' };
});
```

### 1.2 Basic Authentication Middleware

```javascript
// Simple authentication middleware
async function authenticate(request, reply) {
  const token = request.headers.authorization;
  
  if (!token) {
    reply.code(401).send({ error: 'Authentication required' });
    return;
  }
  
  try {
    request.user = await verifyToken(token);
  } catch (err) {
    reply.code(401).send({ error: 'Invalid token' });
  }
}

fastify.addHook('preHandler', authenticate);
```

## 2. Intermediate Middleware Patterns

### 2.1 Route-Specific Middleware

```javascript
// Middleware for specific routes
const routeMiddleware = async (request, reply) => {
  if (!request.user.hasPermission('admin')) {
    reply.code(403).send({ error: 'Access denied' });
    return;
  }
};

fastify.get('/admin', {
  preHandler: routeMiddleware,
  handler: async (request, reply) => {
    return { admin: 'panel' };
  }
});
```

### 2.2 Multiple Middleware Chain

```javascript
// Chaining multiple middleware
const validateInput = async (request, reply) => {
  const { email } = request.body;
  if (!email || !email.includes('@')) {
    reply.code(400).send({ error: 'Invalid email' });
    return;
  }
};

const checkUserExists = async (request, reply) => {
  const user = await findUserByEmail(request.body.email);
  if (user) {
    reply.code(409).send({ error: 'User already exists' });
    return;
  }
};

fastify.post('/register', {
  preHandler: [validateInput, checkUserExists],
  handler: async (request, reply) => {
    // Registration logic
  }
});
```

### 2.3 Error Handling Middleware

```javascript
// Error handling middleware
const errorHandler = (error, request, reply) => {
  request.log.error(error);
  
  if (error.validation) {
    reply.code(400).send({
      error: 'Validation Error',
      messages: error.validation
    });
    return;
  }
  
  if (error.code === 'UNAUTHORIZED') {
    reply.code(401).send({
      error: 'Authentication Error',
      message: error.message
    });
    return;
  }
  
  reply.code(500).send({
    error: 'Internal Server Error',
    message: 'An unexpected error occurred'
  });
};

fastify.setErrorHandler(errorHandler);
```

## 3. Advanced Middleware Techniques

### 3.1 Context-Aware Middleware

```javascript
// Middleware with context sharing
const fp = require('fastify-plugin');

async function contextMiddleware(fastify, opts) {
  fastify.decorateRequest('context', null);
  
  fastify.addHook('onRequest', async (request, reply) => {
    request.context = {
      requestId: request.id,
      timestamp: Date.now(),
      correlationId: request.headers['x-correlation-id'],
      tenant: request.headers['x-tenant-id']
    };
  });
}

module.exports = fp(contextMiddleware);
```

### 3.2 Rate Limiting Middleware

```javascript
// Advanced rate limiting middleware
const fp = require('fastify-plugin');
const Redis = require('ioredis');

async function rateLimiter(fastify, opts) {
  const redis = new Redis(opts.redis);
  
  fastify.decorateRequest('rateLimit', null);
  
  fastify.addHook('onRequest', async (request, reply) => {
    const key = `ratelimit:${request.ip}`;
    const limit = opts.limit || 100;
    const window = opts.window || 60; // seconds
    
    const multi = redis.multi();
    multi.incr(key);
    multi.expire(key, window);
    
    const [count] = await multi.exec();
    
    request.rateLimit = {
      limit,
      remaining: Math.max(0, limit - count[1]),
      reset: Math.ceil(Date.now() / 1000) + window
    };
    
    reply.header('X-RateLimit-Limit', limit);
    reply.header('X-RateLimit-Remaining', request.rateLimit.remaining);
    reply.header('X-RateLimit-Reset', request.rateLimit.reset);
    
    if (count[1] > limit) {
      reply.code(429).send({
        error: 'Too Many Requests',
        message: 'Rate limit exceeded'
      });
      return;
    }
  });
}

module.exports = fp(rateLimiter);
```

### 3.3 Circuit Breaker Middleware

```javascript
// Circuit breaker middleware for external services
const fp = require('fastify-plugin');
const CircuitBreaker = require('opossum');

async function circuitBreakerMiddleware(fastify, opts) {
  const breakers = new Map();
  
  const defaultOptions = {
    timeout: 3000,
    errorThresholdPercentage: 50,
    resetTimeout: 30000
  };
  
  function createBreaker(service, options = {}) {
    const breaker = new CircuitBreaker(async (request) => {
      return await service(request);
    }, { ...defaultOptions, ...options });
    
    breaker.on('open', () => {
      fastify.log.warn(`Circuit breaker opened for service: ${service.name}`);
    });
    
    breaker.on('halfOpen', () => {
      fastify.log.info(`Circuit breaker half-open for service: ${service.name}`);
    });
    
    breaker.on('close', () => {
      fastify.log.info(`Circuit breaker closed for service: ${service.name}`);
    });
    
    return breaker;
  }
  
  fastify.decorate('circuitBreaker', {
    get: (service) => {
      if (!breakers.has(service)) {
        breakers.set(service, createBreaker(service));
      }
      return breakers.get(service);
    }
  });
}

module.exports = fp(circuitBreakerMiddleware);
```

## 4. Expert-Level Middleware

### 4.1 Dynamic Middleware Factory

```javascript
// Dynamic middleware factory with caching
const fp = require('fastify-plugin');

async function dynamicMiddlewareFactory(fastify, opts) {
  const middlewareCache = new Map();
  
  const createMiddleware = (config) => {
    const cacheKey = JSON.stringify(config);
    
    if (middlewareCache.has(cacheKey)) {
      return middlewareCache.get(cacheKey);
    }
    
    const middleware = async (request, reply) => {
      // Dynamic middleware logic based on config
      for (const rule of config.rules) {
        const result = await evaluateRule(rule, request);
        if (!result.success) {
          reply.code(result.code).send(result.error);
          return;
        }
      }
      
      // Add dynamic decorators
      for (const decorator of config.decorators) {
        request[decorator.name] = await decorator.value(request);
      }
    };
    
    middlewareCache.set(cacheKey, middleware);
    return middleware;
  };
  
  fastify.decorate('createMiddleware', createMiddleware);
}

module.exports = fp(dynamicMiddlewareFactory);
```

### 4.2 Observability Middleware

```javascript
// Advanced observability middleware
const fp = require('fastify-plugin');
const opentelemetry = require('@opentelemetry/api');

async function observabilityMiddleware(fastify, opts) {
  const tracer = opentelemetry.trace.getTracer('fastify-app');
  
  fastify.addHook('onRequest', async (request, reply) => {
    const span = tracer.startSpan('http_request', {
      attributes: {
        'http.method': request.method,
        'http.url': request.url,
        'http.host': request.hostname
      }
    });
    
    request.span = span;
    request.context = opentelemetry.trace.setSpan(
      opentelemetry.context.active(),
      span
    );
  });
  
  fastify.addHook('onResponse', async (request, reply) => {
    if (request.span) {
      request.span.setAttributes({
        'http.status_code': reply.statusCode,
        'http.response_content_length': reply.getHeader('content-length'),
        'http.response_time': reply.getResponseTime()
      });
      request.span.end();
    }
  });
  
  fastify.addHook('onError', async (request, reply, error) => {
    if (request.span) {
      request.span.setStatus({
        code: opentelemetry.SpanStatusCode.ERROR,
        message: error.message
      });
      request.span.recordException(error);
    }
  });
}

module.exports = fp(observabilityMiddleware);
```

### 4.3 Transaction Middleware

```javascript
// Advanced transaction middleware
const fp = require('fastify-plugin');

async function transactionMiddleware(fastify, opts) {
  fastify.decorateRequest('transaction', null);
  
  fastify.addHook('onRequest', async (request, reply) => {
    const transaction = await startTransaction();
    request.transaction = transaction;
  });
  
  fastify.addHook('onResponse', async (request, reply) => {
    if (reply.statusCode < 400) {
      await request.transaction.commit();
    } else {
      await request.transaction.rollback();
    }
  });
  
  fastify.addHook('onError', async (request, reply, error) => {
    if (request.transaction) {
      await request.transaction.rollback();
    }
  });
  
  async function startTransaction() {
    const transaction = {
      id: generateTransactionId(),
      operations: [],
      async commit() {
        for (const op of this.operations) {
          await op.execute();
        }
      },
      async rollback() {
        for (const op of this.operations.reverse()) {
          await op.rollback();
        }
      },
      addOperation(operation) {
        this.operations.push(operation);
      }
    };
    
    return transaction;
  }
}

module.exports = fp(transactionMiddleware);
```

## 5. Best Practices

### 5.1 Middleware Organization

```javascript
// middleware/index.js
const fp = require('fastify-plugin');

async function middlewareManager(fastify, opts) {
  // Register all middleware in correct order
  await fastify.register(require('./context'));
  await fastify.register(require('./auth'));
  await fastify.register(require('./rateLimit'));
  await fastify.register(require('./circuitBreaker'));
  await fastify.register(require('./transaction'));
  await fastify.register(require('./observability'));
}

module.exports = fp(middlewareManager);
```

### 5.2 Middleware Configuration

```javascript
// config/middleware.js
module.exports = {
  auth: {
    secret: process.env.JWT_SECRET,
    expiresIn: '1h'
  },
  rateLimit: {
    redis: {
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT
    },
    limit: 100,
    window: 60
  },
  circuitBreaker: {
    timeout: 3000,
    errorThreshold: 50,
    resetTimeout: 30000
  }
};
```

## Related Topics
- [[Fastify-Guide]] - Main Fastify guide
- [[Performance-Optimization]] - Performance tips
- [[Security]] - Security best practices
- [[Error-Handling]] - Error handling strategies

## Tags
#fastify #middleware #nodejs #performance #security #expert-features
