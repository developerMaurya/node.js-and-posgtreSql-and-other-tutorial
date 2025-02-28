# Connection Pooling in Node.js

A comprehensive guide to implementing and managing connection pools for databases and external services in Node.js applications.

## Overview

Connection pooling is a technique of maintaining a pool of reusable connections to improve performance and manage resources efficiently when interacting with databases and external services.

## Database Connection Pooling

### MySQL Connection Pool
```javascript
const mysql = require('mysql2');

// Create connection pool
const pool = mysql.createPool({
    host: 'localhost',
    user: 'user',
    password: 'password',
    database: 'mydb',
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0,
    enableKeepAlive: true,
    keepAliveInitialDelay: 0
});

// Promisify pool queries
const promisePool = pool.promise();

// Example usage
async function getUserById(id) {
    try {
        const [rows] = await promisePool.query(
            'SELECT * FROM users WHERE id = ?',
            [id]
        );
        return rows[0];
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}

// Handle pool errors
pool.on('error', (err) => {
    console.error('Unexpected error on idle client', err);
    process.exit(-1);
});
```

### PostgreSQL Connection Pool
```javascript
const { Pool } = require('pg');

// Create connection pool
const pool = new Pool({
    user: 'user',
    host: 'localhost',
    database: 'mydb',
    password: 'password',
    port: 5432,
    max: 20, // Maximum pool size
    idleTimeoutMillis: 30000, // Close idle clients after 30 seconds
    connectionTimeoutMillis: 2000, // Return an error after 2 seconds if connection could not be established
    maxUses: 7500 // Close a connection after it has been used 7500 times
});

// Example usage with automatic connection handling
async function queryDatabase() {
    const client = await pool.connect();
    try {
        const result = await client.query(
            'SELECT * FROM users WHERE active = $1',
            [true]
        );
        return result.rows;
    } finally {
        client.release();
    }
}

// Pool error handling
pool.on('error', (err, client) => {
    console.error('Unexpected error on idle client', err);
    process.exit(-1);
});
```

### MongoDB Connection Pool
```javascript
const mongoose = require('mongoose');

// Connection pool configuration
const options = {
    poolSize: 10,
    bufferMaxEntries: 0,
    useNewUrlParser: true,
    useUnifiedTopology: true,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    maxPoolSize: 50,
    minPoolSize: 10,
    maxIdleTimeMS: 10000
};

// Connect with pool
mongoose.connect('mongodb://localhost/mydb', options);

// Monitor pool events
mongoose.connection.on('connected', () => {
    console.log('Connected to MongoDB');
});

mongoose.connection.on('error', (err) => {
    console.error('MongoDB connection error:', err);
});

mongoose.connection.on('disconnected', () => {
    console.log('MongoDB disconnected');
});

// Graceful shutdown
process.on('SIGINT', async () => {
    await mongoose.connection.close();
    process.exit(0);
});
```

## HTTP Connection Pooling

### HTTP/HTTPS Agent Pool
```javascript
const https = require('https');
const http = require('http');

// Create custom agents with connection pooling
const httpsAgent = new https.Agent({
    keepAlive: true,
    keepAliveMsecs: 1000,
    maxSockets: 25,
    maxFreeSockets: 10,
    timeout: 60000
});

const httpAgent = new http.Agent({
    keepAlive: true,
    keepAliveMsecs: 1000,
    maxSockets: 25,
    maxFreeSockets: 10
});

// Using with Axios
const axios = require('axios');

const axiosInstance = axios.create({
    httpsAgent,
    httpAgent,
    timeout: 60000,
    maxRedirects: 5
});

// Example request
async function makeRequest(url) {
    try {
        const response = await axiosInstance.get(url);
        return response.data;
    } catch (error) {
        console.error('Request error:', error);
        throw error;
    }
}
```

## Redis Connection Pool

### Redis Pool Implementation
```javascript
const Redis = require('ioredis');
const GenericPool = require('generic-pool');

// Create Redis connection pool
const factory = {
    create: async () => {
        return new Redis({
            host: 'localhost',
            port: 6379,
            password: 'password',
            db: 0,
            retryStrategy: (times) => {
                const delay = Math.min(times * 50, 2000);
                return delay;
            }
        });
    },
    destroy: async (client) => {
        await client.quit();
    }
};

const poolOptions = {
    min: 2, // Minimum number of connections
    max: 10, // Maximum number of connections
    acquireTimeoutMillis: 5000,
    idleTimeoutMillis: 30000,
    evictionRunIntervalMillis: 1000,
    softIdleTimeoutMillis: 300000,
    testOnBorrow: true
};

const redisPool = GenericPool.createPool(factory, poolOptions);

// Example usage
async function cacheOperation(key, value) {
    let client;
    try {
        client = await redisPool.acquire();
        if (value === undefined) {
            return await client.get(key);
        }
        await client.set(key, value);
    } catch (error) {
        console.error('Redis operation error:', error);
        throw error;
    } finally {
        if (client) {
            await redisPool.release(client);
        }
    }
}
```

## Custom Connection Pool Implementation

