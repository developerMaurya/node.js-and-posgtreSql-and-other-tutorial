# Cloud Native Development with Node.js

A comprehensive guide to developing cloud-native applications using Node.js.

## 1. Container Management

### Docker Integration
```javascript
// Dockerfile Generator
class DockerfileGenerator {
  constructor(options = {}) {
    this.options = {
      baseImage: options.baseImage || 'node:18-alpine',
      workDir: options.workDir || '/app',
      ...options
    };
  }

  generate() {
    return `
FROM ${this.options.baseImage}

# Set working directory
WORKDIR ${this.options.workDir}

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application files
COPY . .

# Set environment variables
ENV NODE_ENV=production

# Expose port
EXPOSE ${this.options.port || 3000}

# Start application
CMD ["npm", "start"]
    `.trim();
  }

  generateDevelopment() {
    return `
FROM ${this.options.baseImage}

WORKDIR ${this.options.workDir}

COPY package*.json ./

RUN npm install

COPY . .

ENV NODE_ENV=development

EXPOSE ${this.options.port || 3000}

CMD ["npm", "run", "dev"]
    `.trim();
  }
}

// Docker Compose Generator
class DockerComposeGenerator {
  constructor(options = {}) {
    this.options = {
      version: options.version || '3.8',
      services: options.services || [],
      ...options
    };
  }

  generate() {
    return `
version: '${this.options.version}'

services:
  ${this.generateServices()}
    `.trim();
  }

  generateServices() {
    return this.options.services
      .map(service => this.generateService(service))
      .join('\n\n');
  }

  generateService(service) {
    return `
  ${service.name}:
    build:
      context: ${service.context || '.'}
      dockerfile: ${service.dockerfile || 'Dockerfile'}
    ports:
      - "${service.port}:${service.port}"
    environment:
      ${this.generateEnvironment(service.env)}
    ${service.dependencies ? `depends_on:
      ${service.dependencies.map(d => `- ${d}`).join('\n      ')}` : ''}
    `.trim();
  }

  generateEnvironment(env = {}) {
    return Object.entries(env)
      .map(([key, value]) => `${key}: ${value}`)
      .join('\n      ');
  }
}
```

## 2. Kubernetes Integration

### Kubernetes Resource Management
```javascript
// Kubernetes Manifest Generator
class KubernetesManifestGenerator {
  constructor(options = {}) {
    this.options = {
      apiVersion: options.apiVersion || 'apps/v1',
      ...options
    };
  }

  generateDeployment(service) {
    return {
      apiVersion: this.options.apiVersion,
      kind: 'Deployment',
      metadata: {
        name: service.name,
        labels: {
          app: service.name
        }
      },
      spec: {
        replicas: service.replicas || 1,
        selector: {
          matchLabels: {
            app: service.name
          }
        },
        template: {
          metadata: {
            labels: {
              app: service.name
            }
          },
          spec: {
            containers: [{
              name: service.name,
              image: service.image,
              ports: [{
                containerPort: service.port
              }],
              env: this.generateEnvironmentVariables(service.env),
              resources: service.resources || {
                limits: {
                  cpu: '500m',
                  memory: '512Mi'
                },
                requests: {
                  cpu: '200m',
                  memory: '256Mi'
                }
              }
            }]
          }
        }
      }
    };
  }

  generateService(service) {
    return {
      apiVersion: 'v1',
      kind: 'Service',
      metadata: {
        name: service.name
      },
      spec: {
        selector: {
          app: service.name
        },
        ports: [{
          port: service.port,
          targetPort: service.port
        }],
        type: service.serviceType || 'ClusterIP'
      }
    };
  }

  generateIngress(service) {
    return {
      apiVersion: 'networking.k8s.io/v1',
      kind: 'Ingress',
      metadata: {
        name: service.name,
        annotations: service.ingressAnnotations || {}
      },
      spec: {
        rules: [{
          host: service.host,
          http: {
            paths: [{
              path: service.path || '/',
              pathType: 'Prefix',
              backend: {
                service: {
                  name: service.name,
                  port: {
                    number: service.port
                  }
                }
              }
            }]
          }
        }]
      }
    };
  }

  generateEnvironmentVariables(env = {}) {
    return Object.entries(env).map(([name, value]) => ({
      name,
      value: String(value)
    }));
  }
}

// Kubernetes Client Wrapper
class KubernetesClient {
  constructor(config) {
    this.client = new K8s.Client({ config });
  }

  async createDeployment(deployment) {
    return this.client.apis.apps.v1.namespaces('default')
      .deployments.post({ body: deployment });
  }

  async updateDeployment(name, deployment) {
    return this.client.apis.apps.v1.namespaces('default')
      .deployments(name).put({ body: deployment });
  }

  async deleteDeployment(name) {
    return this.client.apis.apps.v1.namespaces('default')
      .deployments(name).delete();
  }

  async getDeploymentStatus(name) {
    const deployment = await this.client.apis.apps.v1
      .namespaces('default').deployments(name).get();
    
    return {
      availableReplicas: deployment.body.status.availableReplicas,
      readyReplicas: deployment.body.status.readyReplicas,
      updatedReplicas: deployment.body.status.updatedReplicas
    };
  }
}
```

## 3. Cloud Service Integration

