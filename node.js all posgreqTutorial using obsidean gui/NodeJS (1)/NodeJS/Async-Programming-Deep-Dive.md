# Deep Dive into Asynchronous Programming

## What is Asynchronous Programming?
Asynchronous programming is a programming paradigm that allows operations to run independently of the main program flow. Instead of executing code line by line (synchronously), async programming enables certain operations to happen in the background while the rest of the code continues to execute.

Think of it like ordering food at a restaurant:
- Synchronous: You order food and wait at the counter until it's ready (blocking)
- Asynchronous: You order food, get a buzzer, and do other things until your order is ready (non-blocking)

### Key Concepts:
1. **Non-blocking Execution**: Code doesn't halt while waiting for operations to complete
2. **Event Loop**: Manages the execution of async operations (see [[Event-Loop]])
3. **Callback Queue**: Stores completed async operations waiting to be executed
4. **Promise Queue (Microtasks)**: Higher priority queue for Promise resolutions

## Why Do We Need Async Programming?

### 1. Performance and Responsiveness
```javascript
// Synchronous (Blocking) - Bad for performance
const fs = require('fs');
const data = fs.readFileSync('large-file.txt'); // Blocks until file is read
console.log('File read');
processUserRequest(); // Has to wait for file read

// Asynchronous (Non-blocking) - Better performance
fs.readFile('large-file.txt', (err, data) => {
    console.log('File read');
});
processUserRequest(); // Executes immediately
```

### 2. Handling Multiple Operations
```javascript
// Real-world example: E-commerce product page
async function getProductPage(productId) {
    try {
        // Fetch multiple pieces of data concurrently
        const [
            productDetails,
            userReviews,
            relatedProducts,
            inventoryStatus
        ] = await Promise.all([
            fetchProductDetails(productId),
            fetchUserReviews(productId),
            fetchRelatedProducts(productId),
            checkInventory(productId)
        ]);

        return {
            productDetails,
            userReviews,
            relatedProducts,
            inventoryStatus
        };
    } catch (error) {
        handleError(error);
    }
}
```

## When to Use Async vs Sync

### Use Async When:
1. **I/O Operations**
```javascript
// Database operations
async function getUserProfile(userId) {
    const user = await db.users.findOne({ id: userId });
    const posts = await db.posts.find({ userId });
    return { user, posts };
}

// API calls
async function fetchWeatherData(city) {
    const response = await fetch(`https://api.weather.com/${city}`);
    return response.json();
}
```

2. **Long-running Computations**
```javascript
// Image processing
async function processImage(imageData) {
    return new Promise((resolve) => {
        // Simulate heavy processing
        setTimeout(() => {
            const processed = heavyImageProcessing(imageData);
            resolve(processed);
        }, 0);
    });
}
```

3. **Multiple Independent Operations**
```javascript
// Real-world example: Social media feed
async function loadFeed() {
    const [posts, friends, notifications] = await Promise.all([
        fetchPosts(),
        fetchFriendsList(),
        fetchNotifications()
    ]);
    return { posts, friends, notifications };
}
```

### Use Sync When:
1. **Simple Calculations**
```javascript
// Use sync for simple operations
function calculateTotal(items) {
    return items.reduce((sum, item) => sum + item.price, 0);
}
```

2. **Small Data Operations**
```javascript
// Configuration parsing
const config = JSON.parse(fs.readFileSync('config.json'));
```

3. **Initialization Code**
```javascript
// App startup configuration
const initialize = () => {
    const config = loadConfig();
    validateConfig(config);
    setupLogger(config.logging);
    return config;
};
```

## When to Avoid Async

### 1. Simple Operations
```javascript
// Bad - Unnecessary async
async function add(a, b) {
    return a + b;
}

// Good - Keep it sync
function add(a, b) {
    return a + b;
}
```

### 2. Sequential Dependencies
```javascript
// Bad - Using async when operations must be sequential
async function processUser(userId) {
    const user = await db.users.findOne({ id: userId });
    const validated = await validateUser(user); // Depends on user
    const enriched = await enrichUserData(validated); // Depends on validated
    return enriched;
}

