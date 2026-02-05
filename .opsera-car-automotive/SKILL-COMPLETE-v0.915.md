# =============================================================================
# OPSERA CODE-TO-CLOUD ENTERPRISE - COMPLETE SKILL v0.915
# =============================================================================
# This file contains ALL fixes, templates, and embedded files for integration
# into the base skill. Every file is complete and ready to use.
# =============================================================================

# SKILL METADATA
```yaml
name: code-to-cloud-enterprise
version: "0.915"
previous_version: "0.914"
release_date: "2026-02-05"
status: PRODUCTION-VERIFIED
total_rules: 157  # 153 from v0.914 + 4 new (150-153)
total_templates: 15
total_fixes: 5
```

---

# SECTION 1: SESSION ISSUE ANALYSIS

## 1.1 Issue Timeline

```
ISSUE #1: "default backend - 404" error
├── Symptom: Browser shows nginx default backend page
├── Root Cause #1: PLACEHOLDER_REPO_URL not replaced in ArgoCD application
├── Root Cause #2: targetRevision set to 'main' but code on 'car-automotive' branch
├── Impact: ArgoCD cannot sync, no manifests applied, no pods created
└── Fix: Replace placeholder + update branch

ISSUE #2: "CreateContainerConfigError" on all pods
├── Symptom: Pods stuck in error state, 0/1 Ready
├── Root Cause: runAsNonRoot: true without runAsUser/runAsGroup
├── Impact: Kubernetes cannot verify container runs as non-root
└── Fix: Add runAsUser/runAsGroup matching Dockerfile UIDs

ISSUE #3: Cannot see debug workflow output
├── Symptom: Workflow logs don't show kubectl output
├── Root Cause: Output only sent to GITHUB_STEP_SUMMARY, not stdout
├── Impact: Cannot diagnose issues from workflow logs
└── Fix: Use 'tee' command for dual output
```

## 1.2 Files That Were Modified

| File | Issue | Change Made |
|------|-------|-------------|
| `.opsera-car-automotive/argocd/dev/application.yaml` | PLACEHOLDER_REPO_URL | Replaced with actual GitHub URL |
| `.opsera-car-automotive/argocd/dev/application.yaml` | Wrong branch | Changed `main` to `car-automotive` |
| `.opsera-car-automotive/k8s/base/backend-deployment.yaml` | Missing UID | Added `runAsUser: 1001`, `runAsGroup: 1001` |
| `.opsera-car-automotive/k8s/base/frontend-deployment.yaml` | Missing UID | Added `runAsUser: 101`, `runAsGroup: 101` |
| `.github/workflows/verify-pods-car-automotive.yaml` | No stdout | Added `tee` for dual output |
| `.github/workflows/test-urls-car-automotive.yaml` | No stdout | Added `tee` for dual output |

---

# SECTION 2: NEW RULES (150-153)

## RULE 150: NEVER Generate Placeholder URLs in ArgoCD Applications
```yaml
rule_id: 150
severity: CRITICAL
category: ArgoCD Configuration
failure_symptom: "default backend - 404"
detection_command: "grep -r 'PLACEHOLDER' .opsera-*/argocd/"

description: |
  ArgoCD application.yaml MUST contain the actual GitHub repository URL
  at generation time. Never use PLACEHOLDER_REPO_URL or any placeholder
  that requires post-processing replacement.

wrong_pattern: |
  spec:
    source:
      repoURL: PLACEHOLDER_REPO_URL

correct_pattern: |
  spec:
    source:
      repoURL: https://github.com/{{ GITHUB_ORG }}/{{ GITHUB_REPO }}.git

fix_procedure: |
  1. Detect: grep "PLACEHOLDER" .opsera-*/argocd/*/application.yaml
  2. Get repo URL: git remote get-url origin
  3. Replace: sed -i "s|PLACEHOLDER_REPO_URL|<actual-url>|g" application.yaml
  4. Commit and push
  5. Re-run bootstrap workflow
```

## RULE 151: ArgoCD targetRevision Must Match Working Branch
```yaml
rule_id: 151
severity: CRITICAL
category: ArgoCD Configuration
failure_symptom: "ArgoCD shows OutOfSync but no resources created"
detection_command: |
  CURRENT=$(git branch --show-current)
  ARGOCD=$(grep "targetRevision:" .opsera-*/argocd/*/application.yaml | awk '{print $2}')
  [ "$CURRENT" != "$ARGOCD" ] && echo "MISMATCH: $CURRENT vs $ARGOCD"

description: |
  The ArgoCD application's targetRevision MUST match the branch where
  the Kubernetes manifests exist. If CI/CD triggers on 'feature-branch'
  but ArgoCD points to 'main', manifests won't be found.

wrong_pattern: |
  # Working on 'car-automotive' branch but ArgoCD says:
  spec:
    source:
      targetRevision: main

correct_pattern: |
  # Must match the actual working branch:
  spec:
    source:
      targetRevision: car-automotive

fix_procedure: |
  1. Detect: git branch --show-current
  2. Check ArgoCD: grep "targetRevision:" .opsera-*/argocd/*/application.yaml
  3. Update to match current branch
  4. Commit and push
  5. Re-run bootstrap workflow to re-apply ArgoCD application
```

