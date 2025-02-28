# Environment Variables and Configuration in Node.js

A comprehensive guide to managing environment variables and configuration in Node.js applications.

## Process Environment Variables

### Accessing Environment Variables
```javascript
// Access environment variables
console.log(process.env.NODE_ENV);
console.log(process.env.PORT);
console.log(process.env.DATABASE_URL);

// Set default values
const port = process.env.PORT || 3000;
const env = process.env.NODE_ENV || 'development';
```

### Setting Environment Variables

#### Command Line
```bash
# Windows Command Prompt
set NODE_ENV=production
set PORT=3000

# Windows PowerShell
$env:NODE_ENV="production"
$env:PORT=3000

# Unix/Linux/MacOS
export NODE_ENV=production
export PORT=3000
```

#### Package.json Scripts
```json
{
  "scripts": {
    "start": "NODE_ENV=production node app.js",
    "dev": "NODE_ENV=development nodemon app.js",
    "test": "NODE_ENV=test jest"
  }
}
```

## Using .env Files

### Basic Setup
```javascript
// Install dotenv
// $ npm install dotenv

// Load .env file
require('dotenv').config();

// Access variables
const dbUrl = process.env.DATABASE_URL;
const apiKey = process.env.API_KEY;
```

### Sample .env File
```plaintext
# .env
NODE_ENV=development
PORT=3000
DATABASE_URL=mongodb://localhost:27017/myapp
API_KEY=your_api_key_here
JWT_SECRET=your_jwt_secret
```

### Multiple Environments
```plaintext
# .env.development
PORT=3000
DATABASE_URL=mongodb://localhost:27017/dev_db

# .env.production
PORT=80
DATABASE_URL=mongodb://prod-server:27017/prod_db

# .env.test
PORT=3001
DATABASE_URL=mongodb://localhost:27017/test_db
```

## Configuration Management

### Basic Configuration Object
```javascript
// config.js
require('dotenv').config();

const config = {
    app: {
        port: process.env.PORT || 3000,
        env: process.env.NODE_ENV || 'development'
    },
    db: {
        url: process.env.DATABASE_URL,
        options: {
            useNewUrlParser: true,
            useUnifiedTopology: true
        }
    },
    jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: '1d'
    }
};

module.exports = config;
```

### Using Configuration
```javascript
// app.js
const config = require('./config');
const express = require('express');
const app = express();

app.listen(config.app.port, () => {
    console.log(`Server running in ${config.app.env} mode on port ${config.app.port}`);
});
```

## Environment-Specific Configuration

### Configuration by Environment
```javascript
// config/index.js
const development = require('./development');
const production = require('./production');
const test = require('./test');

const env = process.env.NODE_ENV || 'development';

const configs = {
    development,
    production,
    test
};

module.exports = configs[env];
```

### Environment Config Files
```javascript
// config/development.js
module.exports = {
    port: 3000,
    database: {
        url: 'mongodb://localhost:27017/dev_db',
        options: { /* development options */ }
    },
    logging: true
};

// config/production.js
module.exports = {
    port: process.env.PORT || 80,
    database: {
        url: process.env.DATABASE_URL,
        options: { /* production options */ }
    },
    logging: false
};
```

## Security Best Practices

### Protecting Sensitive Data
```javascript
// DON'T commit .env files
// Add to .gitignore
.env
.env.*

// Provide example configuration
// .env.example
NODE_ENV=development
PORT=3000
DATABASE_URL=mongodb://localhost:27017/myapp
API_KEY=your_api_key_here
```

### Validation
```javascript
// Install envalid
// $ npm install envalid

const { cleanEnv, str, num, url } = require('envalid');

const env = cleanEnv(process.env, {
    NODE_ENV: str({ choices: ['development', 'test', 'production'] }),
    PORT: num({ default: 3000 }),
    DATABASE_URL: url(),
    API_KEY: str(),
    JWT_SECRET: str()
});

module.exports = env;
```

## Custom Configuration Manager

### Creating a Config Manager
```javascript
// configManager.js
class ConfigManager {
    constructor() {
        this.config = {};
        this.loadEnv();
    }

    loadEnv() {
        require('dotenv').config();
        
        this.config = {
            app: this.getAppConfig(),
            db: this.getDbConfig(),
            auth: this.getAuthConfig()
        };
    }

    getAppConfig() {
        return {
            port: process.env.PORT || 3000,
            env: process.env.NODE_ENV || 'development',
            debug: process.env.DEBUG === 'true'
        };
    }

    getDbConfig() {
        return {
            url: process.env.DATABASE_URL,
            options: {
                useNewUrlParser: true,
                useUnifiedTopology: true
            }
        };
    }

    getAuthConfig() {
        return {
            jwtSecret: process.env.JWT_SECRET,
            expiresIn: process.env.JWT_EXPIRES_IN || '1d'
        };
    }

    get(key) {
        return this.config[key];
    }
}

module.exports = new ConfigManager();
```

### Using Config Manager
```javascript
// app.js
const config = require('./configManager');
const express = require('express');
const app = express();

const appConfig = config.get('app');
const dbConfig = config.get('db');

app.listen(appConfig.port, () => {
    console.log(`Server running in ${appConfig.env} mode on port ${appConfig.port}`);
});
```

## Related Topics
- [[Security-Best-Practices]] - Security guidelines
- [[Deployment-Strategies]] - Deployment configuration
- [[Database-Integration]] - Database configuration
- [[Logging]] - Logging configuration

## Practice Projects
1. Create a configuration management system
2. Build an environment validator
3. Implement secure secrets management
4. Create a multi-environment setup

## Resources
- [Dotenv Documentation](https://github.com/motdotla/dotenv)
- [Node.js Process Documentation](https://nodejs.org/api/process.html)
- [[Learning-Resources#Configuration|Configuration Resources]]

## Tags
#nodejs #configuration #environment-variables #security #best-practices