// Better - Use regular promises or sync code if possible
function processUser(userId) {
    return db.users.findOne({ id: userId })
        .then(validateUser)
        .then(enrichUserData);
}
```

## Real-World Scenarios

### 1. Chat Application
```javascript
class ChatRoom {
    async initialize() {
        // Connect to WebSocket
        this.socket = await this.connectWebSocket();
        
        // Set up real-time listeners
        this.socket.on('message', async (msg) => {
            await this.saveToHistory(msg);
            await this.updateUI(msg);
            
            // Optional: Process message for notifications
            if (msg.needsNotification) {
                // Don't await - run in background
                this.sendNotification(msg).catch(console.error);
            }
        });
    }

    async sendMessage(content) {
        try {
            // Optimistic UI update
            this.updateUI({ content, status: 'sending' });

            // Send message
            await this.socket.emit('message', content);
            
            // Update UI with success
            this.updateUI({ content, status: 'sent' });
        } catch (error) {
            this.updateUI({ content, status: 'failed' });
            throw error;
        }
    }
}
```

### 2. E-commerce Checkout
```javascript
class CheckoutProcess {
    async processOrder(cart, user) {
        try {
            // 1. Validate stock (must be done first)
            const stockValidation = await this.validateStock(cart.items);
            if (!stockValidation.success) {
                throw new Error('Items out of stock');
            }

            // 2. Process payment and reserve stock concurrently
            const [paymentResult, stockReservation] = await Promise.all([
                this.processPayment(cart.total, user.paymentDetails),
                this.reserveStock(cart.items)
            ]);

            // 3. If both successful, finalize order
            if (paymentResult.success && stockReservation.success) {
                const order = await this.createOrder({
                    items: cart.items,
                    payment: paymentResult,
                    user: user
                });

                // 4. Trigger non-blocking background tasks
                Promise.all([
                    this.sendOrderConfirmation(order, user.email),
                    this.updateInventory(cart.items),
                    this.notifyWarehouse(order)
                ]).catch(console.error); // Handle errors but don't block

                return order;
            }
        } catch (error) {
            // Handle failures and rollback if necessary
            await this.rollbackTransaction(error);
            throw error;
        }
    }
}
```

## Best Practices

1. **Error Handling**
```javascript
async function robustFunction() {
    try {
        const result = await riskyOperation();
        return result;
    } catch (error) {
        // Log error
        logger.error(error);
        
        // Retry logic
        if (error.isRetryable && this.retries < MAX_RETRIES) {
            return await this.retry();
        }
        
        // Fallback
        return await this.fallbackOperation();
    } finally {
        // Cleanup
        await this.cleanup();
    }
}
```

2. **Concurrency Control**
```javascript
class RateLimiter {
    async processItems(items, concurrency = 3) {
        const results = [];
        for (let i = 0; i < items.length; i += concurrency) {
            const batch = items.slice(i, i + concurrency);
            const batchResults = await Promise.all(
                batch.map(item => this.processItem(item))
            );
            results.push(...batchResults);
        }
        return results;
    }
}
```

3. **Cancellation**
```javascript
class SearchComponent {
    async search(query) {
        // Cancel previous search if exists
        if (this.currentSearch) {
            this.currentSearch.cancel();
        }

        // Create new cancellable search
        this.currentSearch = new CancellableSearch();
        
        try {
            const results = await this.currentSearch.execute(query);
            this.displayResults(results);
        } catch (error) {
            if (error.isCancelled) {
                // Ignore cancelled search errors
                return;
            }
            throw error;
        }
    }
}
```

Tags: #nodejs #async #promises #javascript #real-world-examples

### Advanced Database Operations Example:
```javascript
class DatabaseService {
    async performComplexQuery(userId) {
        // Start with parallel queries
        const [userDetails, userPreferences] = await Promise.all([
            this.db.users.findOne({ id: userId }),
            this.db.preferences.find({ userId })
        ]);

        // Sequential operations that depend on previous results
        const enrichedUser = await this.enrichUserData(userDetails);
        
        // Parallel operations with enriched data
        const [
            recommendations,
            notifications,
            analytics
        ] = await Promise.all([
            this.generateRecommendations(enrichedUser),
            this.fetchNotifications(enrichedUser),
            this.trackUserActivity(enrichedUser)
        ]);

        return {
            user: enrichedUser,
            preferences: userPreferences,
            recommendations,
            notifications,
            analytics
        };
    }