## RULE 152: Security Context Must Include runAsUser When runAsNonRoot is True
```yaml
rule_id: 152
severity: CRITICAL
category: Kubernetes Security
failure_symptom: "CreateContainerConfigError"
detection_command: |
  for f in .opsera-*/k8s/base/*-deployment.yaml; do
    if grep -q "runAsNonRoot: true" "$f" && ! grep -q "runAsUser:" "$f"; then
      echo "MISSING runAsUser in $f"
    fi
  done

description: |
  When runAsNonRoot: true is set, Kubernetes needs to verify the container
  will run as non-root. Without runAsUser, it cannot verify this at pod
  creation time, causing CreateContainerConfigError.

wrong_pattern: |
  securityContext:
    runAsNonRoot: true
    allowPrivilegeEscalation: false

correct_pattern: |
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001      # Must match Dockerfile USER
    runAsGroup: 1001     # Must match Dockerfile GROUP
    allowPrivilegeEscalation: false

uid_reference:
  nginxinc/nginx-unprivileged: { uid: 101, gid: 101 }
  node:20-alpine (custom nodejs): { uid: 1001, gid: 1001 }
  python:3.11-slim (custom appuser): { uid: 1000, gid: 1000 }
  eclipse-temurin:17-jre (custom java): { uid: 1000, gid: 1000 }
  gcr.io/distroless/static: { uid: 65532, gid: 65532 }

fix_procedure: |
  1. Check Dockerfile for USER directive and UID
  2. Add runAsUser and runAsGroup to securityContext
  3. Ensure values match Dockerfile UID/GID
  4. Commit and push
  5. ArgoCD will auto-sync (or trigger manual sync)
```

## RULE 153: Debug Workflows Must Print to STDOUT and GITHUB_STEP_SUMMARY
```yaml
rule_id: 153
severity: HIGH
category: Observability
failure_symptom: "Cannot see diagnostic output in workflow logs"
detection_command: |
  grep -l ">> \$GITHUB_STEP_SUMMARY" .github/workflows/*debug*.yaml | \
  xargs grep -L "tee"

description: |
  Debug workflows must output to both STDOUT (visible in logs) and
  GITHUB_STEP_SUMMARY (visible in UI). Using only >> $GITHUB_STEP_SUMMARY
  makes troubleshooting impossible from logs.

wrong_pattern: |
  kubectl get pods -n $NAMESPACE >> $GITHUB_STEP_SUMMARY

correct_pattern: |
  kubectl get pods -n $NAMESPACE 2>&1 | tee -a $GITHUB_STEP_SUMMARY

fix_procedure: |
  1. Find all >> $GITHUB_STEP_SUMMARY usages
  2. Replace with: command 2>&1 | tee -a $GITHUB_STEP_SUMMARY
  3. For echo statements: echo "text" | tee -a $GITHUB_STEP_SUMMARY
  4. Commit and push
```

---

# SECTION 3: COMPLETE EMBEDDED FILES

## 3.1 ArgoCD Application - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/argocd/{{ ENV }}/application.yaml
# RULES: 59, 149, 150, 151
# =============================================================================
# COPY THIS ENTIRE FILE - Replace {{ variables }} with actual values
# =============================================================================

# ArgoCD Application for {{ APP_NAME }} {{ ENV }} environment
# RULE 59: Use destination.name for hub-spoke, NOT destination.server
# RULE 149: NEVER use https://kubernetes.default.svc (deploys to hub!)
# RULE 150: NEVER use PLACEHOLDER_REPO_URL - replaced at generation
# RULE 151: targetRevision MUST match working branch
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ APP_NAME }}-{{ ENV }}
  namespace: argocd
  labels:
    app: {{ APP_NAME }}
    environment: {{ ENV }}
    tenant: {{ TENANT }}
    managed-by: opsera-code-to-cloud
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    # RULE 150: Actual GitHub URL - NEVER use placeholders
    repoURL: https://github.com/{{ GITHUB_ORG }}/{{ GITHUB_REPO }}.git
    # RULE 151: Must match the branch where manifests exist
    targetRevision: {{ GIT_BRANCH }}
    path: .opsera-{{ APP_NAME }}/k8s/overlays/{{ ENV }}
  destination:
    # RULE 149: Use spoke cluster NAME for hub-spoke architecture
    # RULE 59: Never use server URL, always use name
    name: {{ SPOKE_CLUSTER }}
    namespace: {{ TENANT }}-{{ APP_NAME }}-{{ ENV }}
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - Replace=true      # RULE 109: Required for immutable field changes
      - PruneLast=true    # Prune after sync completes
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 3.1.1 Example: car-automotive DEV (ACTUAL FIXED FILE)

