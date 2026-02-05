# Opsera Code-to-Cloud Enterprise Skill Enhancement
# Version: v0.915 (Upgrade from v0.914)
# Date: 2026-02-05
# Status: PRODUCTION-VERIFIED

---

# CRITICAL RULES ADDITION (150-153)

## RULE 150: NEVER Use Placeholder URLs in ArgoCD Applications
```
SEVERITY: CRITICAL
CATEGORY: ArgoCD Configuration
FAILURE_MODE: Application shows "default backend - 404", no pods created

DESCRIPTION:
ArgoCD application.yaml MUST contain the actual GitHub repository URL at generation time.
Never use PLACEHOLDER_REPO_URL or any placeholder that requires post-processing.

DETECTION:
- grep -r "PLACEHOLDER" .opsera-*/argocd/

VALIDATION:
- repoURL must start with "https://github.com/" or "git@github.com:"
- repoURL must end with ".git"
- repoURL must contain actual org/repo names

BAD:
  repoURL: PLACEHOLDER_REPO_URL
  repoURL: ${REPO_URL}
  repoURL: __REPO_URL__

GOOD:
  repoURL: https://github.com/opsera-agent-demos/Car_Automotivee.git
```

## RULE 151: ArgoCD targetRevision Must Match Working Branch
```
SEVERITY: CRITICAL
CATEGORY: ArgoCD Configuration
FAILURE_MODE: ArgoCD syncs wrong branch, manifests not found, no pods created

DESCRIPTION:
The ArgoCD application's targetRevision MUST match the branch where the code and
manifests actually exist. Never hardcode "main" if CI/CD triggers on other branches.

DETECTION:
- Check CI workflow: grep "branches:" .github/workflows/*.yaml
- Check ArgoCD: grep "targetRevision:" .opsera-*/argocd/*/application.yaml
- These MUST match

VALIDATION:
- If CI triggers on [main, feature-branch], ArgoCD must use the active branch
- During generation, use the current git branch as targetRevision

BAD (if working on car-automotive branch):
  targetRevision: main

GOOD:
  targetRevision: car-automotive  # Matches actual working branch
```

## RULE 152: Complete Security Context Required for runAsNonRoot
```
SEVERITY: CRITICAL
CATEGORY: Kubernetes Security
FAILURE_MODE: CreateContainerConfigError, pods stuck in error state

DESCRIPTION:
When runAsNonRoot: true is set, you MUST specify runAsUser and runAsGroup
that match the UID/GID defined in the Dockerfile.

DETECTION:
- grep -A5 "runAsNonRoot: true" .opsera-*/k8s/base/*.yaml
- Verify runAsUser and runAsGroup are present

VALIDATION:
- runAsUser must match Dockerfile USER directive UID
- runAsGroup must match Dockerfile USER directive GID
- Values must be non-zero (non-root)

BAD:
  securityContext:
    runAsNonRoot: true
    allowPrivilegeEscalation: false

GOOD:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    allowPrivilegeEscalation: false
```

## RULE 153: Debug Workflows Must Output to STDOUT
```
SEVERITY: HIGH
CATEGORY: Observability
FAILURE_MODE: Cannot see diagnostic output in workflow logs

DESCRIPTION:
All debug/diagnostic workflows MUST print output to both STDOUT (for logs)
and GITHUB_STEP_SUMMARY (for UI). Use 'tee' command for dual output.

DETECTION:
- grep ">> \$GITHUB_STEP_SUMMARY" .github/workflows/*debug*.yaml
- Should also have corresponding "| tee" or echo statements

BAD:
  kubectl get pods >> $GITHUB_STEP_SUMMARY

GOOD:
  kubectl get pods 2>&1 | tee -a $GITHUB_STEP_SUMMARY
```

---

# DOCKERFILE UID REFERENCE TABLE

