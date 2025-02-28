# Testing in Node.js

Comprehensive guide to testing Node.js applications.

## Testing Frameworks

### 1. Jest
Popular testing framework by Facebook:
```javascript
// sum.js
module.exports = (a, b) => a + b;

// sum.test.js
const sum = require('./sum');

test('adds 1 + 2 to equal 3', () => {
    expect(sum(1, 2)).toBe(3);
});

// Async testing
test('async test', async () => {
    const data = await fetchData();
    expect(data).toBe('peanut butter');
});
```

### 2. Mocha with Chai
```javascript
const chai = require('chai');
const expect = chai.expect;

describe('Array', () => {
    describe('#indexOf()', () => {
        it('should return -1 when value is not present', () => {
            expect([1,2,3].indexOf(4)).to.equal(-1);
        });
    });
});
```

## Types of Tests

### 1. Unit Tests
Testing individual components:
```javascript
describe('UserService', () => {
    it('should create a new user', async () => {
        const user = await UserService.create({
            name: 'John',
            email: 'john@example.com'
        });
        expect(user.name).to.equal('John');
    });
});
```

### 2. Integration Tests
```javascript
const request = require('supertest');
const app = require('../app');

describe('POST /api/users', () => {
    it('should create a new user', async () => {
        const res = await request(app)
            .post('/api/users')
            .send({
                name: 'John',
                email: 'john@example.com'
            });
        expect(res.status).to.equal(201);
        expect(res.body).to.have.property('id');
    });
});
```

### 3. E2E Tests
Using Cypress or Puppeteer for full end-to-end testing.

## Test Coverage
```javascript
// Using Jest
{
    "scripts": {
        "test": "jest --coverage"
    }
}
```

## Mocking
```javascript
// Mock a function
jest.mock('../api');
api.fetchData = jest.fn().mockResolvedValue({ data: 'test' });

// Mock a module
jest.mock('axios');
```

## Best Practices
1. Write testable code
2. Use test-driven development (TDD)
3. Maintain test independence
4. Mock external dependencies
5. Aim for good coverage
6. Keep tests simple and readable

## Related Topics
- [[CI-CD]] - Continuous Integration/Deployment
- [[Error-Handling-Advanced]] - Testing error cases
- [[Code-Quality]] - Maintaining code quality
- [[Performance-Testing]] - Load and stress testing

Tags: #nodejs #testing #jest #mocha #tdd
