# Database Design Patterns and Query Optimization in Node.js

A comprehensive guide to database design patterns and query optimization techniques for both SQL and NoSQL databases in Node.js applications.

## 1. Database Design Patterns

### Repository Pattern
```javascript
class BaseRepository {
    constructor(model) {
        this.model = model;
    }

    async create(data) {
        return this.model.create(data);
    }

    async findById(id) {
        return this.model.findById(id);
    }

    async findOne(conditions) {
        return this.model.findOne(conditions);
    }

    async find(conditions, options = {}) {
        return this.model.find(conditions)
            .skip(options.skip)
            .limit(options.limit)
            .sort(options.sort);
    }

    async update(id, data) {
        return this.model.findByIdAndUpdate(id, data, {
            new: true,
            runValidators: true
        });
    }

    async delete(id) {
        return this.model.findByIdAndDelete(id);
    }

    async count(conditions) {
        return this.model.countDocuments(conditions);
    }
}

// Specific repository implementation
class UserRepository extends BaseRepository {
    constructor(model) {
        super(model);
    }

    async findByEmail(email) {
        return this.findOne({ email });
    }

    async findActiveUsers() {
        return this.find({ status: 'active' });
    }

    async updateLastLogin(userId) {
        return this.update(userId, {
            lastLoginAt: new Date()
        });
    }
}
```

### Unit of Work Pattern
```javascript
class UnitOfWork {
    constructor(mongoose) {
        this.mongoose = mongoose;
        this.repositories = new Map();
    }

    getRepository(name) {
        if (!this.repositories.has(name)) {
            const model = this.mongoose.model(name);
            this.repositories.set(name, new BaseRepository(model));
        }
        return this.repositories.get(name);
    }

    async startTransaction() {
        this.session = await this.mongoose.startSession();
        this.session.startTransaction();
    }

    async commitTransaction() {
        await this.session.commitTransaction();
        await this.session.endSession();
    }

    async rollbackTransaction() {
        await this.session.abortTransaction();
        await this.session.endSession();
    }

    async executeTransaction(work) {
        await this.startTransaction();
        try {
            await work();
            await this.commitTransaction();
        } catch (error) {
            await this.rollbackTransaction();
            throw error;
        }
    }
}

// Usage example
const uow = new UnitOfWork(mongoose);

await uow.executeTransaction(async () => {
    const userRepo = uow.getRepository('User');
    const orderRepo = uow.getRepository('Order');

    const user = await userRepo.create({
        name: 'John Doe',
        email: 'john@example.com'
    });

    await orderRepo.create({
        userId: user._id,
        items: ['item1', 'item2']
    });
});
```

### Query Builder Pattern
```javascript
class QueryBuilder {
    constructor(model) {
        this.model = model;
        this.query = {};
        this.options = {
            select: {},
            sort: {},
            skip: 0,
            limit: 10
        };
    }

    where(conditions) {
        this.query = { ...this.query, ...conditions };
        return this;
    }

    select(fields) {
        this.options.select = fields;
        return this;
    }

    sort(sortBy) {
        this.options.sort = sortBy;
        return this;
    }

    skip(skip) {
        this.options.skip = skip;
        return this;
    }

    limit(limit) {
        this.options.limit = limit;
        return this;
    }

    async execute() {
        return this.model
            .find(this.query)
            .select(this.options.select)
            .sort(this.options.sort)
            .skip(this.options.skip)
            .limit(this.options.limit);
    }

    async count() {
        return this.model.countDocuments(this.query);
    }
}

// Usage
const query = new QueryBuilder(UserModel)
    .where({ status: 'active' })
    .select({ name: 1, email: 1 })
    .sort({ createdAt: -1 })
    .limit(20);

const users = await query.execute();
```

## 2. Query Optimization

### Query Analyzer
```javascript
class QueryAnalyzer {
    constructor(mongoose) {
        this.mongoose = mongoose;
        this.queries = new Map();
    }

    async analyzeQuery(query) {
        const explain = await query.explain('executionStats');
        return {
            executionTime: explain.executionStats.executionTimeMillis,
            docsExamined: explain.executionStats.totalDocsExamined,
            docsReturned: explain.executionStats.nReturned,
            indexesUsed: explain.queryPlanner.winningPlan.inputStage.indexName
        };
    }

    recordQuery(query, stats) {
        const key = JSON.stringify(query);
        if (!this.queries.has(key)) {
            this.queries.set(key, {
                count: 0,
                totalTime: 0,
                avgTime: 0
            });
        }

        const record = this.queries.get(key);
        record.count++;
        record.totalTime += stats.executionTime;
        record.avgTime = record.totalTime / record.count;
    }

    getSlowQueries(threshold = 100) {
        return Array.from(this.queries.entries())
            .filter(([_, stats]) => stats.avgTime > threshold)
            .map(([query, stats]) => ({
                query: JSON.parse(query),
                ...stats
            }));
    }
}
```

