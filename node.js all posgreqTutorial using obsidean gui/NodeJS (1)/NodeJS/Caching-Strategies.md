# Caching Strategies in Node.js

A comprehensive guide to implementing various caching strategies in Node.js applications for improved performance and scalability.

## In-Memory Caching

### Simple Cache Implementation
```javascript
class SimpleCache {
    constructor(options = {}) {
        this.cache = new Map();
        this.ttl = options.ttl || 3600000; // Default 1 hour
        this.checkPeriod = options.checkPeriod || 600000; // Clean up every 10 minutes
        this.startCleanupInterval();
    }

    set(key, value, ttl = this.ttl) {
        this.cache.set(key, {
            value,
            expires: Date.now() + ttl
        });
    }

    get(key) {
        const item = this.cache.get(key);
        if (!item) return null;
        
        if (Date.now() > item.expires) {
            this.cache.delete(key);
            return null;
        }
        
        return item.value;
    }

    delete(key) {
        return this.cache.delete(key);
    }

    clear() {
        this.cache.clear();
    }

    startCleanupInterval() {
        setInterval(() => {
            const now = Date.now();
            for (const [key, item] of this.cache.entries()) {
                if (now > item.expires) {
                    this.cache.delete(key);
                }
            }
        }, this.checkPeriod);
    }
}

// Usage
const cache = new SimpleCache({ ttl: 60000 }); // 1 minute TTL
```

### LRU Cache Implementation
```javascript
const LRU = require('lru-cache');

const cache = new LRU({
    max: 500, // Maximum number of items
    maxAge: 1000 * 60 * 60, // Items live for 1 hour
    updateAgeOnGet: true, // Update item age on access
    length: (item) => {
        // Calculate item size for max memory usage
        return JSON.stringify(item).length;
    }
});

// Middleware for caching API responses
function cacheMiddleware(duration) {
    return (req, res, next) => {
        const key = `__express__${req.originalUrl || req.url}`;
        const cachedBody = cache.get(key);

        if (cachedBody) {
            res.send(cachedBody);
            return;
        }

        const originalSend = res.send;
        res.send = function(body) {
            cache.set(key, body, duration * 1000);
            originalSend.call(this, body);
        };
        next();
    };
}
```

## Redis Caching

### Basic Redis Cache
```javascript
const Redis = require('ioredis');
const redis = new Redis({
    host: 'localhost',
    port: 6379,
    retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
    }
});

class RedisCache {
    constructor(prefix = 'cache:') {
        this.redis = redis;
        this.prefix = prefix;
    }

    async set(key, value, ttl = 3600) {
        const serialized = JSON.stringify(value);
        await this.redis.set(
            this.prefix + key,
            serialized,
            'EX',
            ttl
        );
    }

    async get(key) {
        const value = await this.redis.get(this.prefix + key);
        if (!value) return null;
        return JSON.parse(value);
    }

    async delete(key) {
        await this.redis.del(this.prefix + key);
    }

    async clear(pattern = '*') {
        const keys = await this.redis.keys(this.prefix + pattern);
        if (keys.length > 0) {
            await this.redis.del(keys);
        }
    }
}
```

### Redis Cache Patterns

#### Caching Database Queries
```javascript
class DatabaseCache {
    constructor(redisCache, db) {
        this.cache = redisCache;
        this.db = db;
    }

    async getUser(id) {
        const cacheKey = `user:${id}`;
        
        // Try cache first
        let user = await this.cache.get(cacheKey);
        if (user) {
            return user;
        }

        // Get from database
        user = await this.db.findUser(id);
        if (user) {
            // Cache for 1 hour
            await this.cache.set(cacheKey, user, 3600);
        }
        
        return user;
    }

    async updateUser(id, data) {
        // Update database
        const user = await this.db.updateUser(id, data);
        
        // Update cache
        await this.cache.set(`user:${id}`, user, 3600);
        
        return user;
    }
}
```

#### Cache Aside Pattern
```javascript
class CacheAside {
    constructor(cache, db) {
        this.cache = cache;
        this.db = db;
    }

    async get(key, fetchFn) {
        // Try cache first
        let data = await this.cache.get(key);
        if (data) {
            return data;
        }

        // Cache miss - fetch data
        data = await fetchFn();
        
        // Store in cache
        if (data) {
            await this.cache.set(key, data);
        }
        
        return data;
    }

    async invalidate(key) {
        await this.cache.delete(key);
    }
}

// Usage
const cacheAside = new CacheAside(redisCache, db);
const user = await cacheAside.get('user:123', () => db.findUser('123'));
```

## Distributed Caching

