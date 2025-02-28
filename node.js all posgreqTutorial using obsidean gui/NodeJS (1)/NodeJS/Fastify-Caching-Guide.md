# Fastify Caching Guide

## 1. In-Memory Caching

### 1.1 Basic In-Memory Cache

```javascript
// plugins/cache.js
const fp = require('fastify-plugin');
const NodeCache = require('node-cache');

async function cachePlugin(fastify, opts) {
  const cache = new NodeCache({
    stdTTL: opts.ttl || 300, // 5 minutes default
    checkperiod: opts.checkPeriod || 600 // 10 minutes default
  });

  fastify.decorate('cache', cache);

  // Cache decorator for routes
  fastify.decorateReply('setCache', function (key, data, ttl) {
    return cache.set(key, data, ttl);
  });

  fastify.decorateRequest('getCache', function (key) {
    return cache.get(key);
  });
}

module.exports = fp(cachePlugin);
```

### 1.2 Route-Level Caching

```javascript
// routes/cached-routes.js
async function routes(fastify, opts) {
  fastify.get('/cached-data', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            data: { type: 'string' }
          }
        }
      }
    },
    async handler(request, reply) {
      const cacheKey = 'cached-data';
      const cachedData = request.getCache(cacheKey);

      if (cachedData) {
        return { data: cachedData };
      }

      const data = await fetchExpensiveData();
      reply.setCache(cacheKey, data, 300); // Cache for 5 minutes

      return { data };
    }
  });
}

module.exports = routes;
```

## 2. Redis Caching

### 2.1 Redis Cache Plugin

```javascript
// plugins/redis-cache.js
const fp = require('fastify-plugin');
const Redis = require('ioredis');

async function redisCachePlugin(fastify, opts) {
  const redis = new Redis({
    host: opts.redis.host || 'localhost',
    port: opts.redis.port || 6379,
    password: opts.redis.password,
    db: opts.redis.db || 0,
    keyPrefix: opts.redis.prefix || 'fastify:'
  });

  // Handle Redis connection errors
  redis.on('error', (err) => {
    fastify.log.error('Redis connection error:', err);
  });

  fastify.addHook('onClose', async () => {
    await redis.quit();
  });

  fastify.decorate('redis', redis);

  // Cache helpers
  fastify.decorate('cacheGet', async function (key) {
    const data = await redis.get(key);
    return data ? JSON.parse(data) : null;
  });

  fastify.decorate('cacheSet', async function (key, data, ttl = 300) {
    await redis.set(key, JSON.stringify(data), 'EX', ttl);
  });

  fastify.decorate('cacheDelete', async function (key) {
    await redis.del(key);
  });
}

module.exports = fp(redisCachePlugin);
```

### 2.2 Advanced Redis Caching Strategies

```javascript
// plugins/advanced-redis-cache.js
const fp = require('fastify-plugin');

async function advancedCachePlugin(fastify, opts) {
  // Cache by pattern
  fastify.decorate('cacheByPattern', async function (pattern) {
    const keys = await fastify.redis.keys(pattern);
    const pipeline = fastify.redis.pipeline();
    
    keys.forEach(key => pipeline.get(key));
    const results = await pipeline.exec();
    
    return results.map(([err, data]) => {
      if (err) throw err;
      return JSON.parse(data);
    });
  });

  // Cache with tags
  fastify.decorate('cacheSetWithTags', async function (key, data, tags = [], ttl = 300) {
    const pipeline = fastify.redis.pipeline();
    
    // Store the data
    pipeline.set(key, JSON.stringify(data), 'EX', ttl);
    
    // Store tag associations
    tags.forEach(tag => {
      pipeline.sadd(`tag:${tag}`, key);
    });
    
    await pipeline.exec();
  });

  // Invalidate by tags
  fastify.decorate('invalidateByTags', async function (tags) {
    const pipeline = fastify.redis.pipeline();
    
    for (const tag of tags) {
      const keys = await fastify.redis.smembers(`tag:${tag}`);
      keys.forEach(key => pipeline.del(key));
      pipeline.del(`tag:${tag}`);
    }
    
    await pipeline.exec();
  });
}

module.exports = fp(advancedCachePlugin);
```

## 3. HTTP Caching

### 3.1 HTTP Cache Headers

```javascript
// plugins/http-cache.js
const fp = require('fastify-plugin');

async function httpCachePlugin(fastify, opts) {
  fastify.decorateReply('setHttpCache', function (options = {}) {
    const {
      public = true,
      maxAge = 3600,
      staleWhileRevalidate = 60,
      etag = true
    } = options;

    let cacheControl = public ? 'public' : 'private';
    cacheControl += `, max-age=${maxAge}`;
    
    if (staleWhileRevalidate) {
      cacheControl += `, stale-while-revalidate=${staleWhileRevalidate}`;
    }

    this.header('Cache-Control', cacheControl);
    
    if (etag) {
      this.header('ETag', generateETag(this.send));
    }

    return this;
  });

  // ETag middleware
  fastify.addHook('preHandler', async (request, reply) => {
    const ifNoneMatch = request.headers['if-none-match'];
    if (ifNoneMatch) {
      request.etag = ifNoneMatch;
    }
  });
}

module.exports = fp(httpCachePlugin);
```

### 3.2 Conditional Requests

```javascript
// routes/conditional-routes.js
async function routes(fastify, opts) {
  fastify.get('/cached-resource', async (request, reply) => {
    const resource = await fetchResource();
    const etag = generateETag(resource);

    if (request.etag === etag) {
      reply.code(304).send();
      return;
    }

    reply
      .setHttpCache({ maxAge: 3600 })
      .header('ETag', etag)
      .send(resource);
  });
}

module.exports = routes;
```

