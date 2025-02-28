# Clean Architecture in Node.js

A comprehensive guide to implementing Clean Architecture in Node.js applications.

## 1. Entities Layer

### Core Business Rules
```javascript
// Entity base class
class Entity {
  constructor(id) {
    this._id = id;
    this._createdAt = new Date();
    this._updatedAt = new Date();
  }

  get id() { return this._id; }
  get createdAt() { return this._createdAt; }
  get updatedAt() { return this._updatedAt; }
}

// User entity
class User extends Entity {
  constructor(id, email, password) {
    super(id);
    this._email = email;
    this._password = password;
    this._isActive = true;
  }

  get email() { return this._email; }
  get isActive() { return this._isActive; }

  deactivate() {
    this._isActive = false;
    this._updatedAt = new Date();
  }

  changeEmail(newEmail) {
    if (!this._isActive) {
      throw new Error('Cannot change email of inactive user');
    }
    this._email = newEmail;
    this._updatedAt = new Date();
  }
}

// Product entity
class Product extends Entity {
  constructor(id, name, price, stock) {
    super(id);
    this._name = name;
    this._price = price;
    this._stock = stock;
  }

  get name() { return this._name; }
  get price() { return this._price; }
  get stock() { return this._stock; }

  updateStock(quantity) {
    if (quantity < 0) {
      throw new Error('Stock cannot be negative');
    }
    this._stock = quantity;
    this._updatedAt = new Date();
  }

  updatePrice(price) {
    if (price <= 0) {
      throw new Error('Price must be positive');
    }
    this._price = price;
    this._updatedAt = new Date();
  }
}
```

## 2. Use Cases Layer

### Application Business Rules
```javascript
// Use case interfaces
class UseCase {
  execute(request) {
    throw new Error('Method not implemented');
  }
}

// User registration use case
class RegisterUser extends UseCase {
  constructor(userRepository, passwordHasher, emailService) {
    super();
    this.userRepository = userRepository;
    this.passwordHasher = passwordHasher;
    this.emailService = emailService;
  }

  async execute(request) {
    // Validate request
    if (!request.email || !request.password) {
      throw new Error('Email and password are required');
    }

    // Check if user exists
    const existingUser = await this.userRepository.findByEmail(
      request.email
    );
    if (existingUser) {
      throw new Error('User already exists');
    }

    // Hash password
    const hashedPassword = await this.passwordHasher.hash(
      request.password
    );

    // Create user
    const user = new User(
      crypto.randomUUID(),
      request.email,
      hashedPassword
    );

    // Save user
    await this.userRepository.save(user);

    // Send welcome email
    await this.emailService.sendWelcomeEmail(user.email);

    return {
      id: user.id,
      email: user.email,
      createdAt: user.createdAt
    };
  }
}

// Product management use case
class CreateProduct extends UseCase {
  constructor(productRepository, priceValidator) {
    super();
    this.productRepository = productRepository;
    this.priceValidator = priceValidator;
  }

  async execute(request) {
    // Validate request
    if (!request.name || !request.price) {
      throw new Error('Name and price are required');
    }

    // Validate price
    if (!this.priceValidator.isValid(request.price)) {
      throw new Error('Invalid price');
    }

    // Create product
    const product = new Product(
      crypto.randomUUID(),
      request.name,
      request.price,
      request.stock || 0
    );

    // Save product
    await this.productRepository.save(product);

    return {
      id: product.id,
      name: product.name,
      price: product.price,
      stock: product.stock,
      createdAt: product.createdAt
    };
  }
}
```

## 3. Interface Adapters Layer

### Presentation Layer
```javascript
// Controller base class
class Controller {
  async handle(request) {
    throw new Error('Method not implemented');
  }
}

// User registration controller
class RegisterUserController extends Controller {
  constructor(registerUser) {
    super();
    this.registerUser = registerUser;
  }

  async handle(request) {
    try {
      const result = await this.registerUser.execute({
        email: request.body.email,
        password: request.body.password
      });

      return {
        statusCode: 201,
        body: result
      };
    } catch (error) {
      return {
        statusCode: 400,
        body: { error: error.message }
      };
    }
  }
}

// Product creation controller
class CreateProductController extends Controller {
  constructor(createProduct) {
    super();
    this.createProduct = createProduct;
  }

  async handle(request) {
    try {
      const result = await this.createProduct.execute({
        name: request.body.name,
        price: request.body.price,
        stock: request.body.stock
      });

      return {
        statusCode: 201,
        body: result
      };
    } catch (error) {
      return {
        statusCode: 400,
        body: { error: error.message }
      };
    }
  }
}
```