```yaml
# =============================================================================
# FILE: .opsera-car-automotive/argocd/dev/application.yaml
# THIS IS THE ACTUAL FIXED FILE FROM THIS SESSION
# =============================================================================

# ArgoCD Application for car-automotive DEV environment
# RULE 59: Use destination.name for hub-spoke, NOT destination.server
# RULE 149: NEVER use https://kubernetes.default.svc (deploys to hub!)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: car-automotive-dev
  namespace: argocd
  labels:
    app: car-automotive
    environment: dev
    tenant: opsera
spec:
  project: default
  source:
    # RULE 150: Actual repo URL - was PLACEHOLDER_REPO_URL before fix
    repoURL: https://github.com/opsera-agent-demos/Car_Automotivee.git
    # RULE 151: Changed from 'main' to 'car-automotive' to match working branch
    targetRevision: car-automotive
    path: .opsera-car-automotive/k8s/overlays/dev
  destination:
    # RULE 149: Use spoke cluster name, NOT server URL
    name: opsera-usw2-np
    namespace: opsera-car-automotive-dev
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - Replace=true  # RULE 109: Required for field changes
    automated:
      prune: true
      selfHeal: true
```

---

## 3.2 Backend Deployment - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/base/backend-deployment.yaml
# RULES: 74b, 152
# =============================================================================
# COPY THIS ENTIRE FILE - Replace {{ variables }} with actual values
# =============================================================================

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
          # RULE 152: COMPLETE security context with UID/GID
          # For Node.js with custom user (UID 1001)
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

### 3.2.1 Example: car-automotive Backend (ACTUAL FIXED FILE)

```yaml
# =============================================================================
# FILE: .opsera-car-automotive/k8s/base/backend-deployment.yaml
# THIS IS THE ACTUAL FIXED FILE FROM THIS SESSION
# =============================================================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-automotive-backend
  labels:
    app: car-automotive
    component: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: car-automotive
      component: backend
  template:
    metadata:
      labels:
        app: car-automotive
        component: backend
    spec:
      containers:
        - name: backend
          image: car-automotive-backend:latest
          ports:
            # RULE 74b: Port 8080 for non-root containers
            - containerPort: 8080
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
          livenessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
          # RULE 152: Added runAsUser/runAsGroup - was missing before fix
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            runAsGroup: 1001
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
```

---

## 3.3 Frontend Deployment - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/base/frontend-deployment.yaml
# RULES: 74b, 152
# =============================================================================
# COPY THIS ENTIRE FILE - Replace {{ variables }} with actual values
# =============================================================================

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
          # RULE 152: COMPLETE security context with UID/GID
          # For nginx-unprivileged (UID 101)
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

### 3.3.1 Example: car-automotive Frontend (ACTUAL FIXED FILE)

```yaml
# =============================================================================
# FILE: .opsera-car-automotive/k8s/base/frontend-deployment.yaml
# THIS IS THE ACTUAL FIXED FILE FROM THIS SESSION
# =============================================================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-automotive-frontend
  labels:
    app: car-automotive
    component: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: car-automotive
      component: frontend
  template:
    metadata:
      labels:
        app: car-automotive
        component: frontend
    spec:
      containers:
        - name: frontend
          image: car-automotive-frontend:latest
          ports:
            # RULE 74b: Port 8080 for non-root containers
            - containerPort: 8080
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
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
          # RULE 152: Added runAsUser/runAsGroup - was missing before fix
          securityContext:
            runAsNonRoot: true
            runAsUser: 101
            runAsGroup: 101
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
```

---

## 3.4 Backend Service - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/base/backend-service.yaml
# =============================================================================

apiVersion: v1
kind: Service
metadata:
  name: {{ APP_NAME }}-backend
  labels:
    app: {{ APP_NAME }}
    component: backend
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: {{ APP_NAME }}
    component: backend
```

---

## 3.5 Frontend Service - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/base/frontend-service.yaml
# =============================================================================

apiVersion: v1
kind: Service
metadata:
  name: {{ APP_NAME }}-frontend
  labels:
    app: {{ APP_NAME }}
    component: frontend
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: {{ APP_NAME }}
    component: frontend
```

---

## 3.6 Ingress - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/base/ingress.yaml
# =============================================================================

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ APP_NAME }}
  labels:
    app: {{ APP_NAME }}
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  rules:
    - host: {{ TENANT }}-{{ APP_NAME }}-{{ ENV }}.{{ DOMAIN }}
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: {{ APP_NAME }}-backend
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ APP_NAME }}-frontend
                port:
                  number: 8080
  tls:
    - hosts:
        - {{ TENANT }}-{{ APP_NAME }}-{{ ENV }}.{{ DOMAIN }}
      secretName: {{ APP_NAME }}-tls
```

---

## 3.7 Base Kustomization - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/base/kustomization.yaml
# =============================================================================

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - frontend-deployment.yaml
  - frontend-service.yaml
  - backend-deployment.yaml
  - backend-service.yaml
  - ingress.yaml

commonLabels:
  app.kubernetes.io/name: {{ APP_NAME }}
  app.kubernetes.io/managed-by: opsera-code-to-cloud
```

