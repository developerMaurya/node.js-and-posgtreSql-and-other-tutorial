# Advanced Microservices Patterns in Node.js

A comprehensive guide to implementing advanced microservices patterns in Node.js applications.

## 1. Saga Pattern

### Distributed Saga Implementation
```javascript
class SagaOrchestrator {
  constructor(options = {}) {
    this.options = {
      retryAttempts: options.retryAttempts || 3,
      retryDelay: options.retryDelay || 1000,
      ...options
    };
    
    this.eventBus = new EventBus();
    this.stateStore = new StateStore();
    this.compensations = new Map();
  }

  async start(sagaId, steps) {
    const saga = {
      id: sagaId,
      steps,
      currentStep: 0,
      status: 'started',
      data: {}
    };
    
    await this.stateStore.save(saga);
    return this.execute(saga);
  }

  async execute(saga) {
    while (saga.currentStep < saga.steps.length) {
      const step = saga.steps[saga.currentStep];
      
      try {
        const result = await this.executeStep(saga, step);
        saga.data[step.name] = result;
        saga.currentStep++;
        await this.stateStore.save(saga);
      } catch (error) {
        await this.handleFailure(saga, error);
        return {
          success: false,
          sagaId: saga.id,
          error: error.message
        };
      }
    }
    
    saga.status = 'completed';
    await this.stateStore.save(saga);
    
    return {
      success: true,
      sagaId: saga.id,
      data: saga.data
    };
  }

  async executeStep(saga, step) {
    let attempts = 0;
    
    while (attempts < this.options.retryAttempts) {
      try {
        const result = await this.eventBus.publish(step.command, {
          sagaId: saga.id,
          data: saga.data
        });
        
        this.compensations.set(step.name, step.compensate);
        return result;
      } catch (error) {
        attempts++;
        if (attempts === this.options.retryAttempts) {
          throw error;
        }
        await new Promise(resolve => 
          setTimeout(resolve, this.options.retryDelay)
        );
      }
    }
  }

  async handleFailure(saga, error) {
    saga.status = 'failed';
    saga.error = error.message;
    
    for (let i = saga.currentStep - 1; i >= 0; i--) {
      const step = saga.steps[i];
      const compensation = this.compensations.get(step.name);
      
      if (compensation) {
        try {
          await this.eventBus.publish(compensation, {
            sagaId: saga.id,
            data: saga.data
          });
        } catch (compensationError) {
          // Log compensation failure
          console.error(
            `Compensation failed for step ${step.name}:`,
            compensationError
          );
        }
      }
    }
    
    await this.stateStore.save(saga);
  }
}
```

## 2. CQRS Pattern

### CQRS Implementation
```javascript
class CommandBus {
  constructor() {
    this.handlers = new Map();
  }

  register(commandType, handler) {
    this.handlers.set(commandType, handler);
  }

  async execute(command) {
    const handler = this.handlers.get(command.type);
    
    if (!handler) {
      throw new Error(`No handler for command type: ${command.type}`);
    }
    
    return handler.execute(command);
  }
}

class QueryBus {
  constructor() {
    this.handlers = new Map();
  }

  register(queryType, handler) {
    this.handlers.set(queryType, handler);
  }

  async execute(query) {
    const handler = this.handlers.get(query.type);
    
    if (!handler) {
      throw new Error(`No handler for query type: ${query.type}`);
    }
    
    return handler.execute(query);
  }
}

class EventStore {
  constructor() {
    this.events = [];
    this.subscribers = new Map();
  }

  async append(event) {
    this.events.push({
      ...event,
      timestamp: Date.now()
    });
    
    await this.notify(event);
  }

  async notify(event) {
    const subscribers = this.subscribers.get(event.type) || [];
    
    for (const subscriber of subscribers) {
      await subscriber(event);
    }
  }

  subscribe(eventType, handler) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, []);
    }
    
    this.subscribers.get(eventType).push(handler);
  }

  getEvents(aggregateId) {
    return this.events.filter(e => e.aggregateId === aggregateId);
  }
}

class ReadModelProjector {
  constructor(eventStore) {
    this.eventStore = eventStore;
    this.readModel = new Map();
    this.setupProjections();
  }

  setupProjections() {
    this.eventStore.subscribe('ItemCreated', this.onItemCreated.bind(this));
    this.eventStore.subscribe('ItemUpdated', this.onItemUpdated.bind(this));
    this.eventStore.subscribe('ItemDeleted', this.onItemDeleted.bind(this));
  }

  async onItemCreated(event) {
    this.readModel.set(event.aggregateId, {
      id: event.aggregateId,
      ...event.data
    });
  }

  async onItemUpdated(event) {
    const item = this.readModel.get(event.aggregateId);
    if (item) {
      this.readModel.set(event.aggregateId, {
        ...item,
        ...event.data
      });
    }
  }

  async onItemDeleted(event) {
    this.readModel.delete(event.aggregateId);
  }

  getItem(id) {
    return this.readModel.get(id);
  }

  getAllItems() {
    return Array.from(this.readModel.values());
  }
}
```

