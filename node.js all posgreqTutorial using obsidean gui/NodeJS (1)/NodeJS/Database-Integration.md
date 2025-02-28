# Database Integration in Node.js

Learn how to integrate various databases with Node.js applications.

## Popular Databases

### 1. MongoDB (NoSQL)
Using Mongoose ODM:
```javascript
const mongoose = require('mongoose');

// Connect to MongoDB
mongoose.connect('mongodb://localhost/myapp', {
    useNewUrlParser: true,
    useUnifiedTopology: true
});

// Define a schema
const userSchema = new mongoose.Schema({
    name: String,
    email: { type: String, required: true },
    createdAt: { type: Date, default: Date.now }
});

// Create and use a model
const User = mongoose.model('User', userSchema);
const user = new User({ name: 'John', email: 'john@example.com' });
await user.save();
```

### 2. PostgreSQL (SQL)
Using node-postgres:
```javascript
const { Pool } = require('pg');

const pool = new Pool({
    user: 'dbuser',
    host: 'localhost',
    database: 'myapp',
    password: 'password',
    port: 5432,
});

// Query example
async function getUsers() {
    const { rows } = await pool.query('SELECT * FROM users');
    return rows;
}
```

### 3. MySQL
Using mysql2:
```javascript
const mysql = require('mysql2/promise');

const connection = await mysql.createConnection({
    host: 'localhost',
    user: 'root',
    database: 'myapp'
});

const [rows] = await connection.execute(
    'SELECT * FROM users WHERE id = ?',
    [userId]
);
```

## Best Practices
1. Use connection pooling
2. Implement proper error handling
3. Use parameterized queries
4. Handle connection failures
5. Use ORMs when appropriate

## Security Considerations
- Prevent SQL injection
- Secure connection strings
- Implement proper access controls
- Regular backups
- Data encryption

## Related Topics
- [[Security]] - Database security
- [[ORMs]] - Object-Relational Mapping
- [[Performance]] - Database optimization
- [[Error-Handling-Advanced]] - Managing database errors

Tags: #nodejs #database #mongodb #postgresql #mysql
