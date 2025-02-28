# Microservices Architecture in Node.js

A comprehensive guide to building, deploying, and managing microservices using Node.js, covering service design patterns, communication protocols, service discovery, and containerization.

## 1. Service Design Patterns

### Base Service Template
```javascript
const express = require('express');
const { createLogger } = require('./logger');
const { MetricsCollector } = require('./metrics');
const { HealthCheck } = require('./health');
const { CircuitBreaker } = require('./resilience');

class MicroService {
    constructor(options = {}) {
        this.name = options.name || 'microservice';
        this.version = options.version || '1.0.0';
        this.port = options.port || 3000;
        
        this.app = express();
        this.logger = createLogger(this.name);
        this.metrics = new MetricsCollector(this.name);
        this.health = new HealthCheck();
        
        this.setupMiddleware();
        this.setupRoutes();
    }

    setupMiddleware() {
        this.app.use(express.json());
        this.app.use(this.loggerMiddleware.bind(this));
        this.app.use(this.metricsMiddleware.bind(this));
    }

    setupRoutes() {
        // Health check endpoint
        this.app.get('/health', (req, res) => {
            const status = this.health.getStatus();
            res.status(status.healthy ? 200 : 503).json(status);
        });

        // Metrics endpoint
        this.app.get('/metrics', (req, res) => {
            res.json(this.metrics.getMetrics());
        });

        // Version endpoint
        this.app.get('/version', (req, res) => {
            res.json({
                name: this.name,
                version: this.version
            });
        });
    }

    loggerMiddleware(req, res, next) {
        const start = Date.now();
        res.on('finish', () => {
            const duration = Date.now() - start;
            this.logger.info({
                method: req.method,
                url: req.url,
                status: res.statusCode,
                duration
            });
        });
        next();
    }

    metricsMiddleware(req, res, next) {
        const start = Date.now();
        res.on('finish', () => {
            const duration = Date.now() - start;
            this.metrics.recordRequest({
                method: req.method,
                path: req.path,
                status: res.statusCode,
                duration
            });
        });
        next();
    }

    async start() {
        try {
            await this.health.start();
            await this.metrics.start();
            
            this.server = this.app.listen(this.port, () => {
                this.logger.info(`${this.name} started on port ${this.port}`);
            });
        } catch (error) {
            this.logger.error('Failed to start service:', error);
            throw error;
        }
    }

    async stop() {
        try {
            await this.health.stop();
            await this.metrics.stop();
            
            if (this.server) {
                await new Promise((resolve) => {
                    this.server.close(resolve);
                });
            }
            
            this.logger.info(`${this.name} stopped`);
        } catch (error) {
            this.logger.error('Failed to stop service:', error);
            throw error;
        }
    }
}
```

### Service Registry Pattern
```javascript
const { EventEmitter } = require('events');
const axios = require('axios');

class ServiceRegistry extends EventEmitter {
    constructor(options = {}) {
        super();
        this.services = new Map();
        this.checkInterval = options.checkInterval || 30000;
        this.startHealthChecks();
    }

    register(service) {
        this.services.set(service.id, {
            ...service,
            lastCheck: Date.now(),
            healthy: true
        });
        this.emit('service:registered', service);
    }

    deregister(serviceId) {
        this.services.delete(serviceId);
        this.emit('service:deregistered', serviceId);
    }

    getService(name) {
        const services = Array.from(this.services.values())
            .filter(s => s.name === name && s.healthy);
        
        if (services.length === 0) {
            return null;
        }

        // Simple round-robin load balancing
        const index = Math.floor(Math.random() * services.length);
        return services[index];
    }

    async checkHealth(service) {
        try {
            const response = await axios.get(
                `${service.url}/health`,
                { timeout: 5000 }
            );
            return response.status === 200;
        } catch (error) {
            return false;
        }
    }

    startHealthChecks() {
        setInterval(async () => {
            for (const [id, service] of this.services) {
                const healthy = await this.checkHealth(service);
                
                if (healthy !== service.healthy) {
                    service.healthy = healthy;
                    this.emit(
                        healthy ? 'service:healthy' : 'service:unhealthy',
                        service
                    );
                }
                
                service.lastCheck = Date.now();
            }
        }, this.checkInterval);
    }
}
```

## 2. Inter-service Communication

