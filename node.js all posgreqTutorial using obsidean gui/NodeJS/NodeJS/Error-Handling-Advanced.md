# Advanced Error Handling in Node.js

## 1. Custom Error System

### 1.1 Base Error Classes
```javascript
// Base Application Error
class ApplicationError extends Error {
  constructor(message, options = {}) {
    super(message);
    this.name = this.constructor.name;
    this.timestamp = new Date();
    this.code = options.code || 'INTERNAL_ERROR';
    this.httpStatus = options.httpStatus || 500;
    this.severity = options.severity || 'ERROR';
    this.context = options.context || {};
    this.cause = options.cause;
    
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, this.constructor);
    }
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      severity: this.severity,
      timestamp: this.timestamp,
      context: this.context,
      stack: this.stack,
      cause: this.cause
    };
  }
}

// Domain Specific Errors
class ValidationError extends ApplicationError {
  constructor(message, validationErrors, options = {}) {
    super(message, {
      ...options,
      code: 'VALIDATION_ERROR',
      httpStatus: 400,
      context: { validationErrors }
    });
    this.validationErrors = validationErrors;
  }
}

class BusinessError extends ApplicationError {
  constructor(message, options = {}) {
    super(message, {
      ...options,
      code: 'BUSINESS_RULE_VIOLATION',
      httpStatus: 422
    });
  }
}

class ConcurrencyError extends ApplicationError {
  constructor(message, options = {}) {
    super(message, {
      ...options,
      code: 'CONCURRENCY_CONFLICT',
      httpStatus: 409
    });
  }
}
```

### 1.2 Error Factory
```javascript
// Error Factory
class ErrorFactory {
  static createError(type, message, options = {}) {
    const errorMap = {
      validation: ValidationError,
      business: BusinessError,
      concurrency: ConcurrencyError,
      // Add more error types
    };

    const ErrorClass = errorMap[type] || ApplicationError;
    return new ErrorClass(message, options);
  }

  static wrapError(error, context = {}) {
    if (error instanceof ApplicationError) {
      error.context = { ...error.context, ...context };
      return error;
    }

    return new ApplicationError(error.message, {
      cause: error,
      context
    });
  }
}
```

## 2. Advanced Error Handling Patterns

### 2.1 Error Handler Middleware
```javascript
// Error Handler Middleware
class ErrorHandlerMiddleware {
  constructor(options = {}) {
    this.logger = options.logger;
    this.errorReporter = options.errorReporter;
    this.isDevelopment = options.isDevelopment || false;
  }

  handle() {
    return async (error, req, res, next) => {
      try {
        // Normalize error
        const normalizedError = this.normalizeError(error);

        // Log error
        await this.logError(normalizedError, req);

        // Report error if necessary
        if (this.shouldReportError(normalizedError)) {
          await this.reportError(normalizedError, req);
        }

        // Send response
        this.sendErrorResponse(normalizedError, req, res);
      } catch (handlingError) {
        // Fallback error handling
        this.handleFallback(handlingError, error, res);
      }
    };
  }

  normalizeError(error) {
    if (error instanceof ApplicationError) {
      return error;
    }
    return ErrorFactory.wrapError(error);
  }

  async logError(error, req) {
    const logContext = {
      requestId: req.id,
      url: req.url,
      method: req.method,
      userId: req.user?.id,
      ...error.context
    };

    if (error.severity === 'ERROR' || error.severity === 'CRITICAL') {
      this.logger.error(error.message, { error, ...logContext });
    } else {
      this.logger.warn(error.message, { error, ...logContext });
    }
  }

  shouldReportError(error) {
    return error.severity === 'ERROR' || error.severity === 'CRITICAL';
  }

  async reportError(error, req) {
    await this.errorReporter.report(error, {
      request: {
        id: req.id,
        url: req.url,
        method: req.method,
        headers: req.headers,
        query: req.query,
        body: req.body
      },
      user: req.user
    });
  }

  sendErrorResponse(error, req, res) {
    const response = {
      status: 'error',
      message: error.message,
      code: error.code,
      requestId: req.id
    };

    if (this.isDevelopment) {
      response.stack = error.stack;
      response.context = error.context;
    }

    if (error instanceof ValidationError) {
      response.validationErrors = error.validationErrors;
    }

    res.status(error.httpStatus).json(response);
  }

  handleFallback(handlingError, originalError, res) {
    this.logger.error('Error handler failed', {
      handlingError,
      originalError
    });

    res.status(500).json({
      status: 'error',
      message: 'Internal server error',
      code: 'INTERNAL_ERROR'
    });
  }
}
```

### 2.2 Async Error Boundary
```javascript
// Async Error Boundary
class AsyncErrorBoundary {
  static wrap(fn) {
    return async (req, res, next) => {
      try {
        await fn(req, res, next);
      } catch (error) {
        next(error);
      }
    };
  }

  static wrapClass(Controller) {
    const prototype = Controller.prototype;
    const methodNames = Object.getOwnPropertyNames(prototype)
      .filter(prop => typeof prototype[prop] === 'function');

    methodNames.forEach(methodName => {
      const originalMethod = prototype[methodName];
      prototype[methodName] = AsyncErrorBoundary.wrap(originalMethod);
    });

    return Controller;
  }
}
```

