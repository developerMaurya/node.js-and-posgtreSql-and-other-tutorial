# MongoDB Integration with Node.js

A comprehensive guide to integrating MongoDB with Node.js applications using Mongoose ODM (Object Data Modeling).

## Basic Setup

### 1. Installation and Connection
```javascript
const mongoose = require('mongoose');
require('dotenv').config();

// Connection URI
const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017/myapp';

// Connection options
const options = {
    useNewUrlParser: true,
    useUnifiedTopology: true,
    serverSelectionTimeoutMS: 5000
};

// Connect to MongoDB
mongoose.connect(MONGODB_URI, options)
    .then(() => console.log('Connected to MongoDB'))
    .catch(err => console.error('MongoDB connection error:', err));

// Handle connection events
mongoose.connection.on('connected', () => {
    console.log('Mongoose connected to MongoDB');
});

mongoose.connection.on('error', (err) => {
    console.error('Mongoose connection error:', err);
});

mongoose.connection.on('disconnected', () => {
    console.log('Mongoose disconnected from MongoDB');
});

// Graceful shutdown
process.on('SIGINT', async () => {
    await mongoose.connection.close();
    process.exit(0);
});
```

## Schema and Model Design

### 1. Basic Schema Definition
```javascript
const mongoose = require('mongoose');

// User Schema
const userSchema = new mongoose.Schema({
    username: {
        type: String,
        required: true,
        unique: true,
        trim: true,
        minlength: 3
    },
    email: {
        type: String,
        required: true,
        unique: true,
        lowercase: true,
        validate: {
            validator: function(v) {
                return /\S+@\S+\.\S+/.test(v);
            },
            message: props => `${props.value} is not a valid email!`
        }
    },
    password: {
        type: String,
        required: true,
        minlength: 6
    },
    role: {
        type: String,
        enum: ['user', 'admin'],
        default: 'user'
    },
    isActive: {
        type: Boolean,
        default: true
    },
    createdAt: {
        type: Date,
        default: Date.now
    }
}, {
    timestamps: true
});

// Create model
const User = mongoose.model('User', userSchema);
```

### 2. Schema with Relationships
```javascript
// Post Schema with references
const postSchema = new mongoose.Schema({
    title: {
        type: String,
        required: true,
        trim: true
    },
    content: {
        type: String,
        required: true
    },
    author: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User',
        required: true
    },
    categories: [{
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Category'
    }],
    comments: [{
        user: {
            type: mongoose.Schema.Types.ObjectId,
            ref: 'User'
        },
        text: String,
        createdAt: {
            type: Date,
            default: Date.now
        }
    }],
    tags: [String],
    status: {
        type: String,
        enum: ['draft', 'published'],
        default: 'draft'
    }
}, {
    timestamps: true
});

const Post = mongoose.model('Post', postSchema);
```

## CRUD Operations

### 1. Create Operations
```javascript
// Create single document
async function createUser(userData) {
    try {
        const user = new User(userData);
        return await user.save();
    } catch (error) {
        throw new Error(`Error creating user: ${error.message}`);
    }
}

// Create multiple documents
async function createManyUsers(usersData) {
    try {
        return await User.insertMany(usersData);
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
        const user = await User.findById(id);
        if (!user) throw new Error('User not found');
        return user;
    } catch (error) {
        throw new Error(`Error fetching user: ${error.message}`);
    }
}

// Find with conditions
async function findUsers(query) {
    try {
        return await User.find(query)
            .select('username email role')
            .sort({ createdAt: -1 })
            .limit(10);
    } catch (error) {
        throw new Error(`Error finding users: ${error.message}`);
    }
}

// Populate references
async function getPostWithDetails(postId) {
    try {
        return await Post.findById(postId)
            .populate('author', 'username email')
            .populate('categories', 'name')
            .populate('comments.user', 'username');
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
        const options = { new: true, runValidators: true };
        const user = await User.findByIdAndUpdate(id, updates, options);
        if (!user) throw new Error('User not found');
        return user;
    } catch (error) {
        throw new Error(`Error updating user: ${error.message}`);
    }
}

// Update many
async function updateManyUsers(filter, updates) {
    try {
        return await User.updateMany(filter, updates);
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
        const user = await User.findByIdAndDelete(id);
        if (!user) throw new Error('User not found');
        return user;
    } catch (error) {
        throw new Error(`Error deleting user: ${error.message}`);
    }
}

// Delete many
async function deleteManyUsers(filter) {
    try {
        return await User.deleteMany(filter);
    } catch (error) {
        throw new Error(`Error deleting users: ${error.message}`);
    }
}
```

