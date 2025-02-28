# Hexagonal Architecture (Ports and Adapters) in Node.js

A comprehensive guide to implementing Hexagonal Architecture in Node.js applications.

## 1. Core Domain

### Domain Model Implementation
```javascript
// Core domain entities and value objects
class Order {
  constructor(id, customerId, items) {
    this.id = id;
    this.customerId = customerId;
    this.items = items;
    this.status = 'pending';
  }

  calculateTotal() {
    return this.items.reduce(
      (total, item) => total + item.price * item.quantity,
      0
    );
  }

  submit() {
    if (this.items.length === 0) {
      throw new Error('Order must have at least one item');
    }
    this.status = 'submitted';
  }

  cancel() {
    if (this.status === 'delivered') {
      throw new Error('Cannot cancel delivered order');
    }
    this.status = 'cancelled';
  }
}

class OrderItem {
  constructor(productId, quantity, price) {
    this.productId = productId;
    this.quantity = quantity;
    this.price = price;
  }
}

// Domain services
class OrderService {
  constructor(orderRepository, paymentPort, notificationPort) {
    this.orderRepository = orderRepository;
    this.paymentPort = paymentPort;
    this.notificationPort = notificationPort;
  }

  async createOrder(customerId, items) {
    const order = new Order(
      crypto.randomUUID(),
      customerId,
      items.map(i => new OrderItem(i.productId, i.quantity, i.price))
    );
    
    await this.orderRepository.save(order);
    return order;
  }

  async submitOrder(orderId) {
    const order = await this.orderRepository.findById(orderId);
    if (!order) {
      throw new Error('Order not found');
    }
    
    order.submit();
    
    // Process payment through port
    await this.paymentPort.processPayment({
      orderId: order.id,
      amount: order.calculateTotal()
    });
    
    // Notify through port
    await this.notificationPort.notify({
      type: 'ORDER_SUBMITTED',
      orderId: order.id,
      customerId: order.customerId
    });
    
    await this.orderRepository.save(order);
    return order;
  }
}
```

## 2. Ports (Interfaces)

### Port Definitions
```javascript
// Primary (Driving) Ports
class OrderManagementPort {
  createOrder(customerId, items) {}
  submitOrder(orderId) {}
  cancelOrder(orderId) {}
  getOrder(orderId) {}
}

// Secondary (Driven) Ports
class OrderRepositoryPort {
  save(order) {}
  findById(orderId) {}
  findByCustomerId(customerId) {}
  delete(orderId) {}
}

class PaymentPort {
  processPayment(paymentDetails) {}
  refundPayment(paymentId) {}
  getPaymentStatus(paymentId) {}
}

class NotificationPort {
  notify(notification) {}
  getNotificationStatus(notificationId) {}
}

class ProductCatalogPort {
  getProduct(productId) {}
  checkAvailability(productId, quantity) {}
  reserveStock(productId, quantity) {}
}
```

## 3. Adapters

### Primary (Driving) Adapters
```javascript
// REST API Adapter
class OrderRestAdapter {
  constructor(orderService) {
    this.orderService = orderService;
  }

  async createOrder(req, res) {
    try {
      const order = await this.orderService.createOrder(
        req.body.customerId,
        req.body.items
      );
      res.status(201).json(order);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }

  async submitOrder(req, res) {
    try {
      const order = await this.orderService.submitOrder(
        req.params.orderId
      );
      res.status(200).json(order);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}

// GraphQL Adapter
class OrderGraphQLAdapter {
  constructor(orderService) {
    this.orderService = orderService;
  }

  resolvers = {
    Query: {
      order: async (_, { id }) => {
        return this.orderService.getOrder(id);
      }
    },
    Mutation: {
      createOrder: async (_, { input }) => {
        return this.orderService.createOrder(
          input.customerId,
          input.items
        );
      },
      submitOrder: async (_, { id }) => {
        return this.orderService.submitOrder(id);
      }
    }
  };
}

// Message Queue Adapter
class OrderMessageQueueAdapter {
  constructor(orderService, queueClient) {
    this.orderService = orderService;
    this.queueClient = queueClient;
    this.setupSubscriptions();
  }

  setupSubscriptions() {
    this.queueClient.subscribe('order.create', async (message) => {
      await this.orderService.createOrder(
        message.customerId,
        message.items
      );
    });

    this.queueClient.subscribe('order.submit', async (message) => {
      await this.orderService.submitOrder(message.orderId);
    });
  }
}
```

