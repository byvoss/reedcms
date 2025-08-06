# Deployment Guide

## Overview

Deployment Guide Implementation für ReedCMS. Production Deployment mit Best Practices und Automation.

## Pre-Deployment Checklist

```markdown
# ReedCMS Deployment Checklist

## Environment Preparation
- [ ] Infrastructure provisioned (Kubernetes cluster, databases, storage)
- [ ] SSL certificates obtained and configured
- [ ] DNS records configured
- [ ] Backup systems in place
- [ ] Monitoring and alerting configured
- [ ] Security scanning completed

## Application Configuration
- [ ] Environment variables documented
- [ ] Secrets management configured (Vault, K8s secrets)
- [ ] Database migrations tested
- [ ] Feature flags configured
- [ ] Rate limiting configured
- [ ] CORS settings verified

## Performance
- [ ] Load testing completed
- [ ] Caching strategy implemented
- [ ] CDN configured
- [ ] Database indexes optimized
- [ ] Resource limits set

## Security
- [ ] Security headers configured
- [ ] Authentication/authorization tested
- [ ] Input validation verified
- [ ] HTTPS enforced
- [ ] Secrets rotated
- [ ] Firewall rules configured

## Operational Readiness
- [ ] Runbooks created
- [ ] On-call rotation scheduled
- [ ] Incident response plan
- [ ] Rollback procedure tested
- [ ] Backup/restore tested
- [ ] Disaster recovery plan
```

## Infrastructure Setup

```bash
#!/bin/bash
# infrastructure/setup.sh

set -euo pipefail

# Configuration
CLUSTER_NAME=${CLUSTER_NAME:-reedcms-production}
REGION=${AWS_REGION:-eu-central-1}
NODE_COUNT=${NODE_COUNT:-3}
NODE_TYPE=${NODE_TYPE:-t3.large}

echo "Setting up ReedCMS infrastructure..."

# Create EKS cluster
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --nodes $NODE_COUNT \
  --node-type $NODE_TYPE \
  --managed \
  --alb-ingress-access \
  --asg-access

# Install cluster addons
echo "Installing cluster addons..."

# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Install ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Install Prometheus stack
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring/prometheus-values.yaml

# Create RDS instance
echo "Creating RDS instance..."
aws rds create-db-instance \
  --db-instance-identifier reedcms-production \
  --db-instance-class db.r6g.large \
  --engine postgres \
  --engine-version 15.4 \
  --master-username reedcms \
  --master-user-password "${DB_PASSWORD}" \
  --allocated-storage 100 \
  --storage-encrypted \
  --backup-retention-period 30 \
  --multi-az \
  --no-publicly-accessible \
  --vpc-security-group-ids "${DB_SECURITY_GROUP}"

# Create ElastiCache cluster
echo "Creating ElastiCache cluster..."
aws elasticache create-cache-cluster \
  --cache-cluster-id reedcms-production \
  --cache-node-type cache.r6g.large \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-nodes 3 \
  --cache-subnet-group-name reedcms-subnet-group \
  --security-group-ids "${CACHE_SECURITY_GROUP}"

# Create S3 buckets
echo "Creating S3 buckets..."
aws s3api create-bucket \
  --bucket reedcms-production-assets \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION

aws s3api put-bucket-versioning \
  --bucket reedcms-production-assets \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket reedcms-production-assets \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

echo "Infrastructure setup complete!"
```

## Application Deployment

```yaml
# kubernetes/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reedcms
  namespace: production
  labels:
    app: reedcms
    environment: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: reedcms
      environment: production
  template:
    metadata:
      labels:
        app: reedcms
        environment: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: reedcms
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - reedcms
              topologyKey: kubernetes.io/hostname
      containers:
        - name: reedcms
          image: ghcr.io/reedcms/reedcms:v1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP
          env:
            - name: RUST_LOG
              value: "info,reedcms=debug"
            - name: ENVIRONMENT
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: reedcms-database
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: reedcms-redis
                  key: url
            - name: S3_BUCKET
              value: "reedcms-production-assets"
            - name: AWS_REGION
              value: "eu-central-1"
          envFrom:
            - configMapRef:
                name: reedcms-config
            - secretRef:
                name: reedcms-secrets
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 2Gi
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 30
          volumeMounts:
            - name: config
              mountPath: /etc/reedcms
              readOnly: true
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: config
          configMap:
            name: reedcms-config
        - name: tmp
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: reedcms
  namespace: production
  labels:
    app: reedcms
    environment: production
spec:
  type: ClusterIP
  selector:
    app: reedcms
    environment: production
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
    - name: metrics
      port: 9090
      targetPort: metrics
      protocol: TCP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: reedcms
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: reedcms
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
        - type: Pods
          value: 1
          periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 2
          periodSeconds: 60
      selectPolicy: Max

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: reedcms
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: reedcms
      environment: production
```

