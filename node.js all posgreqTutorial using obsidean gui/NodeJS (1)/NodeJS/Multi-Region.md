# Multi-Region Deployment in Node.js

A comprehensive guide to implementing multi-region deployment strategies for Node.js applications.

## 1. Region Management

### Region Manager Implementation
```javascript
class RegionManager {
  constructor(options = {}) {
    this.options = {
      primaryRegion: options.primaryRegion || 'us-east',
      failoverStrategy: options.failoverStrategy || 'auto',
      healthCheckInterval: options.healthCheckInterval || 30000,
      ...options
    };
    
    this.regions = new Map();
    this.healthStatus = new Map();
    this.setupHealthChecks();
  }

  addRegion(region) {
    this.regions.set(region.id, {
      ...region,
      status: 'active',
      priority: region.id === this.options.primaryRegion ? 1 : 2
    });
    
    this.healthStatus.set(region.id, {
      healthy: true,
      lastCheck: Date.now(),
      failures: 0
    });
  }

  async getActiveRegions() {
    return Array.from(this.regions.values())
      .filter(r => r.status === 'active' && this.isHealthy(r.id))
      .sort((a, b) => a.priority - b.priority);
  }

  isHealthy(regionId) {
    const status = this.healthStatus.get(regionId);
    return status && status.healthy;
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
      const response = await fetch(`${region.endpoint}/health`);
      this.updateHealthStatus(regionId, response.ok);
    } catch (error) {
      this.updateHealthStatus(regionId, false);
    }
  }

  updateHealthStatus(regionId, isHealthy) {
    const status = this.healthStatus.get(regionId);
    if (!status) return;
    
    status.lastCheck = Date.now();
    status.healthy = isHealthy;
    
    if (!isHealthy) {
      status.failures++;
      if (status.failures >= 3) {
        this.handleRegionFailure(regionId);
      }
    } else {
      status.failures = 0;
    }
  }

  async handleRegionFailure(regionId) {
    const region = this.regions.get(regionId);
    if (!region) return;
    
    region.status = 'inactive';
    
    if (regionId === this.options.primaryRegion) {
      await this.failover();
    }
  }

  async failover() {
    if (this.options.failoverStrategy !== 'auto') {
      return;
    }
    
    const activeRegions = await this.getActiveRegions();
    if (activeRegions.length > 0) {
      this.options.primaryRegion = activeRegions[0].id;
      activeRegions[0].priority = 1;
    }
  }
}
```

## 2. Database Replication

### Database Replication Manager
```javascript
class DatabaseReplicationManager {
  constructor(options = {}) {
    this.options = {
      replicationMode: options.replicationMode || 'async',
      consistencyLevel: options.consistencyLevel || 'eventual',
      retryAttempts: options.retryAttempts || 3,
      ...options
    };
    
    this.regions = new Map();
    this.replicationQueue = new Queue();
  }

  addRegion(region) {
    this.regions.set(region.id, {
      ...region,
      database: new DatabaseConnection(region.dbConfig)
    });
  }

  async write(data, options = {}) {
    const primaryRegion = this.getPrimaryRegion();
    
    // Write to primary
    await primaryRegion.database.write(data);
    
    // Replicate to secondaries
    if (this.options.replicationMode === 'sync') {
      await this.replicateSync(data);
    } else {
      this.replicateAsync(data);
    }
  }

  async replicateSync(data) {
    const secondaryRegions = this.getSecondaryRegions();
    const promises = secondaryRegions.map(region =>
      this.replicateToRegion(region, data)
    );
    
    await Promise.all(promises);
  }

  async replicateAsync(data) {
    const secondaryRegions = this.getSecondaryRegions();
    for (const region of secondaryRegions) {
      this.replicationQueue.enqueue({
        region,
        data,
        attempts: 0
      });
    }
  }

  async replicateToRegion(region, data) {
    let attempts = 0;
    while (attempts < this.options.retryAttempts) {
      try {
        await region.database.write(data);
        return;
      } catch (error) {
        attempts++;
        if (attempts === this.options.retryAttempts) {
          this.handleReplicationFailure(region, data);
        }
        await this.delay(Math.pow(2, attempts) * 1000);
      }
    }
  }

  async handleReplicationFailure(region, data) {
    // Log failure
    console.error(`Replication failed for region ${region.id}`);
    
    // Store for later retry
    this.replicationQueue.enqueue({
      region,
      data,
      attempts: 0
    });
  }

  getPrimaryRegion() {
    return Array.from(this.regions.values())
      .find(r => r.role === 'primary');
  }

  getSecondaryRegions() {
    return Array.from(this.regions.values())
      .filter(r => r.role === 'secondary');
  }
}
```

## 3. Data Synchronization

