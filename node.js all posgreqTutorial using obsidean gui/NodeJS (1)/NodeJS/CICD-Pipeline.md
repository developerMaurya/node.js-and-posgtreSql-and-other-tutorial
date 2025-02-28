# CI/CD Pipeline Implementation Guide

A comprehensive guide to implementing Continuous Integration and Continuous Deployment pipelines for Node.js applications.

## 1. GitHub Actions Pipeline

### Basic Workflow
```yaml
# .github/workflows/main.yml
name: Node.js CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x]
        
    steps:
    - uses: actions/checkout@v3
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run tests
      run: npm test
      
    - name: Run linting
      run: npm run lint
      
    - name: Check code coverage
      run: npm run coverage

  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
      
    - name: Log in to registry
      run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u $ --password-stdin
      
    - name: Push image
      run: |
        docker tag myapp:${{ github.sha }} ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}
        docker push ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
        
    - name: Update ECS service
      run: |
        aws ecs update-service --cluster production --service myapp --force-new-deployment
```

## 2. Jenkins Pipeline

### Jenkinsfile Configuration
```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '-p 3000:3000'
        }
    }
    
    environment {
        CI = 'true'
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    npm run lint
                    npm test
                    npm run coverage
                '''
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
                
                script {
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Push') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'registry-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}").push()
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def kubeconfig = credentials('kubeconfig')
                    sh """
                        kubectl --kubeconfig=${kubeconfig} set image deployment/myapp \
                        myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend channel: '#deployments',
                      color: 'good',
                      message: "Deployment successful: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend channel: '#deployments',
                      color: 'danger',
                      message: "Deployment failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}
```

## 3. GitLab CI/CD

### GitLab Pipeline Configuration
```yaml
# .gitlab-ci.yml
image: node:18-alpine

stages:
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHA

cache:
  paths:
    - node_modules/

test:
  stage: test
  script:
    - npm ci
    - npm run lint
    - npm test
    - npm run coverage
  coverage: '/Lines\s*:\s*([0-9.]+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:
  stage: build
  script:
    - npm ci
    - npm run build
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG
  only:
    - main

deploy:staging:
  stage: deploy
  script:
    - kubectl config use-context staging
    - helm upgrade --install myapp ./helm/myapp
      --set image.tag=$IMAGE_TAG
      --namespace staging
  environment:
    name: staging
  only:
    - main

deploy:production:
  stage: deploy
  script:
    - kubectl config use-context production
    - helm upgrade --install myapp ./helm/myapp
      --set image.tag=$IMAGE_TAG
      --namespace production
  environment:
    name: production
  when: manual
  only:
    - main
```

## 4. Azure DevOps Pipeline

### Azure Pipeline Configuration
```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'myregistry'
  imageRepository: 'myapp'
  containerRegistry: 'myregistry.azurecr.io'
  dockerfilePath: '**/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
- stage: Test
  jobs:
  - job: Test
    steps:
    - task: NodeTool@0
      inputs:
        versionSpec: '18.x'
    
    - script: |
        npm ci
        npm run lint
        npm test
        npm run coverage
      displayName: 'Run tests'

- stage: Build
  jobs:
  - job: Build
    steps:
    - task: Docker@2
      inputs:
        command: 'buildAndPush'
        containerRegistry: $(dockerRegistryServiceConnection)
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        tags: |
          $(tag)
          latest

- stage: Deploy
  jobs:
  - deployment: Deploy
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'production-cluster'
              namespace: 'production'
              manifests: |
                kubernetes/deployment.yml
                kubernetes/service.yml
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)
```

## 5. Quality Gates

### SonarQube Configuration
```yaml
# sonar-project.properties
sonar.projectKey=myapp
sonar.projectName=My Node.js Application
sonar.sources=src
sonar.tests=test
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.coverage.exclusions=test/**/*,src/migrations/**/*
sonar.cpd.exclusions=test/**/*

# GitHub Actions Integration
jobs:
  sonarcloud:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: SonarCloud Scan
      uses: SonarSource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## 6. Automated Testing

### Test Configuration
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

// package.json scripts
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:e2e": "jest --config jest.e2e.config.js",
    "lint": "eslint .",
    "coverage": "jest --coverage && coveralls"
  }
}
```

## 7. Deployment Strategies

### Blue-Green Deployment
```yaml
# kubernetes/blue-green-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:blue
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:green
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Switch between blue and green
  ports:
  - port: 80
    targetPort: 3000
```

## Related Topics
- [[Docker-Guide]] - Docker implementation
- [[Kubernetes-Guide]] - Kubernetes setup
- [[Cloud-Deployment]] - Cloud deployment strategies
- [[Testing-Strategies]] - Testing implementation

## Practice Projects
1. Set up a complete CI/CD pipeline
2. Implement automated testing
3. Configure quality gates
4. Deploy using different strategies

## Resources
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [[Learning-Resources#CICD|CI/CD Resources]]

## Tags
#cicd #devops #automation #testing #deployment