## 3. Error Recovery Patterns

### 3.1 Circuit Breaker
```javascript
// Circuit Breaker Implementation
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.failureCount = 0;
    this.lastFailureTime = null;
    this.state = 'CLOSED';
  }

  async execute(operation) {
    if (this.state === 'OPEN') {
      if (this.shouldReset()) {
        this.halfOpen();
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure(error);
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure(error) {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
    }

    throw error;
  }

  shouldReset() {
    return Date.now() - this.lastFailureTime >= this.resetTimeout;
  }

  halfOpen() {
    this.state = 'HALF_OPEN';
    this.failureCount = 0;
  }
}
```

### 3.2 Retry Mechanism
```javascript
// Advanced Retry Mechanism
class RetryMechanism {
  constructor(options = {}) {
    this.maxAttempts = options.maxAttempts || 3;
    this.backoffStrategy = options.backoffStrategy || 'exponential';
    this.baseDelay = options.baseDelay || 1000;
    this.maxDelay = options.maxDelay || 30000;
    this.retryableErrors = options.retryableErrors || [];
  }

  async execute(operation) {
    let lastError;
    let attempt = 0;

    while (attempt < this.maxAttempts) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (!this.isRetryable(error)) {
          throw error;
        }

        attempt++;
        if (attempt === this.maxAttempts) {
          break;
        }

        await this.delay(attempt);
      }
    }

    throw new Error(`Operation failed after ${attempt} attempts`, {
      cause: lastError
    });
  }

  isRetryable(error) {
    return this.retryableErrors.some(ErrorClass =>
      error instanceof ErrorClass
    );
  }

  async delay(attempt) {
    let delay;
    switch (this.backoffStrategy) {
      case 'linear':
        delay = this.baseDelay * attempt;
        break;
      case 'exponential':
        delay = this.baseDelay * Math.pow(2, attempt - 1);
        break;
      case 'fibonacci':
        delay = this.getFibonacciDelay(attempt);
        break;
      default:
        delay = this.baseDelay;
    }

    delay = Math.min(delay, this.maxDelay);
    await new Promise(resolve => setTimeout(resolve, delay));
  }

  getFibonacciDelay(n) {
    let prev = 0, curr = 1;
    for (let i = 0; i < n; i++) {
      const temp = curr;
      curr = prev + curr;
      prev = temp;
    }
    return this.baseDelay * prev;
  }
}
```

## 4. Error Monitoring and Analytics

### 4.1 Error Aggregator
```javascript
// Error Aggregator
class ErrorAggregator {
  constructor(options = {}) {
    this.storage = options.storage;
    this.analysisInterval = options.analysisInterval || 3600000;
    this.startAnalysis();
  }

  async recordError(error, context = {}) {
    const errorRecord = {
      name: error.name,
      message: error.message,
      code: error.code,
      stack: error.stack,
      context,
      timestamp: new Date()
    };

    await this.storage.store(errorRecord);
  }

  async analyzeErrors(timeWindow) {
    const errors = await this.storage.query({
      timeRange: {
        start: new Date(Date.now() - timeWindow),
        end: new Date()
      }
    });

    return {
      totalErrors: errors.length,
      errorsByType: this.groupByType(errors),
      errorsByCode: this.groupByCode(errors),
      timeDistribution: this.analyzeTimeDistribution(errors),
      patterns: this.findPatterns(errors)
    };
  }

  groupByType(errors) {
    return errors.reduce((acc, error) => {
      acc[error.name] = (acc[error.name] || 0) + 1;
      return acc;
    }, {});
  }

  groupByCode(errors) {
    return errors.reduce((acc, error) => {
      acc[error.code] = (acc[error.code] || 0) + 1;
      return acc;
    }, {});
  }

  analyzeTimeDistribution(errors) {
    // Implement time-based analysis
  }

  findPatterns(errors) {
    // Implement pattern recognition
  }

  startAnalysis() {
    setInterval(() => {
      this.analyzeErrors(this.analysisInterval)
        .then(analysis => {
          // Handle analysis results
        })
        .catch(error => {
          // Handle analysis error
        });
    }, this.analysisInterval);
  }
}
```

## Related Topics
- [[Error-Handling-Basics]] - Basic Error Handling
- [[Monitoring]] - System Monitoring
- [[Logging]] - Logging Implementation
- [[Testing]] - Testing Strategies

## Practice Projects
1. Implement custom error handling system
2. Create error monitoring dashboard
3. Build retry mechanism with different strategies
4. Develop error analytics system

## Resources
- [Node.js Error Handling Best Practices](https://nodejs.org/en/docs/guides/error-handling)
- [[Learning-Resources#ErrorHandling|Error Handling Resources]]

## Tags
#error-handling #nodejs #monitoring #reliability #resilience
