# 🔧 READY-TO-USE YAML TEMPLATES

These are copy-paste ready templates for the most critical improvements.

---

## 1. BACKEND CONFIGMAP (apps/satmonitor/base/backend-login/01-configmap.yaml)

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: satmonitor-backend-config
  namespace: satmonitor-app
  labels:
    app.kubernetes.io/name: satmonitor-backend-login
    app.kubernetes.io/part-of: satmonitor
data:
  # Keycloak Configuration
  keycloak_server_url: "https://keycloak.example.com"
  keycloak_realm: "satmonitor"
  keycloak_client_id: "satmonitor-backend"
  
  # Database Configuration
  db_url: "jdbc:postgresql://satmonitor-postgres.satmonitor-app.svc.cluster.local:5432/satmonitor_db"
  db_driver: "org.postgresql.Driver"
  
  # Spring Configuration
  spring_profiles_active: "kubernetes"
  spring_jpa_hibernate_ddl_auto: "validate"
  
  # Logging
  logging_level_root: "INFO"
  logging_level_com_satmonitor: "DEBUG"
```

---

## 2. BACKEND SECRETS (apps/satmonitor/base/backend-login/02-secrets.yaml)

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: satmonitor-backend-secrets
  namespace: satmonitor-app
  labels:
    app.kubernetes.io/name: satmonitor-backend-login
    app.kubernetes.io/part-of: satmonitor
type: Opaque
stringData:
  keycloak_client_secret: "change-me-to-real-value"  # Update in production
  db_username: "satmonitor_app"
  db_password: "change-me-to-real-password"
```

---

## 3. BACKEND SERVICEACCOUNT & RBAC (apps/satmonitor/base/01-rbac.yaml)

```yaml
---
# ServiceAccount for backend pods
apiVersion: v1
kind: ServiceAccount
metadata:
  name: satmonitor-backend-login
  namespace: satmonitor-app
  labels:
    app.kubernetes.io/name: satmonitor-backend-login
    app.kubernetes.io/part-of: satmonitor

---
# Minimal permissions: read config and secrets only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: satmonitor-backend-login
  namespace: satmonitor-app
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
    resourceNames: ["satmonitor-backend-config"]
  
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["satmonitor-backend-secrets"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: satmonitor-backend-login
  namespace: satmonitor-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: satmonitor-backend-login
subjects:
  - kind: ServiceAccount
    name: satmonitor-backend-login
    namespace: satmonitor-app
```

---

## 4. NETWORKPOLICY (apps/satmonitor/base/02-networkpolicy.yaml)

```yaml
---
# Default deny all ingress and egress (explicit allow policy)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: satmonitor-app-default-deny
  namespace: satmonitor-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow ingress from Traefik
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: satmonitor-app-allow-ingress
  namespace: satmonitor-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: traefik

---
# Allow inter-pod communication (backend-to-db, frontend-to-backend, etc.)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: satmonitor-app-allow-internal
  namespace: satmonitor-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}

---
# Allow egress to DNS (required for all pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: satmonitor-app-allow-dns
  namespace: satmonitor-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53

---
# Allow egress to other namespaces (backend to keycloak, frontend to backend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: satmonitor-app-allow-external-egress
  namespace: satmonitor-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # To platform services (Keycloak)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: satmonitor-platform
      ports:
        - protocol: TCP
          port: 8080
    
    # External egress (to DockerHub, etc.)
    - to:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 443
```

---

## 5. CENTRALIZED APP INGRESS (clusters/srvapplis/ingresscontroller/01-application-ingress.yaml)

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: satmonitor-app
  namespace: satmonitor-app
  labels:
    app.kubernetes.io/name: satmonitor-ingress
    app.kubernetes.io/part-of: satmonitor
  annotations:
    # Traefik annotations
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.middlewares: satmonitor-app-security@kubernetescrd
    
    # TLS (when ready)
    # cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  ingressClassName: traefik
  
  # TLS configuration (enable when certificate is ready)
  # tls:
  #   - secretName: satmonitor-tls
  #     hosts:
  #       - satmonitor.example.com

  rules:
    - host: satmonitor.example.com
      http:
        paths:
          # Backend API routes
          - path: /api/v1/auth
            pathType: Prefix
            backend:
              service:
                name: satmonitor-backend-login
                port:
                  number: 8080
          
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: satmonitor-backend-login
                port:
                  number: 8080
          
          # Frontend UI
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: satmonitor-frontend
                port:
                  number: 80
          
          # Root path - redirect to /app
          - path: /
            pathType: Exact
            backend:
              service:
                name: satmonitor-frontend
                port:
                  number: 80
          
          # GeoServer mapping service
          - path: /geoserver
            pathType: Prefix
            backend:
              service:
                name: satmonitor-geoserver
                port:
                  number: 8080