### Cloud Storage Integration
```javascript
// Cloud Storage Adapter
class CloudStorageAdapter {
  constructor(provider, config) {
    this.provider = provider;
    this.client = this.initializeClient(config);
  }

  initializeClient(config) {
    switch (this.provider) {
      case 'aws':
        return new AWS.S3(config);
      case 'gcp':
        return new Storage(config);
      case 'azure':
        return BlobServiceClient.fromConnectionString(config.connectionString);
      default:
        throw new Error(`Unsupported provider: ${this.provider}`);
    }
  }

  async upload(bucket, key, data, options = {}) {
    switch (this.provider) {
      case 'aws':
        return this.client.upload({
          Bucket: bucket,
          Key: key,
          Body: data,
          ...options
        }).promise();
      
      case 'gcp':
        const bucket = this.client.bucket(bucket);
        const file = bucket.file(key);
        return file.save(data, options);
      
      case 'azure':
        const containerClient = this.client.getContainerClient(bucket);
        const blobClient = containerClient.getBlockBlobClient(key);
        return blobClient.upload(data, data.length, options);
    }
  }

  async download(bucket, key) {
    switch (this.provider) {
      case 'aws':
        const response = await this.client.getObject({
          Bucket: bucket,
          Key: key
        }).promise();
        return response.Body;
      
      case 'gcp':
        const [content] = await this.client
          .bucket(bucket)
          .file(key)
          .download();
        return content;
      
      case 'azure':
        const containerClient = this.client.getContainerClient(bucket);
        const blobClient = containerClient.getBlockBlobClient(key);
        const downloadResponse = await blobClient.download();
        return streamToBuffer(downloadResponse.readableStreamBody);
    }
  }
}

// Stream to Buffer utility
async function streamToBuffer(readableStream) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    readableStream.on('data', (data) => {
      chunks.push(data instanceof Buffer ? data : Buffer.from(data));
    });
    readableStream.on('end', () => {
      resolve(Buffer.concat(chunks));
    });
    readableStream.on('error', reject);
  });
}
```

## 4. Service Discovery

### Service Registry Implementation
```javascript
// Service Registry
class ServiceRegistry {
  constructor(options = {}) {
    this.options = {
      ttl: options.ttl || 30000,
      checkInterval: options.checkInterval || 10000,
      ...options
    };
    
    this.services = new Map();
    this.startHealthChecks();
  }

  register(service) {
    service.lastHeartbeat = Date.now();
    this.services.set(service.id, service);
  }

  deregister(serviceId) {
    this.services.delete(serviceId);
  }

  get(serviceId) {
    return this.services.get(serviceId);
  }

  getAll() {
    return Array.from(this.services.values());
  }

  getHealthy() {
    const now = Date.now();
    return Array.from(this.services.values())
      .filter(service => 
        now - service.lastHeartbeat <= this.options.ttl
      );
  }

  heartbeat(serviceId) {
    const service = this.services.get(serviceId);
    if (service) {
      service.lastHeartbeat = Date.now();
    }
  }

  startHealthChecks() {
    setInterval(() => {
      const now = Date.now();
      for (const [id, service] of this.services) {
        if (now - service.lastHeartbeat > this.options.ttl) {
          this.services.delete(id);
        }
      }
    }, this.options.checkInterval);
  }
}

// Service Discovery Client
class ServiceDiscoveryClient {
  constructor(registry) {
    this.registry = registry;
  }

  async findService(criteria) {
    const services = this.registry.getHealthy()
      .filter(service => this.matchesCriteria(service, criteria));
    
    if (services.length === 0) {
      throw new Error('No healthy services found');
    }
    
    return this.selectService(services);
  }

  matchesCriteria(service, criteria) {
    return Object.entries(criteria).every(([key, value]) =>
      service[key] === value
    );
  }

  selectService(services) {
    // Simple round-robin selection
    const index = Math.floor(Math.random() * services.length);
    return services[index];
  }
}
```

## 5. Configuration Management

### Cloud Config Implementation
```javascript
// Cloud Config Client
class CloudConfigClient {
  constructor(options = {}) {
    this.options = {
      refreshInterval: options.refreshInterval || 30000,
      ...options
    };
    
    this.config = new Map();
    this.watchers = new Map();
    this.startRefreshLoop();
  }

  async loadConfig() {
    try {
      const config = await this.fetchConfig();
      this.updateConfig(config);
    } catch (error) {
      console.error('Failed to load config:', error);
    }
  }

  async fetchConfig() {
    // Implement provider-specific config fetching
    throw new Error('Not implemented');
  }

  updateConfig(newConfig) {
    const changes = new Map();
    
    // Detect changes
    for (const [key, value] of Object.entries(newConfig)) {
      const oldValue = this.config.get(key);
      if (oldValue !== value) {
        changes.set(key, {
          oldValue,
          newValue: value
        });
      }
      this.config.set(key, value);
    }
    
    // Notify watchers
    if (changes.size > 0) {
      this.notifyWatchers(changes);
    }
  }

  watch(key, callback) {
    if (!this.watchers.has(key)) {
      this.watchers.set(key, new Set());
    }
    this.watchers.get(key).add(callback);
    
    return () => {
      this.watchers.get(key).delete(callback);
    };
  }

  notifyWatchers(changes) {
    for (const [key, change] of changes) {
      const watchers = this.watchers.get(key);
      if (watchers) {
        watchers.forEach(callback => {
          callback(change.newValue, change.oldValue);
        });
      }
    }
  }

  startRefreshLoop() {
    setInterval(() => {
      this.loadConfig();
    }, this.options.refreshInterval);
  }
}
```

## Related Topics
- [[System-Architecture]] - System Architecture
- [[DevOps-Advanced]] - Advanced DevOps
- [[Monitoring]] - System Monitoring
- [[Security]] - Security Implementation

## Practice Projects
1. Build containerized microservices
2. Implement Kubernetes deployment
3. Create cloud storage service
4. Design service discovery system

## Resources
- [Cloud Native Computing Foundation](https://www.cncf.io/)
- [[Learning-Resources#CloudNative|Cloud Native Resources]]

## Tags
#cloud-native #nodejs #kubernetes #docker #microservices
