# Testing Strategies in Node.js

A comprehensive guide to testing Node.js applications, covering unit testing, integration testing, end-to-end testing, performance testing, and security testing.

## 1. Unit Testing with Jest

### Basic Test Structure
```javascript
const { sum, multiply } = require('../math');

describe('Math Operations', () => {
    test('adds 1 + 2 to equal 3', () => {
        expect(sum(1, 2)).toBe(3);
    });

    test('multiplies 2 * 3 to equal 6', () => {
        expect(multiply(2, 3)).toBe(6);
    });
});
```

### Mocking Functions and Modules
```javascript
const UserService = require('../services/UserService');
const UserRepository = require('../repositories/UserRepository');

// Mock the repository
jest.mock('../repositories/UserRepository');

describe('UserService', () => {
    let userService;
    let mockRepository;

    beforeEach(() => {
        // Clear all mocks
        jest.clearAllMocks();
        
        // Create mock implementation
        mockRepository = {
            findById: jest.fn(),
            save: jest.fn(),
            delete: jest.fn()
        };

        // Initialize service with mock
        userService = new UserService(mockRepository);
    });

    describe('getUser', () => {
        test('should return user when found', async () => {
            const mockUser = { id: 1, name: 'John' };
            mockRepository.findById.mockResolvedValue(mockUser);

            const result = await userService.getUser(1);
            expect(result).toEqual(mockUser);
            expect(mockRepository.findById).toHaveBeenCalledWith(1);
        });

        test('should throw error when user not found', async () => {
            mockRepository.findById.mockResolvedValue(null);

            await expect(userService.getUser(1))
                .rejects
                .toThrow('User not found');
        });
    });
});
```

### Testing Async Code
```javascript
const { fetchUserData } = require('../api/users');

describe('API Calls', () => {
    test('fetchUserData returns user data', async () => {
        const userData = await fetchUserData(1);
        expect(userData).toHaveProperty('id');
        expect(userData).toHaveProperty('name');
    });

    test('fetchUserData handles errors', async () => {
        await expect(fetchUserData(-1))
            .rejects
            .toThrow('Invalid user ID');
    });
});
```

### Custom Matchers
```javascript
expect.extend({
    toBeWithinRange(received, floor, ceiling) {
        const pass = received >= floor && received <= ceiling;
        if (pass) {
            return {
                message: () =>
                    `expected ${received} not to be within range ${floor} - ${ceiling}`,
                pass: true
            };
        } else {
            return {
                message: () =>
                    `expected ${received} to be within range ${floor} - ${ceiling}`,
                pass: false
            };
        }
    }
});

test('numeric ranges', () => {
    expect(100).toBeWithinRange(90, 110);
});
```

## 2. Integration Testing

### API Testing with Supertest
```javascript
const request = require('supertest');
const app = require('../app');
const db = require('../db');

describe('User API', () => {
    beforeAll(async () => {
        await db.connect();
    });

    afterAll(async () => {
        await db.disconnect();
    });

    beforeEach(async () => {
        await db.clear();
    });

    describe('POST /api/users', () => {
        test('creates a new user', async () => {
            const userData = {
                name: 'John Doe',
                email: 'john@example.com'
            };

            const response = await request(app)
                .post('/api/users')
                .send(userData)
                .expect(201);

            expect(response.body).toHaveProperty('id');
            expect(response.body.name).toBe(userData.name);
            expect(response.body.email).toBe(userData.email);
        });

        test('validates required fields', async () => {
            const response = await request(app)
                .post('/api/users')
                .send({})
                .expect(400);

            expect(response.body.errors).toContain('Name is required');
            expect(response.body.errors).toContain('Email is required');
        });
    });
});
```

