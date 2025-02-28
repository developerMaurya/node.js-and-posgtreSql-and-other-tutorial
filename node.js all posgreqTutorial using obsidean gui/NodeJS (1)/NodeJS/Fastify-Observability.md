# Fastify Observability Integration Guide

## 1. Observability Plugin

```javascript
// Observability Plugin
const fp = require('fastify-plugin');

async function observabilityPlugin(fastify, options) {
  const metrics = new MetricsCollector({
    prefix: 'app_',
    defaultLabels: {
      service: options.serviceName
    }
  });

  const tracer = new Tracer({
    serviceName: options.serviceName,
    exporter: options.tracingExporter
  });

  const logger = new LogAggregator({
    serviceName: options.serviceName,
    processors: options.logProcessors
  });

  // Register metrics
  const requestCounter = metrics.createCounter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'path', 'status']
  );

  const requestDuration = metrics.createHistogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'path'],
    [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
  );

  // Add hooks for metrics collection
  fastify.addHook('onRequest', async (request, reply) => {
    request.startTime = process.hrtime();

    // Start trace span
    const span = tracer.startSpan('http_request', {
      attributes: {
        'http.method': request.method,
        'http.url': request.url,
        'http.host': request.hostname
      }
    });

    request.span = span;
  });

  fastify.addHook('onResponse', async (request, reply) => {
    const hrtime = process.hrtime(request.startTime);
    const duration = hrtime[0] + hrtime[1] / 1e9;

    // Record metrics
    requestCounter.inc({
      method: request.method,
      path: request.routerPath || request.url,
      status: reply.statusCode
    });

    requestDuration.observe(
      {
        method: request.method,
        path: request.routerPath || request.url
      },
      duration
    );

    // End trace span
    if (request.span) {
      request.span.setAttributes({
        'http.status_code': reply.statusCode,
        'http.duration': duration
      });
      request.span.end();
    }

    // Log request
    logger.log({
      level: 'info',
      message: 'Request completed',
      method: request.method,
      path: request.url,
      statusCode: reply.statusCode,
      duration,
      traceId: request.span?.context().traceId
    });
  });

  // Error handling
  fastify.setErrorHandler(async (error, request, reply) => {
    // Record error metrics
    requestCounter.inc({
      method: request.method,
      path: request.routerPath || request.url,
      status: error.statusCode || 500
    });

    // Update trace span
    if (request.span) {
      request.span.setStatus({
        code: 'ERROR',
        message: error.message
      });
      request.span.recordException(error);
    }

    // Log error
    logger.log({
      level: 'error',
      message: 'Request failed',
      error: {
        message: error.message,
        stack: error.stack,
        name: error.name
      },
      method: request.method,
      path: request.url,
      traceId: request.span?.context().traceId
    });

    reply.status(error.statusCode || 500).send({
      error: error.name,
      message: error.message,
      statusCode: error.statusCode || 500
    });
  });

  // Health check endpoint
  fastify.get('/health', async (request, reply) => {
    const health = await checkHealth();
    reply.status(health.status === 'up' ? 200 : 503).send(health);
  });

  // Metrics endpoint
  fastify.get('/metrics', async (request, reply) => {
    const metricsData = await metrics.collect();
    reply.type('text/plain').send(metricsData);
  });

  // Decorate fastify instance
  fastify.decorate('metrics', metrics);
  fastify.decorate('tracer', tracer);
  fastify.decorate('logger', logger);
}

module.exports = fp(observabilityPlugin, {
  name: 'observability',
  fastify: '4.x'
});
```

## 2. Usage Example

```javascript
// Main Application
const fastify = require('fastify')({
  logger: true
});

// Register observability plugin
fastify.register(require('./plugins/observability'), {
  serviceName: 'user-service',
  tracingExporter: new JaegerExporter({
    endpoint: 'http://jaeger:14268/api/traces'
  }),
  logProcessors: [
    new ElasticsearchProcessor({
      node: 'http://elasticsearch:9200'
    })
  ]
});

// Example route with tracing
fastify.get('/users/:id', async (request, reply) => {
  const span = request.span;
  
  try {
    // Create child span for database operation
    const dbSpan = fastify.tracer.startSpan('db_query', {
      parent: span
    });

    // Perform database query
    const user = await fastify.db.users.findById(request.params.id);
    
    dbSpan.end();

    if (!user) {
      throw new Error('User not found');
    }

    return user;
  } catch (error) {
    // Error will be handled by error handler
    throw error;
  }
});

// Example route with custom metrics
fastify.post('/orders', async (request, reply) => {
  const span = request.span;
  
  // Create custom counter for orders
  const orderCounter = fastify.metrics.getCounter('orders_total');
  
  try {
    // Create child span for order processing
    const orderSpan = fastify.tracer.startSpan('process_order', {
      parent: span
    });

    // Process order
    const order = await processOrder(request.body);
    
    // Increment order counter
    orderCounter.inc({
      status: 'success',
      type: order.type
    });

    orderSpan.end();
    
    return order;
  } catch (error) {
    // Increment counter with error status
    orderCounter.inc({
      status: 'error',
      type: request.body.type
    });
    
    throw error;
  }
});

// Start server
const start = async () => {
  try {
    await fastify.listen({ port: 3000 });
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
```

## 3. Monitoring Dashboard Configuration

```javascript
// Grafana Dashboard Configuration
{
  "dashboard": {
    "title": "Fastify Service Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(app_http_requests_total[5m])",
            "legendFormat": "{{method}} {{path}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "heatmap",
        "targets": [
          {
            "expr": "rate(app_http_request_duration_seconds_bucket[5m])",
            "legendFormat": "{{le}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(app_http_requests_total{status=~\"5.*\"}[5m])",
            "legendFormat": "{{path}}"
          }
        ]
      }
    ]
  }
}
```

## Related Topics
- [[Fastify-Guide]] - Fastify Framework Guide
- [[Observability-Platform-Part1]] - Core Components
- [[Observability-Platform-Part2]] - Data Processing
- [[Observability-Platform-Part3]] - Storage
- [[Observability-Platform-Part4]] - Visualization

## Tags
#fastify #observability #monitoring #tracing #metrics