### Generic Resource Pool
```javascript
class ResourcePool {
    constructor(factory, options = {}) {
        this.resources = [];
        this.inUse = new Set();
        this.factory = factory;
        this.options = {
            min: options.min || 2,
            max: options.max || 10,
            acquireTimeout: options.acquireTimeout || 5000,
            idleTimeout: options.idleTimeout || 30000,
            ...options
        };
        
        this.initialize();
    }

    async initialize() {
        for (let i = 0; i < this.options.min; i++) {
            const resource = await this.factory.create();
            this.resources.push(resource);
        }
    }

    async acquire() {
        const timeoutPromise = new Promise((_, reject) => {
            setTimeout(() => {
                reject(new Error('Acquire timeout'));
            }, this.options.acquireTimeout);
        });

        const acquirePromise = new Promise(async (resolve, reject) => {
            try {
                // Try to get an available resource
                let resource = this.resources.find(r => !this.inUse.has(r));
                
                if (!resource && this.resources.length < this.options.max) {
                    // Create new resource if pool not at max
                    resource = await this.factory.create();
                    this.resources.push(resource);
                }
                
                if (resource) {
                    this.inUse.add(resource);
                    resolve(resource);
                } else {
                    reject(new Error('No resources available'));
                }
            } catch (error) {
                reject(error);
            }
        });

        return Promise.race([acquirePromise, timeoutPromise]);
    }

    async release(resource) {
        if (this.inUse.has(resource)) {
            this.inUse.delete(resource);
            
            // If we have too many resources, destroy this one
            if (this.resources.length > this.options.min &&
                this.inUse.size < this.resources.length / 2) {
                const index = this.resources.indexOf(resource);
                if (index !== -1) {
                    this.resources.splice(index, 1);
                    await this.factory.destroy(resource);
                }
            }
        }
    }

    async drain() {
        for (const resource of this.resources) {
            await this.factory.destroy(resource);
        }
        this.resources = [];
        this.inUse.clear();
    }
}

// Example usage
const dbFactory = {
    create: async () => {
        // Create database connection
        return new DatabaseConnection();
    },
    destroy: async (connection) => {
        // Close database connection
        await connection.close();
    }
};

const pool = new ResourcePool(dbFactory, {
    min: 5,
    max: 20,
    acquireTimeout: 5000
});
```

## Monitoring and Metrics

### Pool Monitoring
```javascript
class PoolMonitor {
    constructor(pool) {
        this.pool = pool;
        this.metrics = {
            acquired: 0,
            released: 0,
            errors: 0,
            waitTime: []
        };
    }

    startMonitoring() {
        setInterval(() => {
            this.reportMetrics();
        }, 60000); // Report every minute
    }

    recordAcquisition(waitTime) {
        this.metrics.acquired++;
        this.metrics.waitTime.push(waitTime);
    }

    recordRelease() {
        this.metrics.released++;
    }

    recordError() {
        this.metrics.errors++;
    }

    reportMetrics() {
        const avgWaitTime = this.metrics.waitTime.reduce((a, b) => a + b, 0) / this.metrics.waitTime.length;
        
        console.log({
            totalAcquired: this.metrics.acquired,
            totalReleased: this.metrics.released,
            totalErrors: this.metrics.errors,
            averageWaitTime: avgWaitTime,
            currentPoolSize: this.pool.resources.length,
            inUseConnections: this.pool.inUse.size
        });
        
        // Reset metrics
        this.metrics.waitTime = [];
    }
}
```

## Best Practices

### 1. Connection Management
```javascript
// Implement connection health checks
async function checkConnection(connection) {
    try {
        await connection.query('SELECT 1');
        return true;
    } catch (error) {
        return false;
    }
}

// Implement connection refresh
async function refreshConnection(pool, connection) {
    try {
        await pool.release(connection);
        return await pool.acquire();
    } catch (error) {
        console.error('Error refreshing connection:', error);
        throw error;
    }
}
```

### 2. Error Handling
```javascript
// Implement retry mechanism
async function withRetry(operation, maxRetries = 3) {
    let lastError;
    
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await operation();
        } catch (error) {
            lastError = error;
            if (!isRetryableError(error)) {
                throw error;
            }
            await sleep(Math.pow(2, i) * 100); // Exponential backoff
        }
    }
    
    throw lastError;
}

// Usage
const result = await withRetry(async () => {
    const connection = await pool.acquire();
    try {
        return await connection.query('SELECT * FROM users');
    } finally {
        await pool.release(connection);
    }
});
```

## Related Topics
- [[Database-Integration]] - Database integration patterns
- [[Performance-Optimization]] - Performance optimization techniques
- [[Error-Handling-Advanced]] - Error handling strategies
- [[Monitoring]] - Application monitoring

## Practice Projects
1. Build a generic connection pool
2. Implement a database connection manager
3. Create a pool monitoring system
4. Develop a connection pool stress test

## Resources
- [Node.js MySQL Documentation](https://github.com/mysqljs/mysql)
- [Node.js PostgreSQL Documentation](https://node-postgres.com/)
- [[Learning-Resources#ConnectionPooling|Connection Pooling Resources]]

## Tags
#connection-pooling #database #performance #nodejs #resource-management