### Data Access Layer
```javascript
// Repository interfaces
class UserRepository {
  save(user) {
    throw new Error('Method not implemented');
  }

  findById(id) {
    throw new Error('Method not implemented');
  }

  findByEmail(email) {
    throw new Error('Method not implemented');
  }
}

class ProductRepository {
  save(product) {
    throw new Error('Method not implemented');
  }

  findById(id) {
    throw new Error('Method not implemented');
  }

  findByName(name) {
    throw new Error('Method not implemented');
  }
}

// MongoDB implementations
class MongoUserRepository extends UserRepository {
  constructor(database) {
    super();
    this.collection = database.collection('users');
  }

  async save(user) {
    await this.collection.updateOne(
      { _id: user.id },
      {
        $set: {
          email: user.email,
          password: user.password,
          isActive: user.isActive,
          createdAt: user.createdAt,
          updatedAt: user.updatedAt
        }
      },
      { upsert: true }
    );
  }

  async findById(id) {
    const doc = await this.collection.findOne({ _id: id });
    return doc ? this.mapToEntity(doc) : null;
  }

  async findByEmail(email) {
    const doc = await this.collection.findOne({ email });
    return doc ? this.mapToEntity(doc) : null;
  }

  mapToEntity(doc) {
    const user = new User(doc._id, doc.email, doc.password);
    if (!doc.isActive) user.deactivate();
    return user;
  }
}

class MongoProductRepository extends ProductRepository {
  constructor(database) {
    super();
    this.collection = database.collection('products');
  }

  async save(product) {
    await this.collection.updateOne(
      { _id: product.id },
      {
        $set: {
          name: product.name,
          price: product.price,
          stock: product.stock,
          createdAt: product.createdAt,
          updatedAt: product.updatedAt
        }
      },
      { upsert: true }
    );
  }

  async findById(id) {
    const doc = await this.collection.findOne({ _id: id });
    return doc ? this.mapToEntity(doc) : null;
  }

  async findByName(name) {
    const doc = await this.collection.findOne({ name });
    return doc ? this.mapToEntity(doc) : null;
  }

  mapToEntity(doc) {
    return new Product(doc._id, doc.name, doc.price, doc.stock);
  }
}
```

## 4. Frameworks & Drivers Layer

### External Interfaces
```javascript
// Express.js web framework setup
class ExpressRouter {
  constructor(controllers) {
    this.router = express.Router();
    this.controllers = controllers;
    this.setupRoutes();
  }

  setupRoutes() {
    this.router.post('/users', this.adapt(
      this.controllers.registerUser
    ));
    this.router.post('/products', this.adapt(
      this.controllers.createProduct
    ));
  }

  adapt(controller) {
    return async (req, res) => {
      const request = {
        body: req.body,
        query: req.query,
        params: req.params,
        headers: req.headers
      };

      const result = await controller.handle(request);
      res.status(result.statusCode).json(result.body);
    };
  }
}

// MongoDB database setup
class MongoDatabase {
  constructor(config) {
    this.config = config;
  }

  async connect() {
    this.client = await MongoClient.connect(this.config.url);
    this.db = this.client.db(this.config.database);
  }

  async disconnect() {
    await this.client.close();
  }

  collection(name) {
    return this.db.collection(name);
  }
}
```

### External Services
```javascript
// Email service implementation
class NodemailerEmailService {
  constructor(config) {
    this.transporter = nodemailer.createTransport(config);
  }

  async sendWelcomeEmail(email) {
    await this.transporter.sendMail({
      to: email,
      subject: 'Welcome!',
      text: 'Welcome to our platform!'
    });
  }
}

// Password hasher implementation
class BcryptPasswordHasher {
  constructor(saltRounds = 10) {
    this.saltRounds = saltRounds;
  }

  async hash(password) {
    return bcrypt.hash(password, this.saltRounds);
  }

  async compare(password, hash) {
    return bcrypt.compare(password, hash);
  }
}
```

## 5. Dependency Injection

### Container Setup
```javascript
class Container {
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

function setupContainer(config) {
  const container = new Container();

  // Register infrastructure
  container.register('database', () => 
    new MongoDatabase(config.mongodb)
  );
  container.register('emailService', () => 
    new NodemailerEmailService(config.email)
  );
  container.register('passwordHasher', () => 
    new BcryptPasswordHasher()
  );

  // Register repositories
  container.register('userRepository', (c) => 
    new MongoUserRepository(c.resolve('database'))
  );
  container.register('productRepository', (c) => 
    new MongoProductRepository(c.resolve('database'))
  );

  // Register use cases
  container.register('registerUser', (c) => 
    new RegisterUser(
      c.resolve('userRepository'),
      c.resolve('passwordHasher'),
      c.resolve('emailService')
    )
  );
  container.register('createProduct', (c) => 
    new CreateProduct(
      c.resolve('productRepository'),
      new PriceValidator()
    )
  );

  // Register controllers
  container.register('registerUserController', (c) => 
    new RegisterUserController(c.resolve('registerUser'))
  );
  container.register('createProductController', (c) => 
    new CreateProductController(c.resolve('createProduct'))
  );

  return container;
}
```

## Related Topics
- [[Hexagonal-Architecture]] - Hexagonal Architecture
- [[DDD-Implementation]] - Domain-Driven Design
- [[Testing-Strategies]] - Testing Strategies
- [[Dependency-Injection]] - Dependency Injection

## Practice Projects
1. Build user management system
2. Implement product catalog
3. Create order processing system
4. Design inventory management

## Resources
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [[Learning-Resources#Architecture|Architecture Resources]]

## Tags
#clean-architecture #nodejs #ddd #solid #testing
