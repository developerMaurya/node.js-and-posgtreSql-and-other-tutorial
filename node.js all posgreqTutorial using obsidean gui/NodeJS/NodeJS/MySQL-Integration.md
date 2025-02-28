# MySQL Integration with Node.js

A comprehensive guide to integrating MySQL with Node.js applications using the mysql2 package and Sequelize ORM.

## Basic Setup

### 1. MySQL2 Setup
```javascript
const mysql = require('mysql2/promise');
require('dotenv').config();

// Connection configuration
const dbConfig = {
    host: process.env.DB_HOST || 'localhost',
    user: process.env.DB_USER || 'root',
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0
};

// Create connection pool
const pool = mysql.createPool(dbConfig);

// Test connection
async function testConnection() {
    try {
        const connection = await pool.getConnection();
        console.log('Connected to MySQL database');
        connection.release();
    } catch (error) {
        console.error('Error connecting to database:', error);
        throw error;
    }
}

testConnection();
```

### 2. Sequelize Setup
```javascript
const { Sequelize } = require('sequelize');

// Create Sequelize instance
const sequelize = new Sequelize(
    process.env.DB_NAME,
    process.env.DB_USER,
    process.env.DB_PASSWORD,
    {
        host: process.env.DB_HOST,
        dialect: 'mysql',
        pool: {
            max: 10,
            min: 0,
            acquire: 30000,
            idle: 10000
        },
        logging: false // Set to console.log for SQL query logging
    }
);

// Test connection
async function testSequelizeConnection() {
    try {
        await sequelize.authenticate();
        console.log('Connection established successfully.');
    } catch (error) {
        console.error('Unable to connect to database:', error);
        throw error;
    }
}

testSequelizeConnection();
```

## Model Definition

### 1. Sequelize Models
```javascript
const { DataTypes } = require('sequelize');

// User Model
const User = sequelize.define('User', {
    id: {
        type: DataTypes.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    username: {
        type: DataTypes.STRING,
        allowNull: false,
        unique: true,
        validate: {
            len: [3, 50]
        }
    },
    email: {
        type: DataTypes.STRING,
        allowNull: false,
        unique: true,
        validate: {
            isEmail: true
        }
    },
    password: {
        type: DataTypes.STRING,
        allowNull: false
    },
    role: {
        type: DataTypes.ENUM('user', 'admin'),
        defaultValue: 'user'
    },
    isActive: {
        type: DataTypes.BOOLEAN,
        defaultValue: true
    }
}, {
    timestamps: true,
    paranoid: true // Soft deletes
});

// Post Model
const Post = sequelize.define('Post', {
    id: {
        type: DataTypes.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    title: {
        type: DataTypes.STRING,
        allowNull: false
    },
    content: {
        type: DataTypes.TEXT,
        allowNull: false
    },
    status: {
        type: DataTypes.ENUM('draft', 'published'),
        defaultValue: 'draft'
    }
});

// Define relationships
User.hasMany(Post);
Post.belongsTo(User);
```

## CRUD Operations

### 1. Create Operations
```javascript
// Create single record
async function createUser(userData) {
    try {
        return await User.create(userData);
    } catch (error) {
        throw new Error(`Error creating user: ${error.message}`);
    }
}

// Create multiple records
async function createManyUsers(usersData) {
    try {
        return await User.bulkCreate(usersData, {
            validate: true
        });
    } catch (error) {
        throw new Error(`Error creating users: ${error.message}`);
    }
}
```

### 2. Read Operations
```javascript
// Find by ID
async function getUserById(id) {
    try {
        const user = await User.findByPk(id);
        if (!user) throw new Error('User not found');
        return user;
    } catch (error) {
        throw new Error(`Error fetching user: ${error.message}`);
    }
}

// Find with conditions
async function findUsers(query) {
    try {
        return await User.findAll({
            where: query,
            attributes: ['username', 'email', 'role'],
            order: [['createdAt', 'DESC']],
            limit: 10
        });
    } catch (error) {
        throw new Error(`Error finding users: ${error.message}`);
    }
}

// Find with associations
async function getPostWithAuthor(postId) {
    try {
        return await Post.findByPk(postId, {
            include: [{
                model: User,
                attributes: ['username', 'email']
            }]
        });
    } catch (error) {
        throw new Error(`Error fetching post: ${error.message}`);
    }
}
```

