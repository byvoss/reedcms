# CI/CD Pipeline

## Overview

CI/CD Pipeline Implementation für ReedCMS. Automated Build, Test und Deployment mit GitHub Actions.

## GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *' # Nightly builds

env:
  RUST_VERSION: 1.75.0
  CARGO_TERM_COLOR: always
  RUSTFLAGS: '-D warnings'
  SQLX_OFFLINE: true

jobs:
  # Code quality checks
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: ${{ env.RUST_VERSION }}
          components: rustfmt, clippy
      
      - name: Cache dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: quality-${{ runner.os }}
      
      - name: Check formatting
        run: cargo fmt --all -- --check
      
      - name: Run Clippy
        run: cargo clippy --all-targets --all-features -- -D warnings
      
      - name: Check dependencies
        run: |
          cargo install cargo-audit cargo-outdated
          cargo audit
          cargo outdated --exit-code 1
  
  # Security scanning
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run security audit
        uses: actions-rs/audit-check@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: SAST with Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >
            p/rust
            p/security-audit
            p/secrets
      
      - name: Dependency scanning
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
  
  # Unit tests
  test:
    name: Tests (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        rust: [stable, beta]
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: reedcms_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: ${{ matrix.rust }}
      
      - name: Cache dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: test-${{ runner.os }}-${{ matrix.rust }}
      
      - name: Run migrations
        run: |
          cargo install sqlx-cli --no-default-features --features postgres
          sqlx migrate run
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/reedcms_test
      
      - name: Run tests
        run: cargo test --all-features --workspace
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/reedcms_test
          REDIS_URL: redis://localhost:6379
          RUST_BACKTRACE: full
      
      - name: Generate test report
        if: always()
        run: |
          cargo install cargo-junit
          cargo test --all-features --no-fail-fast -- -Z unstable-options --format json | cargo-junit > results.xml
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.rust }}
          path: results.xml
  
  # Integration tests
  integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [quality, test]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Start services
        run: docker-compose up -d
      
      - name: Wait for services
        run: |
          timeout 60 bash -c 'until docker-compose ps | grep -q "healthy"; do sleep 1; done'
      
      - name: Run integration tests
        run: cargo test --test '*' --features integration
      
      - name: Collect logs
        if: failure()
        run: docker-compose logs > docker-logs.txt
      
      - name: Upload logs
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: integration-logs
          path: docker-logs.txt
  
  # Coverage
  coverage:
    name: Code Coverage
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: nightly
          components: llvm-tools-preview
      
      - name: Install grcov
        run: cargo install grcov
      
      - name: Run tests with coverage
        run: |
          export CARGO_INCREMENTAL=0
          export RUSTFLAGS="-Cinstrument-coverage"
          export LLVM_PROFILE_FILE="cargo-test-%p-%m.profraw"
          cargo test --all-features
      
      - name: Generate coverage report
        run: |
          grcov . --binary-path ./target/debug/deps/ -s . -t lcov --branch --ignore-not-existing --ignore '../*' --ignore "/*" -o coverage.lcov
      
      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.lcov
          flags: unittests
          fail_ci_if_error: true
  
  # Build artifacts
  build:
    name: Build (${{ matrix.target }})
    runs-on: ${{ matrix.os }}
    needs: [quality, test]
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
          - target: x86_64-apple-darwin
            os: macos-latest
          - target: x86_64-pc-windows-msvc
            os: windows-latest
          - target: aarch64-unknown-linux-gnu
            os: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
      
      - name: Install cross-compilation tools
        if: matrix.target == 'aarch64-unknown-linux-gnu'
        run: |
          sudo apt-get update
          sudo apt-get install -y gcc-aarch64-linux-gnu
      
      - name: Build release
        run: cargo build --release --target ${{ matrix.target }}
        env:
          CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER: aarch64-linux-gnu-gcc
      
      - name: Package artifacts
        run: |
          mkdir -p dist
          cp target/${{ matrix.target }}/release/reedcms* dist/
          cp -r config dist/
          cp README.md LICENSE dist/
          tar czf reedcms-${{ matrix.target }}.tar.gz -C dist .
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: reedcms-${{ matrix.target }}
          path: reedcms-${{ matrix.target }}.tar.gz

