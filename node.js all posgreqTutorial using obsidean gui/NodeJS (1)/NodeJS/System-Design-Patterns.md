# System Design Patterns in Node.js

A comprehensive guide to implementing common system design patterns in Node.js applications.

## 1. Layered Architecture Pattern

### Overview
The Layered Architecture pattern organizes code into distinct layers, each with specific responsibilities.

```javascript
// 1. Presentation Layer (Routes)
class UserRoutes {
    constructor(userController) {
        this.controller = userController;
    }

    setupRoutes(app) {
        app.post('/users', (req, res) => this.controller.createUser(req, res));
        app.get('/users/:id', (req, res) => this.controller.getUser(req, res));
    }
}

// 2. Business Layer (Controllers)
class UserController {
    constructor(userService) {
        this.userService = userService;
    }

    async createUser(req, res) {
        try {
            const user = await this.userService.createUser(req.body);
            res.status(201).json(user);
        } catch (error) {
            res.status(400).json({ error: error.message });
        }
    }

    async getUser(req, res) {
        try {
            const user = await this.userService.getUser(req.params.id);
            res.json(user);
        } catch (error) {
            res.status(404).json({ error: error.message });
        }
    }
}

// 3. Service Layer
class UserService {
    constructor(userRepository) {
        this.userRepository = userRepository;
    }

    async createUser(userData) {
        // Business logic validation
        if (!userData.email || !userData.password) {
            throw new Error('Email and password are required');
        }

        // Hash password
        userData.password = await bcrypt.hash(userData.password, 10);
        
        return this.userRepository.create(userData);
    }

    async getUser(id) {
        const user = await this.userRepository.findById(id);
        if (!user) {
            throw new Error('User not found');
        }
        return user;
    }
}

// 4. Data Access Layer (Repository)
class UserRepository {
    constructor(database) {
        this.db = database;
    }

    async create(userData) {
        return this.db.users.create(userData);
    }

    async findById(id) {
        return this.db.users.findOne({ where: { id } });
    }
}

// Usage
const setupApplication = (app, database) => {
    const userRepository = new UserRepository(database);
    const userService = new UserService(userRepository);
    const userController = new UserController(userService);
    const userRoutes = new UserRoutes(userController);
    
    userRoutes.setupRoutes(app);
};
```

### Benefits
- Clear separation of concerns
- Improved maintainability
- Easier testing
- Better code organization

### Best Practices
1. Keep layers loosely coupled
2. Implement dependency injection
3. Use interfaces between layers
4. Handle errors at appropriate layers
5. Maintain unidirectional data flow

## 2. Event-Driven Architecture Pattern

### Overview
Event-Driven Architecture (EDA) is a pattern where components communicate through events.

```javascript
// Event Emitter Implementation
class EventBus {
    constructor() {
        this.events = new Map();
    }

    subscribe(eventName, callback) {
        if (!this.events.has(eventName)) {
            this.events.set(eventName, []);
        }
        this.events.get(eventName).push(callback);
        
        return () => this.unsubscribe(eventName, callback);
    }

    unsubscribe(eventName, callback) {
        if (!this.events.has(eventName)) return;
        
        const callbacks = this.events.get(eventName);
        const index = callbacks.indexOf(callback);
        if (index !== -1) {
            callbacks.splice(index, 1);
        }
    }

    publish(eventName, data) {
        if (!this.events.has(eventName)) return;
        
        this.events.get(eventName).forEach(callback => {
            try {
                callback(data);
            } catch (error) {
                console.error(`Error in event handler for ${eventName}:`, error);
            }
        });
    }
}

// Order Processing Example
class OrderService {
    constructor(eventBus) {
        this.eventBus = eventBus;
    }

    async createOrder(orderData) {
        // Create order
        const order = await this.saveOrder(orderData);
        
        // Publish event
        this.eventBus.publish('orderCreated', order);
        
        return order;
    }
}

class InventoryService {
    constructor(eventBus) {
        this.eventBus = eventBus;
        this.setupSubscriptions();
    }

    setupSubscriptions() {
        this.eventBus.subscribe('orderCreated', this.handleOrderCreated.bind(this));
    }

    async handleOrderCreated(order) {
        // Update inventory
        await this.updateInventory(order.items);
        
        // Publish inventory updated event
        this.eventBus.publish('inventoryUpdated', {
            orderId: order.id,
            status: 'updated'
        });
    }
}

class NotificationService {
    constructor(eventBus) {
        this.eventBus = eventBus;
        this.setupSubscriptions();
    }

    setupSubscriptions() {
        this.eventBus.subscribe('orderCreated', this.sendOrderConfirmation.bind(this));
        this.eventBus.subscribe('inventoryUpdated', this.sendInventoryUpdate.bind(this));
    }

    async sendOrderConfirmation(order) {
        // Send confirmation email
        await this.sendEmail(order.customerEmail, 'Order Confirmation', {
            orderId: order.id,
            items: order.items
        });
    }

    async sendInventoryUpdate(update) {
        // Send inventory update notification
        await this.sendAdminNotification('Inventory Updated', update);
    }
}

// Usage
const eventBus = new EventBus();
const orderService = new OrderService(eventBus);
const inventoryService = new InventoryService(eventBus);
const notificationService = new NotificationService(eventBus);
```

