# Global Scale Architecture in Node.js

A comprehensive guide to implementing globally scalable Node.js applications.

## 1. Global Load Balancer

### Load Balancer Implementation
```javascript
const dns = require('dns').promises;
const { createProxyServer } = require('http-proxy');

class GlobalLoadBalancer {
  constructor(options = {}) {
    this.options = {
      healthCheckInterval: options.healthCheckInterval || 30000,
      failureThreshold: options.failureThreshold || 3,
      successThreshold: options.successThreshold || 2,
      ...options
    };
    
    this.regions = new Map();
    this.healthStatus = new Map();
    this.proxy = createProxyServer({});
    
    this.setupHealthChecks();
  }

  addRegion(region) {
    this.regions.set(region.id, region);
    this.healthStatus.set(region.id, {
      healthy: true,
      failures: 0,
      successes: 0
    });
  }

  async route(req, res) {
    const region = await this.selectRegion(req);
    if (!region) {
      return res.status(503).json({ error: 'No healthy regions available' });
    }
    
    this.proxy.web(req, res, {
      target: region.url,
      timeout: 5000
    });
  }

  async selectRegion(req) {
    const clientIp = req.ip;
    const clientRegion = await this.getClientRegion(clientIp);
    
    // Get healthy regions
    const healthyRegions = Array.from(this.regions.values())
      .filter(r => this.isRegionHealthy(r.id));
    
    if (healthyRegions.length === 0) {
      return null;
    }
    
    // Find nearest region
    return this.findNearestRegion(clientRegion, healthyRegions);
  }

  async getClientRegion(ip) {
    // Implement IP geolocation
    return 'us-east';
  }

  isRegionHealthy(regionId) {
    const status = this.healthStatus.get(regionId);
    return status && status.healthy;
  }

  findNearestRegion(clientRegion, regions) {
    // Implement region proximity calculation
    return regions[0];
  }

  setupHealthChecks() {
    setInterval(async () => {
      for (const [regionId, region] of this.regions) {
        await this.checkRegionHealth(regionId, region);
      }
    }, this.options.healthCheckInterval);
  }

  async checkRegionHealth(regionId, region) {
    try {
      const response = await fetch(`${region.url}/health`);
      this.updateHealthStatus(regionId, response.ok);
    } catch (error) {
      this.updateHealthStatus(regionId, false);
    }
  }

  updateHealthStatus(regionId, isHealthy) {
    const status = this.healthStatus.get(regionId);
    if (!status) return;
    
    if (isHealthy) {
      status.successes++;
      status.failures = 0;
      if (status.successes >= this.options.successThreshold) {
        status.healthy = true;
      }
    } else {
      status.failures++;
      status.successes = 0;
      if (status.failures >= this.options.failureThreshold) {
        status.healthy = false;
      }
    }
  }
}
```

## 2. Global Data Distribution

### Data Distribution System
```javascript
class GlobalDataDistributor {
  constructor(options = {}) {
    this.options = {
      replicationMode: options.replicationMode || 'async',
      consistencyLevel: options.consistencyLevel || 'eventual',
      ...options
    };
    
    this.regions = new Map();
    this.primaryRegion = null;
  }

  addRegion(region) {
    this.regions.set(region.id, {
      ...region,
      dataStore: new RegionalDataStore(region.id)
    });
    
    if (!this.primaryRegion) {
      this.primaryRegion = region.id;
    }
  }

  async write(key, value, options = {}) {
    const primaryStore = this.regions.get(this.primaryRegion).dataStore;
    await primaryStore.write(key, value);
    
    if (this.options.replicationMode === 'sync') {
      await this.replicateSync(key, value);
    } else {
      this.replicateAsync(key, value);
    }
  }

  async read(key, options = {}) {
    const preferredRegion = options.region || this.getNearestRegion();
    const store = this.regions.get(preferredRegion).dataStore;
    
    if (this.options.consistencyLevel === 'strong') {
      return this.readWithStrongConsistency(key);
    }
    
    return store.read(key);
  }

  async replicateSync(key, value) {
    const promises = [];
    for (const [regionId, region] of this.regions) {
      if (regionId !== this.primaryRegion) {
        promises.push(region.dataStore.write(key, value));
      }
    }
    await Promise.all(promises);
  }

  async replicateAsync(key, value) {
    for (const [regionId, region] of this.regions) {
      if (regionId !== this.primaryRegion) {
        region.dataStore.write(key, value).catch(console.error);
      }
    }
  }

  async readWithStrongConsistency(key) {
    const values = await Promise.all(
      Array.from(this.regions.values())
        .map(r => r.dataStore.read(key))
    );
    
    // Implement consensus mechanism
    return values[0];
  }

  getNearestRegion() {
    // Implement region selection logic
    return this.primaryRegion;
  }
}

class RegionalDataStore {
  constructor(regionId) {
    this.regionId = regionId;
    this.store = new Map();
    this.version = 0;
  }

  async write(key, value) {
    this.version++;
    this.store.set(key, {
      value,
      version: this.version,
      timestamp: Date.now()
    });
  }

  async read(key) {
    return this.store.get(key);
  }
}
```

## 3. Cache Distribution