## 3. Event Sourcing

### Event Sourcing Implementation
```javascript
class Aggregate {
  constructor(id) {
    this.id = id;
    this.version = 0;
    this.changes = [];
  }

  getUncommittedChanges() {
    return this.changes;
  }

  markChangesAsCommitted() {
    this.changes = [];
  }

  loadFromHistory(events) {
    for (const event of events) {
      this.applyChange(event, false);
    }
  }

  applyChange(event, isNew = true) {
    this.apply(event);
    this.version++;
    
    if (isNew) {
      this.changes.push(event);
    }
  }

  apply(event) {
    const handler = `on${event.type}`;
    if (typeof this[handler] === 'function') {
      this[handler](event);
    }
  }
}

class OrderAggregate extends Aggregate {
  constructor(id) {
    super(id);
    this.items = [];
    this.status = 'new';
  }

  createOrder(data) {
    this.applyChange({
      type: 'OrderCreated',
      aggregateId: this.id,
      data
    });
  }

  addItem(item) {
    this.applyChange({
      type: 'ItemAdded',
      aggregateId: this.id,
      data: item
    });
  }

  removeItem(itemId) {
    this.applyChange({
      type: 'ItemRemoved',
      aggregateId: this.id,
      data: { itemId }
    });
  }

  submitOrder() {
    if (this.status !== 'new') {
      throw new Error('Order already submitted');
    }
    
    this.applyChange({
      type: 'OrderSubmitted',
      aggregateId: this.id
    });
  }

  onOrderCreated(event) {
    Object.assign(this, event.data);
  }

  onItemAdded(event) {
    this.items.push(event.data);
  }

  onItemRemoved(event) {
    this.items = this.items.filter(
      item => item.id !== event.data.itemId
    );
  }

  onOrderSubmitted() {
    this.status = 'submitted';
  }
}
```

## 4. API Gateway Pattern