---

## 3.8 Overlay Kustomization (DEV) - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/overlays/dev/kustomization.yaml
# =============================================================================

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: {{ TENANT }}-{{ APP_NAME }}-dev

resources:
  - ../../base
  - namespace.yaml

# RULE 106: Use labels with includeSelectors: false, NOT commonLabels
labels:
  - pairs:
      environment: dev
    includeSelectors: false

images:
  - name: {{ APP_NAME }}-frontend
    newName: {{ AWS_ACCOUNT_ID }}.dkr.ecr.{{ AWS_REGION }}.amazonaws.com/{{ TENANT }}/{{ APP_NAME }}-frontend
    newTag: latest
  - name: {{ APP_NAME }}-backend
    newName: {{ AWS_ACCOUNT_ID }}.dkr.ecr.{{ AWS_REGION }}.amazonaws.com/{{ TENANT }}/{{ APP_NAME }}-backend
    newTag: latest

patches:
  # Update ingress host for dev environment
  - target:
      kind: Ingress
      name: {{ APP_NAME }}
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: {{ TENANT }}-{{ APP_NAME }}-dev.{{ DOMAIN }}
```

---

## 3.9 Namespace - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/k8s/overlays/dev/namespace.yaml
# =============================================================================

apiVersion: v1
kind: Namespace
metadata:
  name: {{ TENANT }}-{{ APP_NAME }}-dev
  labels:
    app: {{ APP_NAME }}
    environment: dev
    tenant: {{ TENANT }}
```

---

## 3.10 Dockerfile Backend (Node.js) - COMPLETE FILE

```dockerfile
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/Dockerfiles/Dockerfile.backend
# RULES: 73, 74b, 152-compatible (UID 1001)
# =============================================================================

# Multi-stage build for Node.js backend
# RULE 74b: Use port 8080 for non-root containers in Kubernetes
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# RULE 73: Use npm install --legacy-peer-deps instead of npm ci
RUN npm install --legacy-peer-deps --only=production && npm cache clean --force

# Production stage
FROM node:20-alpine

# RULE 152: Create non-root user with UID 1001 (matches securityContext)
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

# Health check on port 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8080/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "server.js"]
```

---

## 3.11 Dockerfile Frontend (React/Nginx) - COMPLETE FILE

```dockerfile
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/Dockerfiles/Dockerfile.frontend
# RULES: 73, 74, 74b, 152-compatible (UID 101)
# =============================================================================

# Multi-stage build for React frontend
# RULE 74b: Use port 8080 for non-root containers in Kubernetes
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# RULE 73: Use npm install --legacy-peer-deps instead of npm ci
RUN npm install --legacy-peer-deps && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build

# RULE 74: Use nginx-unprivileged for non-root security
# RULE 152: nginx-unprivileged uses UID 101 (matches securityContext)
FROM nginxinc/nginx-unprivileged:alpine

# Set working directory
WORKDIR /usr/share/nginx/html

# Remove default nginx static assets
USER root
RUN rm -rf ./*

# Copy built assets from builder
COPY --from=builder /app/dist .

# Copy nginx configuration (updated for port 8080)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# RULE 74b: nginx-unprivileged uses port 8080 by default
EXPOSE 8080

# Health check on port 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080 || exit 1

# Switch back to non-root user (UID 101)
USER nginx

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

---

## 3.12 Nginx Configuration - COMPLETE FILE

```nginx
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/nginx.conf
# RULE: 74b (port 8080)
# =============================================================================

