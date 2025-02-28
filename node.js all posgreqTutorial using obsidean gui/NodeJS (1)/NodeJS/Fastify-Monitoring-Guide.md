# Fastify Monitoring and Feature Flag Guide

## 1. Prometheus Integration

### 1.1 Basic Setup

```javascript
// prometheus-plugin.js
const fp = require('fastify-plugin');
const prometheus = require('prom-client');

async function prometheusPlugin(fastify, opts) {
  // Enable default metrics collection
  const collectDefaultMetrics = prometheus.collectDefaultMetrics;
  collectDefaultMetrics({ prefix: 'node_' });

  // Create custom metrics
  const httpRequestDuration = new prometheus.Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route', 'status_code'],
    buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
  });

  const httpRequestTotal = new prometheus.Counter({
    name: 'http_request_total',
    help: 'Total number of HTTP requests',
    labelNames: ['method', 'route', 'status_code']
  });

  // Add timing hooks
  fastify.addHook('onRequest', (request, reply, done) => {
    request.startTime = process.hrtime();
    done();
  });

  fastify.addHook('onResponse', (request, reply, done) => {
    const hrtime = process.hrtime(request.startTime);
    const responseTimeInSeconds = hrtime[0] + hrtime[1] / 1e9;

    httpRequestDuration.labels(
      request.method,
      request.routerPath || request.raw.url,
      reply.statusCode
    ).observe(responseTimeInSeconds);

    httpRequestTotal.labels(
      request.method,
      request.routerPath || request.raw.url,
      reply.statusCode
    ).inc();

    done();
  });

  // Expose metrics endpoint
  fastify.get('/metrics', async (request, reply) => {
    reply.header('Content-Type', prometheus.register.contentType);
    return prometheus.register.metrics();
  });

  // Decorate fastify instance with prometheus registry
  fastify.decorate('prometheus', prometheus);
}

module.exports = fp(prometheusPlugin);
```

### 1.2 Custom Business Metrics

```javascript
// business-metrics.js
const fp = require('fastify-plugin');

async function businessMetrics(fastify, opts) {
  const prometheus = fastify.prometheus;

  // Business-specific metrics
  const activeUsers = new prometheus.Gauge({
    name: 'business_active_users',
    help: 'Number of currently active users'
  });

  const orderValue = new prometheus.Histogram({
    name: 'business_order_value_dollars',
    help: 'Value of orders in dollars',
    buckets: [10, 50, 100, 500, 1000, 5000]
  });

  const userRegistrations = new prometheus.Counter({
    name: 'business_user_registrations_total',
    help: 'Total number of user registrations'
  });

  // Decorate fastify with business metrics
  fastify.decorate('businessMetrics', {
    activeUsers,
    orderValue,
    userRegistrations
  });
}

module.exports = fp(businessMetrics);
```

## 2. Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "id": null,
    "title": "Fastify Application Dashboard",
    "tags": ["fastify", "nodejs"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Request Duration",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ]
      },
      {
        "title": "Request Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_request_total[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_request_total{status_code=~\"5.*\"}[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ]
      },
      {
        "title": "Business Metrics",
        "type": "stat",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "business_active_users",
            "legendFormat": "Active Users"
          },
          {
            "expr": "rate(business_user_registrations_total[24h])",
            "legendFormat": "Registrations/Day"
          }
        ]
      }
    ]
  }
}
```

## 3. Flagsmith Integration

### 3.1 Basic Setup

```javascript
// flagsmith-plugin.js
const fp = require('fastify-plugin');
const { Flagsmith } = require('flagsmith-nodejs');

async function flagsmithPlugin(fastify, opts) {
  const flagsmith = new Flagsmith({
    environmentKey: opts.environmentKey,
    enableAnalytics: true,
    defaultFlags: opts.defaultFlags || {}
  });

  // Initialize Flagsmith
  await flagsmith.init();

  // Add hook to identify user
  fastify.addHook('preHandler', async (request, reply) => {
    if (request.user) {
      request.flagsmith = await flagsmith.getIdentity(request.user.id, {
        traits: {
          email: request.user.email,
          plan: request.user.plan
        }
      });
    } else {
      request.flagsmith = await flagsmith.getEnvironmentFlags();
    }
  });

  // Decorate fastify with flagsmith instance
  fastify.decorate('flagsmith', flagsmith);
}

