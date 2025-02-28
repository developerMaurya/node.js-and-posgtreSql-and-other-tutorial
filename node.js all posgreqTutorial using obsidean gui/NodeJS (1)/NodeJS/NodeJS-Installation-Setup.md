# Node.js Installation and Setup

A comprehensive guide to installing and setting up Node.js for development.

## Installation Methods

### 1. Direct Download
1. Visit [Node.js official website](https://nodejs.org)
2. Choose LTS (Long Term Support) version for stability
3. Download and run the installer
4. Verify installation:
```bash
node --version
npm --version
```

### 2. Using Node Version Manager (nvm)
```bash
# Windows (nvm-windows)
# 1. Download nvm-windows from GitHub
# 2. Install nvm-windows
nvm list available
nvm install latest
nvm install lts
nvm use <version>

# List installed versions
nvm list

# Switch between versions
nvm use 16.14.0
```

## Development Environment Setup

### 1. IDE/Text Editor
Recommended options:
- Visual Studio Code
- WebStorm
- Atom
- Sublime Text

#### VS Code Extensions
- Node.js Extension Pack
- ESLint
- Prettier
- Node.js Debugging
- npm Intelligence

### 2. Environment Configuration
```javascript
// .env file
NODE_ENV=development
PORT=3000
DATABASE_URL=mongodb://localhost:27017/myapp

// Loading environment variables
require('dotenv').config();
console.log(process.env.NODE_ENV);
```

### 3. Project Initialization
```bash
# Create new directory
mkdir my-node-project
cd my-node-project

# Initialize npm project
npm init

# Or skip questions with defaults
npm init -y
```

## Basic Configuration Files

### 1. Package.json
```json
{
  "name": "my-node-project",
  "version": "1.0.0",
  "description": "My Node.js project",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.17.1"
  },
  "devDependencies": {
    "nodemon": "^2.0.15",
    "jest": "^27.0.0"
  }
}
```

### 2. .gitignore
```plaintext
# Dependencies
node_modules/

# Environment variables
.env
.env.local

# Logs
logs
*.log
npm-debug.log*

# Runtime data
pids
*.pid
*.seed

# Directory for instrumented libs
lib-cov

# Coverage directory
coverage/

# Build output
dist/
build/

# IDE specific files
.idea/
.vscode/
*.swp
*.swo
```

### 3. ESLint Configuration
```javascript
// .eslintrc.js
module.exports = {
    "env": {
        "node": true,
        "es2021": true
    },
    "extends": "eslint:recommended",
    "parserOptions": {
        "ecmaVersion": 12
    },
    "rules": {
        "indent": ["error", 4],
        "semi": ["error", "always"]
    }
}
```

## Project Structure
```plaintext
my-node-project/
├── src/
│   ├── index.js        # Application entry point
│   ├── config/         # Configuration files
│   ├── routes/         # Route definitions
│   ├── controllers/    # Route controllers
│   ├── models/         # Data models
│   ├── middleware/     # Custom middleware
│   ├── utils/          # Utility functions
│   └── services/       # Business logic
├── tests/              # Test files
├── public/             # Static files
├── package.json        # Project metadata
├── .env               # Environment variables
├── .gitignore         # Git ignore rules
└── README.md          # Project documentation
```

## Basic Application Setup

### 1. Entry Point (index.js)
```javascript
const express = require('express');
const dotenv = require('dotenv');

// Load environment variables
dotenv.config();

const app = express();
const port = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.get('/', (req, res) => {
    res.send('Hello World!');
});

// Start server
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
```

## Debugging Setup

### 1. VS Code Launch Configuration
```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Launch Program",
            "skipFiles": [
                "<node_internals>/**"
            ],
            "program": "${workspaceFolder}/src/index.js"
        }
    ]
}
```

### 2. Node.js Inspector
```bash
# Start app with inspector
node --inspect index.js

# Break on first line
node --inspect-brk index.js
```

## Best Practices

1. **Version Control**
   - Initialize Git repository
   - Create meaningful commits
   - Use feature branches

2. **Security**
   - Keep dependencies updated
   - Use security linting
   - Never commit sensitive data

3. **Performance**
   - Use appropriate Node.js version
   - Configure memory limits
   - Enable compression

## Related Topics
- [[npm-Basics]] - Package management
- [[NodeJS-Fundamentals]] - Core concepts
- [[Debugging-Profiling]] - Advanced debugging
- [[Development-Tools]] - Additional tools and utilities

Tags: #nodejs #setup #installation #development #configuration
