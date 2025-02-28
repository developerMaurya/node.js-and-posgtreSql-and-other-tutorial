# Domain-Driven Design in Node.js

A comprehensive guide to implementing Domain-Driven Design (DDD) in Node.js applications.

## 1. Domain Layer

### Entity Implementation
```javascript
class Entity {
  constructor(id) {
    this._id = id;
    this._events = [];
  }

  get id() {
    return this._id;
  }

  addDomainEvent(event) {
    this._events.push(event);
  }

  clearEvents() {
    this._events = [];
  }

  getEvents() {
    return [...this._events];
  }
}

class User extends Entity {
  constructor(id, email, name) {
    super(id);
    this._email = email;
    this._name = name;
    this._status = 'inactive';
  }

  activate() {
    if (this._status === 'active') {
      throw new Error('User already active');
    }
    
    this._status = 'active';
    this.addDomainEvent(new UserActivated(this.id));
  }

  updateEmail(email) {
    if (!this.isValidEmail(email)) {
      throw new Error('Invalid email');
    }
    
    const oldEmail = this._email;
    this._email = email;
    
    this.addDomainEvent(new UserEmailUpdated(this.id, oldEmail, email));
  }

  isValidEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

### Value Object Implementation
```javascript
class ValueObject {
  equals(other) {
    return JSON.stringify(this) === JSON.stringify(other);
  }
}

class Address extends ValueObject {
  constructor(street, city, country, postalCode) {
    super();
    this._street = street;
    this._city = city;
    this._country = country;
    this._postalCode = postalCode;
    
    this.validate();
  }

  validate() {
    if (!this._street || !this._city || !this._country || !this._postalCode) {
      throw new Error('All address fields are required');
    }
  }

  toString() {
    return `${this._street}, ${this._city}, ${this._country} ${this._postalCode}`;
  }
}

class Money extends ValueObject {
  constructor(amount, currency) {
    super();
    this._amount = amount;
    this._currency = currency;
    
    this.validate();
  }

  validate() {
    if (typeof this._amount !== 'number' || this._amount < 0) {
      throw new Error('Amount must be a positive number');
    }
    
    if (!['USD', 'EUR', 'GBP'].includes(this._currency)) {
      throw new Error('Invalid currency');
    }
  }

  add(other) {
    if (this._currency !== other._currency) {
      throw new Error('Cannot add different currencies');
    }
    
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other) {
    if (this._currency !== other._currency) {
      throw new Error('Cannot subtract different currencies');
    }
    
    return new Money(this._amount - other._amount, this._currency);
  }
}
```

### Aggregate Root Implementation
```javascript
class Order extends Entity {
  constructor(id, userId) {
    super(id);
    this._userId = userId;
    this._items = new Map();
    this._status = 'draft';
    this._total = new Money(0, 'USD');
  }

  addItem(product, quantity) {
    if (this._status !== 'draft') {
      throw new Error('Cannot modify confirmed order');
    }
    
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    
    const existingQuantity = this._items.get(product.id) || 0;
    this._items.set(product.id, existingQuantity + quantity);
    
    this._total = this._total.add(product.price.multiply(quantity));
    
    this.addDomainEvent(new OrderItemAdded(this.id, product.id, quantity));
  }

  removeItem(productId, quantity) {
    if (this._status !== 'draft') {
      throw new Error('Cannot modify confirmed order');
    }
    
    const existingQuantity = this._items.get(productId);
    if (!existingQuantity) {
      throw new Error('Product not in order');
    }
    
    if (quantity > existingQuantity) {
      throw new Error('Cannot remove more than existing quantity');
    }
    
    const newQuantity = existingQuantity - quantity;
    if (newQuantity === 0) {
      this._items.delete(productId);
    } else {
      this._items.set(productId, newQuantity);
    }
    
    this.addDomainEvent(new OrderItemRemoved(this.id, productId, quantity));
  }

  confirm() {
    if (this._status !== 'draft') {
      throw new Error('Order already confirmed');
    }
    
    if (this._items.size === 0) {
      throw new Error('Cannot confirm empty order');
    }
    
    this._status = 'confirmed';
    this.addDomainEvent(new OrderConfirmed(this.id));
  }
}
```

## 2. Repository Layer

### Repository Implementation
```javascript
class Repository {
  constructor(unitOfWork) {
    this._unitOfWork = unitOfWork;
  }

  async save(entity) {
    await this._unitOfWork.registerNew(entity);
  }

  async update(entity) {
    await this._unitOfWork.registerDirty(entity);
  }

  async delete(entity) {
    await this._unitOfWork.registerDeleted(entity);
  }
}

class UserRepository extends Repository {
  constructor(unitOfWork, userMapper) {
    super(unitOfWork);
    this._mapper = userMapper;
  }

  async findById(id) {
    const data = await this._unitOfWork.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    
    return data ? this._mapper.toDomain(data) : null;
  }

  async findByEmail(email) {
    const data = await this._unitOfWork.query(
      'SELECT * FROM users WHERE email = ?',
      [email]
    );
    
    return data ? this._mapper.toDomain(data) : null;
  }

