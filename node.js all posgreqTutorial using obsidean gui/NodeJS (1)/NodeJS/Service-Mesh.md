# Service Mesh Implementation in Node.js

A comprehensive guide to implementing service mesh patterns in Node.js applications.

## 1. Sidecar Pattern

### Sidecar Implementation
```javascript
class Sidecar {
  constructor(options = {}) {
    this.options = {
      servicePort: options.servicePort || 3000,
      adminPort: options.adminPort || 9901,
      metricsInterval: options.metricsInterval || 5000,
      ...options
    };
    
    this.metrics = new MetricsCollector();
    this.proxy = new ProxyServer();
    this.discovery = new ServiceDiscovery();
    this.healthChecker = new HealthChecker();
  }

  async start() {
    // Start admin server
    await this.startAdminServer();
    
    // Start proxy server
    await this.startProxyServer();
    
    // Start metrics collection
    this.startMetricsCollection();
    
    // Start health checks
    this.startHealthChecks();
  }

  async startAdminServer() {
    const app = express();
    
    app.get('/metrics', (req, res) => {
      res.json(this.metrics.getMetrics());
    });
    
    app.get('/health', (req, res) => {
      res.json(this.healthChecker.getStatus());
    });
    
    app.listen(this.options.adminPort);
  }

  async startProxyServer() {
    this.proxy.on('request', async (req, res) => {
      const target = await this.discovery.resolveService(req);
      if (!target) {
        return res.status(503).json({ error: 'Service Unavailable' });
      }
      
      const start = Date.now();
      try {
        await this.proxy.forward(req, res, target);
        this.metrics.recordSuccess(target, Date.now() - start);
      } catch (error) {
        this.metrics.recordError(target, error);
        res.status(500).json({ error: 'Internal Error' });
      }
    });
    
    await this.proxy.listen(this.options.servicePort);
  }

  startMetricsCollection() {
    setInterval(() => {
      this.metrics.collect();
    }, this.options.metricsInterval);
  }

  startHealthChecks() {
    this.healthChecker.start();
  }
}
```

## 2. Service Discovery

### Service Discovery Implementation
```javascript
class ServiceDiscovery {
  constructor(options = {}) {
    this.options = {
      refreshInterval: options.refreshInterval || 30000,
      ...options
    };
    
    this.services = new Map();
    this.watchers = new Map();
    this.setupRefresh();
  }

  async registerService(service) {
    this.services.set(service.id, {
      ...service,
      status: 'healthy',
      lastSeen: Date.now()
    });
    
    // Notify watchers
    this.notifyWatchers(service.type);
  }

  async deregisterService(serviceId) {
    this.services.delete(serviceId);
  }

  async resolveService(request) {
    const serviceType = this.getServiceType(request);
    const services = this.getHealthyServices(serviceType);
    
    if (services.length === 0) {
      return null;
    }
    
    return this.selectService(services);
  }

  getHealthyServices(type) {
    return Array.from(this.services.values())
      .filter(s => s.type === type && s.status === 'healthy');
  }

  selectService(services) {
    // Implement service selection strategy
    return services[0];
  }

  watch(serviceType, callback) {
    if (!this.watchers.has(serviceType)) {
      this.watchers.set(serviceType, new Set());
    }
    
    this.watchers.get(serviceType).add(callback);
  }

  notifyWatchers(serviceType) {
    const watchers = this.watchers.get(serviceType);
    if (!watchers) return;
    
    const services = this.getHealthyServices(serviceType);
    for (const callback of watchers) {
      callback(services);
    }
  }

  setupRefresh() {
    setInterval(() => {
      this.refresh();
    }, this.options.refreshInterval);
  }

  async refresh() {
    const now = Date.now();
    for (const [id, service] of this.services) {
      if (now - service.lastSeen > this.options.refreshInterval * 2) {
        this.services.delete(id);
        this.notifyWatchers(service.type);
      }
    }
  }
}
```

## 3. Circuit Breaking