# .github/workflows/cd.yml
name: CD Pipeline

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  # Create release
  release:
    name: Create Release
    runs-on: ubuntu-latest
    outputs:
      upload_url: ${{ steps.create_release.outputs.upload_url }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate changelog
        id: changelog
        run: |
          echo "CHANGELOG<<EOF" >> $GITHUB_OUTPUT
          git log --pretty=format:"- %s" $(git describe --tags --abbrev=0)..HEAD >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create Release
        id: create_release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: |
            ## Changes
            ${{ steps.changelog.outputs.CHANGELOG }}
            
            ## Docker Images
            - `ghcr.io/reedcms/reedcms:${{ github.ref_name }}`
            - `ghcr.io/reedcms/reedcms:latest`
          draft: false
          prerelease: false
  
  # Build and push Docker images
  docker:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: release
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=sha
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.ref_name }}
            COMMIT=${{ github.sha }}
  
  # Deploy to staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: docker
    if: github.event_name == 'push' || github.event.inputs.environment == 'staging'
    environment:
      name: staging
      url: https://staging.reedcms.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Kubernetes
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          
          helm upgrade --install reedcms-staging ./helm/reedcms \
            --namespace staging \
            --create-namespace \
            --set image.tag=${{ github.ref_name }} \
            --set ingress.hosts[0].host=staging.reedcms.com \
            --values ./helm/values.staging.yaml \
            --wait
      
      - name: Run smoke tests
        run: |
          npm install -g newman
          newman run tests/postman/smoke-tests.json \
            --environment tests/postman/staging.json \
            --reporters cli,junit \
            --reporter-junit-export smoke-results.xml
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: smoke-test-results
          path: smoke-results.xml
  
  # Deploy to production
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [docker, deploy-staging]
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://reedcms.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Kubernetes
        run: |
          echo "${{ secrets.KUBECONFIG_PROD }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          
          # Blue-green deployment
          helm upgrade --install reedcms-green ./helm/reedcms \
            --namespace production \
            --set image.tag=${{ github.ref_name }} \
            --set service.selector=green \
            --values ./helm/values.production.yaml \
            --wait
          
          # Run health checks
          kubectl wait --for=condition=ready pod -l app=reedcms,version=green -n production --timeout=300s
          
          # Switch traffic
          kubectl patch service reedcms -n production -p '{"spec":{"selector":{"version":"green"}}}'
          
          # Wait and verify
          sleep 60
          curl -f https://reedcms.com/health || exit 1
          
          # Cleanup old deployment
          helm uninstall reedcms-blue -n production || true
      
      - name: Create deployment record
        run: |
          curl -X POST https://api.reedcms.com/deployments \
            -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "version": "${{ github.ref_name }}",
              "commit": "${{ github.sha }}",
              "environment": "production",
              "deployed_by": "${{ github.actor }}",
              "deployed_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
            }'
```

## Dockerfile

```dockerfile
# Multi-stage Dockerfile
FROM rust:1.75 AS builder

# Install dependencies
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Copy manifests
COPY Cargo.toml Cargo.lock ./

# Build dependencies
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Copy source code
COPY . .

# Build application
RUN cargo build --release --bin reedcms

# Runtime stage
FROM debian:bookworm-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r reedcms && useradd -r -g reedcms reedcms

# Copy binary from builder
COPY --from=builder /app/target/release/reedcms /usr/local/bin/reedcms

# Copy configuration
COPY --from=builder /app/config /etc/reedcms

# Set ownership
RUN chown -R reedcms:reedcms /etc/reedcms

# Create data directory
RUN mkdir -p /var/lib/reedcms && chown -R reedcms:reedcms /var/lib/reedcms

# Switch to non-root user
USER reedcms

# Expose ports
EXPOSE 8080 9090

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["reedcms", "health"]

# Entry point
ENTRYPOINT ["reedcms"]
CMD ["serve"]
```

## Helm Chart

```yaml
# helm/reedcms/Chart.yaml
apiVersion: v2
name: reedcms
description: A Helm chart for ReedCMS
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled

# helm/reedcms/values.yaml
replicaCount: 2

image:
  repository: ghcr.io/reedcms/reedcms
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: reedcms.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: reedcms-tls
      hosts:
        - reedcms.com

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    database: reedcms
    username: reedcms
    existingSecret: reedcms-postgresql
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: true
    existingSecret: reedcms-redis
  master:
    persistence:
      size: 5Gi

