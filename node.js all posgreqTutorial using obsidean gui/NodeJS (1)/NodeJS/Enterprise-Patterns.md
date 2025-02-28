# Enterprise Patterns in Node.js

A comprehensive guide to implementing enterprise-level design patterns in Node.js applications.

## 1. Repository Pattern

### Implementation
```javascript
// Generic Repository Interface
class Repository {
  findById(id) { throw new Error('Not implemented'); }
  findAll(criteria) { throw new Error('Not implemented'); }
  create(entity) { throw new Error('Not implemented'); }
  update(id, entity) { throw new Error('Not implemented'); }
  delete(id) { throw new Error('Not implemented'); }
}

// MongoDB Repository Implementation
class MongoRepository extends Repository {
  constructor(model) {
    super();
    this.model = model;
  }

  async findById(id) {
    return this.model.findById(id);
  }

  async findAll(criteria = {}) {
    return this.model.find(criteria);
  }

  async create(entity) {
    const model = new this.model(entity);
    return model.save();
  }

  async update(id, entity) {
    return this.model.findByIdAndUpdate(
      id,
      entity,
      { new: true }
    );
  }

  async delete(id) {
    return this.model.findByIdAndDelete(id);
  }
}

// Unit of Work Pattern
class UnitOfWork {
  constructor(connection) {
    this.connection = connection;
    this.repositories = new Map();
  }

  getRepository(name) {
    if (!this.repositories.has(name)) {
      const model = this.connection.model(name);
      this.repositories.set(
        name,
        new MongoRepository(model)
      );
    }
    return this.repositories.get(name);
  }

  async beginTransaction() {
    this.session = await this.connection.startSession();
    this.session.startTransaction();
  }

  async commit() {
    await this.session.commitTransaction();
    await this.session.endSession();
  }

  async rollback() {
    await this.session.abortTransaction();
    await this.session.endSession();
  }
}
```

## 2. Service Layer Pattern

### Implementation
```javascript
// Service Interface
class Service {
  constructor(unitOfWork) {
    this.unitOfWork = unitOfWork;
  }
}

// Order Service Implementation
class OrderService extends Service {
  constructor(unitOfWork, eventEmitter) {
    super(unitOfWork);
    this.eventEmitter = eventEmitter;
  }

  async createOrder(orderData) {
    const orderRepo = this.unitOfWork.getRepository('Order');
    const productRepo = this.unitOfWork.getRepository('Product');

    try {
      await this.unitOfWork.beginTransaction();

      // Validate products
      for (const item of orderData.items) {
        const product = await productRepo.findById(item.productId);
        if (!product || product.stock < item.quantity) {
          throw new Error('Insufficient stock');
        }
      }

      // Create order
      const order = await orderRepo.create({
        ...orderData,
        status: 'pending'
      });

      // Update product stock
      for (const item of orderData.items) {
        await productRepo.update(
          item.productId,
          { $inc: { stock: -item.quantity } }
        );
      }

      await this.unitOfWork.commit();

      // Emit event
      this.eventEmitter.emit('orderCreated', order);

      return order;
    } catch (error) {
      await this.unitOfWork.rollback();
      throw error;
    }
  }

  async processOrder(orderId) {
    const orderRepo = this.unitOfWork.getRepository('Order');

    try {
      await this.unitOfWork.beginTransaction();

      const order = await orderRepo.findById(orderId);
      if (!order) {
        throw new Error('Order not found');
      }

      order.status = 'processing';
      await orderRepo.update(orderId, order);

      await this.unitOfWork.commit();

      // Emit event
      this.eventEmitter.emit('orderProcessing', order);

      return order;
    } catch (error) {
      await this.unitOfWork.rollback();
      throw error;
    }
  }
}
```

## 3. Specification Pattern