## Advanced Features

### 1. Middleware (Hooks)
```javascript
// Pre-save middleware
userSchema.pre('save', async function(next) {
    if (this.isModified('password')) {
        this.password = await bcrypt.hash(this.password, 10);
    }
    next();
});

// Post-save middleware
userSchema.post('save', function(doc, next) {
    console.log(`User ${doc.username} has been saved`);
    next();
});
```

### 2. Instance Methods
```javascript
userSchema.methods.comparePassword = async function(candidatePassword) {
    return await bcrypt.compare(candidatePassword, this.password);
};

userSchema.methods.generateAuthToken = function() {
    return jwt.sign({ id: this._id }, process.env.JWT_SECRET, {
        expiresIn: '1h'
    });
};
```

### 3. Static Methods
```javascript
userSchema.statics.findByEmail = function(email) {
    return this.findOne({ email });
};

userSchema.statics.findActiveUsers = function() {
    return this.find({ isActive: true });
};
```

### 4. Query Helpers
```javascript
userSchema.query.byRole = function(role) {
    return this.where({ role });
};

userSchema.query.activeOnly = function() {
    return this.where({ isActive: true });
};
```

## Aggregation Pipeline

### 1. Basic Aggregation
```javascript
async function getUserStats() {
    try {
        return await User.aggregate([
            {
                $group: {
                    _id: '$role',
                    count: { $sum: 1 },
                    avgCreatedAt: { $avg: '$createdAt' }
                }
            },
            {
                $sort: { count: -1 }
            }
        ]);
    } catch (error) {
        throw new Error(`Error getting user stats: ${error.message}`);
    }
}
```

### 2. Complex Aggregation
```javascript
async function getPostAnalytics() {
    try {
        return await Post.aggregate([
            {
                $match: {
                    status: 'published'
                }
            },
            {
                $lookup: {
                    from: 'users',
                    localField: 'author',
                    foreignField: '_id',
                    as: 'authorDetails'
                }
            },
            {
                $unwind: '$authorDetails'
            },
            {
                $group: {
                    _id: '$authorDetails._id',
                    authorName: { $first: '$authorDetails.username' },
                    totalPosts: { $sum: 1 },
                    avgComments: { $avg: { $size: '$comments' } }
                }
            },
            {
                $sort: { totalPosts: -1 }
            }
        ]);
    } catch (error) {
        throw new Error(`Error getting post analytics: ${error.message}`);
    }
}
```

## Error Handling

### 1. Custom Error Handler
```javascript
const handleMongoError = (error) => {
    if (error.name === 'ValidationError') {
        const errors = Object.values(error.errors).map(err => err.message);
        return {
            status: 400,
            message: 'Validation Error',
            errors
        };
    }

    if (error.code === 11000) {
        const field = Object.keys(error.keyPattern)[0];
        return {
            status: 409,
            message: `Duplicate ${field}`,
            error: `${field} already exists`
        };
    }

    return {
        status: 500,
        message: 'Database Error',
        error: error.message
    };
};
```

## Best Practices

1. Always use try-catch blocks for async operations
2. Implement proper indexing for better performance
3. Use schema validation for data integrity
4. Implement proper error handling
5. Use middleware for common operations
6. Keep models organized and modular
7. Use environment variables for configuration
8. Implement proper connection handling

## Related Topics
- [[Database-Design]] - Database design principles
- [[API-Design]] - RESTful API design
- [[Error-Handling-Advanced]] - Error handling strategies
- [[Performance-Optimization]] - Database performance tips

Tags: #nodejs #mongodb #mongoose #database #backend