### API Gateway Implementation
```javascript
class ApiGateway {
  constructor(options = {}) {
    this.options = {
      timeout: options.timeout || 5000,
      retryAttempts: options.retryAttempts || 3,
      ...options
    };
    
    this.registry = new ServiceRegistry();
    this.circuitBreaker = new CircuitBreaker();
    this.cache = new Cache();
  }

  async handleRequest(req) {
    const route = this.getRoute(req);
    
    if (!route) {
      throw new Error('Route not found');
    }
    
    try {
      // Authentication
      await this.authenticate(req);
      
      // Rate Limiting
      await this.checkRateLimit(req);
      
      // Cache Check
      const cachedResponse = await this.checkCache(req);
      if (cachedResponse) {
        return cachedResponse;
      }
      
      // Service Discovery
      const service = await this.registry.getService(route.serviceId);
      
      // Load Balancing
      const instance = await this.loadBalancer.select(service);
      
      // Circuit Breaking
      const response = await this.circuitBreaker.execute(
        async () => this.forwardRequest(req, instance)
      );
      
      // Cache Response
      await this.cacheResponse(req, response);
      
      return response;
    } catch (error) {
      return this.handleError(error);
    }
  }

  async authenticate(req) {
    const token = req.headers.authorization;
    if (!token) {
      throw new Error('Authentication required');
    }
    
    try {
      const user = await this.authService.verify(token);
      req.user = user;
    } catch (error) {
      throw new Error('Invalid authentication');
    }
  }

  async checkRateLimit(req) {
    const key = `${req.ip}:${req.path}`;
    const limit = await this.rateLimiter.check(key);
    
    if (!limit.allowed) {
      throw new Error('Rate limit exceeded');
    }
  }

  async checkCache(req) {
    if (req.method !== 'GET') {
      return null;
    }
    
    const key = this.getCacheKey(req);
    return this.cache.get(key);
  }

  async forwardRequest(req, instance) {
    const url = this.buildServiceUrl(instance, req.path);
    
    const response = await fetch(url, {
      method: req.method,
      headers: this.filterHeaders(req.headers),
      body: req.body,
      timeout: this.options.timeout
    });
    
    return {
      status: response.status,
      headers: response.headers,
      body: await response.json()
    };
  }

  async cacheResponse(req, response) {
    if (req.method !== 'GET' || !this.isCacheable(response)) {
      return;
    }
    
    const key = this.getCacheKey(req);
    await this.cache.set(key, response);
  }

  handleError(error) {
    // Log error
    console.error('API Gateway Error:', error);
    
    return {
      status: this.getErrorStatus(error),
      body: {
        error: this.getErrorMessage(error)
      }
    };
  }

  getRoute(req) {
    return this.routes.find(route => 
      route.method === req.method && 
      route.path.test(req.path)
    );
  }

  getCacheKey(req) {
    return `${req.method}:${req.path}:${
      JSON.stringify(req.query)
    }`;
  }

  isCacheable(response) {
    return response.status === 200 && 
           response.headers['cache-control'] !== 'no-cache';
  }

  filterHeaders(headers) {
    const filtered = { ...headers };
    delete filtered.host;
    delete filtered.connection;
    return filtered;
  }

  getErrorStatus(error) {
    switch (error.constructor.name) {
      case 'AuthenticationError':
        return 401;
      case 'AuthorizationError':
        return 403;
      case 'RateLimitError':
        return 429;
      case 'ServiceUnavailableError':
        return 503;
      default:
        return 500;
    }
  }

  getErrorMessage(error) {
    return this.options.debug ? 
      error.message : 
      'Internal Server Error';
  }
}
```

## 5. Service Mesh Integration

### Service Mesh Client Implementation
```javascript
class ServiceMeshClient {
  constructor(options = {}) {
    this.options = {
      serviceName: options.serviceName,
      namespace: options.namespace || 'default',
      ...options
    };
    
    this.discovery = new ServiceDiscovery();
    this.loadBalancer = new LoadBalancer();
    this.circuitBreaker = new CircuitBreaker();
    this.metrics = new MetricsCollector();
  }

  async request(service, options) {
    const instances = await this.discovery.getInstances(service);
    
    if (instances.length === 0) {
      throw new Error(`No instances found for service: ${service}`);
    }
    
    const instance = this.loadBalancer.select(instances);
    
    return this.circuitBreaker.execute(async () => {
      const start = Date.now();
      
      try {
        const response = await this.sendRequest(instance, options);
        
        this.metrics.recordSuccess(
          service,
          Date.now() - start
        );
        
        return response;
      } catch (error) {
        this.metrics.recordError(service, error);
        throw error;
      }
    });
  }

  async sendRequest(instance, options) {
    const url = this.buildUrl(instance, options.path);
    
    const response = await fetch(url, {
      method: options.method || 'GET',
      headers: this.buildHeaders(options.headers),
      body: options.body,
      timeout: options.timeout || 5000
    });
    
    if (!response.ok) {
      throw new Error(
        `Service request failed: ${response.status}`
      );
    }
    
    return response;
  }

  buildUrl(instance, path) {
    return `${instance.protocol}://${instance.host}:${
      instance.port
    }${path}`;
  }

  buildHeaders(headers = {}) {
    return {
      'x-service-name': this.options.serviceName,
      'x-request-id': this.generateRequestId(),
      ...headers
    };
  }

  generateRequestId() {
    return `${this.options.serviceName}-${
      Date.now()
    }-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

## Related Topics
- [[Service-Mesh]] - Service Mesh Implementation
- [[Event-Sourcing]] - Event Sourcing Pattern
- [[DDD-Implementation]] - Domain-Driven Design
- [[Traffic-Management]] - Traffic Management

## Practice Projects
1. Build distributed saga system
2. Implement CQRS architecture
3. Create API gateway
4. Design service mesh client

## Resources
- [Microservices Patterns](https://microservices.io/patterns/index.html)
- [[Learning-Resources#Microservices|Microservices Resources]]

## Tags
#microservices #nodejs #patterns #architecture #distributed-systems