### Implementation
```javascript
// Specification Interface
class Specification {
  isSatisfiedBy(candidate) {
    throw new Error('Not implemented');
  }

  and(other) {
    return new AndSpecification(this, other);
  }

  or(other) {
    return new OrSpecification(this, other);
  }

  not() {
    return new NotSpecification(this);
  }
}

// Composite Specifications
class AndSpecification extends Specification {
  constructor(...specifications) {
    super();
    this.specifications = specifications;
  }

  isSatisfiedBy(candidate) {
    return this.specifications.every(spec => 
      spec.isSatisfiedBy(candidate)
    );
  }
}

class OrSpecification extends Specification {
  constructor(...specifications) {
    super();
    this.specifications = specifications;
  }

  isSatisfiedBy(candidate) {
    return this.specifications.some(spec => 
      spec.isSatisfiedBy(candidate)
    );
  }
}

class NotSpecification extends Specification {
  constructor(specification) {
    super();
    this.specification = specification;
  }

  isSatisfiedBy(candidate) {
    return !this.specification.isSatisfiedBy(candidate);
  }
}

// Example Specifications
class PriceRangeSpecification extends Specification {
  constructor(min, max) {
    super();
    this.min = min;
    this.max = max;
  }

  isSatisfiedBy(product) {
    return product.price >= this.min && 
           product.price <= this.max;
  }
}

class InStockSpecification extends Specification {
  constructor(minStock = 1) {
    super();
    this.minStock = minStock;
  }

  isSatisfiedBy(product) {
    return product.stock >= this.minStock;
  }
}

// Usage Example
const affordableSpec = new PriceRangeSpecification(0, 100);
const inStockSpec = new InStockSpecification(5);
const availableSpec = affordableSpec.and(inStockSpec);
```

## 4. Command Pattern

### Implementation
```javascript
// Command Interface
class Command {
  execute() {
    throw new Error('Not implemented');
  }

  undo() {
    throw new Error('Not implemented');
  }
}

// Command Handler
class CommandHandler {
  constructor() {
    this.commands = new Map();
  }

  register(commandName, handler) {
    this.commands.set(commandName, handler);
  }

  async execute(command) {
    const handler = this.commands.get(command.constructor.name);
    if (!handler) {
      throw new Error(`No handler for command: ${command.constructor.name}`);
    }
    return handler.execute(command);
  }
}

// Example Commands
class CreateOrderCommand extends Command {
  constructor(orderData) {
    super();
    this.orderData = orderData;
  }
}

class ProcessOrderCommand extends Command {
  constructor(orderId) {
    super();
    this.orderId = orderId;
  }
}

// Command Handlers
class CreateOrderHandler {
  constructor(orderService) {
    this.orderService = orderService;
  }

  async execute(command) {
    return this.orderService.createOrder(command.orderData);
  }
}

class ProcessOrderHandler {
  constructor(orderService) {
    this.orderService = orderService;
  }

  async execute(command) {
    return this.orderService.processOrder(command.orderId);
  }
}
```

## 5. Observer Pattern

### Implementation
```javascript
// Event Emitter Implementation
class EventEmitter {
  constructor() {
    this.listeners = new Map();
  }

  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event).add(callback);
    
    return () => this.off(event, callback);
  }

  off(event, callback) {
    const listeners = this.listeners.get(event);
    if (listeners) {
      listeners.delete(callback);
    }
  }

  emit(event, data) {
    const listeners = this.listeners.get(event);
    if (listeners) {
      for (const listener of listeners) {
        listener(data);
      }
    }
  }
}

// Example Usage
class OrderProcessor {
  constructor(eventEmitter) {
    this.eventEmitter = eventEmitter;
    this.setupListeners();
  }

  setupListeners() {
    this.eventEmitter.on('orderCreated', this.handleOrderCreated.bind(this));
    this.eventEmitter.on('orderProcessing', this.handleOrderProcessing.bind(this));
  }

  async handleOrderCreated(order) {
    // Send confirmation email
    await this.sendOrderConfirmation(order);
    
    // Update inventory
    await this.updateInventory(order);
  }

  async handleOrderProcessing(order) {
    // Notify shipping department
    await this.notifyShipping(order);
    
    // Update order status
    await this.updateOrderStatus(order);
  }
}
```

## Related Topics
- [[System-Architecture]] - System Architecture
- [[DDD-Implementation]] - Domain-Driven Design
- [[Clean-Architecture]] - Clean Architecture
- [[Testing-Strategies]] - Testing Strategies

## Practice Projects
1. Build enterprise order management system
2. Implement repository pattern with MongoDB
3. Create specification-based filtering system
4. Design event-driven architecture

## Resources
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [[Learning-Resources#Enterprise|Enterprise Architecture Resources]]

## Tags
#enterprise-patterns #nodejs #design-patterns #architecture #integration