### 3. Update Operations
```javascript
// Update by ID
async function updateUser(id, updates) {
    try {
        const [updated] = await User.update(updates, {
            where: { id },
            returning: true
        });
        
        if (!updated) throw new Error('User not found');
        return await User.findByPk(id);
    } catch (error) {
        throw new Error(`Error updating user: ${error.message}`);
    }
}

// Update many
async function updateManyUsers(filter, updates) {
    try {
        const [count] = await User.update(updates, {
            where: filter
        });
        return count;
    } catch (error) {
        throw new Error(`Error updating users: ${error.message}`);
    }
}
```

### 4. Delete Operations
```javascript
// Delete by ID
async function deleteUser(id) {
    try {
        const deleted = await User.destroy({
            where: { id }
        });
        if (!deleted) throw new Error('User not found');
        return deleted;
    } catch (error) {
        throw new Error(`Error deleting user: ${error.message}`);
    }
}

// Delete many
async function deleteManyUsers(filter) {
    try {
        return await User.destroy({
            where: filter
        });
    } catch (error) {
        throw new Error(`Error deleting users: ${error.message}`);
    }
}
```

## Advanced Features

### 1. Hooks (Lifecycle Events)
```javascript
User.beforeCreate(async (user) => {
    if (user.password) {
        user.password = await bcrypt.hash(user.password, 10);
    }
});

User.afterCreate(async (user) => {
    console.log(`User ${user.username} created successfully`);
});
```

### 2. Instance Methods
```javascript
User.prototype.comparePassword = async function(candidatePassword) {
    return await bcrypt.compare(candidatePassword, this.password);
};

User.prototype.toJSON = function() {
    const values = { ...this.get() };
    delete values.password;
    return values;
};
```

### 3. Class Methods
```javascript
User.findByEmail = async function(email) {
    return await this.findOne({ where: { email } });
};

User.findActiveUsers = async function() {
    return await this.findAll({ where: { isActive: true } });
};
```

## Transactions

### 1. Basic Transaction
```javascript
async function createUserWithPosts(userData, posts) {
    const t = await sequelize.transaction();

    try {
        const user = await User.create(userData, { transaction: t });
        
        const postPromises = posts.map(post => 
            Post.create({ ...post, userId: user.id }, { transaction: t })
        );
        
        await Promise.all(postPromises);
        
        await t.commit();
        return user;
    } catch (error) {
        await t.rollback();
        throw new Error(`Transaction failed: ${error.message}`);
    }
}
```

### 2. Managed Transaction
```javascript
async function transferPoints(fromUserId, toUserId, points) {
    try {
        const result = await sequelize.transaction(async (t) => {
            const fromUser = await User.findByPk(fromUserId, { transaction: t });
            const toUser = await User.findByPk(toUserId, { transaction: t });

            if (fromUser.points < points) {
                throw new Error('Insufficient points');
            }

            await fromUser.decrement('points', { by: points, transaction: t });
            await toUser.increment('points', { by: points, transaction: t });

            return { fromUser, toUser };
        });

        return result;
    } catch (error) {
        throw new Error(`Transfer failed: ${error.message}`);
    }
}
```

## Raw Queries

### 1. Using Raw Queries
```javascript
async function complexQuery() {
    try {
        const [results] = await sequelize.query(`
            SELECT 
                users.username,
                COUNT(posts.id) as post_count,
                MAX(posts.createdAt) as last_post_date
            FROM users
            LEFT JOIN posts ON users.id = posts.userId
            GROUP BY users.id
            HAVING post_count > 0
            ORDER BY post_count DESC
        `, {
            type: QueryTypes.SELECT
        });
        
        return results;
    } catch (error) {
        throw new Error(`Query failed: ${error.message}`);
    }
}
```

## Best Practices

1. Use connection pools for better performance
2. Implement proper error handling
3. Use transactions for data integrity
4. Implement proper indexing
5. Use migrations for schema changes
6. Keep models organized and modular
7. Use environment variables for configuration
8. Implement proper logging

## Related Topics
- [[Database-Design]] - Database design principles
- [[API-Design]] - RESTful API design
- [[Error-Handling-Advanced]] - Error handling strategies
- [[Performance-Optimization]] - Database performance tips

Tags: #nodejs #mysql #sequelize #database #backend