### Index Optimizer
```javascript
class IndexOptimizer {
    constructor(model) {
        this.model = model;
        this.indexStats = new Map();
    }

    async analyzeIndexes() {
        const stats = await this.model.collection.stats();
        const indexes = await this.model.collection.indexes();

        return {
            totalDocs: stats.count,
            indexes: indexes.map(index => ({
                name: index.name,
                fields: index.key,
                size: stats.indexSizes[index.name]
            }))
        };
    }

    async suggestIndexes(query) {
        const explain = await this.model.find(query).explain();
        const winningPlan = explain.queryPlanner.winningPlan;

        if (winningPlan.stage === 'COLLSCAN') {
            return this.suggestIndexForScan(query);
        }

        return null;
    }

    suggestIndexForScan(query) {
        const fields = Object.keys(query);
        return {
            suggestion: 'Create index',
            fields: fields,
            reason: 'Collection scan detected'
        };
    }
}
```

### Query Cache
```javascript
class QueryCache {
    constructor(redis) {
        this.redis = redis;
        this.prefix = 'query:';
        this.defaultTTL = 3600; // 1 hour
    }

    generateKey(query, options = {}) {
        return this.prefix + JSON.stringify({
            query,
            options
        });
    }

    async get(key) {
        const cached = await this.redis.get(key);
        return cached ? JSON.parse(cached) : null;
    }

    async set(key, value, ttl = this.defaultTTL) {
        await this.redis.set(
            key,
            JSON.stringify(value),
            'EX',
            ttl
        );
    }

    async wrap(key, ttl, fetchFn) {
        let result = await this.get(key);

        if (!result) {
            result = await fetchFn();
            await this.set(key, result, ttl);
        }

        return result;
    }

    async invalidate(pattern) {
        const keys = await this.redis.keys(this.prefix + pattern);
        if (keys.length > 0) {
            await this.redis.del(keys);
        }
    }
}
```

### Connection Pool Manager
```javascript
class ConnectionPoolManager {
    constructor(options = {}) {
        this.options = {
            minSize: options.minSize || 5,
            maxSize: options.maxSize || 20,
            acquireTimeout: options.acquireTimeout || 10000,
            idleTimeout: options.idleTimeout || 30000,
            ...options
        };

        this.pool = [];
        this.active = new Set();
        this.waiting = [];
    }

    async initialize() {
        for (let i = 0; i < this.options.minSize; i++) {
            const connection = await this.createConnection();
            this.pool.push(connection);
        }
    }

    async acquire() {
        if (this.pool.length > 0) {
            const connection = this.pool.pop();
            this.active.add(connection);
            return connection;
        }

        if (this.active.size < this.options.maxSize) {
            const connection = await this.createConnection();
            this.active.add(connection);
            return connection;
        }

        return new Promise((resolve, reject) => {
            const timeout = setTimeout(() => {
                reject(new Error('Connection acquisition timeout'));
            }, this.options.acquireTimeout);

            this.waiting.push({ resolve, reject, timeout });
        });
    }

    async release(connection) {
        this.active.delete(connection);

        if (this.waiting.length > 0) {
            const { resolve, timeout } = this.waiting.shift();
            clearTimeout(timeout);
            this.active.add(connection);
            resolve(connection);
        } else {
            this.pool.push(connection);
        }
    }

    async createConnection() {
        // Implement actual connection creation logic
        return {};
    }
}
```

## 3. Performance Monitoring

### Query Performance Monitor
```javascript
class QueryPerformanceMonitor {
    constructor() {
        this.queries = new Map();
        this.slowQueryThreshold = 100; // ms
    }

    startQuery(query) {
        const start = process.hrtime();
        const queryId = Date.now() + Math.random();

        this.queries.set(queryId, {
            query,
            start,
            stack: new Error().stack
        });

        return queryId;
    }

    endQuery(queryId) {
        const queryData = this.queries.get(queryId);
        if (!queryData) return;

        const [seconds, nanoseconds] = process.hrtime(queryData.start);
        const duration = seconds * 1000 + nanoseconds / 1000000;

        if (duration > this.slowQueryThreshold) {
            this.logSlowQuery(queryData.query, duration, queryData.stack);
        }

        this.queries.delete(queryId);
        return duration;
    }

    logSlowQuery(query, duration, stack) {
        console.warn(`Slow Query Detected:
            Query: ${JSON.stringify(query)}
            Duration: ${duration}ms
            Stack: ${stack}
        `);
    }

    getQueryStats() {
        return Array.from(this.queries.values()).map(data => ({
            query: data.query,
            duration: process.hrtime(data.start)
        }));
    }
}
```

## Related Topics
- [[Database-Optimization]] - Advanced optimization techniques
- [[Caching-Strategies]] - Database caching patterns
- [[Connection-Pooling]] - Connection pool management
- [[Query-Performance]] - Query performance tuning

## Practice Projects
1. Build a query optimization tool
2. Create a database monitoring dashboard
3. Implement a smart caching system
4. Develop an index suggestion tool

## Resources
- [MongoDB Performance Best Practices](https://docs.mongodb.com/manual/core/query-optimization/)
- [MySQL Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [[Learning-Resources#Database|Database Resources]]

## Tags
#database #optimization #patterns #performance #nodejs