    async enrichUserData(user) {
        // Simulate complex data enrichment
        await this.simulateDelay(100);
        return {
            ...user,
            lastActive: new Date(),
            status: 'active'
        };
    }
}
```

### E-commerce Checkout with Robust Error Handling and Transaction Management
```javascript
class CheckoutProcess {
    constructor() {
        this.paymentService = new PaymentService();
        this.inventoryService = new InventoryService();
        this.notificationService = new NotificationService();
        this.transactionManager = new TransactionManager();
    }

    async processOrder(cart, user) {
        // Start a transaction session
        const session = await this.transactionManager.startSession();

        try {
            // 1. Pre-checkout validation
            await this.validateCheckoutEligibility(cart, user);

            // 2. Stock validation with timeout
            const stockValidation = await Promise.race([
                this.validateStock(cart.items),
                this.timeout(5000, 'Stock validation timeout')
            ]);

            if (!stockValidation.success) {
                throw new StockValidationError(stockValidation.message);
            }

            // 3. Process payment and reserve stock concurrently
            const [paymentResult, stockReservation] = await Promise.all([
                this.processPaymentWithRetry(cart.total, user.paymentDetails),
                this.reserveStockWithLock(cart.items, session)
            ]);

            // 4. Order creation and post-processing
            const order = await this.createOrderInTransaction(
                cart, 
                paymentResult, 
                user, 
                session
            );

            // 5. Commit transaction
            await session.commitTransaction();

            // 6. Post-order processing (background tasks)
            this.processPostOrderTasks(order, user);

            return {
                success: true,
                orderId: order.id,
                status: 'completed'
            };

        } catch (error) {
            // Rollback transaction if needed
            if (session.isActive()) {
                await session.abortTransaction();
            }

            // Handle specific error types
            if (error instanceof PaymentError) {
                return this.handlePaymentError(error);
            } else if (error instanceof StockError) {
                return this.handleStockError(error);
            }

            // Generic error handling
            return {
                success: false,
                error: error.message
            };
        }
    }

    async processPaymentWithRetry(amount, paymentDetails, maxRetries = 3) {
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                return await this.paymentService.processPayment(amount, paymentDetails);
            } catch (error) {
                if (attempt === maxRetries) throw error;
                await this.delay(attempt * 1000); // Exponential backoff
            }
        }
    }

    async processPostOrderTasks(order, user) {
        Promise.allSettled([
            this.notificationService.sendOrderConfirmation(order, user.email),
            this.inventoryService.updateInventory(order.items),
            this.notificationService.notifyWarehouse(order),
            this.analyticsService.trackPurchase(order)
        ]).catch(error => {
            // Log errors but don't fail the order
            console.error('Post-order task failed:', error);
            this.errorTrackingService.capture(error);
        });
    }

    // Utility methods
    timeout(ms, message) {
        return new Promise((_, reject) => {
            setTimeout(() => reject(new Error(message)), ms);
        });
    }

    delay(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Usage example
const checkout = new CheckoutProcess();
try {
    const result = await checkout.processOrder(cart, user);
    if (result.success) {
        console.log(`Order ${result.orderId} processed successfully`);
    } else {
        console.error('Order processing failed:', result.error);
    }
} catch (error) {
    console.error('Unexpected error during checkout:', error);
}
```

### Advanced Error Handling with Retries and Circuit Breaker
```javascript
class CircuitBreaker {
    constructor(fn, options = {}) {
        this.fn = fn;
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 60000;
        this.failures = 0;
        this.lastFailureTime = null;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF-OPEN
    }

    async execute(...args) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
                this.state = 'HALF-OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }

        try {
            const result = await this.fn(...args);
            if (this.state === 'HALF-OPEN') {
                this.reset();
            }
            return result;
        } catch (error) {
            this.failures++;
            this.lastFailureTime = Date.now();

            if (this.failures >= this.failureThreshold) {
                this.state = 'OPEN';
            }
            throw error;
        }
    }

    reset() {
        this.failures = 0;
        this.state = 'CLOSED';
    }
}

