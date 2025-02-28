# Kubernetes Guide for Node.js Applications

A comprehensive guide to deploying and managing Node.js applications on Kubernetes.

## 1. Basic Kubernetes Setup

### Pod Configuration
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-app
  labels:
    app: nodejs
spec:
  containers:
  - name: nodejs
    image: nodejs-app:1.0.0
    ports:
    - containerPort: 3000
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
```

## 2. Deployment Configuration

### Basic Deployment
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
    spec:
      containers:
      - name: nodejs
        image: nodejs-app:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
```

### Rolling Updates
```yaml
# deployment.yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

## 3. Service Configuration

### Service Types
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: nodejs
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-internal
spec:
  type: ClusterIP
  ports:
  - port: 3000
  selector:
    app: nodejs
```

## 4. ConfigMaps and Secrets

### Configuration Management
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  API_URL: "https://api.example.com"
  REDIS_HOST: "redis-service"

# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-url: base64encodedstring
  api-key: base64encodedstring
```

## 5. Persistent Storage

### Volume Configuration
```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 6. Horizontal Pod Autoscaling

### Autoscaling Configuration
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodejs-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## 7. Ingress Configuration

### Ingress Rules
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nodejs-service
            port:
              number: 80
```

## 8. Monitoring and Logging

### Prometheus Configuration
```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'nodejs'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            regex: nodejs
            action: keep
```

### Logging Setup
```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_key time
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
```

## 9. Production Best Practices

### Network Policies
```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nodejs-network-policy
spec:
  podSelector:
    matchLabels:
      app: nodejs
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          type: frontend
    ports:
    - protocol: TCP
      port: 3000
```

### Resource Quotas
```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
```

## Related Topics
- [[Docker-Guide]] - Docker containerization
- [[CICD-Pipeline]] - CI/CD implementation
- [[Cloud-Deployment]] - Cloud deployment strategies
- [[Service-Mesh]] - Service mesh implementation

## Practice Projects
1. Deploy a microservices application on Kubernetes
2. Implement auto-scaling and load balancing
3. Set up monitoring and logging
4. Configure CI/CD pipeline with Kubernetes

## Resources
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Node.js Kubernetes Best Practices](https://learnk8s.io/nodejs-kubernetes-guide)
- [[Learning-Resources#Kubernetes|Kubernetes Resources]]

## Tags
#kubernetes #nodejs #devops #containerization #orchestration