### Secondary (Driven) Adapters
```javascript
// MongoDB Repository Adapter
class MongoOrderRepositoryAdapter {
  constructor(mongoClient) {
    this.collection = mongoClient.collection('orders');
  }

  async save(order) {
    await this.collection.updateOne(
      { _id: order.id },
      { $set: order },
      { upsert: true }
    );
    return order;
  }

  async findById(orderId) {
    const doc = await this.collection.findOne({ _id: orderId });
    if (!doc) return null;
    return this.mapToOrder(doc);
  }

  mapToOrder(doc) {
    return new Order(
      doc._id,
      doc.customerId,
      doc.items.map(i => new OrderItem(
        i.productId,
        i.quantity,
        i.price
      ))
    );
  }
}

// Payment Gateway Adapter
class StripePaymentAdapter {
  constructor(stripeClient) {
    this.stripe = stripeClient;
  }

  async processPayment(paymentDetails) {
    const paymentIntent = await this.stripe.paymentIntents.create({
      amount: paymentDetails.amount * 100, // Convert to cents
      currency: 'usd',
      metadata: {
        orderId: paymentDetails.orderId
      }
    });
    
    return {
      paymentId: paymentIntent.id,
      status: paymentIntent.status
    };
  }

  async refundPayment(paymentId) {
    const refund = await this.stripe.refunds.create({
      payment_intent: paymentId
    });
    
    return {
      refundId: refund.id,
      status: refund.status
    };
  }
}

// Notification Service Adapter
class EmailNotificationAdapter {
  constructor(emailClient) {
    this.emailClient = emailClient;
  }

  async notify(notification) {
    const template = this.getTemplate(notification.type);
    
    await this.emailClient.send({
      to: notification.customerId,
      template: template,
      data: {
        orderId: notification.orderId
      }
    });
  }

  getTemplate(type) {
    const templates = {
      ORDER_SUBMITTED: 'order-confirmation',
      ORDER_CANCELLED: 'order-cancellation'
    };
    return templates[type];
  }
}
```

## 4. Application Configuration

### Dependency Injection Setup
```javascript
class ApplicationContainer {
  constructor() {
    this.dependencies = new Map();
  }

  register(token, factory) {
    this.dependencies.set(token, factory);
  }

  resolve(token) {
    const factory = this.dependencies.get(token);
    if (!factory) {
      throw new Error(`No dependency registered for token: ${token}`);
    }
    return factory(this);
  }
}

// Application setup
function setupApplication() {
  const container = new ApplicationContainer();

  // Register infrastructure dependencies
  container.register('mongoClient', () => new MongoClient());
  container.register('stripeClient', () => new StripeClient());
  container.register('emailClient', () => new EmailClient());
  container.register('queueClient', () => new QueueClient());

  // Register adapters
  container.register('orderRepository', (c) => 
    new MongoOrderRepositoryAdapter(c.resolve('mongoClient'))
  );
  container.register('paymentAdapter', (c) => 
    new StripePaymentAdapter(c.resolve('stripeClient'))
  );
  container.register('notificationAdapter', (c) => 
    new EmailNotificationAdapter(c.resolve('emailClient'))
  );

  // Register domain services
  container.register('orderService', (c) => 
    new OrderService(
      c.resolve('orderRepository'),
      c.resolve('paymentAdapter'),
      c.resolve('notificationAdapter')
    )
  );

  // Register primary adapters
  container.register('orderRestAdapter', (c) => 
    new OrderRestAdapter(c.resolve('orderService'))
  );
  container.register('orderGraphQLAdapter', (c) => 
    new OrderGraphQLAdapter(c.resolve('orderService'))
  );
  container.register('orderMessageQueueAdapter', (c) => 
    new OrderMessageQueueAdapter(
      c.resolve('orderService'),
      c.resolve('queueClient')
    )
  );

  return container;
}
```