server {
    # RULE 74b: Listen on 8080 for non-root containers
    listen 8080;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # SPA routing - serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy to backend service
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
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
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
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## 3.13 Verify Pods Workflow - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .github/workflows/verify-pods-{{ APP_NAME }}.yaml
# RULES: 50b, 153
# =============================================================================

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

      # RULE 153: Print to STDOUT AND GITHUB_STEP_SUMMARY using tee
      - name: Get Pod Status
        run: |
          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"

          echo "=========================================="
          echo "### Pod Status in $NAMESPACE" | tee -a $GITHUB_STEP_SUMMARY
          echo "=========================================="
          echo "" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          kubectl get pods -n $NAMESPACE -o wide 2>&1 | tee -a $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

          echo ""
          echo "### Deployments"
          kubectl get deployments -n $NAMESPACE 2>&1 | tee -a $GITHUB_STEP_SUMMARY

          echo ""
          echo "### Services"
          kubectl get services -n $NAMESPACE 2>&1 | tee -a $GITHUB_STEP_SUMMARY

          echo ""
          echo "### Ingress"
          kubectl get ingress -n $NAMESPACE 2>&1 | tee -a $GITHUB_STEP_SUMMARY

      - name: Get Events (for troubleshooting)
        run: |
          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"

          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Recent Events" | tee -a $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp' 2>&1 | tail -15 | tee -a $GITHUB_STEP_SUMMARY
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

      - name: Describe Failing Pods
        if: always()
        run: |
          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"

          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Failing Pod Details" | tee -a $GITHUB_STEP_SUMMARY

          for POD in $(kubectl get pods -n $NAMESPACE --field-selector=status.phase!=Running -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
            echo "" | tee -a $GITHUB_STEP_SUMMARY
            echo "#### Describing: $POD" | tee -a $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
            kubectl describe pod $POD -n $NAMESPACE 2>&1 | tail -50 | tee -a $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
          done
```

---

## 3.14 Test URLs Workflow - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .github/workflows/test-urls-{{ APP_NAME }}.yaml
# RULES: 50b, 153
# =============================================================================

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
      # RULE 153: Print to STDOUT AND GITHUB_STEP_SUMMARY
      - name: Test Application URL
        run: |
          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}"

          echo "==========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "### URL Accessibility Test" | tee -a $GITHUB_STEP_SUMMARY
          echo "==========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Test | URL | Status | Result |" >> $GITHUB_STEP_SUMMARY
          echo "|------|-----|--------|--------|" >> $GITHUB_STEP_SUMMARY

          echo ""
          echo "Testing Frontend: $URL"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL" --max-time 15 2>/dev/null || echo "000")
          echo "HTTP Status: $STATUS"

          if [ "$STATUS" = "200" ]; then
            echo "| Frontend | $URL | $STATUS | SUCCESS |" >> $GITHUB_STEP_SUMMARY
            echo "Result: SUCCESS"
          elif [ "$STATUS" = "404" ]; then
            echo "| Frontend | $URL | $STATUS | NOT FOUND |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED - 404 (check pods/ingress)"
          elif [ "$STATUS" = "503" ]; then
            echo "| Frontend | $URL | $STATUS | UNAVAILABLE |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED - 503 (pods not ready)"
          else
            echo "| Frontend | $URL | $STATUS | FAILED |" >> $GITHUB_STEP_SUMMARY
            echo "Result: FAILED"
          fi

      - name: Test API Health
        run: |
          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}/api/health"

          echo ""
          echo "Testing API: $URL"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL" --max-time 15 2>/dev/null || echo "000")
          echo "HTTP Status: $STATUS"

          if [ "$STATUS" = "200" ]; then
            echo "| API Health | $URL | $STATUS | HEALTHY |" >> $GITHUB_STEP_SUMMARY
            echo "Result: HEALTHY"
          else
            echo "| API Health | $URL | $STATUS | UNHEALTHY |" >> $GITHUB_STEP_SUMMARY
            echo "Result: UNHEALTHY"
          fi

      - name: Test with curl verbose (on failure)
        if: failure()
        run: |
          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}"

          echo ""
          echo "### Verbose curl output for debugging:"
          curl -v "$URL" --max-time 15 2>&1 || true
```

---

## 3.15 Diagnostics Workflow - COMPLETE FILE

```yaml
# =============================================================================
# FILE: .github/workflows/diagnostics-{{ APP_NAME }}.yaml
# RULES: 50b, 153
# =============================================================================

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

      # RULE 153: All diagnostics print to STDOUT
      - name: Stage 1 - ECR Check
        run: |
          echo "==========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 1: ECR Repositories" | tee -a $GITHUB_STEP_SUMMARY
          echo "==========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          for COMPONENT in frontend backend; do
            REPO="${{ env.TENANT }}/${{ env.APP_NAME }}-${COMPONENT}"
            echo "Checking: $REPO"

            if aws ecr describe-repositories --repository-names "$REPO" &>/dev/null; then
              IMAGES=$(aws ecr list-images --repository-name "$REPO" --query 'imageIds[*].imageTag' --output text 2>/dev/null | wc -w)
              echo "- ECR $COMPONENT: EXISTS ($IMAGES images)" | tee -a $GITHUB_STEP_SUMMARY
            else
              echo "- ECR $COMPONENT: NOT FOUND" | tee -a $GITHUB_STEP_SUMMARY
            fi
          done

      - name: Stage 2 - EKS Namespace Check
        run: |
          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 2: EKS Namespace" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          aws eks update-kubeconfig --name ${{ env.SPOKE_CLUSTER }} --region ${{ env.AWS_REGION }}

          NAMESPACE="${{ env.TENANT }}-${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"
          echo "Checking: $NAMESPACE"

          if kubectl get namespace $NAMESPACE &>/dev/null; then
            echo "- Namespace: EXISTS" | tee -a $GITHUB_STEP_SUMMARY
            PODS=$(kubectl get pods -n $NAMESPACE --no-headers 2>/dev/null | wc -l)
            RUNNING=$(kubectl get pods -n $NAMESPACE --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
            echo "- Total Pods: $PODS" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Running Pods: $RUNNING" | tee -a $GITHUB_STEP_SUMMARY
          else
            echo "- Namespace: NOT FOUND" | tee -a $GITHUB_STEP_SUMMARY
          fi

      - name: Stage 3 - ArgoCD Application Status
        run: |
          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 3: ArgoCD Application" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          aws eks update-kubeconfig --name ${{ env.HUB_CLUSTER }} --region ${{ env.AWS_REGION }}

          APP_NAME="${{ env.APP_NAME }}-${{ inputs.environment || 'dev' }}"
          echo "Checking: $APP_NAME"

          if kubectl get application $APP_NAME -n argocd &>/dev/null; then
            SYNC=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.status.sync.status}')
            HEALTH=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.status.health.status}')
            REPO=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.spec.source.repoURL}')
            BRANCH=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.spec.source.targetRevision}')
            PATH=$(kubectl get application $APP_NAME -n argocd -o jsonpath='{.spec.source.path}')

            echo "- Application: EXISTS" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Sync Status: $SYNC" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Health Status: $HEALTH" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Repository: $REPO" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Branch: $BRANCH" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Path: $PATH" | tee -a $GITHUB_STEP_SUMMARY

            # RULE 150 Check
            if [[ "$REPO" == *"PLACEHOLDER"* ]]; then
              echo "- **ERROR**: PLACEHOLDER_REPO_URL detected! (RULE 150 violation)" | tee -a $GITHUB_STEP_SUMMARY
            fi

            # Show sync errors if any
            if [ "$SYNC" != "Synced" ]; then
              echo "" | tee -a $GITHUB_STEP_SUMMARY
              echo "#### Sync Errors:" | tee -a $GITHUB_STEP_SUMMARY
              kubectl get application $APP_NAME -n argocd -o jsonpath='{.status.conditions[*].message}' 2>/dev/null | tee -a $GITHUB_STEP_SUMMARY
            fi
          else
            echo "- Application: NOT FOUND" | tee -a $GITHUB_STEP_SUMMARY
          fi

      - name: Stage 4 - URL Health Check
        run: |
          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Stage 4: URL Health Check" | tee -a $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          ENV="${{ inputs.environment || 'dev' }}"
          URL="https://${{ env.TENANT }}-${{ env.APP_NAME }}-${ENV}.${{ env.DOMAIN }}"

          echo "Testing: $URL"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL" --max-time 15 2>/dev/null || echo "000")

          echo "- URL: $URL" | tee -a $GITHUB_STEP_SUMMARY
          echo "- HTTP Status: $STATUS" | tee -a $GITHUB_STEP_SUMMARY

          if [ "$STATUS" = "200" ]; then
            echo "- Result: **HEALTHY**" | tee -a $GITHUB_STEP_SUMMARY
          elif [ "$STATUS" = "404" ]; then
            echo "- Result: **FAILED** - 404 Not Found" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Suggestion: Check pods are running and ingress is configured" | tee -a $GITHUB_STEP_SUMMARY
          elif [ "$STATUS" = "503" ]; then
            echo "- Result: **FAILED** - 503 Service Unavailable" | tee -a $GITHUB_STEP_SUMMARY
            echo "- Suggestion: Pods may be starting or unhealthy" | tee -a $GITHUB_STEP_SUMMARY
          else
            echo "- Result: **FAILED** - Status $STATUS" | tee -a $GITHUB_STEP_SUMMARY
          fi

      - name: Summary
        run: |
          echo "" | tee -a $GITHUB_STEP_SUMMARY
          echo "==========================================" | tee -a $GITHUB_STEP_SUMMARY
          echo "### Diagnostics Complete" | tee -a $GITHUB_STEP_SUMMARY
          echo "==========================================" | tee -a $GITHUB_STEP_SUMMARY
```

---

# SECTION 4: ERROR PATTERNS & AUTOMATIC FIXES

```yaml
# =============================================================================
# ERROR PATTERN DATABASE
# Use this to automatically detect and suggest fixes
# =============================================================================

ERROR_PATTERNS:

  - pattern: "default backend - 404"
    symptom: "Browser shows nginx default backend page"
    causes:
      - name: "PLACEHOLDER_REPO_URL not replaced"
        detection: "grep -q 'PLACEHOLDER' .opsera-*/argocd/*/application.yaml"
        fix: |
          REPO_URL=$(git remote get-url origin)
          sed -i "s|PLACEHOLDER_REPO_URL|${REPO_URL}|g" .opsera-*/argocd/*/application.yaml
        rule: 150

      - name: "ArgoCD targeting wrong branch"
        detection: |
          CURRENT=$(git branch --show-current)
          ARGOCD=$(grep "targetRevision:" .opsera-*/argocd/*/application.yaml | awk '{print $2}')
          [ "$CURRENT" != "$ARGOCD" ]
        fix: |
          CURRENT=$(git branch --show-current)
          sed -i "s|targetRevision:.*|targetRevision: ${CURRENT}|g" .opsera-*/argocd/*/application.yaml
        rule: 151

      - name: "No pods running"
        detection: "kubectl get pods -n $NAMESPACE --no-headers | wc -l | grep -q '^0$'"
        fix: "Re-run bootstrap workflow to apply ArgoCD application"

  - pattern: "CreateContainerConfigError"
    symptom: "Pods stuck in error state, 0/1 Ready"
    causes:
      - name: "Missing runAsUser in securityContext"
        detection: |
          grep -q "runAsNonRoot: true" .opsera-*/k8s/base/*-deployment.yaml && \
          ! grep -q "runAsUser:" .opsera-*/k8s/base/*-deployment.yaml
        fix: |
          # For backend (Node.js UID 1001):
          sed -i '/runAsNonRoot: true/a\            runAsUser: 1001\n            runAsGroup: 1001' backend-deployment.yaml
          # For frontend (nginx UID 101):
          sed -i '/runAsNonRoot: true/a\            runAsUser: 101\n            runAsGroup: 101' frontend-deployment.yaml
        rule: 152

  - pattern: "ImagePullBackOff"
    symptom: "Pods cannot pull container image"
    causes:
      - name: "ECR repository doesn't exist"
        detection: "aws ecr describe-repositories --repository-names $REPO 2>&1 | grep -q 'RepositoryNotFoundException'"
        fix: "Run bootstrap workflow to create ECR repositories"

      - name: "Image tag doesn't exist"
        detection: "aws ecr describe-images --repository-name $REPO --image-ids imageTag=$TAG 2>&1 | grep -q 'ImageNotFoundException'"
        fix: "Run CI workflow to build and push images"

  - pattern: "CrashLoopBackOff"
    symptom: "Pods keep restarting"
    causes:
      - name: "Application startup failure"
        detection: "kubectl logs $POD -n $NAMESPACE | grep -i 'error\\|exception\\|fatal'"
        fix: "Check application logs and fix startup issues"

      - name: "Port mismatch"
        detection: |
          CONTAINER_PORT=$(grep "containerPort:" deployment.yaml | awk '{print $2}')
          ENV_PORT=$(grep "value:" deployment.yaml | grep -A1 "PORT" | tail -1 | awk '{print $2}' | tr -d '"')
          [ "$CONTAINER_PORT" != "$ENV_PORT" ]
        fix: "Ensure containerPort matches PORT environment variable (should be 8080)"
```

---

# SECTION 5: VALIDATION SCRIPT

```bash
#!/bin/bash
# =============================================================================
# FILE: .opsera-{{ APP_NAME }}/scripts/validate-manifests.sh
# Run this before deployment to catch common issues
# =============================================================================

set -e

APP_NAME="${1:-$(basename $(pwd) | sed 's/^_//')}"
echo "Validating manifests for: $APP_NAME"

ERRORS=0

# RULE 150: Check for placeholder URLs
echo "Checking RULE 150 (no placeholders)..."
if grep -rq "PLACEHOLDER" .opsera-*/argocd/ 2>/dev/null; then
    echo "  ERROR: PLACEHOLDER values found in ArgoCD manifests"
    grep -r "PLACEHOLDER" .opsera-*/argocd/
    ERRORS=$((ERRORS + 1))
else
    echo "  OK: No placeholders found"
fi

# RULE 151: Check branch consistency
echo "Checking RULE 151 (branch match)..."
CURRENT_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
ARGOCD_BRANCH=$(grep "targetRevision:" .opsera-*/argocd/*/application.yaml 2>/dev/null | head -1 | awk '{print $2}')
if [ "$CURRENT_BRANCH" != "$ARGOCD_BRANCH" ] && [ -n "$ARGOCD_BRANCH" ]; then
    echo "  WARNING: Branch mismatch - Current: $CURRENT_BRANCH, ArgoCD: $ARGOCD_BRANCH"
else
    echo "  OK: Branch matches ($CURRENT_BRANCH)"
fi

# RULE 152: Check security context completeness
echo "Checking RULE 152 (security context)..."
for file in .opsera-*/k8s/base/*-deployment.yaml; do
    if [ -f "$file" ]; then
        if grep -q "runAsNonRoot: true" "$file"; then
            if ! grep -q "runAsUser:" "$file"; then
                echo "  ERROR: $file has runAsNonRoot but missing runAsUser"
                ERRORS=$((ERRORS + 1))
            elif ! grep -q "runAsGroup:" "$file"; then
                echo "  ERROR: $file has runAsNonRoot but missing runAsGroup"
                ERRORS=$((ERRORS + 1))
            else
                echo "  OK: $file has complete security context"
            fi
        fi
    fi
done

# Check UID values match expected
echo "Checking UID values..."
BACKEND_UID=$(grep -A2 "runAsUser:" .opsera-*/k8s/base/backend-deployment.yaml 2>/dev/null | head -1 | awk '{print $2}')
FRONTEND_UID=$(grep -A2 "runAsUser:" .opsera-*/k8s/base/frontend-deployment.yaml 2>/dev/null | head -1 | awk '{print $2}')

if [ "$BACKEND_UID" = "1001" ]; then
    echo "  OK: Backend UID is 1001 (Node.js)"
elif [ "$BACKEND_UID" = "1000" ]; then
    echo "  OK: Backend UID is 1000 (Python/Java)"
elif [ -n "$BACKEND_UID" ]; then
    echo "  INFO: Backend UID is $BACKEND_UID (verify matches Dockerfile)"
fi

if [ "$FRONTEND_UID" = "101" ]; then
    echo "  OK: Frontend UID is 101 (nginx-unprivileged)"
elif [ -n "$FRONTEND_UID" ]; then
    echo "  WARNING: Frontend UID is $FRONTEND_UID (expected 101 for nginx-unprivileged)"
fi

# Summary
echo ""
echo "==========================================="
if [ $ERRORS -gt 0 ]; then
    echo "VALIDATION FAILED: $ERRORS error(s) found"
    exit 1
else
    echo "VALIDATION PASSED: All checks passed"
    exit 0
fi
```

---

# SECTION 6: QUICK REFERENCE CARD

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    OPSERA CODE-TO-CLOUD v0.915 QUICK REFERENCE               ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  COMMON ERRORS & FIXES                                                       ║
║  ────────────────────                                                        ║
║                                                                              ║
║  "default backend - 404"                                                     ║
║    → Check: grep "PLACEHOLDER" .opsera-*/argocd/*/application.yaml           ║
║    → Check: git branch vs ArgoCD targetRevision                              ║
║    → Fix: Replace placeholder URL, update branch, re-run bootstrap           ║
║                                                                              ║
║  "CreateContainerConfigError"                                                ║
║    → Check: grep -A5 "runAsNonRoot" .opsera-*/k8s/base/*-deployment.yaml     ║
║    → Fix: Add runAsUser/runAsGroup matching Dockerfile UID                   ║
║                                                                              ║
║  UID REFERENCE                                                               ║
║  ─────────────                                                               ║
║    nginx-unprivileged  →  runAsUser: 101,  runAsGroup: 101                   ║
║    node:alpine custom  →  runAsUser: 1001, runAsGroup: 1001                  ║
║    python:slim custom  →  runAsUser: 1000, runAsGroup: 1000                  ║
║    distroless/static   →  runAsUser: 65532, runAsGroup: 65532                ║
║                                                                              ║
║  REQUIRED SECURITY CONTEXT                                                   ║
║  ─────────────────────────                                                   ║
║    securityContext:                                                          ║
║      runAsNonRoot: true                                                      ║
║      runAsUser: <UID>      # REQUIRED with runAsNonRoot                      ║
║      runAsGroup: <GID>     # REQUIRED with runAsNonRoot                      ║
║      allowPrivilegeEscalation: false                                         ║
║                                                                              ║
║  DEBUG WORKFLOW OUTPUT (RULE 153)                                            ║
║  ─────────────────────────────────                                           ║
║    WRONG:  kubectl get pods >> $GITHUB_STEP_SUMMARY                          ║
║    RIGHT:  kubectl get pods 2>&1 | tee -a $GITHUB_STEP_SUMMARY               ║
║                                                                              ║
║  ARGOCD APPLICATION (RULES 150, 151)                                         ║
║  ────────────────────────────────────                                        ║
║    spec:                                                                     ║
║      source:                                                                 ║
║        repoURL: https://github.com/ORG/REPO.git  # Never PLACEHOLDER         ║
║        targetRevision: <current-branch>          # Must match git branch     ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

# SECTION 7: INTEGRATION CHECKLIST

```markdown
## Steps to Integrate into Base Skill v0.914 → v0.915

### 1. Add New Rules
- [ ] Add RULE 150 to mandatory-rules.yaml
- [ ] Add RULE 151 to mandatory-rules.yaml
- [ ] Add RULE 152 to mandatory-rules.yaml
- [ ] Add RULE 153 to mandatory-rules.yaml

### 2. Update Templates
- [ ] Replace ArgoCD application template (Section 3.1)
- [ ] Replace backend-deployment template (Section 3.2)
- [ ] Replace frontend-deployment template (Section 3.3)
- [ ] Replace verify-pods workflow template (Section 3.13)
- [ ] Replace test-urls workflow template (Section 3.14)
- [ ] Replace diagnostics workflow template (Section 3.15)

### 3. Add New Components
- [ ] Add UID reference table to skill knowledge base
- [ ] Add error pattern database (Section 4)
- [ ] Add validation script (Section 5)
- [ ] Add quick reference card (Section 6)

### 4. Update Generation Logic
- [ ] Never generate PLACEHOLDER_REPO_URL - always use actual URL
- [ ] Use current git branch for targetRevision
- [ ] Always include runAsUser/runAsGroup with runAsNonRoot
- [ ] Always use tee in debug workflows

### 5. Test
- [ ] Deploy new application using updated skill
- [ ] Verify no PLACEHOLDER values in generated files
- [ ] Verify security context is complete
- [ ] Verify debug workflows show output in logs
- [ ] Verify application is accessible

### 6. Version Update
- [ ] Update skill version to v0.915
- [ ] Update changelog with new rules
- [ ] Document breaking changes (none expected)
```

---

# END OF SKILL ENHANCEMENT FILE
