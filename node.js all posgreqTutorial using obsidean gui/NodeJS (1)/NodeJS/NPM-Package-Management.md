# NPM Package Management

A comprehensive guide to managing packages and dependencies using Node Package Manager (NPM).

## NPM Basics

### Initializing a Project
```bash
# Create a new package.json
npm init

# Create with default values
npm init -y
```

### Package.json Structure
```json
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "Project description",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "jest",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.17.1",
    "mongoose": "^6.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.12",
    "jest": "^27.0.6"
  }
}
```

## Installing Packages

### Basic Installation
```bash
# Install a package
npm install package-name

# Install specific version
npm install package-name@1.2.3

# Install multiple packages
npm install package1 package2 package3

# Install as dev dependency
npm install --save-dev package-name

# Install globally
npm install -g package-name
```

### Dependency Types
```bash
# Production dependencies
npm install express mongoose

# Development dependencies
npm install --save-dev nodemon jest eslint

# Optional dependencies
npm install --save-optional some-package

# Peer dependencies (defined in package.json)
{
  "peerDependencies": {
    "react": ">=16.8.0",
    "react-dom": ">=16.8.0"
  }
}
```

## Managing Dependencies

### Updating Packages
```bash
# Check outdated packages
npm outdated

# Update packages
npm update

# Update specific package
npm update package-name

# Update to latest version (including major)
npm install package-name@latest
```

### Removing Packages
```bash
# Remove a package
npm uninstall package-name

# Remove global package
npm uninstall -g package-name

# Remove dev dependency
npm uninstall --save-dev package-name
```

## Version Control

### Understanding Semantic Versioning
```plaintext
"package-name": "^1.2.3"
# ^ = Compatible with version 1.x.x
# ~ = Compatible with version 1.2.x
# * = Latest version
# 1.2.3 = Exact version

Major.Minor.Patch
1.0.0 = Major version (breaking changes)
0.1.0 = Minor version (new features)
0.0.1 = Patch version (bug fixes)
```

### Lock File
```json
// package-lock.json
{
  "name": "project",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "dependencies": {
    "package-name": {
      "version": "1.2.3",
      "resolved": "https://registry.npmjs.org/package-name/-/package-name-1.2.3.tgz",
      "integrity": "sha512-..."
    }
  }
}
```

## NPM Scripts

### Basic Scripts
```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "lint": "eslint .",
    "build": "tsc"
  }
}
```

### Running Scripts
```bash
# Run a script
npm run script-name

# Run start script
npm start

# Run test script
npm test

# Run multiple scripts
npm run lint && npm run test
```

### Custom Scripts
```json
{
  "scripts": {
    "custom": "echo \"Custom script\"",
    "prebuild": "echo \"Pre-build script\"",
    "build": "tsc",
    "postbuild": "echo \"Post-build script\"",
    "deploy": "npm run build && node deploy.js"
  }
}
```

## Project Configuration

### NPM Configuration
```bash
# Set config value
npm config set key value

# Get config value
npm config get key

# List all config
npm config list

# Delete config value
npm config delete key
```

### .npmrc File
```plaintext
# .npmrc
registry=https://registry.npmjs.org/
save-exact=true
package-lock=true
```

## Working with Private Packages

### Scoped Packages
```bash
# Install scoped package
npm install @scope/package-name

# Publish scoped package
npm publish --access public
```

### Private Registry
```bash
# Set registry
npm config set registry https://private-registry.com

# Login to registry
npm login --registry=https://private-registry.com
```

## Security

### Auditing Dependencies
```bash
# Run security audit
npm audit

# Fix security issues
npm audit fix

# Fix security issues (including breaking changes)
npm audit fix --force
```

### Package Integrity
```bash
# Verify packages
npm verify-cache

# Clean cache
npm cache clean --force

# Check package integrity
npm cache verify
```

## Best Practices

### Project Structure
```plaintext
my-project/
├── node_modules/
├── src/
│   ├── index.js
│   └── components/
├── tests/
├── .gitignore
├── package.json
├── package-lock.json
└── README.md
```

### .gitignore
```plaintext
# Node modules
node_modules/

# Environment variables
.env
.env.*

# Logs
logs
*.log
npm-debug.log*

# Build output
dist/
build/

# IDE files
.vscode/
.idea/
```

## Related Topics
- [[Dependency-Management]] - Advanced dependency management
- [[Package-Publishing]] - Publishing NPM packages
- [[Security-Best-Practices]] - NPM security
- [[Monorepo-Management]] - Managing multiple packages

## Practice Projects
1. Create and publish an NPM package
2. Set up a project with custom scripts
3. Implement automated dependency updates
4. Create a monorepo structure

## Resources
- [NPM Documentation](https://docs.npmjs.com/)
- [Semantic Versioning](https://semver.org/)
- [[Learning-Resources#NPM|NPM Learning Resources]]

## Tags
#npm #package-management #dependencies #node-modules #versioning