## 5. Testing Strategy

### Test Implementation
```javascript
// Domain Tests
describe('Order Domain', () => {
  test('should calculate total correctly', () => {
    const order = new Order('1', 'customer1', [
      new OrderItem('product1', 2, 10),
      new OrderItem('product2', 1, 20)
    ]);
    
    expect(order.calculateTotal()).toBe(40);
  });

  test('should not submit empty order', () => {
    const order = new Order('1', 'customer1', []);
    
    expect(() => order.submit())
      .toThrow('Order must have at least one item');
  });
});

// Port Mock Implementation
class MockOrderRepository {
  constructor() {
    this.orders = new Map();
  }

  async save(order) {
    this.orders.set(order.id, order);
    return order;
  }

  async findById(orderId) {
    return this.orders.get(orderId);
  }
}

// Service Tests
describe('Order Service', () => {
  let orderService;
  let mockRepository;
  let mockPaymentPort;
  let mockNotificationPort;

  beforeEach(() => {
    mockRepository = new MockOrderRepository();
    mockPaymentPort = {
      processPayment: jest.fn()
    };
    mockNotificationPort = {
      notify: jest.fn()
    };
    
    orderService = new OrderService(
      mockRepository,
      mockPaymentPort,
      mockNotificationPort
    );
  });

  test('should create and submit order', async () => {
    const order = await orderService.createOrder(
      'customer1',
      [{ productId: 'product1', quantity: 1, price: 10 }]
    );
    
    expect(order.id).toBeDefined();
    expect(order.status).toBe('pending');
    
    await orderService.submitOrder(order.id);
    
    expect(mockPaymentPort.processPayment).toHaveBeenCalled();
    expect(mockNotificationPort.notify).toHaveBeenCalled();
  });
});

// Adapter Tests
describe('Order REST Adapter', () => {
  let adapter;
  let mockOrderService;
  let mockRequest;
  let mockResponse;

  beforeEach(() => {
    mockOrderService = {
      createOrder: jest.fn(),
      submitOrder: jest.fn()
    };
    
    adapter = new OrderRestAdapter(mockOrderService);
    
    mockRequest = {
      body: {},
      params: {}
    };
    
    mockResponse = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn()
    };
  });

  test('should handle create order request', async () => {
    mockRequest.body = {
      customerId: 'customer1',
      items: [{ productId: 'product1', quantity: 1, price: 10 }]
    };
    
    mockOrderService.createOrder.mockResolvedValue({
      id: '1',
      status: 'pending'
    });
    
    await adapter.createOrder(mockRequest, mockResponse);
    
    expect(mockResponse.status).toHaveBeenCalledWith(201);
    expect(mockResponse.json).toHaveBeenCalledWith({
      id: '1',
      status: 'pending'
    });
  });
});
```

## Related Topics
- [[DDD-Implementation]] - Domain-Driven Design
- [[Advanced-Microservices]] - Advanced Microservices Patterns
- [[Clean-Architecture]] - Clean Architecture
- [[Testing-Strategies]] - Testing Strategies

## Practice Projects
1. Build order management system
2. Implement payment processing system
3. Create notification service
4. Design product catalog

## Resources
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [[Learning-Resources#Architecture|Architecture Resources]]

## Tags
#hexagonal-architecture #nodejs #ddd #ports-and-adapters #testing