```yaml
# RULE 152 REFERENCE: Map base images to UIDs for securityContext
DOCKERFILE_UID_MAP:
  # Frontend Images
  nginxinc/nginx-unprivileged:
    user: nginx
    uid: 101
    gid: 101
    use_case: "React, Vue, Angular, static sites"

  nginx:alpine:
    user: nginx
    uid: 101
    gid: 101
    note: "Requires custom config for port 8080"

  # Backend Images - Node.js
  node:20-alpine:
    user: nodejs (custom)
    uid: 1001
    gid: 1001
    dockerfile_setup: |
      RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
      USER nodejs

  node:18-alpine:
    user: nodejs (custom)
    uid: 1001
    gid: 1001
    dockerfile_setup: |
      RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
      USER nodejs

  # Backend Images - Python
  python:3.11-slim:
    user: appuser (custom)
    uid: 1000
    gid: 1000
    dockerfile_setup: |
      RUN useradd -m -u 1000 appuser
      USER appuser

  python:3.12-slim:
    user: appuser (custom)
    uid: 1000
    gid: 1000
    dockerfile_setup: |
      RUN useradd -m -u 1000 appuser
      USER appuser

  # Backend Images - Java
  eclipse-temurin:17-jre:
    user: java (custom)
    uid: 1000
    gid: 1000
    dockerfile_setup: |
      RUN useradd -m -u 1000 java
      USER java

  openjdk:17-slim:
    user: java (custom)
    uid: 1000
    gid: 1000
    dockerfile_setup: |
      RUN useradd -m -u 1000 java
      USER java

  # Backend Images - Go
  golang:1.21-alpine:
    user: appuser (custom)
    uid: 1000
    gid: 1000
    dockerfile_setup: |
      RUN adduser -D -u 1000 appuser
      USER appuser

  # Scratch/Distroless (Go binaries)
  gcr.io/distroless/static:
    user: nonroot
    uid: 65532
    gid: 65532
    note: "Built-in nonroot user"
```

---

# EMBEDDED TEMPLATES

## TEMPLATE: ArgoCD Application (RULES 59, 149, 150, 151)

```yaml
# ============================================================================
# ARGOCD APPLICATION TEMPLATE
# Rules Applied: 59, 149, 150, 151
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 59: Use destination.name for hub-spoke, NOT destination.server
# RULE 149: NEVER use https://kubernetes.default.svc (deploys to hub!)
# RULE 150: NEVER use PLACEHOLDER_REPO_URL - use actual URL at generation
# RULE 151: targetRevision MUST match working branch
# ============================================================================

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ APP_NAME }}-{{ ENVIRONMENT }}
  namespace: argocd
  labels:
    app: {{ APP_NAME }}
    environment: {{ ENVIRONMENT }}
    tenant: {{ TENANT }}
    managed-by: opsera-code-to-cloud
spec:
  project: default
  source:
    # RULE 150: Actual repo URL - NEVER use placeholders
    repoURL: https://github.com/{{ GITHUB_ORG }}/{{ GITHUB_REPO }}.git
    # RULE 151: Must match the branch where code/manifests exist
    targetRevision: {{ GIT_BRANCH }}
    path: .opsera-{{ APP_NAME }}/k8s/overlays/{{ ENVIRONMENT }}
  destination:
    # RULE 149: Use spoke cluster NAME, never server URL
    # RULE 59: Hub-spoke pattern requires destination.name
    name: {{ SPOKE_CLUSTER }}
    namespace: {{ TENANT }}-{{ APP_NAME }}-{{ ENVIRONMENT }}
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - Replace=true  # RULE 109: Required for immutable field changes
    automated:
      prune: true
      selfHeal: true
```

## TEMPLATE: Backend Deployment - Node.js (RULES 74b, 152)

```yaml
# ============================================================================
# BACKEND DEPLOYMENT TEMPLATE - NODE.JS
# Rules Applied: 74b, 152
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 74b: Use port 8080 for non-root containers
# RULE 152: Complete securityContext with runAsUser/runAsGroup
# ============================================================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ APP_NAME }}-backend
  labels:
    app: {{ APP_NAME }}
    component: backend
    managed-by: opsera-code-to-cloud
spec:
  replicas: 2
  selector:
    matchLabels:
      app: {{ APP_NAME }}
      component: backend
  template:
    metadata:
      labels:
        app: {{ APP_NAME }}
        component: backend
    spec:
      containers:
        - name: backend
          image: {{ APP_NAME }}-backend:latest
          ports:
            # RULE 74b: Port 8080 for non-root containers
            - containerPort: 8080
              name: http
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
            - name: NODE_ENV
              value: "production"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          # RULE 152: Complete security context - Node.js uses UID 1001
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            runAsGroup: 1001
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
```