### Circuit Breaker Implementation
```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.options = {
      failureThreshold: options.failureThreshold || 5,
      resetTimeout: options.resetTimeout || 30000,
      requestTimeout: options.requestTimeout || 5000,
      ...options
    };
    
    this.state = 'closed';
    this.failures = 0;
    this.lastFailureTime = null;
  }

  async executeRequest(request) {
    if (!this.canExecute()) {
      throw new Error('Circuit is open');
    }
    
    try {
      const result = await this.executeWithTimeout(request);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  canExecute() {
    if (this.state === 'closed') {
      return true;
    }
    
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.options.resetTimeout) {
        this.state = 'half-open';
        return true;
      }
      return false;
    }
    
    return true;
  }

  async executeWithTimeout(request) {
    return Promise.race([
      request(),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Timeout')), this.options.requestTimeout)
      )
    ]);
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'closed';
  }

  onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();
    
    if (this.failures >= this.options.failureThreshold) {
      this.state = 'open';
    }
  }
}
```

## 4. Load Balancing

### Load Balancer Implementation
```javascript
class LoadBalancer {
  constructor(options = {}) {
    this.options = {
      strategy: options.strategy || 'round-robin',
      healthCheckInterval: options.healthCheckInterval || 5000,
      ...options
    };
    
    this.backends = new Map();
    this.healthChecker = new HealthChecker();
    this.setupHealthChecks();
  }

  addBackend(backend) {
    this.backends.set(backend.id, {
      ...backend,
      healthy: true,
      lastUsed: 0,
      activeRequests: 0
    });
  }

  removeBackend(backendId) {
    this.backends.delete(backendId);
  }

  async selectBackend() {
    const healthyBackends = Array.from(this.backends.values())
      .filter(b => b.healthy);
    
    if (healthyBackends.length === 0) {
      throw new Error('No healthy backends available');
    }
    
    switch (this.options.strategy) {
      case 'round-robin':
        return this.roundRobin(healthyBackends);
      case 'least-connections':
        return this.leastConnections(healthyBackends);
      case 'random':
        return this.random(healthyBackends);
      default:
        return this.roundRobin(healthyBackends);
    }
  }

  roundRobin(backends) {
    const sorted = backends.sort((a, b) => a.lastUsed - b.lastUsed);
    const selected = sorted[0];
    selected.lastUsed = Date.now();
    return selected;
  }

  leastConnections(backends) {
    return backends.sort((a, b) => 
      a.activeRequests - b.activeRequests
    )[0];
  }

  random(backends) {
    const index = Math.floor(Math.random() * backends.length);
    return backends[index];
  }

  setupHealthChecks() {
    setInterval(async () => {
      for (const [id, backend] of this.backends) {
        const healthy = await this.healthChecker.check(backend);
        backend.healthy = healthy;
      }
    }, this.options.healthCheckInterval);
  }
}
```

## 5. Observability

### Observability Implementation
```javascript
class Observability {
  constructor(options = {}) {
    this.options = {
      metricsInterval: options.metricsInterval || 5000,
      traceSampleRate: options.traceSampleRate || 0.1,
      ...options
    };
    
    this.metrics = new MetricsCollector();
    this.tracer = new Tracer();
    this.logger = new StructuredLogger();
  }

  middleware() {
    return async (req, res, next) => {
      const span = this.tracer.startSpan();
      const start = Date.now();
      
      // Add trace context
      req.traceContext = span.context();
      
      // Capture request metrics
      this.metrics.incrementCounter('http_requests_total');
      
      res.on('finish', () => {
        const duration = Date.now() - start;
        
        // Record metrics
        this.metrics.recordHistogram('http_request_duration', duration);
        this.metrics.incrementCounter('http_responses_total', {
          status: res.statusCode
        });
        
        // End span
        span.end();
        
        // Log request
        this.logger.info('Request completed', {
          method: req.method,
          path: req.path,
          status: res.statusCode,
          duration
        });
      });
      
      next();
    };
  }

  async collectMetrics() {
    const metrics = await this.metrics.collect();
    return {
      timestamp: Date.now(),
      metrics
    };
  }

  async getTraces() {
    return this.tracer.getTraces();
  }

  async getLogs(query) {
    return this.logger.query(query);
  }
}
```

## Related Topics
- [[Global-Scaling]] - Global Scale Architecture
- [[Multi-Region]] - Multi-Region Deployment
- [[Traffic-Management]] - Traffic Management
- [[High-Availability]] - High Availability

## Practice Projects
1. Build service mesh proxy
2. Implement service discovery
3. Create circuit breaker
4. Design observability system

## Resources
- [Service Mesh Architecture](https://istio.io/latest/docs/concepts/what-is-istio/)
- [[Learning-Resources#ServiceMesh|Service Mesh Resources]]

## Tags
#service-mesh #nodejs #microservices #observability #load-balancing