// Usage example
const apiCall = new CircuitBreaker(
    async (url) => {
        const response = await fetch(url);
        if (!response.ok) throw new Error('API call failed');
        return response.json();
    },
    { failureThreshold: 3, resetTimeout: 30000 }
);
```

### Queue Processing with Rate Limiting
```javascript
class AsyncQueue {
    constructor(concurrency = 3) {
        this.concurrency = concurrency;
        this.queue = [];
        this.running = 0;
    }

    async add(task) {
        return new Promise((resolve, reject) => {
            this.queue.push({ task, resolve, reject });
            this.processNext();
        });
    }

    async processNext() {
        if (this.running >= this.concurrency || this.queue.length === 0) {
            return;
        }

        this.running++;
        const { task, resolve, reject } = this.queue.shift();

        try {
            const result = await task();
            resolve(result);
        } catch (error) {
            reject(error);
        } finally {
            this.running--;
            this.processNext();
        }
    }
}

// Usage example
const queue = new AsyncQueue(2);
const tasks = [
    () => fetch('api/endpoint1'),
    () => fetch('api/endpoint2'),
    () => fetch('api/endpoint3')
];

tasks.forEach(task => queue.add(task));
```

### Async Iterator Pattern
```javascript
class AsyncDataSource {
    constructor(data, batchSize = 100) {
        this.data = data;
        this.batchSize = batchSize;
    }

    async *[Symbol.asyncIterator]() {
        for (let i = 0; i < this.data.length; i += this.batchSize) {
            const batch = this.data.slice(i, i + this.batchSize);
            yield await this.processBatch(batch);
        }
    }

    async processBatch(batch) {
        // Simulate processing delay
        await new Promise(resolve => setTimeout(resolve, 100));
        return batch.map(item => ({ ...item, processed: true }));
    }
}

// Usage example
async function processLargeDataset() {
    const source = new AsyncDataSource(Array.from({ length: 1000 }));
    
    for await (const batch of source) {
        console.log('Processed batch:', batch.length);
    }
}
```

## Memory and Performance Considerations

### 1. Memory Management
```javascript
class MemoryEfficientProcessor {
    async processLargeFile(filePath) {
        const fileStream = fs.createReadStream(filePath);
        const rl = readline.createInterface({
            input: fileStream,
            crlfDelay: Infinity
        });

        let batch = [];
        const batchSize = 1000;

        for await (const line of rl) {
            batch.push(line);
            
            if (batch.length >= batchSize) {
                await this.processBatch(batch);
                batch = []; // Free memory
            }
        }

        if (batch.length > 0) {
            await this.processBatch(batch);
        }
    }

    async processBatch(batch) {
        // Process batch of lines
        const results = await Promise.all(
            batch.map(line => this.processLine(line))
        );
        
        // Stream results to output
        await this.streamResults(results);
    }
}
```

### 2. Performance Optimization
```javascript
class OptimizedDataProcessor {
    constructor() {
        this.cache = new Map();
        this.pendingPromises = new Map();
    }

    async getData(key) {
        // Check cache first
        if (this.cache.has(key)) {
            return this.cache.get(key);
        }

        // Check if request is already pending
        if (this.pendingPromises.has(key)) {
            return this.pendingPromises.get(key);
        }

        // Create new request
        const promise = this.fetchData(key)
            .then(data => {
                this.cache.set(key, data);
                this.pendingPromises.delete(key);
                return data;
            })
            .catch(error => {
                this.pendingPromises.delete(key);
                throw error;
            });

        this.pendingPromises.set(key, promise);
        return promise;
    }

    async fetchData(key) {
        // Simulate API call
        await this.delay(100);
        return { key, data: 'some data' };
    }
}
```

Tags: #nodejs #async #promises #javascript #real-world-examples #advanced-patterns #performance