module.exports = fp(flagsmithPlugin);
```

### 3.2 Feature Flag Usage

```javascript
// routes/features.js
async function featureRoutes(fastify, opts) {
  fastify.get('/api/features', async (request, reply) => {
    const isNewFeatureEnabled = request.flagsmith.isFeatureEnabled('new_feature');
    const betaValue = request.flagsmith.getValue('beta_percentage');

    return {
      features: {
        newFeature: isNewFeatureEnabled,
        betaPercentage: betaValue
      }
    };
  });

  fastify.get('/api/premium-feature', async (request, reply) => {
    if (!request.flagsmith.isFeatureEnabled('premium_access')) {
      reply.code(403).send({ error: 'Feature not available' });
      return;
    }

    // Premium feature implementation
    return { data: 'Premium content' };
  });
}

module.exports = featureRoutes;
```

## 4. Advanced Logging Configuration

### 4.1 Structured Logging

```javascript
// logging-config.js
const fp = require('fastify-plugin');

async function loggingConfig(fastify, opts) {
  // Configure custom serializers
  fastify.addHook('onRequest', async (request, reply) => {
    request.log = request.log.child({
      requestId: request.id,
      userId: request.user?.id,
      path: request.routerPath,
      service: opts.serviceName
    });
  });

  // Add error logging
  fastify.setErrorHandler(async (error, request, reply) => {
    request.log.error({
      err: error,
      stack: error.stack,
      params: request.params,
      body: request.body,
      query: request.query
    }, 'Request failed');

    reply.status(error.statusCode || 500).send({
      error: error.name,
      message: error.message,
      statusCode: error.statusCode || 500
    });
  });

  // Add performance logging
  fastify.addHook('onResponse', async (request, reply) => {
    const responseTime = reply.getResponseTime();
    
    request.log.info({
      responseTime,
      statusCode: reply.statusCode,
      method: request.method,
      url: request.url,
      userAgent: request.headers['user-agent']
    }, 'Request completed');
  });
}

module.exports = fp(loggingConfig);
```

### 4.2 Log Aggregation Setup

```javascript
// log-aggregation.js
const fp = require('fastify-plugin');
const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');

async function logAggregation(fastify, opts) {
  const esTransport = new ElasticsearchTransport({
    level: 'info',
    index: opts.elasticsearchIndex,
    clientOpts: {
      node: opts.elasticsearchUrl,
      auth: {
        username: opts.elasticsearchUsername,
        password: opts.elasticsearchPassword
      }
    }
  });

  const logger = winston.createLogger({
    transports: [
      new winston.transports.Console(),
      esTransport
    ]
  });

  // Replace Fastify logger
  fastify.decorate('logger', logger);
}

module.exports = fp(logAggregation);
```

## 5. Application Integration

```javascript
// app.js
const fastify = require('fastify')({
  logger: {
    level: 'info',
    serializers: {
      req: (req) => ({
        method: req.method,
        url: req.url,
        headers: req.headers,
        hostname: req.hostname,
        remoteAddress: req.ip
      })
    }
  }
});

// Register plugins
async function start() {
  // Prometheus
  await fastify.register(require('./plugins/prometheus-plugin'));
  await fastify.register(require('./plugins/business-metrics'));

  // Flagsmith
  await fastify.register(require('./plugins/flagsmith-plugin'), {
    environmentKey: process.env.FLAGSMITH_KEY,
    defaultFlags: {
      new_feature: false,
      beta_percentage: 0
    }
  });

  // Logging
  await fastify.register(require('./plugins/logging-config'), {
    serviceName: 'my-service'
  });

  await fastify.register(require('./plugins/log-aggregation'), {
    elasticsearchUrl: process.env.ELASTICSEARCH_URL,
    elasticsearchIndex: 'fastify-logs',
    elasticsearchUsername: process.env.ELASTICSEARCH_USERNAME,
    elasticsearchPassword: process.env.ELASTICSEARCH_PASSWORD
  });

  // Start server
  try {
    await fastify.listen({ port: 3000 });
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
}

start();
```

## 6. Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - FLAGSMITH_KEY=your_key
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=changeme
    depends_on:
      - prometheus
      - grafana
      - elasticsearch

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.15.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
```

## Related Topics
- [[Fastify-Guide]] - Main Fastify guide
- [[Performance-Optimization]] - Performance tips
- [[Security]] - Security best practices
- [[Observability-Platform-Part1]] - Core observability components

## Tags
#fastify #monitoring #prometheus #grafana #flagsmith #logging #observability
