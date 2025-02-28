# Advanced Database Optimization in Node.js

A deep dive into advanced database optimization techniques, focusing on practical implementations for both SQL and NoSQL databases.

## 1. Advanced Query Optimization

### Smart Query Builder
```javascript
class SmartQueryBuilder {
    constructor(model) {
        this.model = model;
        this.queryAnalyzer = new QueryAnalyzer();
        this.indexOptimizer = new IndexOptimizer(model);
        this.queryCache = new QueryCache();
    }

    async buildOptimizedQuery(params) {
        const { filters, sort, pagination, select } = this.parseParams(params);
        
        // Build base query
        let query = this.model.find(filters);

        // Analyze and optimize
        const analysis = await this.queryAnalyzer.analyzeQuery(query);
        const suggestions = await this.indexOptimizer.suggestIndexes(filters);

        // Apply optimizations
        if (suggestions) {
            query = this.applyIndexOptimizations(query, suggestions);
        }

        // Apply sorting and pagination
        query = this.applySortAndPagination(query, sort, pagination);

        // Apply field selection
        if (select) {
            query = query.select(select);
        }

        return query;
    }

    parseParams(params) {
        return {
            filters: this.buildFilters(params.filters),
            sort: params.sort || { createdAt: -1 },
            pagination: {
                page: parseInt(params.page) || 1,
                limit: parseInt(params.limit) || 10
            },
            select: params.select
        };
    }

    buildFilters(filters) {
        const parsedFilters = {};

        for (const [key, value] of Object.entries(filters)) {
            if (typeof value === 'object') {
                parsedFilters[key] = this.parseOperator(value);
            } else {
                parsedFilters[key] = value;
            }
        }

        return parsedFilters;
    }

    parseOperator(value) {
        const operators = {
            eq: '$eq',
            ne: '$ne',
            gt: '$gt',
            gte: '$gte',
            lt: '$lt',
            lte: '$lte',
            in: '$in',
            nin: '$nin',
            regex: '$regex'
        };

        const parsed = {};
        for (const [op, val] of Object.entries(value)) {
            if (operators[op]) {
                parsed[operators[op]] = val;
            }
        }

        return parsed;
    }

    applySortAndPagination(query, sort, pagination) {
        const { page, limit } = pagination;
        const skip = (page - 1) * limit;

        return query
            .sort(sort)
            .skip(skip)
            .limit(limit);
    }

    applyIndexOptimizations(query, suggestions) {
        if (suggestions.hint) {
            query = query.hint(suggestions.hint);
        }
        return query;
    }

    async execute(params) {
        const query = await this.buildOptimizedQuery(params);
        const cacheKey = this.queryCache.generateKey(params);

        return this.queryCache.wrap(cacheKey, 3600, async () => {
            const [data, total] = await Promise.all([
                query.exec(),
                this.model.countDocuments(query.getQuery())
            ]);

            return {
                data,
                total,
                page: params.page || 1,
                limit: params.limit || 10
            };
        });
    }
}
```

### Batch Processing Optimizer
```javascript
class BatchProcessor {
    constructor(options = {}) {
        this.batchSize = options.batchSize || 1000;
        this.concurrency = options.concurrency || 5;
        this.timeout = options.timeout || 30000;
    }

    async processBatch(items, processFn) {
        const batches = this.createBatches(items);
        const results = [];

        for (let i = 0; i < batches.length; i += this.concurrency) {
            const batchPromises = batches
                .slice(i, i + this.concurrency)
                .map(batch => this.processWithTimeout(batch, processFn));

            const batchResults = await Promise.all(batchPromises);
            results.push(...batchResults.flat());
        }

        return results;
    }

    createBatches(items) {
        const batches = [];
        for (let i = 0; i < items.length; i += this.batchSize) {
            batches.push(items.slice(i, i + this.batchSize));
        }
        return batches;
    }

    async processWithTimeout(batch, processFn) {
        return Promise.race([
            processFn(batch),
            new Promise((_, reject) => 
                setTimeout(() => reject(new Error('Batch timeout')), this.timeout)
            )
        ]);
    }
}

// Usage example
const batchProcessor = new BatchProcessor({
    batchSize: 1000,
    concurrency: 5,
    timeout: 30000
});

const items = await Model.find().lean();
const results = await batchProcessor.processBatch(items, async batch => {
    return Model.updateMany(
        { _id: { $in: batch.map(item => item._id) } },
        { $set: { processed: true } }
    );
});
```

