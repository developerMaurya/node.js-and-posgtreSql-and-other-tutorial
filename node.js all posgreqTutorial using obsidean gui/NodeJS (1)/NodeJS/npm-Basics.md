# npm (Node Package Manager) Basics

npm is the default package manager for Node.js. This guide covers essential npm concepts and commands.

## Package Management

### 1. Installing Packages
```bash
# Install package as dependency
npm install express
npm i express  # Shorthand

# Install specific version
npm install express@4.17.1

# Install as dev dependency
npm install --save-dev jest

# Install globally
npm install -g nodemon

# Install all dependencies from package.json
npm install
```

### 2. Updating Packages
```bash
# Check outdated packages
npm outdated

# Update specific package
npm update express

# Update all packages
npm update

# Update to major version (careful!)
npm install express@latest
```

### 3. Removing Packages
```bash
# Remove local package
npm uninstall express

# Remove global package
npm uninstall -g nodemon

# Remove dev dependency
npm uninstall --save-dev jest
```

## Package.json

### 1. Basic Structure
```json
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "Project description",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "lint": "eslint ."
  },
  "dependencies": {
    "express": "^4.17.1",
    "mongoose": "^6.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.15",
    "jest": "^27.0.0",
    "eslint": "^8.0.0"
  }
}
```

### 2. Version Numbers
```plaintext
"package": "1.2.3"
           ^ ^ ^
           | | patch: backwards-compatible bug fixes
           | minor: backwards-compatible features
           major: breaking changes

^1.2.3 : Accept any 1.x.x version
~1.2.3 : Accept any 1.2.x version
1.2.3  : Accept only exact version
*      : Accept any version
```

## npm Scripts

### 1. Basic Scripts
```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "build": "tsc",
    "lint": "eslint .",
    "format": "prettier --write ."
  }
}
```

### 2. Running Scripts
```bash
# Run script
npm run dev

# Special scripts (can omit 'run')
npm start
npm test

# Pass arguments
npm run test -- --watch

# Run multiple scripts
npm run lint && npm run test
```

### 3. Custom Scripts
```json
{
  "scripts": {
    "prebuild": "rm -rf dist",
    "build": "tsc",
    "postbuild": "cp package.json dist/",
    "custom": "echo \"Custom script\" && npm run build"
  }
}
```

## Dependencies Management

### 1. Types of Dependencies
```json
{
  "dependencies": {
    // Production dependencies
    "express": "^4.17.1"
  },
  "devDependencies": {
    // Development-only dependencies
    "jest": "^27.0.0"
  },
  "peerDependencies": {
    // Required peer dependencies
    "react": ">=16.8.0"
  },
  "optionalDependencies": {
    // Optional dependencies
    "image-optimizer": "^1.0.0"
  }
}
```

### 2. Lock File (package-lock.json)
- Ensures consistent installations
- Locks exact versions
- Should be committed to version control

## npm Configuration

### 1. npm Config
```bash
# View all configs
npm config list

# Set config
npm config set init-author-name "Your Name"

# Get specific config
npm config get init-author-name

# Delete config
npm config delete init-author-name
```

### 2. .npmrc File
```plaintext
# .npmrc
init-author-name=Your Name
init-license=MIT
save-exact=true
```

## Security

### 1. Security Auditing
```bash
# Run security audit
npm audit

# Fix security issues
npm audit fix

# Fix security issues including breaking changes
npm audit fix --force
```

### 2. Publishing Packages
```bash
# Login to npm
npm login

# Publish package
npm publish

# Update package
npm version patch
npm publish
```

## Best Practices

1. **Version Control**
   - Include package.json and package-lock.json
   - Exclude node_modules/
   - Use .npmignore for published packages

2. **Dependencies**
   - Regular updates
   - Minimal dependencies
   - Exact versions for critical dependencies

3. **Scripts**
   - Meaningful script names
   - Document complex scripts
   - Use pre/post hooks

## Common Issues and Solutions

### 1. Permission Issues
```bash
# Fix EACCES errors
sudo chown -R $USER ~/.npm
sudo chown -R $USER /usr/local/lib/node_modules
```

### 2. Cache Issues
```bash
# Clear npm cache
npm cache clean --force

# Verify cache
npm cache verify
```

## Related Topics
- [[NodeJS-Installation-Setup]] - Initial setup
- [[Package-Management]] - Advanced package management
- [[Security]] - Security best practices
- [[Deployment]] - Deployment considerations

Tags: #npm #package-manager #nodejs #dependencies #scripts