### Message Broker Integration
```javascript
const amqp = require('amqplib');

class MessageBroker {
    constructor(options = {}) {
        this.url = options.url || 'amqp://localhost';
        this.exchanges = new Map();
        this.queues = new Map();
        this.connection = null;
        this.channel = null;
    }

    async connect() {
        try {
            this.connection = await amqp.connect(this.url);
            this.channel = await this.connection.createChannel();
            
            // Handle connection closure
            this.connection.on('close', () => {
                this.reconnect();
            });
        } catch (error) {
            console.error('Failed to connect to RabbitMQ:', error);
            throw error;
        }
    }

    async reconnect() {
        try {
            await this.connect();
            // Restore exchanges and queues
            for (const [name, exchange] of this.exchanges) {
                await this.createExchange(name, exchange.type);
            }
            for (const [name, queue] of this.queues) {
                await this.createQueue(name, queue.options);
            }
        } catch (error) {
            console.error('Failed to reconnect:', error);
            setTimeout(() => this.reconnect(), 5000);
        }
    }

    async createExchange(name, type = 'topic') {
        await this.channel.assertExchange(name, type, {
            durable: true
        });
        this.exchanges.set(name, { type });
    }

    async createQueue(name, options = {}) {
        const queue = await this.channel.assertQueue(name, {
            durable: true,
            ...options
        });
        this.queues.set(name, { options });
        return queue;
    }

    async publish(exchange, routingKey, message) {
        return this.channel.publish(
            exchange,
            routingKey,
            Buffer.from(JSON.stringify(message)),
            { persistent: true }
        );
    }

    async subscribe(queue, handler) {
        return this.channel.consume(queue, async (msg) => {
            if (!msg) return;
            
            try {
                const content = JSON.parse(msg.content.toString());
                await handler(content);
                this.channel.ack(msg);
            } catch (error) {
                console.error('Error processing message:', error);
                this.channel.nack(msg);
            }
        });
    }

    async close() {
        if (this.channel) {
            await this.channel.close();
        }
        if (this.connection) {
            await this.connection.close();
        }
    }
}
```

### HTTP Client with Circuit Breaker
```javascript
const axios = require('axios');

class CircuitBreaker {
    constructor(options = {}) {
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 60000;
        this.failureCount = 0;
        this.lastFailureTime = null;
        this.state = 'CLOSED';
    }

    async execute(request) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
                this.state = 'HALF_OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }

        try {
            const response = await request();
            
            if (this.state === 'HALF_OPEN') {
                this.state = 'CLOSED';
                this.failureCount = 0;
            }
            
            return response;
        } catch (error) {
            this.handleFailure();
            throw error;
        }
    }

    handleFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();

        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}

class ServiceClient {
    constructor(options = {}) {
        this.baseURL = options.baseURL;
        this.timeout = options.timeout || 5000;
        this.circuitBreaker = new CircuitBreaker(options.circuitBreaker);
        
        this.client = axios.create({
            baseURL: this.baseURL,
            timeout: this.timeout
        });
    }

    async request(config) {
        return this.circuitBreaker.execute(async () => {
            try {
                const response = await this.client.request(config);
                return response.data;
            } catch (error) {
                if (error.response) {
                    throw new Error(
                        `Service returned ${error.response.status}`
                    );
                }
                throw error;
            }
        });
    }

    async get(path, config = {}) {
        return this.request({ ...config, method: 'GET', url: path });
    }

    async post(path, data, config = {}) {
        return this.request({
            ...config,
            method: 'POST',
            url: path,
            data
        });
    }
}
```

## 3. Service Discovery

### Consul Integration
```javascript
const Consul = require('consul');

class ServiceDiscovery {
    constructor(options = {}) {
        this.consul = new Consul({
            host: options.host || 'localhost',
            port: options.port || 8500
        });
        this.serviceId = options.serviceId;
        this.serviceName = options.serviceName;
        this.servicePort = options.servicePort;
    }

    async register() {
        const registration = {
            id: this.serviceId,
            name: this.serviceName,
            port: this.servicePort,
            check: {
                http: `http://localhost:${this.servicePort}/health`,
                interval: '10s',
                timeout: '5s'
            }
        };

        try {
            await this.consul.agent.service.register(registration);
            console.log(`Service registered: ${this.serviceId}`);
        } catch (error) {
            console.error('Failed to register service:', error);
            throw error;
        }
    }

    async deregister() {
        try {
            await this.consul.agent.service.deregister(this.serviceId);
            console.log(`Service deregistered: ${this.serviceId}`);
        } catch (error) {
            console.error('Failed to deregister service:', error);
            throw error;
        }
    }

    async findService(name) {
        try {
            const result = await this.consul.catalog.service.nodes(name);
            return result.map(node => ({
                id: node.ServiceID,
                address: node.ServiceAddress,
                port: node.ServicePort
            }));
        } catch (error) {
            console.error('Failed to find service:', error);
            throw error;
        }
    }

    async watchService(name, callback) {
        const watch = this.consul.watch({
            method: this.consul.catalog.service.nodes,
            options: { service: name }
        });

        watch.on('change', nodes => {
            const services = nodes.map(node => ({
                id: node.ServiceID,
                address: node.ServiceAddress,
                port: node.ServicePort
            }));
            callback(services);
        });

        watch.on('error', error => {
            console.error('Watch error:', error);
        });

        return watch;
    }
}
```

## 4. API Gateway Pattern

### Gateway Implementation
```javascript
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');