## TEMPLATE: Backend Deployment - Python (RULES 74b, 152)

```yaml
# ============================================================================
# BACKEND DEPLOYMENT TEMPLATE - PYTHON
# Rules Applied: 74b, 152
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ APP_NAME }}-backend
  labels:
    app: {{ APP_NAME }}
    component: backend
    managed-by: opsera-code-to-cloud
spec:
  replicas: 2
  selector:
    matchLabels:
      app: {{ APP_NAME }}
      component: backend
  template:
    metadata:
      labels:
        app: {{ APP_NAME }}
        component: backend
    spec:
      containers:
        - name: backend
          image: {{ APP_NAME }}-backend:latest
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
            - name: PYTHONUNBUFFERED
              value: "1"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
          # RULE 152: Complete security context - Python uses UID 1000
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            runAsGroup: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
```

## TEMPLATE: Frontend Deployment - Nginx (RULES 74b, 152)

```yaml
# ============================================================================
# FRONTEND DEPLOYMENT TEMPLATE - NGINX-UNPRIVILEGED
# Rules Applied: 74b, 152
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 74b: Use port 8080 for non-root containers
# RULE 152: Complete securityContext - nginx-unprivileged uses UID 101
# ============================================================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ APP_NAME }}-frontend
  labels:
    app: {{ APP_NAME }}
    component: frontend
    managed-by: opsera-code-to-cloud
spec:
  replicas: 2
  selector:
    matchLabels:
      app: {{ APP_NAME }}
      component: frontend
  template:
    metadata:
      labels:
        app: {{ APP_NAME }}
        component: frontend
    spec:
      containers:
        - name: frontend
          image: {{ APP_NAME }}-frontend:latest
          ports:
            # RULE 74b: Port 8080 for non-root containers
            - containerPort: 8080
              name: http
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          # RULE 152: Complete security context - nginx-unprivileged uses UID 101
          securityContext:
            runAsNonRoot: true
            runAsUser: 101
            runAsGroup: 101
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
```

## TEMPLATE: Dockerfile - Node.js Backend (RULES 73, 74b, 152-compatible)

```dockerfile
# ============================================================================
# DOCKERFILE TEMPLATE - NODE.JS BACKEND
# Rules Applied: 73, 74b, 152-compatible
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 73: Use npm install --legacy-peer-deps
# RULE 74b: Use port 8080 for non-root containers
# RULE 152: UID 1001 for securityContext compatibility
# ============================================================================

FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# RULE 73: Use npm install --legacy-peer-deps
RUN npm install --legacy-peer-deps --only=production && npm cache clean --force

# Production stage
FROM node:20-alpine

# RULE 152: Create non-root user with UID 1001
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app

# Copy dependencies from builder
COPY --from=builder /app/node_modules ./node_modules

# Copy application code
COPY . .

# Change ownership to non-root user
RUN chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# RULE 74b: Use port 8080 for non-root containers
ENV PORT=8080
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8080/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "server.js"]
```

## TEMPLATE: Dockerfile - React Frontend (RULES 73, 74, 74b, 152-compatible)

```dockerfile
# ============================================================================
# DOCKERFILE TEMPLATE - REACT FRONTEND (NGINX-UNPRIVILEGED)
# Rules Applied: 73, 74, 74b, 152-compatible
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 73: Use npm install --legacy-peer-deps
# RULE 74: Use nginx-unprivileged for non-root security
# RULE 74b: Use port 8080 for non-root containers
# RULE 152: UID 101 for securityContext compatibility
# ============================================================================

FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# RULE 73: Use npm install --legacy-peer-deps
RUN npm install --legacy-peer-deps && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build

# RULE 74: Use nginx-unprivileged for non-root security
FROM nginxinc/nginx-unprivileged:alpine

# Set working directory
WORKDIR /usr/share/nginx/html

# Remove default nginx static assets (requires root temporarily)
USER root
RUN rm -rf ./*

# Copy built assets from builder
COPY --from=builder /app/dist .

# Copy nginx configuration (must listen on 8080)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# RULE 74b: nginx-unprivileged uses port 8080 by default
EXPOSE 8080

# Health check on port 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080 || exit 1

# RULE 152: Switch back to nginx user (UID 101)
USER nginx

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

## TEMPLATE: nginx.conf for Frontend (RULE 74b)

```nginx
# ============================================================================
# NGINX CONFIGURATION - NON-ROOT (PORT 8080)
# Rules Applied: 74b
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================