  async save(user) {
    const data = this._mapper.toPersistence(user);
    if (user.id) {
      await this._unitOfWork.registerDirty(data);
    } else {
      await this._unitOfWork.registerNew(data);
    }
    
    // Handle domain events
    for (const event of user.getEvents()) {
      await this._unitOfWork.publishEvent(event);
    }
    user.clearEvents();
  }
}
```

## 3. Application Layer

### Application Service Implementation
```javascript
class ApplicationService {
  constructor(unitOfWork) {
    this._unitOfWork = unitOfWork;
  }

  async execute(command) {
    await this._unitOfWork.begin();
    
    try {
      const result = await this.handleCommand(command);
      await this._unitOfWork.commit();
      return result;
    } catch (error) {
      await this._unitOfWork.rollback();
      throw error;
    }
  }
}

class UserService extends ApplicationService {
  constructor(unitOfWork, userRepository, eventPublisher) {
    super(unitOfWork);
    this._userRepository = userRepository;
    this._eventPublisher = eventPublisher;
  }

  async createUser(command) {
    const existingUser = await this._userRepository.findByEmail(command.email);
    if (existingUser) {
      throw new Error('User already exists');
    }
    
    const user = new User(
      command.id,
      command.email,
      command.name
    );
    
    await this._userRepository.save(user);
    
    // Handle domain events
    for (const event of user.getEvents()) {
      await this._eventPublisher.publish(event);
    }
    
    return user;
  }

  async updateUser(command) {
    const user = await this._userRepository.findById(command.id);
    if (!user) {
      throw new Error('User not found');
    }
    
    if (command.email) {
      user.updateEmail(command.email);
    }
    
    await this._userRepository.save(user);
    
    // Handle domain events
    for (const event of user.getEvents()) {
      await this._eventPublisher.publish(event);
    }
    
    return user;
  }
}
```

## 4. Domain Events

### Event Implementation
```javascript
class DomainEvent {
  constructor() {
    this._occurredOn = new Date();
  }

  get occurredOn() {
    return this._occurredOn;
  }
}

class UserActivated extends DomainEvent {
  constructor(userId) {
    super();
    this._userId = userId;
  }

  get userId() {
    return this._userId;
  }
}

class UserEmailUpdated extends DomainEvent {
  constructor(userId, oldEmail, newEmail) {
    super();
    this._userId = userId;
    this._oldEmail = oldEmail;
    this._newEmail = newEmail;
  }
}
```

### Event Publisher Implementation
```javascript
class EventPublisher {
  constructor() {
    this._handlers = new Map();
  }

  subscribe(eventType, handler) {
    if (!this._handlers.has(eventType)) {
      this._handlers.set(eventType, []);
    }
    
    this._handlers.get(eventType).push(handler);
  }

  async publish(event) {
    const handlers = this._handlers.get(event.constructor.name) || [];
    
    for (const handler of handlers) {
      await handler.handle(event);
    }
  }
}

class UserActivatedHandler {
  async handle(event) {
    // Handle user activated event
    console.log(`User ${event.userId} was activated`);
  }
}

class UserEmailUpdatedHandler {
  async handle(event) {
    // Handle email updated event
    console.log(`User ${event.userId} email was updated`);
  }
}
```

## 5. Unit of Work

### Unit of Work Implementation
```javascript
class UnitOfWork {
  constructor(connection) {
    this._connection = connection;
    this._newObjects = new Set();
    this._dirtyObjects = new Set();
    this._deletedObjects = new Set();
  }

  async begin() {
    await this._connection.beginTransaction();
  }

  async commit() {
    try {
      // Handle new objects
      for (const obj of this._newObjects) {
        await this.insertObject(obj);
      }
      
      // Handle dirty objects
      for (const obj of this._dirtyObjects) {
        await this.updateObject(obj);
      }
      
      // Handle deleted objects
      for (const obj of this._deletedObjects) {
        await this.deleteObject(obj);
      }
      
      await this._connection.commit();
      
      this.clear();
    } catch (error) {
      await this.rollback();
      throw error;
    }
  }

  async rollback() {
    await this._connection.rollback();
    this.clear();
  }

  clear() {
    this._newObjects.clear();
    this._dirtyObjects.clear();
    this._deletedObjects.clear();
  }

  registerNew(object) {
    this._newObjects.add(object);
  }

  registerDirty(object) {
    this._dirtyObjects.add(object);
  }

  registerDeleted(object) {
    this._deletedObjects.add(object);
  }

  async query(sql, params) {
    return this._connection.query(sql, params);
  }
}
```

## Related Topics
- [[Event-Sourcing]] - Event Sourcing Pattern
- [[CQRS-Advanced]] - Advanced CQRS
- [[System-Architecture]] - System Architecture
- [[Enterprise-Patterns]] - Enterprise Patterns

## Practice Projects
1. Build e-commerce system using DDD
2. Implement banking system
3. Create inventory management system
4. Design booking system

## Resources
- [Domain-Driven Design Fundamentals](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [[Learning-Resources#DDD|DDD Resources]]

## Tags
#ddd #nodejs #architecture #domain-driven-design #enterprise