---
# Optional: Traefik Middleware for security headers
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: satmonitor-app-security
  namespace: satmonitor-app
spec:
  headers:
    customResponseHeaders:
      X-Content-Type-Options: nosniff
      X-Frame-Options: DENY
      X-XSS-Protection: "1; mode=block"
      Referrer-Policy: strict-origin-when-cross-origin
```

---

## 6. PLATFORM KEYCLOAK INGRESS (platform/keycloak/07-keycloak-ingress.yaml)

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak-ingress
  namespace: satmonitor-platform
  labels:
    app.kubernetes.io/name: keycloak
    app.kubernetes.io/part-of: satmonitor-platform
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure

spec:
  ingressClassName: traefik
  
  rules:
    # Platform services on separate subdomain
    - host: keycloak.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: keycloak
                port:
                  number: 8080
```

---

## 7. NAMESPACE DEFINITION (apps/satmonitor/base/00-namespace.yaml)

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: satmonitor-app
  labels:
    app.kubernetes.io/name: satmonitor
    app.kubernetes.io/part-of: satmonitor
    environment: application
    pod-security.kubernetes.io/enforce: baseline
```

---

## 8. PLATFORM NAMESPACE (platform/00-namespaces.yaml)

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: satmonitor-platform
  labels:
    app.kubernetes.io/part-of: satmonitor-platform
    environment: platform
    pod-security.kubernetes.io/enforce: baseline
```

---

## 9. POSTGRES PVC (apps/satmonitor/base/postgres/02-pvc.yaml)

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: satmonitor-postgres-data
  namespace: satmonitor-app
  labels:
    app: satmonitor-postgres
    app.kubernetes.io/part-of: satmonitor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path  # k3s default; change to your StorageClass
  resources:
    requests:
      storage: 10Gi
```

---

## 10. POSTGRES SECRET (apps/satmonitor/base/postgres/01-secret.yaml)

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: satmonitor-postgres-secret
  namespace: satmonitor-app
  labels:
    app: satmonitor-postgres
    app.kubernetes.io/part-of: satmonitor
type: Opaque
stringData:
  POSTGRES_DB: satmonitor_db
  POSTGRES_USER: satmonitor_app
  POSTGRES_PASSWORD: "change-me-to-real-password"
  POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=en_US.UTF-8"
```

---

## 11. DEV OVERLAY KUSTOMIZATION (apps/satmonitor/overlays/dev/kustomization.yaml)

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: satmonitor-dev

commonLabels:
  environment: dev
  app.kubernetes.io/part-of: satmonitor

bases:
  - ../../base

# Patches for dev environment
patchesJson6902:
  # Reduce replicas for dev (save resources)
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-backend-login
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
  
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-frontend
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
  
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-geoserver
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1

# Override ConfigMaps for dev environment
configMapGenerator:
  - name: satmonitor-backend-config
    behavior: replace
    literals:
      - keycloak_server_url=https://keycloak.dev.example.com
      - keycloak_realm=satmonitor-dev
      - keycloak_client_id=satmonitor-backend-dev
      - db_url=jdbc:postgresql://satmonitor-postgres.satmonitor-dev.svc.cluster.local:5432/satmonitor_db_dev
      - spring_profiles_active=kubernetes,dev
      - logging_level_com_satmonitor=DEBUG

# Override Secrets for dev environment
secretGenerator:
  - name: satmonitor-backend-secrets
    behavior: replace
    literals:
      - keycloak_client_secret=dev-secret-change-me
      - db_username=satmonitor_dev
      - db_password=dev-password-change-me
  
  - name: satmonitor-postgres-secret
    behavior: replace
    literals:
      - POSTGRES_DB=satmonitor_db_dev
      - POSTGRES_USER=satmonitor_dev
      - POSTGRES_PASSWORD=dev-password-change-me