## Database Migration

```bash
#!/bin/bash
# scripts/migrate-production.sh

set -euo pipefail

echo "Running production database migrations..."

# Check current version
CURRENT_VERSION=$(kubectl exec -n production deployment/reedcms -- reedcms db version)
echo "Current database version: $CURRENT_VERSION"

# Backup database
echo "Creating database backup..."
BACKUP_NAME="reedcms-backup-$(date +%Y%m%d-%H%M%S)"
kubectl exec -n production deployment/reedcms -- \
  pg_dump $DATABASE_URL > backups/$BACKUP_NAME.sql

# Run migrations in transaction
echo "Running migrations..."
kubectl exec -n production deployment/reedcms -- \
  reedcms db migrate --target production

# Verify migration
NEW_VERSION=$(kubectl exec -n production deployment/reedcms -- reedcms db version)
echo "New database version: $NEW_VERSION"

# Run post-migration checks
echo "Running post-migration checks..."
kubectl exec -n production deployment/reedcms -- \
  reedcms db check

echo "Migration completed successfully!"
```

## Zero-Downtime Deployment

```bash
#!/bin/bash
# scripts/deploy-zero-downtime.sh

set -euo pipefail

VERSION=${1:-latest}
NAMESPACE=${2:-production}

echo "Starting zero-downtime deployment of version $VERSION..."

# Pre-deployment checks
echo "Running pre-deployment checks..."
./scripts/pre-deploy-check.sh $NAMESPACE

# Create new deployment
echo "Creating new deployment..."
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reedcms-$VERSION
  namespace: $NAMESPACE
  labels:
    app: reedcms
    version: $VERSION
spec:
  replicas: 3
  selector:
    matchLabels:
      app: reedcms
      version: $VERSION
  template:
    metadata:
      labels:
        app: reedcms
        version: $VERSION
    spec:
      containers:
        - name: reedcms
          image: ghcr.io/reedcms/reedcms:$VERSION
          # ... (same as production deployment)
EOF

# Wait for new deployment to be ready
echo "Waiting for new deployment to be ready..."
kubectl wait --for=condition=available --timeout=600s \
  deployment/reedcms-$VERSION -n $NAMESPACE

# Run smoke tests
echo "Running smoke tests..."
./scripts/smoke-test.sh $NAMESPACE $VERSION

# Gradually shift traffic
echo "Shifting traffic to new version..."
for PERCENT in 10 25 50 75 100; do
  echo "Routing $PERCENT% traffic to new version..."
  kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: reedcms
  namespace: $NAMESPACE
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "$PERCENT"
spec:
  rules:
    - host: reedcms.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: reedcms-$VERSION
                port:
                  number: 80
EOF
  
  # Monitor error rate
  sleep 60
  ERROR_RATE=$(./scripts/check-error-rate.sh)
  if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
    echo "Error rate too high: $ERROR_RATE. Rolling back..."
    ./scripts/rollback.sh $NAMESPACE
    exit 1
  fi
done

# Update main service
echo "Updating main service..."
kubectl patch service reedcms -n $NAMESPACE \
  -p '{"spec":{"selector":{"version":"'$VERSION'"}}}'

# Scale down old deployment
echo "Scaling down old deployment..."
OLD_DEPLOYMENT=$(kubectl get deployments -n $NAMESPACE \
  -l app=reedcms \
  --field-selector metadata.name!=reedcms-$VERSION \
  -o jsonpath='{.items[0].metadata.name}')

if [ ! -z "$OLD_DEPLOYMENT" ]; then
  kubectl scale deployment $OLD_DEPLOYMENT -n $NAMESPACE --replicas=0
  # Keep for 24 hours for quick rollback
  kubectl annotate deployment $OLD_DEPLOYMENT -n $NAMESPACE \
    reedcms.io/cleanup-after="$(date -d '+24 hours' --iso-8601)"
fi

echo "Deployment completed successfully!"
```

## Monitoring and Alerts

```yaml
# monitoring/production-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: reedcms-production
  namespace: monitoring
spec:
  groups:
    - name: production
      interval: 30s
      rules:
        # Deployment health
        - alert: DeploymentDown
          expr: |
            kube_deployment_status_replicas_available{deployment="reedcms", namespace="production"}
            < kube_deployment_spec_replicas{deployment="reedcms", namespace="production"}
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "ReedCMS deployment has insufficient replicas"
            description: "{{ $value }} replicas available, expected {{ $labels.spec_replicas }}"
            runbook_url: "https://wiki.reedcms.com/runbooks/deployment-down"
        
        # High error rate
        - alert: HighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{job="reedcms",status=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{job="reedcms"}[5m]))
            ) > 0.01
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "High 5xx error rate in production"
            description: "Error rate is {{ $value | humanizePercentage }}"
            dashboard_url: "https://grafana.reedcms.com/d/production-overview"
        
        # High latency
        - alert: HighLatency
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{job="reedcms"}[5m])) by (le)
            ) > 0.5
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "High request latency in production"
            description: "95th percentile latency is {{ $value }}s"
        
        # Database connection pool
        - alert: DatabaseConnectionPoolExhaustion
          expr: |
            reedcms_db_connections_active / reedcms_db_connections_max > 0.9
          for: 5m
          labels:
            severity: critical
            team: database
          annotations:
            summary: "Database connection pool near exhaustion"
            description: "{{ $value | humanizePercentage }} of connections in use"
```