### Benefits
- Loose coupling between components
- Improved scalability
- Better fault isolation
- Asynchronous processing
- Easy to extend functionality

### Best Practices
1. Use meaningful event names
2. Handle event failures gracefully
3. Implement event versioning
4. Consider event persistence
5. Monitor event flow

## 3. Microservices Pattern

### Overview
The Microservices pattern structures an application as a collection of loosely coupled, independently deployable services.

```javascript
// API Gateway Service
class APIGateway {
    constructor() {
        this.services = new Map();
        this.setupRoutes();
    }

    registerService(name, url) {
        this.services.set(name, url);
    }

    setupRoutes() {
        this.app = express();
        this.app.use(express.json());
        
        // Authentication middleware
        this.app.use(this.authenticate);
        
        // Route handlers
        this.app.use('/users', this.handleUserRequests.bind(this));
        this.app.use('/orders', this.handleOrderRequests.bind(this));
    }

    async handleUserRequests(req, res) {
        try {
            const response = await this.forwardRequest('user-service', req);
            res.json(response.data);
        } catch (error) {
            res.status(500).json({ error: error.message });
        }
    }

    async handleOrderRequests(req, res) {
        try {
            const response = await this.forwardRequest('order-service', req);
            res.json(response.data);
        } catch (error) {
            res.status(500).json({ error: error.message });
        }
    }

    async forwardRequest(serviceName, req) {
        const serviceUrl = this.services.get(serviceName);
        if (!serviceUrl) {
            throw new Error(`Service ${serviceName} not found`);
        }

        return axios({
            method: req.method,
            url: `${serviceUrl}${req.path}`,
            data: req.body,
            headers: this.filterHeaders(req.headers)
        });
    }
}

// Service Discovery
class ServiceRegistry {
    constructor() {
        this.services = new Map();
        this.healthChecks = new Map();
    }

    register(name, instance) {
        if (!this.services.has(name)) {
            this.services.set(name, new Set());
        }
        this.services.get(name).add(instance);
        this.setupHealthCheck(name, instance);
    }

    deregister(name, instance) {
        if (this.services.has(name)) {
            this.services.get(name).delete(instance);
        }
        this.removeHealthCheck(name, instance);
    }

    getInstance(name) {
        if (!this.services.has(name)) {
            throw new Error(`Service ${name} not found`);
        }
        
        const instances = Array.from(this.services.get(name));
        return instances[Math.floor(Math.random() * instances.length)];
    }

    setupHealthCheck(name, instance) {
        const check = setInterval(async () => {
            try {
                await axios.get(`${instance}/health`);
            } catch (error) {
                this.deregister(name, instance);
            }
        }, 30000);
        
        this.healthChecks.set(`${name}-${instance}`, check);
    }

    removeHealthCheck(name, instance) {
        const key = `${name}-${instance}`;
        if (this.healthChecks.has(key)) {
            clearInterval(this.healthChecks.get(key));
            this.healthChecks.delete(key);
        }
    }
}

// Circuit Breaker
class CircuitBreaker {
    constructor(service, options = {}) {
        this.service = service;
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 60000;
        this.state = 'CLOSED';
        this.failures = 0;
        this.lastFailureTime = null;
    }

    async execute(request) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
                this.state = 'HALF-OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }

        try {
            const response = await this.service.execute(request);
            this.onSuccess();
            return response;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    onSuccess() {
        this.failures = 0;
        this.state = 'CLOSED';
    }

    onFailure() {
        this.failures++;
        this.lastFailureTime = Date.now();

        if (this.failures >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}
```

### Benefits
- Independent deployment
- Technology diversity
- Fault isolation
- Scalability
- Team autonomy