### Global Cache System
```javascript
class GlobalCacheSystem {
  constructor(options = {}) {
    this.options = {
      ttl: options.ttl || 3600,
      maxSize: options.maxSize || 1000,
      ...options
    };
    
    this.regions = new Map();
    this.router = new CacheRouter();
  }

  addRegion(region) {
    this.regions.set(region.id, {
      ...region,
      cache: new RegionalCache(this.options)
    });
  }

  async get(key, options = {}) {
    const region = this.router.getRegion(key, options);
    const cache = this.regions.get(region).cache;
    
    const value = await cache.get(key);
    if (value) {
      return value;
    }
    
    // Cache miss - check other regions
    return this.getFromOtherRegions(key, region);
  }

  async set(key, value, options = {}) {
    const region = this.router.getRegion(key, options);
    const cache = this.regions.get(region).cache;
    
    await cache.set(key, value, options);
    
    // Replicate to nearby regions
    await this.replicateToNearbyRegions(key, value, region);
  }

  async getFromOtherRegions(key, excludeRegion) {
    const nearbyRegions = this.router.getNearbyRegions(excludeRegion);
    
    for (const regionId of nearbyRegions) {
      const cache = this.regions.get(regionId).cache;
      const value = await cache.get(key);
      if (value) {
        // Replicate to original region
        const originalCache = this.regions.get(excludeRegion).cache;
        await originalCache.set(key, value);
        return value;
      }
    }
    
    return null;
  }

  async replicateToNearbyRegions(key, value, sourceRegion) {
    const nearbyRegions = this.router.getNearbyRegions(sourceRegion);
    
    const promises = nearbyRegions.map(regionId => {
      const cache = this.regions.get(regionId).cache;
      return cache.set(key, value);
    });
    
    await Promise.all(promises);
  }
}

class RegionalCache {
  constructor(options) {
    this.cache = new Map();
    this.options = options;
  }

  async get(key) {
    const entry = this.cache.get(key);
    if (!entry) return null;
    
    if (Date.now() > entry.expiry) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.value;
  }

  async set(key, value, options = {}) {
    const ttl = options.ttl || this.options.ttl;
    
    this.cache.set(key, {
      value,
      expiry: Date.now() + (ttl * 1000)
    });
    
    // Cleanup if cache is too large
    if (this.cache.size > this.options.maxSize) {
      this.cleanup();
    }
  }

  cleanup() {
    const entries = Array.from(this.cache.entries());
    entries.sort((a, b) => a[1].expiry - b[1].expiry);
    
    const deleteCount = Math.floor(this.options.maxSize * 0.2);
    for (let i = 0; i < deleteCount; i++) {
      this.cache.delete(entries[i][0]);
    }
  }
}
```

## 4. Request Routing

### Global Request Router
```javascript
class GlobalRequestRouter {
  constructor(options = {}) {
    this.options = {
      routingStrategy: options.routingStrategy || 'latency',
      ...options
    };
    
    this.regions = new Map();
    this.metrics = new MetricsCollector();
  }

  addRegion(region) {
    this.regions.set(region.id, {
      ...region,
      metrics: {
        latency: new MovingAverage(100),
        errorRate: new MovingAverage(100),
        requestCount: 0
      }
    });
  }

  async routeRequest(request) {
    const clientRegion = await this.getClientRegion(request);
    const availableRegions = this.getAvailableRegions();
    
    switch (this.options.routingStrategy) {
      case 'latency':
        return this.routeByLatency(availableRegions, clientRegion);
      case 'geoproximity':
        return this.routeByGeoproximity(availableRegions, clientRegion);
      case 'loadbalance':
        return this.routeByLoad(availableRegions);
      default:
        return this.routeByLatency(availableRegions, clientRegion);
    }
  }

  async getClientRegion(request) {
    // Implement client region detection
    return 'us-east';
  }

  getAvailableRegions() {
    return Array.from(this.regions.entries())
      .filter(([_, region]) => region.metrics.errorRate.average() < 0.1)
      .map(([id, region]) => ({
        id,
        ...region
      }));
  }

  routeByLatency(regions, clientRegion) {
    return regions
      .sort((a, b) => 
        a.metrics.latency.average() - b.metrics.latency.average()
      )[0];
  }

  routeByGeoproximity(regions, clientRegion) {
    return regions
      .sort((a, b) => 
        this.calculateDistance(clientRegion, a.location) -
        this.calculateDistance(clientRegion, b.location)
      )[0];
  }

  routeByLoad(regions) {
    return regions
      .sort((a, b) => 
        a.metrics.requestCount - b.metrics.requestCount
      )[0];
  }

  calculateDistance(point1, point2) {
    // Implement distance calculation
    return 0;
  }

  async recordMetrics(regionId, metrics) {
    const region = this.regions.get(regionId);
    if (!region) return;
    
    region.metrics.latency.add(metrics.latency);
    region.metrics.errorRate.add(metrics.error ? 1 : 0);
    region.metrics.requestCount++;
  }
}

class MovingAverage {
  constructor(size) {
    this.size = size;
    this.values = [];
  }

  add(value) {
    this.values.push(value);
    if (this.values.length > this.size) {
      this.values.shift();
    }
  }

  average() {
    if (this.values.length === 0) return 0;
    return this.values.reduce((a, b) => a + b) / this.values.length;
  }
}
```

## Related Topics
- [[Multi-Region]] - Multi-Region Deployment
- [[Load-Balancing]] - Advanced Load Balancing
- [[Traffic-Management]] - Traffic Management
- [[High-Availability]] - High Availability

## Practice Projects
1. Build global load balancer
2. Implement data distribution system
3. Create global cache system
4. Design request routing system

## Resources
- [Global Scale Architecture](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/web-scale)
- [[Learning-Resources#GlobalScale|Global Scale Resources]]

## Tags
#global-scale #nodejs #architecture #distributed-systems #load-balancing