```

---

## 12. PROD OVERLAY KUSTOMIZATION (apps/satmonitor/overlays/prod/kustomization.yaml)

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: satmonitor-prod

commonLabels:
  environment: prod
  app.kubernetes.io/part-of: satmonitor

bases:
  - ../../base

# Patches for prod environment
patchesJson6902:
  # Increase replicas for prod (HA)
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-backend-login
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
  
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-frontend
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
  
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-geoserver
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
  
  # Increase resource limits for prod
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: satmonitor-backend-login
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/cpu
        value: 2000m
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 2Gi
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: 500m
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: 1Gi

# Override ConfigMaps for prod environment
configMapGenerator:
  - name: satmonitor-backend-config
    behavior: replace
    literals:
      - keycloak_server_url=https://keycloak.example.com
      - keycloak_realm=satmonitor
      - keycloak_client_id=satmonitor-backend
      - db_url=jdbc:postgresql://satmonitor-postgres-prod.satmonitor-prod.svc.cluster.local:5432/satmonitor_db_prod
      - spring_profiles_active=kubernetes,prod
      - logging_level_com_satmonitor=WARN

# NOTE: Secrets should NOT be in Git in production
# Use external secret management (Sealed Secrets, Vault, or CI/CD)
# secretGenerator:
#   - name: satmonitor-backend-secrets
#     behavior: replace
#     literals:
#       - keycloak_client_secret=$SECRET_FROM_VAULT
#       - db_username=$DB_USERNAME_FROM_VAULT
#       - db_password=$DB_PASSWORD_FROM_VAULT
```

---

## 13. BASE KUSTOMIZATION (apps/satmonitor/base/kustomization.yaml)

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Don't set namespace here - let overlays control namespace
commonLabels:
  app.kubernetes.io/part-of: satmonitor
  app.kubernetes.io/managed-by: kustomize

resources:
  - 00-namespace.yaml
  - 01-rbac.yaml
  - 02-networkpolicy.yaml
  - backend-login/
  - frontend/
  - geoserver/
  - postgres/
  - ingress/

commonAnnotations:
  description: "SatMonitor Application Stack"
  documentation: "https://github.com/your-org/satmonitor-gitops"

# Jenkins will update these image tags
images:
  - name: mariem360/satmonitor-backend-login
    newTag: "latest"  # Jenkins updates this to commit hash
  - name: mariem360/satmonitor-frontend
    newTag: "latest"
  - name: mariem360/satmonitor-geoserver
    newTag: "latest"
  - name: postgres
    newTag: "15"  # Pin Postgres version explicitly
```

---

## 🎯 COPY-PASTE INSTRUCTIONS

1. **Backend ConfigMap:**
   - Copy section #1
   - Create file: `apps/satmonitor/base/backend-login/01-configmap.yaml`
   - Update: Change example.com to your domain

2. **Backend Secrets:**
   - Copy section #2
   - Create file: `apps/satmonitor/base/backend-login/02-secrets.yaml`
   - **IMPORTANT**: Don't commit to Git! Use secret management system

3. **RBAC:**
   - Copy section #3
   - Create file: `apps/satmonitor/base/01-rbac.yaml`

4. **NetworkPolicy:**
   - Copy section #4
   - Create file: `apps/satmonitor/base/02-networkpolicy.yaml`

5. **Continue with sections 5-12 for remaining files**

---

## ⚠️ IMPORTANT SECURITY NOTES

1. **Never commit secrets with real passwords to Git**
2. **Use one of these for production secrets:**
   - Sealed Secrets (recommended for k3s)
   - HashiCorp Vault
   - AWS Secrets Manager
   - External Secrets Operator

3. **For CI/CD secrets:**
   - Store in GitHub Secrets or Jenkins Credentials
   - Inject at deployment time via environment variables

4. **For dev environment:**
   - It's OK to commit placeholder secrets with "change-me" values
   - Document the actual values in 1Password or similar

---

## 🧪 QUICK TEST AFTER APPLYING

```bash
# Apply configuration
kubectl apply -k apps/satmonitor/base

# Check all resources created
kubectl get all -n satmonitor-app

# Check if pods are running
kubectl get pods -n satmonitor-app -w

# View backend logs
kubectl logs -f deployment/satmonitor-backend-login -n satmonitor-app

# Test backend endpoint
kubectl port-forward -n satmonitor-app svc/satmonitor-backend-login 8080:8080
# In another terminal: curl http://localhost:8080/actuator/health

# Test ingress
curl https://satmonitor.example.com/app
```
