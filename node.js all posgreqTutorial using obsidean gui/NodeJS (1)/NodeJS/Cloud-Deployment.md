# Cloud Deployment Strategies Guide

A comprehensive guide to deploying Node.js applications across different cloud platforms.

## 1. AWS Deployment

### Elastic Beanstalk Configuration
```yaml
# .elasticbeanstalk/config.yml
branch-defaults:
  main:
    environment: production
    group_suffix: null

global:
  application_name: myapp
  branch: null
  default_ec2_keyname: null
  default_platform: Node.js 18
  default_region: us-east-1
  include_git_submodules: true
  instance_profile: null
  platform_name: null
  platform_version: null
  profile: null
  repository: null
  sc: git
  workspace_type: Application

# Dockerfile.aws.json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "myapp:latest",
    "Update": "true"
  },
  "Ports": [
    {
      "ContainerPort": "3000"
    }
  ],
  "Logging": "/var/log/nginx"
}
```

### ECS Configuration
```yaml
# task-definition.json
{
  "family": "myapp",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "myapp:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "hostPort": 80
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## 2. Azure Deployment

### App Service Configuration
```yaml
# azure-webapp-config.yml
name: myapp
configurations:
  - name: production
    resourceGroup: myapp-prod
    appServicePlan: myapp-plan
    runtime: 'NODE|18-lts'
    region: eastus
    sku: P1v2
    settings:
      - name: WEBSITE_NODE_DEFAULT_VERSION
        value: '~18'
      - name: NODE_ENV
        value: production

# Web.config
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="iisnode" path="server.js" verb="*" modules="iisnode"/>
    </handlers>
    <rewrite>
      <rules>
        <rule name="NodeInspector" patternSyntax="ECMAScript" stopProcessing="true">
          <match url="^server.js\/debug[\/]?" />
        </rule>
        <rule name="StaticContent">
          <action type="Rewrite" url="public{REQUEST_URI}"/>
        </rule>
        <rule name="DynamicContent">
          <conditions>
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="True"/>
          </conditions>
          <action type="Rewrite" url="server.js"/>
        </rule>
      </rules>
    </rewrite>
    <security>
      <requestFiltering>
        <hiddenSegments>
          <remove segment="bin"/>
        </hiddenSegments>
      </requestFiltering>
    </security>
    <httpErrors existingResponse="PassThrough" />
  </system.webServer>
</configuration>
```

## 3. Google Cloud Platform

### App Engine Configuration
```yaml
# app.yaml
runtime: nodejs18
env: standard

instance_class: F1

automatic_scaling:
  target_cpu_utilization: 0.65
  min_instances: 1
  max_instances: 10
  target_throughput_utilization: 0.6

env_variables:
  NODE_ENV: 'production'
  REDIS_HOST: '10.0.0.1'
  REDIS_PORT: '6379'

handlers:
- url: /.*
  script: auto
  secure: always

network:
  session_affinity: true
```

### Cloud Run Configuration
```yaml
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/myapp', '.']

- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/myapp']

- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - 'run'
  - 'deploy'
  - 'myapp'
  - '--image'
  - 'gcr.io/$PROJECT_ID/myapp'
  - '--region'
  - 'us-central1'
  - '--platform'
  - 'managed'
  - '--allow-unauthenticated'
```

## 4. Heroku Deployment

### Heroku Configuration
```json
// package.json
{
  "name": "myapp",
  "version": "1.0.0",
  "engines": {
    "node": "18.x"
  },
  "scripts": {
    "start": "node server.js",
    "heroku-postbuild": "npm run build"
  }
}

// Procfile
web: npm start

// app.json
{
  "name": "myapp",
  "description": "My Node.js Application",
  "keywords": [
    "node",
    "express",
    "static"
  ],
  "website": "https://myapp.herokuapp.com/",
  "repository": "https://github.com/username/myapp",
  "env": {
    "NODE_ENV": {
      "description": "Environment configuration",
      "value": "production"
    }
  },
  "addons": [
    "heroku-postgresql",
    "heroku-redis"
  ]
}
```

## 5. Multi-Cloud Strategy

### Terraform Configuration
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}

provider "google" {
  project = "myproject"
  region  = "us-central1"
}

# AWS Resources
resource "aws_elastic_beanstalk_application" "myapp" {
  name        = "myapp"
  description = "My Node.js Application"
}

# Azure Resources
resource "azurerm_app_service" "myapp" {
  name                = "myapp"
  location            = "eastus"
  resource_group_name = azurerm_resource_group.main.name
  app_service_plan_id = azurerm_app_service_plan.main.id

  site_config {
    linux_fx_version = "NODE|18-lts"
    always_on        = true
  }
}

# GCP Resources
resource "google_cloud_run_service" "myapp" {
  name     = "myapp"
  location = "us-central1"

  template {
    spec {
      containers {
        image = "gcr.io/myproject/myapp"
      }
    }
  }
}
```

## 6. Serverless Deployment

### AWS Lambda Configuration
```yaml
# serverless.yml
service: myapp

provider:
  name: aws
  runtime: nodejs18.x
  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'us-east-1'}

functions:
  api:
    handler: src/handler.api
    events:
      - http:
          path: /{proxy+}
          method: ANY
    environment:
      NODE_ENV: production

plugins:
  - serverless-http
  - serverless-offline

custom:
  serverless-offline:
    port: 3000
```

### Azure Functions Configuration
```json
// host.json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[2.*, 3.0.0)"
  }
}

// function.json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
```

## 7. Monitoring and Logging

### CloudWatch Configuration
```javascript
// winston-cloudwatch.js
const winston = require('winston');
const WinstonCloudWatch = require('winston-cloudwatch');

const logger = winston.createLogger({
  transports: [
    new WinstonCloudWatch({
      logGroupName: 'myapp',
      logStreamName: `${process.env.NODE_ENV}-${process.env.AWS_REGION}`,
      awsRegion: process.env.AWS_REGION
    })
  ]
});

// Application Insights
const appInsights = require('applicationinsights');
appInsights.setup(process.env.APPINSIGHTS_INSTRUMENTATIONKEY)
    .setAutoDependencyCorrelation(true)
    .setAutoCollectRequests(true)
    .setAutoCollectPerformance(true)
    .setAutoCollectExceptions(true)
    .setAutoCollectDependencies(true)
    .setAutoCollectConsole(true)
    .setUseDiskRetryCaching(true)
    .start();
```

## Related Topics
- [[Docker-Guide]] - Docker implementation
- [[Kubernetes-Guide]] - Kubernetes setup
- [[CICD-Pipeline]] - CI/CD implementation
- [[Security-Best-Practices]] - Security guidelines

## Practice Projects
1. Deploy to multiple cloud providers
2. Implement serverless architecture
3. Set up cloud monitoring
4. Configure auto-scaling

## Resources
- [AWS Documentation](https://docs.aws.amazon.com/)
- [Azure Documentation](https://docs.microsoft.com/azure/)
- [GCP Documentation](https://cloud.google.com/docs)
- [[Learning-Resources#Cloud|Cloud Resources]]

## Tags
#cloud #deployment #devops #serverless #monitoring