### Consistent Hashing
```javascript
class ConsistentHashing {
    constructor(nodes = [], replicas = 3) {
        this.replicas = replicas;
        this.ring = {};
        this.keys = [];
        
        for (const node of nodes) {
            this.addNode(node);
        }
    }

    addNode(node) {
        for (let i = 0; i < this.replicas; i++) {
            const hash = this.getHash(`${node}:${i}`);
            this.ring[hash] = node;
            this.keys.push(hash);
        }
        this.keys.sort((a, b) => a - b);
    }

    removeNode(node) {
        for (let i = 0; i < this.replicas; i++) {
            const hash = this.getHash(`${node}:${i}`);
            delete this.ring[hash];
            this.keys = this.keys.filter(key => key !== hash);
        }
    }

    getNode(key) {
        if (this.keys.length === 0) return null;
        
        const hash = this.getHash(key);
        const pos = this.findPosition(hash);
        
        return this.ring[this.keys[pos]];
    }

    getHash(key) {
        let hash = 0;
        for (let i = 0; i < key.length; i++) {
            hash = ((hash << 5) + hash) + key.charCodeAt(i);
            hash = hash & hash;
        }
        return Math.abs(hash);
    }

    findPosition(hash) {
        let left = 0;
        let right = this.keys.length - 1;
        
        if (hash <= this.keys[left]) return left;
        if (hash >= this.keys[right]) return 0;
        
        while (left + 1 < right) {
            const mid = Math.floor((left + right) / 2);
            if (this.keys[mid] === hash) {
                return mid;
            }
            if (this.keys[mid] > hash) {
                right = mid;
            } else {
                left = mid;
            }
        }
        
        return right;
    }
}
```

### Multi-Layer Caching
```javascript
class MultiLayerCache {
    constructor(options = {}) {
        this.layers = [
            new MemoryCache(options.memory),
            new RedisCache(options.redis)
        ];
    }

    async get(key) {
        for (let i = 0; i < this.layers.length; i++) {
            const value = await this.layers[i].get(key);
            if (value !== null) {
                // Populate upper layers
                for (let j = 0; j < i; j++) {
                    await this.layers[j].set(key, value);
                }
                return value;
            }
        }
        return null;
    }

    async set(key, value, ttl) {
        // Set in all layers
        await Promise.all(
            this.layers.map(layer => layer.set(key, value, ttl))
        );
    }

    async invalidate(key) {
        await Promise.all(
            this.layers.map(layer => layer.delete(key))
        );
    }
}
```

## Cache Invalidation Strategies

### Time-Based Invalidation
```javascript
class TimedCache {
    constructor(ttl = 3600) {
        this.cache = new Map();
        this.ttl = ttl;
    }

    set(key, value) {
        this.cache.set(key, {
            value,
            timestamp: Date.now()
        });
    }

    get(key) {
        const item = this.cache.get(key);
        if (!item) return null;

        if (Date.now() - item.timestamp > this.ttl * 1000) {
            this.cache.delete(key);
            return null;
        }

        return item.value;
    }
}
```

### Version-Based Invalidation
```javascript
class VersionedCache {
    constructor(redis) {
        this.redis = redis;
        this.prefix = 'cache:';
        this.versionPrefix = 'version:';
    }

    async get(key, version) {
        const currentVersion = await this.redis.get(
            this.versionPrefix + key
        );
        
        if (currentVersion !== version) {
            return null;
        }
        
        return await this.redis.get(this.prefix + key);
    }

    async set(key, value, version) {
        await Promise.all([
            this.redis.set(this.prefix + key, value),
            this.redis.set(this.versionPrefix + key, version)
        ]);
    }

    async invalidate(key) {
        const version = await this.redis.incr(
            this.versionPrefix + key
        );
        return version;
    }
}
```

## Monitoring and Metrics

### Cache Statistics
```javascript
class CacheStats {
    constructor(cache) {
        this.cache = cache;
        this.stats = {
            hits: 0,
            misses: 0,
            sets: 0,
            deletes: 0
        };
    }

    async get(key) {
        const value = await this.cache.get(key);
        if (value === null) {
            this.stats.misses++;
        } else {
            this.stats.hits++;
        }
        return value;
    }

    async set(key, value, ttl) {
        this.stats.sets++;
        return await this.cache.set(key, value, ttl);
    }

    async delete(key) {
        this.stats.deletes++;
        return await this.cache.delete(key);
    }

    getStats() {
        const total = this.stats.hits + this.stats.misses;
        return {
            ...this.stats,
            hitRate: total ? (this.stats.hits / total) : 0
        };
    }
}
```

## Best Practices

### 1. Cache Keys
```javascript
class CacheKeyBuilder {
    static buildKey(...parts) {
        return parts
            .map(part => String(part).replace(/[^a-zA-Z0-9]/g, '_'))
            .join(':');
    }

    static buildVersionedKey(key, version) {
        return `${key}:v${version}`;
    }

    static buildPrefixedKey(prefix, key) {
        return `${prefix}:${key}`;
    }
}
```

### 2. Error Handling
```javascript
class ResilientCache {
    constructor(cache, fallback) {
        this.cache = cache;
        this.fallback = fallback;
    }

    async get(key) {
        try {
            const value = await this.cache.get(key);
            if (value !== null) {
                return value;
            }
        } catch (error) {
            console.error('Cache error:', error);
        }

        return await this.fallback(key);
    }

    async set(key, value, ttl) {
        try {
            await this.cache.set(key, value, ttl);
        } catch (error) {
            console.error('Cache set error:', error);
        }
    }
}
```

## Related Topics
- [[Redis-Integration]] - Redis implementation details
- [[Performance-Optimization]] - Performance optimization techniques
- [[Distributed-Systems]] - Distributed system patterns
- [[Memory-Management]] - Memory management strategies

## Practice Projects
1. Build a multi-layer caching system
2. Implement a distributed cache with consistent hashing
3. Create a cache monitoring dashboard
4. Develop a cache warming system

## Resources
- [Redis Documentation](https://redis.io/documentation)
- [Node-Cache Documentation](https://github.com/node-cache/node-cache)
- [[Learning-Resources#Caching|Caching Resources]]

## Tags
#caching #performance #redis #distributed-systems #nodejs