server {
    # RULE 74b: Listen on 8080 for non-root containers
    listen 8080;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # SPA routing - serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy (if backend is in same cluster)
    location /api/ {
        proxy_pass http://{{ APP_NAME }}-backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

## TEMPLATE: Verify Pods Workflow (RULE 153)

```yaml
# ============================================================================
# DEBUG WORKFLOW - VERIFY PODS
# Rules Applied: 50b, 153
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 50b: Essential debug workflow for quick pod status checks
# RULE 153: Print to STDOUT and GITHUB_STEP_SUMMARY
# ============================================================================

name: "Debug: Verify Pods {{ APP_NAME }}"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, prod]
        default: dev

env:
  APP_NAME: {{ APP_NAME }}
  TENANT: {{ TENANT }}
  AWS_REGION: {{ AWS_REGION }}
  SPOKE_CLUSTER: {{ SPOKE_CLUSTER }}

permissions:
  contents: read
  id-token: write

jobs:
  verify-pods:
    name: "Verify Pods"
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Install kubectl
        run: |
          curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
          chmod +x kubectl && sudo mv kubectl /usr/local/bin/

      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig --name ${{ env.SPOKE_CLUSTER }} --region ${{ env.AWS_REGION }}

      # RULE 153: Print to STDOUT and GITHUB_STEP_SUMMARY
      - name: Get Pod Status
        run: |
          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"

          echo "========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Pod Status in $NAMESPACE" | tee -a $GITHUB_STEP_SUMMARY
          echo "========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          kubectl get pods -n $NAMESPACE -o wide 2>&1 | tee -a $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

          echo ""
          echo "### Deployment Status"
          kubectl get deployments -n $NAMESPACE -o wide 2>&1 | tee -a $GITHUB_STEP_SUMMARY

          echo ""
          echo "### Service Status"
          kubectl get services -n $NAMESPACE -o wide 2>&1 | tee -a $GITHUB_STEP_SUMMARY

          echo ""
          echo "### Ingress Status"
          kubectl get ingress -n $NAMESPACE -o wide 2>&1 | tee -a $GITHUB_STEP_SUMMARY

      # RULE 153: Show events for troubleshooting
      - name: Get Recent Events
        run: |
          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"

          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Recent Events (last 10)" | tee -a $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp' 2>&1 | tail -20 | tee -a $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

      - name: Get Pod Logs (last 50 lines)
        run: |
          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"

          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Pod Logs" | tee -a $GITHUB_STEP_SUMMARY

          for POD in $(kubectl get pods -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
            echo "" | tee -a $GITHUB_STEP_SUMMARY
            echo "#### Pod: $POD" | tee -a $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
            kubectl logs $POD -n $NAMESPACE --tail=50 2>&1 | tee -a $GITHUB_STEP_SUMMARY || echo "No logs available" | tee -a $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
          done
```

## TEMPLATE: Test URLs Workflow (RULE 153)

```yaml
# ============================================================================
# DEBUG WORKFLOW - TEST URLS
# Rules Applied: 50b, 153
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================
# RULE 50b: Essential debug workflow for URL accessibility testing
# RULE 153: Print to STDOUT and GITHUB_STEP_SUMMARY
# ============================================================================

name: "Debug: Test URLs {{ APP_NAME }}"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, prod]
        default: dev

env:
  APP_NAME: {{ APP_NAME }}
  TENANT: {{ TENANT }}
  DOMAIN: {{ DOMAIN }}

jobs:
  test-urls:
    name: "Test URLs"
    runs-on: ubuntu-latest
    steps:
      # RULE 153: Print to STDOUT and GITHUB_STEP_SUMMARY
      - name: Test Application URL
        run: |
          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}"

          echo "========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "### URL Accessibility Test" | tee -a $GITHUB_STEP_SUMMARY
          echo "========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Environment | URL | Status | Result |" >> $GITHUB_STEP_SUMMARY
          echo "|-------------|-----|--------|--------|" >> $GITHUB_STEP_SUMMARY

          echo "Testing: $URL"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL" --max-time 15 || echo "000")
          echo "HTTP Status Code: $STATUS"

          if [ "$STATUS" = "200" ]; then
            echo "| ${ENV^^} | $URL | $STATUS | SUCCESS |" >> $GITHUB_STEP_SUMMARY
            echo "Result: SUCCESS - Application is accessible"
          elif [ "$STATUS" = "301" ] || [ "$STATUS" = "302" ]; then
            echo "| ${ENV^^} | $URL | $STATUS | REDIRECT |" >> $GITHUB_STEP_SUMMARY
            echo "Result: REDIRECT - Following redirect..."
          elif [ "$STATUS" = "404" ]; then
            echo "| ${ENV^^} | $URL | $STATUS | NOT FOUND |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED - 404 Not Found (check ingress/pods)"
          elif [ "$STATUS" = "503" ]; then
            echo "| ${ENV^^} | $URL | $STATUS | UNAVAILABLE |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED - 503 Service Unavailable (pods not ready)"
          elif [ "$STATUS" = "000" ]; then
            echo "| ${ENV^^} | $URL | $STATUS | TIMEOUT |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED - Connection timeout (DNS or network issue)"
          else
            echo "| ${ENV^^} | $URL | $STATUS | ERROR |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED - Unexpected status code"
          fi

      - name: Test API Health Endpoint
        run: |
          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}/api/health"

          echo ""
          echo "Testing API Health: $URL"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL" --max-time 15 || echo "000")
          echo "API Health Status: $STATUS"

          if [ "$STATUS" = "200" ]; then
            echo "| ${ENV^^} API | $URL | $STATUS | HEALTHY |" >> $GITHUB_STEP_SUMMARY
            echo "API Result: HEALTHY"
          else
            echo "| ${ENV^^} API | $URL | $STATUS | UNHEALTHY |" >> $GITHUB_STEP_SUMMARY
            echo "API Result: UNHEALTHY or UNREACHABLE"
          fi
```

## TEMPLATE: Diagnostics Workflow (RULES 50b, 153)

```yaml
# ============================================================================
# DEBUG WORKFLOW - FULL DIAGNOSTICS
# Rules Applied: 50b, 153
# Generated by: Opsera Code-to-Cloud Enterprise v0.915
# ============================================================================

name: "Diagnostics: {{ APP_NAME }}"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, prod]
        default: dev

env:
  APP_NAME: {{ APP_NAME }}
  TENANT: {{ TENANT }}
  AWS_REGION: {{ AWS_REGION }}
  HUB_CLUSTER: {{ HUB_CLUSTER }}
  SPOKE_CLUSTER: {{ SPOKE_CLUSTER }}
  DOMAIN: {{ DOMAIN }}

permissions:
  contents: read
  id-token: write

jobs:
  diagnostics:
    name: "Full Pipeline Diagnostics"
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Install kubectl
        run: |
          curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
          chmod +x kubectl && sudo mv kubectl /usr/local/bin/

      # RULE 153: All output to STDOUT and GITHUB_STEP_SUMMARY
      - name: Stage 1 - ECR Check
        run: |
          echo "### Stage 1: ECR Repositories" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          for COMPONENT in frontend backend; do
            REPO="${{ env.TENANT }}/${{ env.APP_NAME }}-${COMPONENT}"
            echo "Checking ECR: $REPO"

            if aws ecr describe-repositories --repository-names "$REPO" &>/dev/null; then
              IMAGES=$(aws ecr list-images --repository-name "$REPO" --query 'imageIds[*].imageTag' --output text | wc -w)
              echo "- ECR $COMPONENT: EXISTS ($IMAGES images)" | tee -a $GITHUB_STEP_SUMMARY
            else
              echo "- ECR $COMPONENT: NOT FOUND" | tee -a $GITHUB_STEP_SUMMARY
            fi
          done

      - name: Stage 2 - EKS/Namespace Check
        run: |
          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 2: EKS Namespace" | tee -a $GITHUB_STEP_SUMMARY

          aws eks update-kubeconfig --name ${{ env.SPOKE_CLUSTER }} --region ${{ env.AWS_REGION }}

          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"
          echo "Checking namespace: $NAMESPACE"

          if kubectl get namespace $NAMESPACE &>/dev/null; then
            echo "- Namespace: EXISTS" | tee -a $GITHUB_STEP_SUMMARY
            PODS=$(kubectl get pods -n $NAMESPACE --no-headers 2>/dev/null | wc -l)
            echo "- Pods: $PODS" | tee -a $GITHUB_STEP_SUMMARY
          else
            echo "- Namespace: NOT FOUND" | tee -a $GITHUB_STEP_SUMMARY
          fi

      - name: Stage 3 - ArgoCD Status
        run: |
          aws eks update-kubeconfig --name ${{ env.HUB_CLUSTER }} --region ${{ env.AWS_REGION }}

          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 3: ArgoCD Application" | tee -a $GITHUB_STEP_SUMMARY

          APP_NAME="${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"
          echo "Checking ArgoCD app: $APP_NAME"

          if kubectl get application $APP_NAME -n argocd &>/dev/null; then
            SYNC=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.status.sync.status}')
            HEALTH=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.status.health.status}')
            REPO=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.spec.source.repoURL}')
            BRANCH=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.spec.source.targetRevision}')

            echo "- Application: EXISTS" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Sync Status: $SYNC" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Health Status: $HEALTH" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Repository: $REPO" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Branch: $BRANCH" | tee -a $GITHUB_STEP_SUMMARY

            # RULE 150 Check
            if [[ "$REPO" == *"PLACEHOLDER"* ]]; then
              echo "- WARNING: PLACEHOLDER_REPO_URL detected! (RULE 150 violation)" | tee -a $GITHUB_STEP_SUMMARY
            fi
          else
            echo "- Application: NOT FOUND" | tee -a $GITHUB_STEP_SUMMARY
          fi

      - name: Stage 4 - Health Check
        run: |
          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 4: Health Endpoints" | tee -a $GITHUB_STEP_SUMMARY

          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}"

          echo "Testing: $URL"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL" --max-time 10 || echo "000")

          echo "- URL: $URL" | tee -a $GITHUB_STEP_SUMMARY
          echo "- HTTP Status: $STATUS" | tee -a $GITHUB_STEP_SUMMARY

          if [ "$STATUS" = "200" ] || [ "$STATUS" = "301" ] || [ "$STATUS" = "302" ]; then
            echo "- Result: HEALTHY" | tee -a $GITHUB_STEP_SUMMARY
          else
            echo "- Result: UNHEALTHY or UNREACHABLE" | tee -a $GITHUB_STEP_SUMMARY
          fi
```

---

# VALIDATION FUNCTIONS

```yaml
# ============================================================================
# PRE-DEPLOYMENT VALIDATION CHECKS
# Run these before applying any manifests
# ============================================================================

VALIDATION_CHECKS:

  # RULE 150: Check for placeholder URLs
  check_no_placeholders:
    command: |
      if grep -r "PLACEHOLDER" .opsera-*/argocd/ .opsera-*/k8s/; then
        echo "ERROR: Placeholder values found! Replace before deployment."
        exit 1
      fi
    error_message: "RULE 150 VIOLATION: PLACEHOLDER values found in manifests"

  # RULE 151: Check branch consistency
  check_branch_consistency:
    command: |
      CURRENT_BRANCH=$(git branch --show-current)
      ARGOCD_BRANCH=$(grep "targetRevision:" .opsera-*/argocd/*/application.yaml | head -1 | awk '{print $2}')
      if [ "$CURRENT_BRANCH" != "$ARGOCD_BRANCH" ]; then
        echo "WARNING: Current branch ($CURRENT_BRANCH) differs from ArgoCD targetRevision ($ARGOCD_BRANCH)"
      fi
    error_message: "RULE 151 WARNING: Branch mismatch detected"

  # RULE 152: Check security context completeness
  check_security_context:
    command: |
      for file in .opsera-*/k8s/base/*-deployment.yaml; do
        if grep -q "runAsNonRoot: true" "$file"; then
          if ! grep -q "runAsUser:" "$file"; then
            echo "ERROR: $file has runAsNonRoot but missing runAsUser"
            exit 1
          fi
          if ! grep -q "runAsGroup:" "$file"; then
            echo "ERROR: $file has runAsNonRoot but missing runAsGroup"
            exit 1
          fi
        fi
      done
    error_message: "RULE 152 VIOLATION: Incomplete securityContext found"
```

---

# ERROR PATTERN RECOGNITION

```yaml
# ============================================================================
# COMMON ERROR PATTERNS AND SOLUTIONS
# ============================================================================

ERROR_PATTERNS:

  "default backend - 404":
    causes:
      - "ArgoCD not syncing (PLACEHOLDER_REPO_URL)"
      - "ArgoCD targeting wrong branch"
      - "Ingress not created"
      - "No pods running"
    diagnostics:
      - "Run verify-pods workflow"
      - "Check ArgoCD application status"
      - "Verify ingress exists"
    solutions:
      - "RULE 150: Replace PLACEHOLDER_REPO_URL with actual URL"
      - "RULE 151: Update targetRevision to match working branch"

  "CreateContainerConfigError":
    causes:
      - "runAsNonRoot: true without runAsUser"
      - "UID mismatch between Dockerfile and securityContext"
      - "Missing secrets/configmaps"
    diagnostics:
      - "kubectl describe pod <pod-name>"
      - "Check events in namespace"
    solutions:
      - "RULE 152: Add runAsUser and runAsGroup matching Dockerfile"
      - "Verify UID from Dockerfile USER directive"

  "ImagePullBackOff":
    causes:
      - "ECR repository doesn't exist"
      - "Image tag doesn't exist"
      - "IAM permissions issue"
    diagnostics:
      - "Check ECR repository exists"
      - "Verify image tag in ECR"
      - "Check node IAM role permissions"
    solutions:
      - "Run bootstrap workflow to create ECR repos"
      - "Run CI workflow to build and push images"

  "CrashLoopBackOff":
    causes:
      - "Application startup failure"
      - "Missing environment variables"
      - "Port mismatch"
    diagnostics:
      - "kubectl logs <pod-name>"
      - "Check PORT environment variable"
    solutions:
      - "RULE 74b: Ensure PORT=8080 in deployment"
      - "Check application logs for startup errors"
```

---

# INTEGRATION CHECKLIST

```markdown
## To integrate this skill enhancement into base v0.914:

1. [ ] Add RULES 150-153 to mandatory rules list
2. [ ] Update ArgoCD application template (never use placeholders)
3. [ ] Update deployment templates (always include complete securityContext)
4. [ ] Update Dockerfile templates (use consistent UIDs)
5. [ ] Update debug workflow templates (use tee for dual output)
6. [ ] Add UID reference table to skill knowledge base
7. [ ] Add validation checks to bootstrap workflow
8. [ ] Add error pattern recognition for quick troubleshooting
9. [ ] Update version to v0.915

## Files to update in base skill:
- templates/argocd/application.yaml
- templates/k8s/backend-deployment.yaml
- templates/k8s/frontend-deployment.yaml
- templates/dockerfiles/Dockerfile.backend.node
- templates/dockerfiles/Dockerfile.frontend.nginx
- templates/workflows/verify-pods.yaml
- templates/workflows/test-urls.yaml
- templates/workflows/diagnostics.yaml
- rules/mandatory-rules.yaml
- knowledge/uid-reference.yaml
- knowledge/error-patterns.yaml
```

---

# VERSION HISTORY

| Version | Date | Changes |
|---------|------|---------|
| v0.914 | 2026-02-04 | Previous stable release |
| v0.915 | 2026-02-05 | Added RULES 150-153, fixed placeholder handling, security context, debug output |