## 4. Distributed Caching

### 4.1 Cluster-Aware Caching

```javascript
// plugins/distributed-cache.js
const fp = require('fastify-plugin');
const Redis = require('ioredis');
const Cluster = require('ioredis').Cluster;

async function distributedCachePlugin(fastify, opts) {
  const cluster = new Cluster([
    {
      host: opts.redis.host,
      port: opts.redis.port
    }
  ], {
    redisOptions: {
      password: opts.redis.password
    },
    scaleReads: 'slave'
  });

  fastify.decorate('distributedCache', {
    async get(key) {
      const data = await cluster.get(key);
      return data ? JSON.parse(data) : null;
    },

    async set(key, data, ttl = 300) {
      await cluster.set(key, JSON.stringify(data), 'EX', ttl);
    },

    async delete(key) {
      await cluster.del(key);
    },

    async invalidatePattern(pattern) {
      const keys = await cluster.keys(pattern);
      if (keys.length > 0) {
        await cluster.del(...keys);
      }
    }
  });
}

module.exports = fp(distributedCachePlugin);
```

### 4.2 Cache Synchronization

```javascript
// plugins/cache-sync.js
const fp = require('fastify-plugin');

async function cacheSyncPlugin(fastify, opts) {
  const pubsub = fastify.redis.duplicate();

  pubsub.subscribe('cache:invalidate');

  pubsub.on('message', async (channel, message) => {
    const { pattern, tags } = JSON.parse(message);

    if (pattern) {
      await fastify.distributedCache.invalidatePattern(pattern);
    }

    if (tags) {
      await fastify.invalidateByTags(tags);
    }
  });

  fastify.decorate('invalidateCache', async function (options = {}) {
    const message = JSON.stringify(options);
    await fastify.redis.publish('cache:invalidate', message);
  });

  fastify.addHook('onClose', async () => {
    await pubsub.quit();
  });
}

module.exports = fp(cacheSyncPlugin);
```

## 5. Cache Management

### 5.1 Cache Monitoring

```javascript
// plugins/cache-monitor.js
const fp = require('fastify-plugin');

async function cacheMonitorPlugin(fastify, opts) {
  const metrics = {
    hits: 0,
    misses: 0,
    sets: 0,
    deletes: 0
  };

  fastify.decorate('cacheMetrics', {
    getMetrics() {
      return { ...metrics };
    },

    recordHit() {
      metrics.hits++;
    },

    recordMiss() {
      metrics.misses++;
    },

    recordSet() {
      metrics.sets++;
    },

    recordDelete() {
      metrics.deletes++;
    }
  });

  // Expose metrics endpoint
  fastify.get('/cache/metrics', async () => {
    return fastify.cacheMetrics.getMetrics();
  });
}

module.exports = fp(cacheMonitorPlugin);
```

### 5.2 Cache Warmup

```javascript
// plugins/cache-warmup.js
const fp = require('fastify-plugin');

async function cacheWarmupPlugin(fastify, opts) {
  fastify.decorate('warmupCache', async function (patterns) {
    const warmupTasks = patterns.map(async pattern => {
      const data = await fetchDataForPattern(pattern);
      await fastify.cacheSet(pattern, data);
    });

    await Promise.all(warmupTasks);
  });

  // Warmup on startup
  if (opts.warmupPatterns) {
    fastify.addHook('onReady', async () => {
      await fastify.warmupCache(opts.warmupPatterns);
    });
  }
}

module.exports = fp(cacheWarmupPlugin);
```

## 6. Usage Example

```javascript
// app.js
const fastify = require('fastify')();

// Register cache plugins
fastify.register(require('./plugins/redis-cache'), {
  redis: {
    host: 'localhost',
    port: 6379,
    prefix: 'myapp:'
  }
});

fastify.register(require('./plugins/advanced-redis-cache'));
fastify.register(require('./plugins/http-cache'));
fastify.register(require('./plugins/distributed-cache'));
fastify.register(require('./plugins/cache-sync'));
fastify.register(require('./plugins/cache-monitor'));
fastify.register(require('./plugins/cache-warmup'), {
  warmupPatterns: ['popular:*', 'config:*']
});

// Example route with multiple caching strategies
fastify.get('/api/products/:id', async (request, reply) => {
  const cacheKey = `product:${request.params.id}`;
  
  // Try cache first
  const cachedProduct = await fastify.cacheGet(cacheKey);
  if (cachedProduct) {
    fastify.cacheMetrics.recordHit();
    return cachedProduct;
  }

  fastify.cacheMetrics.recordMiss();
  
  // Fetch from database
  const product = await fetchProduct(request.params.id);
  
  // Cache with tags
  await fastify.cacheSetWithTags(
    cacheKey,
    product,
    [`category:${product.category}`, 'products'],
    3600
  );
  
  // Set HTTP cache headers
  reply.setHttpCache({
    maxAge: 3600,
    staleWhileRevalidate: 60
  });
  
  return product;
});

// Start server
fastify.listen({ port: 3000 }, (err) => {
  if (err) throw err;
});
```

## Related Topics
- [[Performance-Optimization]] - Performance tips
- [[Redis-Integration]] - Redis setup and configuration
- [[Distributed-Systems]] - Distributed caching strategies
- [[Monitoring]] - Monitoring and metrics

## Tags
#fastify #caching #redis #performance #distributed-systems