## Rollback Procedure

```bash
#!/bin/bash
# scripts/rollback.sh

set -euo pipefail

NAMESPACE=${1:-production}
VERSION=${2:-previous}

echo "Initiating rollback in $NAMESPACE..."

# Get previous deployment
if [ "$VERSION" == "previous" ]; then
  VERSION=$(kubectl get deployments -n $NAMESPACE \
    -l app=reedcms \
    --sort-by=.metadata.creationTimestamp \
    -o jsonpath='{.items[-2].metadata.labels.version}')
fi

echo "Rolling back to version: $VERSION"

# Scale up previous deployment
kubectl scale deployment reedcms-$VERSION -n $NAMESPACE --replicas=3

# Wait for rollback deployment to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/reedcms-$VERSION -n $NAMESPACE

# Switch service
kubectl patch service reedcms -n $NAMESPACE \
  -p '{"spec":{"selector":{"version":"'$VERSION'"}}}'

# Verify rollback
sleep 30
./scripts/smoke-test.sh $NAMESPACE $VERSION

echo "Rollback completed successfully!"

# Send notification
curl -X POST $SLACK_WEBHOOK_URL \
  -H 'Content-type: application/json' \
  -d '{
    "text": "Production rollback completed",
    "attachments": [{
      "color": "warning",
      "fields": [{
        "title": "Environment",
        "value": "'$NAMESPACE'",
        "short": true
      }, {
        "title": "Version",
        "value": "'$VERSION'",
        "short": true
      }]
    }]
  }'
```

## Post-Deployment Verification

```bash
#!/bin/bash
# scripts/post-deploy-verify.sh

set -euo pipefail

ENVIRONMENT=${1:-production}
VERSION=${2:-latest}

echo "Running post-deployment verification..."

# Health checks
echo "Checking application health..."
curl -f https://reedcms.com/health || exit 1

# Functional tests
echo "Running functional tests..."
newman run tests/postman/production-tests.json \
  --environment tests/postman/production-env.json \
  --reporters cli,json \
  --reporter-json-export test-results.json

# Performance baseline
echo "Checking performance baseline..."
RESPONSE_TIME=$(curl -w "%{time_total}" -o /dev/null -s https://reedcms.com/api/v1/health)
if (( $(echo "$RESPONSE_TIME > 1.0" | bc -l) )); then
  echo "WARNING: Response time above threshold: ${RESPONSE_TIME}s"
fi

# Check critical endpoints
CRITICAL_ENDPOINTS=(
  "/api/v1/content"
  "/api/v1/users/me"
  "/api/v1/search"
)

for endpoint in "${CRITICAL_ENDPOINTS[@]}"; do
  echo "Checking $endpoint..."
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://reedcms.com$endpoint)
  if [ "$STATUS" != "200" ] && [ "$STATUS" != "401" ]; then
    echo "ERROR: $endpoint returned status $STATUS"
    exit 1
  fi
done

# Verify version
DEPLOYED_VERSION=$(curl -s https://reedcms.com/api/v1/version | jq -r .version)
if [ "$DEPLOYED_VERSION" != "$VERSION" ]; then
  echo "ERROR: Deployed version mismatch. Expected: $VERSION, Got: $DEPLOYED_VERSION"
  exit 1
fi

echo "Post-deployment verification completed successfully!"

# Update deployment tracker
curl -X POST https://deployments.reedcms.com/api/v1/deployments \
  -H "Authorization: Bearer $DEPLOYMENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "environment": "'$ENVIRONMENT'",
    "version": "'$VERSION'",
    "status": "success",
    "verified_at": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
  }'
```

## Summary

Diese Deployment Guide Implementation bietet:
- **Infrastructure Automation** - IaC mit Terraform/eksctl
- **Kubernetes Deployment** - Production-Ready Manifests
- **Zero-Downtime Strategy** - Blue-Green, Canary Deployments
- **Monitoring Integration** - Prometheus, Grafana, Alerts
- **Rollback Procedures** - Quick Recovery Methods
- **Verification Steps** - Post-Deployment Validation

Complete Deployment Guide für Production Excellence.