config:
  environment: production
  logLevel: info
  features:
    - search
    - caching
    - plugins

# helm/reedcms/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "reedcms.fullname" . }}
  labels:
    {{- include "reedcms.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "reedcms.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "reedcms.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "reedcms.serviceAccountName" . }}
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
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: RUST_LOG
              value: {{ .Values.config.logLevel }}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "reedcms.fullname" . }}-database
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "reedcms.fullname" . }}-redis
                  key: url
          volumeMounts:
            - name: config
              mountPath: /etc/reedcms
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: {{ include "reedcms.fullname" . }}
```

## Deployment Scripts

```bash
#!/bin/bash
# scripts/deploy.sh

set -euo pipefail

ENVIRONMENT=${1:-staging}
VERSION=${2:-latest}

echo "Deploying ReedCMS $VERSION to $ENVIRONMENT"

# Validate environment
case $ENVIRONMENT in
  staging|production)
    ;;
  *)
    echo "Invalid environment: $ENVIRONMENT"
    exit 1
    ;;
esac

# Load environment config
source "deploy/config/$ENVIRONMENT.env"

# Pre-deployment checks
echo "Running pre-deployment checks..."
./scripts/pre-deploy-check.sh $ENVIRONMENT

# Database migrations
echo "Running database migrations..."
kubectl exec -n $NAMESPACE deployment/reedcms -- reedcms migrate

# Deploy
echo "Deploying application..."
helm upgrade --install reedcms ./helm/reedcms \
  --namespace $NAMESPACE \
  --set image.tag=$VERSION \
  --values helm/values.$ENVIRONMENT.yaml \
  --wait

# Post-deployment verification
echo "Verifying deployment..."
./scripts/verify-deployment.sh $ENVIRONMENT

echo "Deployment complete!"
```

## Monitoring Integration

```yaml
# .github/workflows/monitoring.yml
name: Deployment Monitoring

on:
  deployment_status:

jobs:
  notify:
    runs-on: ubuntu-latest
    if: github.event.deployment_status.state == 'success' || github.event.deployment_status.state == 'failure'
    
    steps:
      - name: Send metrics
        run: |
          curl -X POST https://metrics.reedcms.com/deployments \
            -H "Authorization: Bearer ${{ secrets.METRICS_TOKEN }}" \
            -d '{
              "environment": "${{ github.event.deployment.environment }}",
              "status": "${{ github.event.deployment_status.state }}",
              "version": "${{ github.event.deployment.ref }}",
              "duration": ${{ github.event.deployment_status.created_at - github.event.deployment.created_at }}
            }'
      
      - name: Update status page
        if: github.event.deployment.environment == 'production'
        run: |
          STATUS="operational"
          if [ "${{ github.event.deployment_status.state }}" != "success" ]; then
            STATUS="partial_outage"
          fi
          
          curl -X PATCH https://status.reedcms.com/api/v1/components/api \
            -H "Authorization: Bearer ${{ secrets.STATUS_PAGE_TOKEN }}" \
            -d "{\"status\": \"$STATUS\"}"
```

## Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "reedcms/maintainers"
    labels:
      - "dependencies"
      - "rust"
  
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "github-actions"
  
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "docker"

# .github/renovate.json
{
  "extends": [
    "config:base",
    ":dependencyDashboard",
    ":semanticCommits"
  ],
  "packageRules": [
    {
      "matchPackagePatterns": ["*"],
      "rangeStrategy": "bump"
    },
    {
      "matchDepTypes": ["devDependencies"],
      "automerge": true
    }
  ],
  "vulnerabilityAlerts": {
    "labels": ["security"],
    "assignees": ["@security-team"]
  }
}
```

## Summary

Diese CI/CD Pipeline Implementation bietet:
- **Automated Testing** - Unit, Integration, E2E Tests
- **Code Quality** - Linting, Formatting, Security Scans
- **Build Automation** - Multi-Platform Builds
- **Container Registry** - Docker Image Management
- **Kubernetes Deployment** - Helm Charts, GitOps
- **Monitoring Integration** - Metrics, Status Updates

Complete CI/CD Pipeline für Professional Deployment.