## 2. Advanced Indexing Strategies

### Dynamic Index Manager
```javascript
class DynamicIndexManager {
    constructor(model) {
        this.model = model;
        this.indexStats = new Map();
        this.indexUsageThreshold = 0.1; // 10% usage threshold
    }

    async analyzeIndexUsage() {
        const stats = await this.model.collection.aggregate([
            { $indexStats: {} }
        ]).toArray();

        stats.forEach(stat => {
            this.indexStats.set(stat.name, {
                ops: stat.accesses.ops,
                since: stat.accesses.since
            });
        });

        return this.getIndexRecommendations();
    }

    async getIndexRecommendations() {
        const recommendations = [];
        const totalOps = Array.from(this.indexStats.values())
            .reduce((sum, stat) => sum + stat.ops, 0);

        for (const [name, stats] of this.indexStats) {
            const usage = stats.ops / totalOps;

            if (usage < this.indexUsageThreshold) {
                recommendations.push({
                    index: name,
                    recommendation: 'drop',
                    reason: `Low usage (${(usage * 100).toFixed(2)}%)`
                });
            }
        }

        return recommendations;
    }

    async createOptimalIndexes(queryPatterns) {
        const recommendations = [];

        for (const pattern of queryPatterns) {
            const analysis = await this.analyzeQueryPattern(pattern);
            if (analysis.needsIndex) {
                recommendations.push({
                    fields: analysis.fields,
                    type: analysis.type,
                    reason: analysis.reason
                });
            }
        }

        return recommendations;
    }

    async analyzeQueryPattern(pattern) {
        const explain = await this.model.find(pattern.query)
            .explain('executionStats');

        return {
            needsIndex: explain.executionStats.totalDocsExamined > 
                       explain.executionStats.nReturned * 2,
            fields: Object.keys(pattern.query),
            type: this.determineIndexType(pattern),
            reason: 'High document scan ratio'
        };
    }

    determineIndexType(pattern) {
        if (pattern.sort) {
            return 'compound';
        }
        if (Object.keys(pattern.query).length > 1) {
            return 'compound';
        }
        return 'single';
    }
}
```

## 3. Query Performance Profiling

### Advanced Query Profiler
```javascript
class QueryProfiler {
    constructor() {
        this.profiles = new Map();
        this.slowQueryThreshold = 100; // ms
        this.setupProfiler();
    }

    setupProfiler() {
        if (process.env.NODE_ENV === 'development') {
            mongoose.set('debug', (collection, method, query, doc) => {
                const start = process.hrtime();
                const queryId = this.startProfiling(collection, method, query);

                return () => {
                    this.endProfiling(queryId, start);
                };
            });
        }
    }

    startProfiling(collection, method, query) {
        const queryId = Date.now() + Math.random();
        this.profiles.set(queryId, {
            collection,
            method,
            query,
            stack: new Error().stack,
            start: process.hrtime()
        });
        return queryId;
    }

    endProfiling(queryId, start) {
        const profile = this.profiles.get(queryId);
        if (!profile) return;

        const [seconds, nanoseconds] = process.hrtime(start);
        const duration = seconds * 1000 + nanoseconds / 1000000;

        profile.duration = duration;
        profile.end = new Date();

        if (duration > this.slowQueryThreshold) {
            this.handleSlowQuery(profile);
        }

        this.profiles.delete(queryId);
    }

    handleSlowQuery(profile) {
        console.warn(`Slow Query Detected:
            Collection: ${profile.collection}
            Method: ${profile.method}
            Query: ${JSON.stringify(profile.query)}
            Duration: ${profile.duration}ms
            Stack: ${profile.stack}
        `);

        // Store slow query for analysis
        this.storeSlowQuery(profile);
    }

    async storeSlowQuery(profile) {
        try {
            await SlowQuery.create({
                collection: profile.collection,
                method: profile.method,
                query: profile.query,
                duration: profile.duration,
                stack: profile.stack,
                timestamp: new Date()
            });
        } catch (error) {
            console.error('Failed to store slow query:', error);
        }
    }

    getQueryStats() {
        const stats = {
            total: 0,
            slow: 0,
            avgDuration: 0,
            collections: new Map()
        };

        for (const profile of this.profiles.values()) {
            stats.total++;
            if (profile.duration > this.slowQueryThreshold) {
                stats.slow++;
            }
            stats.avgDuration += profile.duration;

            const collStats = stats.collections.get(profile.collection) || {
                queries: 0,
                totalDuration: 0
            };
            collStats.queries++;
            collStats.totalDuration += profile.duration;
            stats.collections.set(profile.collection, collStats);
        }

        stats.avgDuration /= stats.total;
        return stats;
    }
}
```

