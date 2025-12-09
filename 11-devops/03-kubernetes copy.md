# Kubernetes 容器编排

Kubernetes (K8s) 是生产环境容器编排的事实标准。本文深入讲解 K8s 核心概念和 Node.js 应用的部署实践。

## 目录
- [K8s 核心概念](#k8s-核心概念)
- [Pod 与 Deployment](#pod-与-deployment)
- [Service 与 Ingress](#service-与-ingress)
- [配置管理](#配置管理)
- [自动扩缩容](#自动扩缩容)
- [Helm](#helm)
- [生产最佳实践](#生产最佳实践)
- [常见面试题](#常见面试题)

---

## K8s 核心概念

### 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
├─────────────────────────────────────────────────────────────────┤
│                         Control Plane                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐  │
│  │ API Server  │ │  Scheduler  │ │ Controller  │ │   etcd    │  │
│  └─────────────┘ └─────────────┘ │   Manager   │ │           │  │
│                                  └─────────────┘ └───────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                          Worker Nodes                            │
│  ┌──────────────────────────┐ ┌──────────────────────────┐      │
│  │         Node 1           │ │         Node 2           │      │
│  │  ┌─────┐ ┌─────┐ ┌─────┐│ │  ┌─────┐ ┌─────┐ ┌─────┐│      │
│  │  │ Pod │ │ Pod │ │ Pod ││ │  │ Pod │ │ Pod │ │ Pod ││      │
│  │  └─────┘ └─────┘ └─────┘│ │  └─────┘ └─────┘ └─────┘│      │
│  │  ┌─────────┐ ┌─────────┐│ │  ┌─────────┐ ┌─────────┐│      │
│  │  │ kubelet │ │ kube-   ││ │  │ kubelet │ │ kube-   ││      │
│  │  │         │ │ proxy   ││ │  │         │ │ proxy   ││      │
│  │  └─────────┘ └─────────┘│ │  └─────────┘ └─────────┘│      │
│  └──────────────────────────┘ └──────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 核心组件

| 组件 | 作用 | 位置 |
|------|------|------|
| **API Server** | 集群入口，REST API | Control Plane |
| **etcd** | 分布式 KV 存储，存储集群状态 | Control Plane |
| **Scheduler** | Pod 调度到 Node | Control Plane |
| **Controller Manager** | 运行各种控制器 | Control Plane |
| **kubelet** | 管理 Node 上的 Pod | Worker Node |
| **kube-proxy** | 网络代理，Service 实现 | Worker Node |

### 核心资源对象

```yaml
# 核心资源层次
Namespace
  └── Deployment / StatefulSet / DaemonSet
        └── ReplicaSet
              └── Pod
                    └── Container

# 服务发现
Service → Endpoints → Pods
Ingress → Service → Pods

# 配置
ConfigMap - 非敏感配置
Secret - 敏感信息

# 存储
PersistentVolume (PV) → PersistentVolumeClaim (PVC) → Pod
```

---

## Pod 与 Deployment

### Pod 基础

Pod 是 K8s 最小的部署单元，包含一个或多个容器。

```yaml
# pod.yaml - Node.js 应用 Pod
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-app
  labels:
    app: nodejs-app
    version: v1
spec:
  containers:
    - name: app
      image: my-app:1.0.0
      ports:
        - containerPort: 3000
          name: http
      env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
          cpu: "500m"
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
  restartPolicy: Always
```

### Deployment

Deployment 管理 Pod 的声明式更新。

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: production
  labels:
    app: nodejs-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多多 1 个 Pod
      maxUnavailable: 0  # 不允许不可用
  template:
    metadata:
      labels:
        app: nodejs-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      serviceAccountName: nodejs-app
      
      # 亲和性：尽量分布在不同节点
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: nodejs-app
                topologyKey: kubernetes.io/hostname
      
      # 容器配置
      containers:
        - name: app
          image: ghcr.io/myorg/nodejs-app:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
              name: http
          
          # 环境变量
          envFrom:
            - configMapRef:
                name: nodejs-app-config
            - secretRef:
                name: nodejs-app-secrets
          
          # 资源限制
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          
          # 健康检查
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          
          # 优雅关闭
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
          
          # Volume 挂载
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
      
      # 优雅终止时间
      terminationGracePeriodSeconds: 30
      
      # Volume 定义
      volumes:
        - name: config-volume
          configMap:
            name: nodejs-app-config
      
      # 镜像拉取凭证
      imagePullSecrets:
        - name: ghcr-secret
```

### Node.js 优雅关闭

```typescript
// src/main.ts - NestJS 优雅关闭
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 启用优雅关闭
  app.enableShutdownHooks();

  // 健康检查端点
  app.use('/health', (req, res) => {
    res.status(200).json({ status: 'healthy' });
  });

  // 就绪检查端点
  let isReady = false;
  app.use('/ready', (req, res) => {
    if (isReady) {
      res.status(200).json({ status: 'ready' });
    } else {
      res.status(503).json({ status: 'not ready' });
    }
  });

  await app.listen(3000);
  isReady = true;

  console.log('Application is running on port 3000');
}

bootstrap();

// 处理信号
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown');
  
  // 标记为不就绪（停止接收新流量）
  // readiness probe 会失败，K8s 会从 Service 中移除
  
  // 等待现有请求完成
  // preStop hook 的 sleep 给了这个时间
  
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('SIGINT received');
  process.exit(0);
});
```

### StatefulSet（有状态应用）

```yaml
# statefulset.yaml - 有状态应用（如数据库）
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
  
  # 持久化存储
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp2
        resources:
          requests:
            storage: 20Gi
```

---

## Service 与 Ingress

### Service 类型

```yaml
# ClusterIP（默认）- 集群内部访问
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
      name: http

---
# NodePort - 通过节点端口访问
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app-nodeport
spec:
  type: NodePort
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080  # 30000-32767

---
# LoadBalancer - 云平台负载均衡器
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app-lb
  annotations:
    # AWS 注解
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000

---
# Headless Service（用于 StatefulSet）
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

### Ingress

```yaml
# ingress.yaml - NGINX Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    # 速率限制
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    # 证书管理
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nodejs-app
                port:
                  number: 80

---
# 多服务路由
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          # /users/* -> user-service
          - path: /users(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          # /orders/* -> order-service
          - path: /orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          # 默认 -> api-gateway
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 80
```

### AWS ALB Ingress

```yaml
# alb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-app-alb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxx:certificate/xxx
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nodejs-app
                port:
                  number: 80
```

---

## 配置管理

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodejs-app-config
  namespace: production
data:
  # 单个值
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  
  # JSON 配置
  app.config.json: |
    {
      "port": 3000,
      "cors": {
        "origin": ["https://example.com"],
        "credentials": true
      },
      "rateLimit": {
        "windowMs": 60000,
        "max": 100
      }
    }
  
  # YAML 配置
  app.config.yaml: |
    database:
      pool:
        min: 5
        max: 20
    cache:
      ttl: 3600
      prefix: "app:"
```

### Secret

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nodejs-app-secrets
  namespace: production
type: Opaque
stringData:  # 明文（base64 自动编码）
  DATABASE_URL: "postgresql://user:pass@postgres:5432/db"
  REDIS_URL: "redis://redis:6379"
  JWT_SECRET: "your-jwt-secret-key"

---
# 或使用 data（base64 编码）
apiVersion: v1
kind: Secret
metadata:
  name: nodejs-app-secrets
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0Bwb3N0Z3Jlczo1NDMyL2Ri
  JWT_SECRET: eW91ci1qd3Qtc2VjcmV0LWtleQ==
```

### 使用 ConfigMap 和 Secret

```yaml
# deployment 中引用
spec:
  containers:
    - name: app
      # 方式 1：envFrom（整个 ConfigMap/Secret）
      envFrom:
        - configMapRef:
            name: nodejs-app-config
        - secretRef:
            name: nodejs-app-secrets
      
      # 方式 2：单独引用
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: nodejs-app-secrets
              key: DATABASE_URL
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: nodejs-app-config
              key: LOG_LEVEL
      
      # 方式 3：挂载为文件
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: secrets-volume
          mountPath: /app/secrets
          readOnly: true
  
  volumes:
    - name: config-volume
      configMap:
        name: nodejs-app-config
        items:
          - key: app.config.json
            path: config.json
    - name: secrets-volume
      secret:
        secretName: nodejs-app-secrets
```

### External Secrets（AWS Secrets Manager 集成）

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: nodejs-app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: nodejs-app-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: production/nodejs-app
        property: database_url
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/nodejs-app
        property: jwt_secret
```

---

## 自动扩缩容

### HPA（Horizontal Pod Autoscaler）

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodejs-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-app
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
    # CPU 使用率
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # 内存使用率
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # 自定义指标（每秒请求数）
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1000
  
  behavior:
    # 扩容行为
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
    
    # 缩容行为
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 分钟稳定期
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### VPA（Vertical Pod Autoscaler）

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nodejs-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-app
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, Off
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2000m
          memory: 2Gi
        controlledResources: ["cpu", "memory"]
```

### KEDA（事件驱动自动扩缩）

```yaml
# keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nodejs-app-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: nodejs-app
  minReplicaCount: 1
  maxReplicaCount: 50
  pollingInterval: 30
  cooldownPeriod: 300
  
  triggers:
    # SQS 队列长度
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/xxx/my-queue
        queueLength: "10"
        awsRegion: us-east-1
      authenticationRef:
        name: keda-aws-credentials
    
    # Kafka consumer lag
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-consumer-group
        topic: my-topic
        lagThreshold: "100"
    
    # Prometheus 指标
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_total
        threshold: "1000"
        query: sum(rate(http_requests_total{app="nodejs-app"}[2m]))
```

### Cluster Autoscaler

```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - name: cluster-autoscaler
          image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.26.0
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --nodes=2:10:my-node-group
            - --scale-down-enabled=true
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --scale-down-utilization-threshold=0.5
            - --skip-nodes-with-local-storage=false
            - --skip-nodes-with-system-pods=false
          resources:
            requests:
              cpu: 100m
              memory: 300Mi
```

---

## Helm

### Helm Chart 结构

```
my-nodejs-app/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt
└── charts/  # 依赖
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: nodejs-app
description: A Helm chart for Node.js application
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: ghcr.io/myorg/nodejs-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ghcr-secret

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "3000"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: nodejs-app
          topologyKey: kubernetes.io/hostname

# 应用配置
config:
  NODE_ENV: production
  LOG_LEVEL: info
  CACHE_TTL: "3600"

# 敏感配置（通过 External Secrets）
secrets:
  existingSecret: nodejs-app-secrets

# 健康检查
healthCheck:
  liveness:
    path: /health
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 5

# 依赖
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp

redis:
  enabled: true
  architecture: standalone
```

### templates/deployment.yaml

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nodejs-app.fullname" . }}
  labels:
    {{- include "nodejs-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "nodejs-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "nodejs-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "nodejs-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "nodejs-app.fullname" . }}-config
            {{- if .Values.secrets.existingSecret }}
            - secretRef:
                name: {{ .Values.secrets.existingSecret }}
            {{- end }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.healthCheck.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.healthCheck.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.readiness.periodSeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
      terminationGracePeriodSeconds: 30
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Helm 命令

```bash
# 创建 Chart
helm create my-nodejs-app

# 更新依赖
helm dependency update ./my-nodejs-app

# 本地渲染模板（调试）
helm template my-release ./my-nodejs-app -f values-staging.yaml

# 安装
helm install my-release ./my-nodejs-app \
  --namespace production \
  --create-namespace \
  -f values-production.yaml

# 升级
helm upgrade my-release ./my-nodejs-app \
  --namespace production \
  -f values-production.yaml

# 回滚
helm rollback my-release 1 --namespace production

# 查看历史
helm history my-release --namespace production

# 卸载
helm uninstall my-release --namespace production

# 打包
helm package ./my-nodejs-app

# 推送到仓库
helm push my-nodejs-app-1.0.0.tgz oci://ghcr.io/myorg/charts
```

---

## 生产最佳实践

### 1. 资源管理

```yaml
# 始终设置资源限制
resources:
  requests:
    memory: "256Mi"  # 保证调度
    cpu: "250m"
  limits:
    memory: "512Mi"  # 防止 OOM
    cpu: "1000m"

# 使用 LimitRange 设置默认值
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      type: Container

# ResourceQuota 限制 Namespace 总资源
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

### 2. Pod 安全

```yaml
# PodSecurityContext
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  
  volumes:
    - name: tmp
      emptyDir: {}
```

### 3. 网络策略

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nodejs-app-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: nodejs-app
  policyTypes:
    - Ingress
    - Egress
  
  # 入站规则
  ingress:
    # 允许来自 Ingress Controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
    # 允许同 Namespace 的 Pod
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 3000
  
  # 出站规则
  egress:
    # 允许访问数据库
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # 允许访问 Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # 允许 DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # 允许访问外部 HTTPS
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

### 4. Pod 分布策略

```yaml
# PodDisruptionBudget - 确保高可用
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nodejs-app-pdb
  namespace: production
spec:
  minAvailable: 2  # 或使用 maxUnavailable: 1
  selector:
    matchLabels:
      app: nodejs-app

---
# 拓扑分布约束
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: nodejs-app
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: nodejs-app
```

### 5. 监控和日志

```yaml
# ServiceMonitor（Prometheus Operator）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nodejs-app
  namespace: production
spec:
  selector:
    matchLabels:
      app: nodejs-app
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
  namespaceSelector:
    matchNames:
      - production
```

---

## 常见面试题

### 1. Pod 的生命周期？

**阶段**：
1. **Pending**：已创建，等待调度
2. **Running**：已调度到 Node，至少一个容器运行中
3. **Succeeded**：所有容器成功终止（Job）
4. **Failed**：所有容器终止，至少一个失败
5. **Unknown**：无法获取状态

**容器状态**：
- Waiting：等待启动
- Running：运行中
- Terminated：已终止

### 2. Service 如何实现负载均衡？

**实现方式**：
1. **kube-proxy + iptables**（默认）：
   - kube-proxy 监听 Service/Endpoint 变化
   - 在每个 Node 上创建 iptables 规则
   - 基于随机/轮询分发流量

2. **kube-proxy + IPVS**：
   - 使用 Linux IPVS 内核模块
   - 支持更多负载均衡算法
   - 性能更好

3. **Service Mesh**（如 Istio）：
   - 完整的 L7 负载均衡
   - 更细粒度的流量控制

### 3. 如何实现零停机部署？

**策略**：

1. **RollingUpdate**：
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

2. **健康检查**：
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
```

3. **优雅关闭**：
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 30
```

4. **PodDisruptionBudget**：
```yaml
minAvailable: 2
```

### 4. ConfigMap 更新后 Pod 如何感知？

**方式**：

1. **Volume 挂载**（自动更新，有延迟）：
   - kubelet 定期同步（默认 1 分钟）
   - 应用需要监听文件变化

2. **环境变量**（不自动更新）：
   - 需要重启 Pod

3. **最佳实践**：
   - 使用 checksum annotation 触发滚动更新
```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### 5. 如何限制 Pod 资源使用？

**三层限制**：

1. **Pod 级别**：
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

2. **Namespace 级别**（LimitRange）：
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
spec:
  limits:
    - default:
        cpu: 500m
      defaultRequest:
        cpu: 100m
      type: Container
```

3. **集群级别**（ResourceQuota）：
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    limits.memory: 8Gi
```

---

## 总结

### K8s 学习路径

```
基础（必须）
├── Pod、Deployment、Service
├── ConfigMap、Secret
├── 资源限制
└── 健康检查

进阶（重要）
├── Ingress
├── HPA
├── Helm
└── 网络策略

高级（架构师）
├── Operator 开发
├── Service Mesh
├── 多集群管理
└── 安全加固
```

### 关键配置清单

- [ ] 资源 requests 和 limits
- [ ] Liveness 和 Readiness Probe
- [ ] Pod 反亲和性
- [ ] PodDisruptionBudget
- [ ] 网络策略
- [ ] 安全上下文
- [ ] HPA/VPA
- [ ] 优雅关闭

---

**上一篇**：[CI/CD](./02-cicd.md)  
**下一篇**：[Infrastructure as Code](./04-infrastructure-as-code.md)