### Database Integration Tests
```javascript
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');
const UserModel = require('../models/User');

describe('User Model', () => {
    let mongoServer;

    beforeAll(async () => {
        mongoServer = await MongoMemoryServer.create();
        await mongoose.connect(mongoServer.getUri());
    });

    afterAll(async () => {
        await mongoose.disconnect();
        await mongoServer.stop();
    });

    beforeEach(async () => {
        await UserModel.deleteMany({});
    });

    test('creates & saves user successfully', async () => {
        const user = new UserModel({
            name: 'John Doe',
            email: 'john@example.com',
            password: 'password123'
        });

        const savedUser = await user.save();
        expect(savedUser._id).toBeDefined();
        expect(savedUser.name).toBe(user.name);
        expect(savedUser.email).toBe(user.email);
    });

    test('fails to save user without required fields', async () => {
        const userWithoutRequiredField = new UserModel({ name: 'John Doe' });
        let err;
        
        try {
            await userWithoutRequiredField.save();
        } catch (error) {
            err = error;
        }

        expect(err).toBeInstanceOf(mongoose.Error.ValidationError);
    });
});
```

## 3. End-to-End Testing with Cypress

### Basic E2E Test
```javascript
// cypress/integration/auth.spec.js
describe('Authentication Flow', () => {
    beforeEach(() => {
        cy.visit('/login');
    });

    it('should login successfully', () => {
        cy.get('[data-test=email-input]')
            .type('user@example.com');
        
        cy.get('[data-test=password-input]')
            .type('password123');
        
        cy.get('[data-test=login-button]')
            .click();

        cy.url().should('include', '/dashboard');
        cy.get('[data-test=welcome-message]')
            .should('contain', 'Welcome');
    });

    it('should show error for invalid credentials', () => {
        cy.get('[data-test=email-input]')
            .type('invalid@example.com');
        
        cy.get('[data-test=password-input]')
            .type('wrongpassword');
        
        cy.get('[data-test=login-button]')
            .click();

        cy.get('[data-test=error-message]')
            .should('be.visible')
            .and('contain', 'Invalid credentials');
    });
});
```

### Custom Commands
```javascript
// cypress/support/commands.js
Cypress.Commands.add('login', (email, password) => {
    cy.visit('/login');
    cy.get('[data-test=email-input]').type(email);
    cy.get('[data-test=password-input]').type(password);
    cy.get('[data-test=login-button]').click();
});

// Usage in test
describe('Protected Routes', () => {
    beforeEach(() => {
        cy.login('user@example.com', 'password123');
    });

    it('accesses protected route', () => {
        cy.visit('/protected');
        cy.get('[data-test=protected-content]')
            .should('be.visible');
    });
});
```

## 4. Performance Testing

### Load Testing with Artillery
```javascript
// artillery/scenarios.yml
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60
      arrivalRate: 5
      rampTo: 50
      name: "Ramp up load"

scenarios:
  - name: "API endpoints"
    flow:
      - get:
          url: "/api/users"
          headers:
            Authorization: "Bearer {{ token }}"
      - think: 1
      - post:
          url: "/api/orders"
          json:
            productId: "{{ productId }}"
            quantity: "{{ quantity }}"

  - name: "Search functionality"
    flow:
      - get:
          url: "/api/products/search?q=laptop"
      - think: 2
      - get:
          url: "/api/products/{{ productId }}"
```

### Custom Performance Tests
```javascript
const autocannon = require('autocannon');

async function runLoadTest() {
    const result = await autocannon({
        url: 'http://localhost:3000',
        connections: 100,
        duration: 30,
        pipelining: 1,
        requests: [
            {
                method: 'GET',
                path: '/api/users'
            },
            {
                method: 'POST',
                path: '/api/orders',
                headers: {
                    'content-type': 'application/json'
                },
                body: JSON.stringify({
                    productId: 1,
                    quantity: 1
                })
            }
        ]
    });

    console.log(result);
}

runLoadTest().catch(console.error);
```

## 5. Security Testing