class ApiGateway {
    constructor(options = {}) {
        this.app = express();
        this.port = options.port || 8000;
        this.serviceRegistry = options.serviceRegistry;
        
        this.setupMiddleware();
        this.setupRoutes();
    }

    setupMiddleware() {
        // Rate limiting
        this.app.use(rateLimit({
            windowMs: 15 * 60 * 1000, // 15 minutes
            max: 100 // limit each IP to 100 requests per windowMs
        }));

        // Authentication
        this.app.use(this.authenticate.bind(this));

        // Request logging
        this.app.use((req, res, next) => {
            console.log(`${req.method} ${req.url}`);
            next();
        });
    }

    authenticate(req, res, next) {
        const token = req.headers.authorization;
        if (!token) {
            return res.status(401).json({
                error: 'Authentication required'
            });
        }
        // Implement your authentication logic here
        next();
    }

    createServiceProxy(serviceName) {
        return createProxyMiddleware({
            target: 'http://localhost:3000', // Default target
            router: async (req) => {
                const service = await this.serviceRegistry.getService(
                    serviceName
                );
                if (!service) {
                    throw new Error(`Service ${serviceName} not found`);
                }
                return `http://${service.address}:${service.port}`;
            },
            pathRewrite: {
                [`^/api/${serviceName}`]: ''
            },
            changeOrigin: true
        });
    }

    setupRoutes() {
        // User service routes
        this.app.use(
            '/api/users',
            this.createServiceProxy('user-service')
        );

        // Order service routes
        this.app.use(
            '/api/orders',
            this.createServiceProxy('order-service')
        );

        // Product service routes
        this.app.use(
            '/api/products',
            this.createServiceProxy('product-service')
        );

        // Error handling
        this.app.use((err, req, res, next) => {
            console.error('Gateway Error:', err);
            res.status(500).json({
                error: 'Internal Server Error'
            });
        });
    }

    async start() {
        return new Promise((resolve) => {
            this.server = this.app.listen(this.port, () => {
                console.log(`API Gateway started on port ${this.port}`);
                resolve();
            });
        });
    }

    async stop() {
        if (this.server) {
            return new Promise((resolve) => {
                this.server.close(resolve);
            });
        }
    }
}
```

## 5. Docker Integration

### Dockerfile Example
```dockerfile
# Base image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Set environment variables
ENV NODE_ENV=production

# Start command
CMD ["node", "src/index.js"]
```

### Docker Compose Configuration
```yaml
version: '3.8'

services:
  api-gateway:
    build: ./gateway
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
    depends_on:
      - consul
      - rabbitmq

  user-service:
    build: ./user-service
    environment:
      - NODE_ENV=production
      - RABBITMQ_URL=amqp://rabbitmq
      - CONSUL_HOST=consul
    depends_on:
      - consul
      - rabbitmq
      - user-db

  order-service:
    build: ./order-service
    environment:
      - NODE_ENV=production
      - RABBITMQ_URL=amqp://rabbitmq
      - CONSUL_HOST=consul
    depends_on:
      - consul
      - rabbitmq
      - order-db

  consul:
    image: consul:latest
    ports:
      - "8500:8500"

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  user-db:
    image: postgres:13
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=userdb

  order-db:
    image: postgres:13
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=orderdb
```

## Related Topics
- [[Service-Mesh]] - Service mesh implementation
- [[API-Gateway]] - API Gateway patterns
- [[Event-Driven]] - Event-driven architecture
- [[Container-Orchestration]] - Container orchestration with Kubernetes

## Practice Projects
1. Build a microservices-based e-commerce system
2. Create a distributed logging system
3. Implement a service mesh
4. Develop a scalable message processing system

## Resources
- [Microservices.io Patterns](https://microservices.io/patterns/)
- [Node.js Microservices](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/)
- [[Learning-Resources#Microservices|Microservices Resources]]

## Tags
#microservices #nodejs #docker #architecture #patterns