## 4. Connection and Pool Optimization

### Advanced Connection Pool
```javascript
class AdvancedConnectionPool {
    constructor(options = {}) {
        this.options = {
            minSize: options.minSize || 5,
            maxSize: options.maxSize || 20,
            acquireTimeout: options.acquireTimeout || 10000,
            idleTimeout: options.idleTimeout || 30000,
            maxWaitingClients: options.maxWaitingClients || 50,
            ...options
        };

        this.pool = [];
        this.active = new Set();
        this.waiting = [];
        this.metrics = {
            created: 0,
            acquired: 0,
            released: 0,
            destroyed: 0,
            timeouts: 0
        };
    }

    async initialize() {
        for (let i = 0; i < this.options.minSize; i++) {
            const connection = await this.createConnection();
            this.pool.push(connection);
            this.metrics.created++;
        }

        this.startHealthCheck();
    }

    async acquire() {
        if (this.waiting.length >= this.options.maxWaitingClients) {
            throw new Error('Too many waiting clients');
        }

        if (this.pool.length > 0) {
            const connection = this.pool.pop();
            this.active.add(connection);
            this.metrics.acquired++;
            return connection;
        }

        if (this.active.size < this.options.maxSize) {
            const connection = await this.createConnection();
            this.active.add(connection);
            this.metrics.created++;
            this.metrics.acquired++;
            return connection;
        }

        return new Promise((resolve, reject) => {
            const timeout = setTimeout(() => {
                const index = this.waiting.findIndex(w => w.timeout === timeout);
                if (index !== -1) {
                    this.waiting.splice(index, 1);
                }
                this.metrics.timeouts++;
                reject(new Error('Connection acquisition timeout'));
            }, this.options.acquireTimeout);

            this.waiting.push({ resolve, reject, timeout });
        });
    }

    async release(connection) {
        if (!this.active.has(connection)) {
            return;
        }

        this.active.delete(connection);
        this.metrics.released++;

        if (this.waiting.length > 0) {
            const { resolve, timeout } = this.waiting.shift();
            clearTimeout(timeout);
            this.active.add(connection);
            this.metrics.acquired++;
            resolve(connection);
        } else if (this.pool.length < this.options.minSize) {
            this.pool.push(connection);
        } else {
            await this.destroyConnection(connection);
            this.metrics.destroyed++;
        }
    }

    async healthCheck() {
        const now = Date.now();

        // Check idle connections
        for (const connection of this.pool) {
            if (now - connection.lastUsed > this.options.idleTimeout) {
                this.pool = this.pool.filter(c => c !== connection);
                await this.destroyConnection(connection);
                this.metrics.destroyed++;
            }
        }

        // Ensure minimum pool size
        while (this.pool.length + this.active.size < this.options.minSize) {
            const connection = await this.createConnection();
            this.pool.push(connection);
            this.metrics.created++;
        }
    }

    startHealthCheck() {
        setInterval(() => {
            this.healthCheck().catch(console.error);
        }, 30000); // Run every 30 seconds
    }

    getMetrics() {
        return {
            ...this.metrics,
            poolSize: this.pool.length,
            activeConnections: this.active.size,
            waitingClients: this.waiting.length
        };
    }

    async createConnection() {
        // Implement actual connection creation logic
        return {
            id: Date.now() + Math.random(),
            lastUsed: Date.now()
        };
    }

    async destroyConnection(connection) {
        // Implement actual connection destruction logic
    }
}
```

## Related Topics
- [[Query-Optimization]] - Advanced query optimization techniques
- [[Index-Management]] - Index management strategies
- [[Connection-Pooling]] - Connection pool patterns
- [[Performance-Monitoring]] - Database performance monitoring

## Practice Projects
1. Build an advanced query optimization tool
2. Create a dynamic index management system
3. Implement a sophisticated connection pool
4. Develop a comprehensive query profiler

## Resources
- [MongoDB Performance Best Practices](https://docs.mongodb.com/manual/core/query-optimization/)
- [MySQL Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [[Learning-Resources#Database|Database Resources]]

## Tags
#database #optimization #performance #nodejs #advanced