### Data Sync Manager
```javascript
class DataSyncManager {
  constructor(options = {}) {
    this.options = {
      syncInterval: options.syncInterval || 300000,
      batchSize: options.batchSize || 1000,
      ...options
    };
    
    this.regions = new Map();
    this.syncQueue = new Queue();
    this.setupSync();
  }

  addRegion(region) {
    this.regions.set(region.id, {
      ...region,
      lastSync: null,
      syncStatus: 'idle'
    });
  }

  setupSync() {
    setInterval(async () => {
      await this.syncAll();
    }, this.options.syncInterval);
  }

  async syncAll() {
    const regions = Array.from(this.regions.values());
    for (const region of regions) {
      if (region.syncStatus === 'idle') {
        await this.syncRegion(region);
      }
    }
  }

  async syncRegion(region) {
    region.syncStatus = 'syncing';
    
    try {
      const changes = await this.getChanges(region);
      for (const batch of this.batchChanges(changes)) {
        await this.applyChanges(region, batch);
      }
      
      region.lastSync = Date.now();
      region.syncStatus = 'idle';
    } catch (error) {
      region.syncStatus = 'error';
      this.handleSyncError(region, error);
    }
  }

  async getChanges(region) {
    const primaryRegion = this.getPrimaryRegion();
    return primaryRegion.database.getChangesSince(region.lastSync);
  }

  batchChanges(changes) {
    const batches = [];
    for (let i = 0; i < changes.length; i += this.options.batchSize) {
      batches.push(changes.slice(i, i + this.options.batchSize));
    }
    return batches;
  }

  async applyChanges(region, changes) {
    await region.database.applyChanges(changes);
  }

  handleSyncError(region, error) {
    console.error(`Sync failed for region ${region.id}:`, error);
    this.syncQueue.enqueue({
      region,
      retryAt: Date.now() + 300000 // 5 minutes
    });
  }
}
```

## 4. Deployment Orchestration

### Deployment Orchestrator
```javascript
class DeploymentOrchestrator {
  constructor(options = {}) {
    this.options = {
      deploymentStrategy: options.deploymentStrategy || 'rolling',
      healthCheckTimeout: options.healthCheckTimeout || 300000,
      rollbackThreshold: options.rollbackThreshold || 0.2,
      ...options
    };
    
    this.regions = new Map();
    this.deploymentStatus = new Map();
  }

  async deploy(version, config) {
    const regions = Array.from(this.regions.values());
    
    switch (this.options.deploymentStrategy) {
      case 'rolling':
        return this.rollingDeploy(regions, version, config);
      case 'blue-green':
        return this.blueGreenDeploy(regions, version, config);
      case 'canary':
        return this.canaryDeploy(regions, version, config);
    }
  }

  async rollingDeploy(regions, version, config) {
    for (const region of regions) {
      await this.deployToRegion(region, version, config);
      
      const isHealthy = await this.waitForHealth(region);
      if (!isHealthy) {
        await this.rollback(region);
        throw new Error(`Deployment failed in region ${region.id}`);
      }
    }
  }

  async blueGreenDeploy(regions, version, config) {
    // Create new environment
    const greenEnvironment = await this.createEnvironment(version, config);
    
    for (const region of regions) {
      await this.deployToEnvironment(region, greenEnvironment);
      
      const isHealthy = await this.waitForHealth(region);
      if (!isHealthy) {
        await this.rollback(region);
        throw new Error(`Deployment failed in region ${region.id}`);
      }
      
      await this.switchTraffic(region, greenEnvironment);
    }
  }

  async canaryDeploy(regions, version, config) {
    // Select canary region
    const canaryRegion = regions[0];
    
    // Deploy to canary
    await this.deployToRegion(canaryRegion, version, config);
    
    // Monitor canary
    const isCanaryHealthy = await this.monitorCanary(canaryRegion);
    if (!isCanaryHealthy) {
      await this.rollback(canaryRegion);
      throw new Error('Canary deployment failed');
    }
    
    // Deploy to remaining regions
    for (const region of regions.slice(1)) {
      await this.deployToRegion(region, version, config);
      
      const isHealthy = await this.waitForHealth(region);
      if (!isHealthy) {
        await this.rollback(region);
        throw new Error(`Deployment failed in region ${region.id}`);
      }
    }
  }

  async deployToRegion(region, version, config) {
    this.deploymentStatus.set(region.id, {
      status: 'deploying',
      version,
      startTime: Date.now()
    });
    
    try {
      await region.deployer.deploy(version, config);
      this.deploymentStatus.get(region.id).status = 'deployed';
    } catch (error) {
      this.deploymentStatus.get(region.id).status = 'failed';
      throw error;
    }
  }

  async waitForHealth(region) {
    const startTime = Date.now();
    while (Date.now() - startTime < this.options.healthCheckTimeout) {
      const isHealthy = await region.healthChecker.check();
      if (isHealthy) return true;
      await this.delay(5000);
    }
    return false;
  }

  async monitorCanary(region) {
    const metrics = await region.metrics.collect();
    return metrics.errorRate < this.options.rollbackThreshold;
  }

  async rollback(region) {
    const status = this.deploymentStatus.get(region.id);
    if (!status) return;
    
    try {
      await region.deployer.rollback(status.version);
      status.status = 'rolledback';
    } catch (error) {
      status.status = 'rollback_failed';
      throw error;
    }
  }
}
```

## Related Topics
- [[Global-Scaling]] - Global Scale Architecture
- [[Load-Balancing]] - Advanced Load Balancing
- [[Traffic-Management]] - Traffic Management
- [[High-Availability]] - High Availability

## Practice Projects
1. Build multi-region deployment system
2. Implement database replication
3. Create data synchronization system
4. Design deployment orchestrator

## Resources
- [Multi-Region Architecture](https://aws.amazon.com/solutions/implementations/multi-region-application-architecture/)
- [[Learning-Resources#MultiRegion|Multi-Region Resources]]

## Tags
#multi-region #nodejs #deployment #replication #orchestration