### OWASP ZAP Integration
```javascript
const ZapClient = require('zaproxy');

class SecurityTester {
    constructor(options = {}) {
        this.zaproxy = new ZapClient({
            apiKey: options.apiKey,
            proxy: {
                host: 'localhost',
                port: 8080
            }
        });
    }

    async scanUrl(url) {
        try {
            // Start spider scan
            const spiderScan = await this.zaproxy.spider.scan({
                url,
                maxChildren: 10
            });

            // Wait for spider to complete
            await this.waitForSpider(spiderScan.scan);

            // Start active scan
            const activeScan = await this.zaproxy.ascan.scan({
                url,
                recurse: true
            });

            // Wait for active scan to complete
            await this.waitForActiveScan(activeScan.scan);

            // Get alerts
            const alerts = await this.zaproxy.core.alerts();
            return this.formatAlerts(alerts);
        } catch (error) {
            console.error('Security scan failed:', error);
            throw error;
        }
    }

    async waitForSpider(scanId) {
        while (true) {
            const status = await this.zaproxy.spider.status(scanId);
            if (status.status === 100) break;
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
    }

    async waitForActiveScan(scanId) {
        while (true) {
            const status = await this.zaproxy.ascan.status(scanId);
            if (status.status === 100) break;
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
    }

    formatAlerts(alerts) {
        return alerts.map(alert => ({
            risk: alert.risk,
            confidence: alert.confidence,
            url: alert.url,
            name: alert.name,
            description: alert.description,
            solution: alert.solution
        }));
    }
}
```

### Security Test Suite
```javascript
const { SecurityTester } = require('./SecurityTester');
const app = require('../app');

describe('Security Tests', () => {
    let server;
    let securityTester;

    beforeAll(async () => {
        server = app.listen(3000);
        securityTester = new SecurityTester({
            apiKey: process.env.ZAP_API_KEY
        });
    });

    afterAll(async () => {
        await server.close();
    });

    test('should not have high-risk vulnerabilities', async () => {
        const alerts = await securityTester.scanUrl(
            'http://localhost:3000'
        );

        const highRiskAlerts = alerts.filter(
            alert => alert.risk === 'High'
        );

        expect(highRiskAlerts).toHaveLength(0);
    });

    test('should have secure headers', async () => {
        const response = await fetch('http://localhost:3000');
        const headers = response.headers;

        expect(headers.get('x-frame-options')).toBe('DENY');
        expect(headers.get('x-xss-protection')).toBe('1; mode=block');
        expect(headers.get('x-content-type-options'))
            .toBe('nosniff');
    });
});
```

## 6. Test Coverage and Reporting

### Jest Coverage Configuration
```javascript
// jest.config.js
module.exports = {
    collectCoverage: true,
    coverageDirectory: 'coverage',
    coverageReporters: ['text', 'lcov', 'clover'],
    coverageThreshold: {
        global: {
            branches: 80,
            functions: 80,
            lines: 80,
            statements: 80
        }
    },
    testEnvironment: 'node',
    testMatch: ['**/__tests__/**/*.js', '**/?(*.)+(spec|test).js'],
    setupFilesAfterEnv: ['./jest.setup.js']
};
```

### Test Report Generator
```javascript
class TestReporter {
    constructor(globalConfig, options) {
        this._globalConfig = globalConfig;
        this._options = options;
    }

    onRunComplete(contexts, results) {
        console.log('\nTest Summary:');
        console.log('--------------');
        console.log(`Total Tests: ${results.numTotalTests}`);
        console.log(`Passed: ${results.numPassedTests}`);
        console.log(`Failed: ${results.numFailedTests}`);
        console.log(`Skipped: ${results.numPendingTests}`);
        console.log(`Duration: ${results.startTime}ms`);

        if (results.numFailedTests > 0) {
            console.log('\nFailed Tests:');
            results.testResults.forEach(testResult => {
                testResult.testResults.forEach(test => {
                    if (test.status === 'failed') {
                        console.log(`\n❌ ${test.fullName}`);
                        console.log(`   ${test.failureMessages[0]}`);
                    }
                });
            });
        }
    }
}
```

## Related Topics
- [[Test-Driven-Development]] - TDD practices
- [[API-Testing]] - API testing strategies
- [[Performance-Testing]] - Load and stress testing
- [[Security-Testing]] - Security testing tools

## Practice Projects
1. Build a test-driven REST API
2. Create an automated E2E test suite
3. Implement a performance testing framework
4. Develop a security testing pipeline

## Resources
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Cypress Documentation](https://docs.cypress.io)
- [[Learning-Resources#Testing|Testing Resources]]

## Tags
#testing #jest #cypress #security #performance