### Best Practices
1. Design around business capabilities
2. Implement service discovery
3. Use circuit breakers
4. Maintain API versioning
5. Implement proper monitoring
6. Handle distributed transactions

## 4. CQRS (Command Query Responsibility Segregation) Pattern

### Overview
CQRS separates read and write operations into different models.

```javascript
// Command (Write) Models
class CreateUserCommand {
    constructor(userData) {
        this.email = userData.email;
        this.password = userData.password;
        this.name = userData.name;
    }

    validate() {
        if (!this.email || !this.password || !this.name) {
            throw new Error('Missing required fields');
        }
    }
}

class UpdateUserCommand {
    constructor(userId, updates) {
        this.userId = userId;
        this.updates = updates;
    }

    validate() {
        if (!this.userId) {
            throw new Error('User ID is required');
        }
    }
}

// Query (Read) Models
class UserDetailsQuery {
    constructor(userId) {
        this.userId = userId;
    }
}

class UserListQuery {
    constructor(filters, pagination) {
        this.filters = filters;
        this.pagination = pagination;
    }
}

// Command Handler
class UserCommandHandler {
    constructor(writeDb) {
        this.writeDb = writeDb;
    }

    async handle(command) {
        if (command instanceof CreateUserCommand) {
            return this.handleCreate(command);
        }
        if (command instanceof UpdateUserCommand) {
            return this.handleUpdate(command);
        }
        throw new Error('Unknown command');
    }

    async handleCreate(command) {
        command.validate();
        
        const user = await this.writeDb.users.create({
            email: command.email,
            password: await bcrypt.hash(command.password, 10),
            name: command.name
        });

        // Publish event for read model update
        await this.publishEvent('UserCreated', user);
        
        return user.id;
    }

    async handleUpdate(command) {
        command.validate();
        
        const updated = await this.writeDb.users.update(
            { id: command.userId },
            command.updates
        );

        if (updated) {
            await this.publishEvent('UserUpdated', {
                id: command.userId,
                updates: command.updates
            });
        }

        return updated;
    }
}

// Query Handler
class UserQueryHandler {
    constructor(readDb) {
        this.readDb = readDb;
    }

    async handle(query) {
        if (query instanceof UserDetailsQuery) {
            return this.handleDetails(query);
        }
        if (query instanceof UserListQuery) {
            return this.handleList(query);
        }
        throw new Error('Unknown query');
    }

    async handleDetails(query) {
        return this.readDb.users.findOne({
            where: { id: query.userId },
            attributes: ['id', 'name', 'email', 'createdAt']
        });
    }

    async handleList(query) {
        const { filters, pagination } = query;
        
        return this.readDb.users.findAll({
            where: filters,
            limit: pagination.limit,
            offset: pagination.offset,
            attributes: ['id', 'name', 'email', 'createdAt']
        });
    }
}

// Usage Example
class UserService {
    constructor(commandHandler, queryHandler) {
        this.commandHandler = commandHandler;
        this.queryHandler = queryHandler;
    }

    async createUser(userData) {
        const command = new CreateUserCommand(userData);
        return this.commandHandler.handle(command);
    }

    async updateUser(userId, updates) {
        const command = new UpdateUserCommand(userId, updates);
        return this.commandHandler.handle(command);
    }

    async getUserDetails(userId) {
        const query = new UserDetailsQuery(userId);
        return this.queryHandler.handle(query);
    }

    async listUsers(filters, pagination) {
        const query = new UserListQuery(filters, pagination);
        return this.queryHandler.handle(query);
    }
}
```

### Benefits
- Optimized read and write operations
- Better scalability
- Improved performance
- Simplified queries
- Better security control

### Best Practices
1. Separate read and write databases
2. Use event sourcing when appropriate
3. Maintain eventual consistency
4. Handle concurrent updates
5. Implement proper validation

## Related Topics
- [[Event-Sourcing]] - Event-based state management
- [[Domain-Driven-Design]] - Strategic design patterns
- [[API-Design]] - RESTful API design patterns
- [[Microservices]] - Service decomposition patterns

## Practice Projects
1. Build a CQRS-based user management system
2. Create an event-driven notification service
3. Implement a microservices-based e-commerce platform
4. Design a layered architecture application

## Resources
- [Node.js Design Patterns Book](https://www.nodejsdesignpatterns.com/)
- [Martin Fowler's Architecture Guide](https://martinfowler.com/architecture/)
- [[Learning-Resources#Architecture|Architecture Resources]]

## Tags
#architecture #design-patterns #nodejs #system-design