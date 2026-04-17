# ⚡ QUICK REFERENCE: Production GitOps Checklist

## 🔴 CRITICAL ISSUES (Fix FIRST)

- [ ] **Keycloak Namespace Mismatch**: Deployed in platform/ but namespace: satmonitor-dev
  - **Action**: Move to namespace: satmonitor-platform (0.5 day)
  - **File**: platform/keycloak/04-keycloak-deployment.yaml

- [ ] **Backend Discovery URL Broken**: Pointing to non-existent satmonitor-discovery:8761
  - **Action**: Use ConfigMap + reference Keycloak in platform namespace (0.5 day)
  - **File**: apps/satmonitor/base/backend-login/*config.yaml

- [ ] **Secrets Hardcoded in YAML**: DB password visible in postgres/deployment.yaml
  - **Action**: Move to Secrets files (0.5 day)
  - **File**: apps/satmonitor/base/postgres/01-secret.yaml

- [ ] **No Resource Limits**: Pods can consume all node memory
  - **Action**: Add requests/limits to all Deployments (1 day)
  - **File**: All deployment.yaml files

- [ ] **No Health Probes**: Dead pods not replaced
  - **Action**: Add liveness/readiness/startup probes (1 day)
  - **File**: All deployment.yaml files

---

## 🟡 HIGH PRIORITY (Week 1-2)

- [ ] **No HA**: replicas: 1 everywhere = single point of failure
  - **Fix**: Backend/Frontend/Keycloak → replicas: 2, use overlays for dev: 1
  - **Effort**: 0.5 day per service

- [ ] **No Persistence**: Postgres/Keycloak data ephemeral
  - **Fix**: Add PVC manifests, mount volumeMounts
  - **Effort**: 0.5 day per service

- [ ] **Cross-Namespace Ingress Routing**: Keycloak mixed with app routes
  - **Fix**: Separate Ingress for platform (keycloak) and app services
  - **Effort**: 0.5 day

- [ ] **Empty Overlays**: dev/prod not differentiated
  - **Fix**: Populate with patches (hostname, replicas, resources)
  - **Effort**: 1 day

- [ ] **No RBAC**: No ServiceAccounts or Roles defined
  - **Fix**: Create ServiceAccount + Role + RoleBinding per component
  - **Effort**: 0.5 day

---

## 🟢 MEDIUM PRIORITY (Week 3-4)

- [ ] **No NetworkPolicies**: No traffic isolation between namespaces
  - **Fix**: Add NetworkPolicy to each namespace
  - **Effort**: 1 day

- [ ] **No ConfigMaps**: Configuration hardcoded in Deployments
  - **Fix**: Extract to ConfigMap for each service
  - **Effort**: 1 day

- [ ] **Inconsistent Labels**: app: vs app.kubernetes.io/name:
  - **Fix**: Standardize all to app.kubernetes.io/* labels
  - **Effort**: 0.5 day

- [ ] **Image Pull Policy**: Always = unnecessary pulls
  - **Fix**: Change to IfNotPresent
  - **Effort**: 0.25 day

- [ ] **No Pod Anti-Affinity**: Pods can collide on same node
  - **Fix**: Add podAntiAffinity rules
  - **Effort**: 0.5 day

---

## 🔵 NICE TO HAVE (Week 5+)

- [ ] **TLS/HTTPS**: No encryption for traffic
  - **Fix**: Install cert-manager, configure Let's Encrypt (requires DNS setup)
  - **Effort**: 1-2 days

- [ ] **Pod Disruption Budgets**: Maintenance can break services
  - **Fix**: Add PDB for critical services
  - **Effort**: 0.5 day

- [ ] **Horizontal Pod Autoscaling**: No auto-scaling
  - **Fix**: Create HPA with CPU/memory targets (prod overlay)
  - **Effort**: 0.5 day

- [ ] **Secret Management**: Secrets hardcoded in Git
  - **Fix**: Use Sealed Secrets or Vault (recommended: Sealed Secrets)
  - **Effort**: 1 day + ops training

- [ ] **Monitoring Annotations**: Not visible to Prometheus
  - **Fix**: Add prometheus.io/* annotations to Services
  - **Effort**: 0.5 day

---

## 📋 FILE CREATION/UPDATE SUMMARY

### NEW FILES TO CREATE

**Platform Namespace & RBAC**
```
platform/
  ├── 00-namespaces.yaml                    # Create satmonitor-platform namespace
```

**Keycloak Improvements**
```
platform/keycloak/
  ├── 00-namespace.yaml                     # Update: namespace: satmonitor-platform
  ├── 05-keycloak-release.yaml              # NEW: Update service namespace
  ├── 06-keycloak-rbac.yaml                 # NEW: ServiceAccount + RBAC + PDB
  ├── 07-keycloak-ingress.yaml              # NEW: Platform ingress (separate)
```

**App Namespace & RBAC**
```
apps/satmonitor/base/
  ├── 00-namespace.yaml                     # NEW: Explicit namespace definition
  ├── 01-rbac.yaml                          # NEW: ServiceAccounts + Roles
  ├── 02-networkpolicy.yaml                 # NEW: Traffic policies
```

**Backend Improvements**
```
apps/satmonitor/base/backend-login/
  ├── 01-configmap.yaml                     # NEW: Environment config
  ├── 02-secrets.yaml                       # NEW: Keycloak credentials
  ├── 03-deployment.yaml                    # UPDATE: Probes + resources + affinity
  ├── 04-service.yaml                       # UPDATE: Consistent labels
```

**Frontend Improvements**
```
apps/satmonitor/base/frontend/
  ├── 01-configmap.yaml                     # NEW: API endpoint config
  ├── 02-deployment.yaml                    # UPDATE: Probes + resources
  ├── 03-service.yaml                       # UPDATE: Consistent labels
```

**GeoServer Improvements**
```
apps/satmonitor/base/geoserver/
  ├── 01-configmap.yaml                     # NEW: Configuration
  ├── 02-pvc.yaml                           # NEW: Persistent storage
  ├── 03-deployment.yaml                    # UPDATE: Probes + resources + mount
  ├── 04-service.yaml                       # UPDATE: Consistent labels
```

**Postgres Improvements**
```
apps/satmonitor/base/postgres/
  ├── 01-secret.yaml                        # NEW: DB credentials
  ├── 02-pvc.yaml                           # NEW: Persistent storage
  ├── 03-deployment.yaml                    # UPDATE: Probes + resources + mount
  ├── 04-service.yaml                       # No changes needed
```

**Ingress Consolidation**
```
clusters/srvapplis/
  └── ingresscontroller/                    # NEW FOLDER
      ├── kustomization.yaml
      ├── 00-namespace.yaml
      └── 01-application-ingress.yaml       # Single ingress for all app routes
```

**Overlay Implementation**
```
apps/satmonitor/overlays/
  ├── dev/
  │   ├── kustomization.yaml                # UPDATE: Add patches
  │   ├── namespace-patch.yaml
  │   ├── replicas-patch.yaml               # replicas: 1
  │   ├── config-patch.yaml                 # Dev endpoints
  │   └── resources-patch.yaml              # Lower limits
  │
  └── prod/
      ├── kustomization.yaml
      ├── namespace-patch.yaml
      ├── replicas-patch.yaml               # replicas: 3+
      ├── config-patch.yaml                 # Prod endpoints
      ├── resources-patch.yaml              # Higher limits
      └── hpa.yaml                          # NEW: Auto-scaling
```

---

## 📊 COMPLEXITY BY COMPONENT

| Component | Priority | Complexity | Time | Files |
|-----------|----------|-----------|------|-------|
| Keycloak | Critical | Medium | 4 hrs | 4 new |
| Backend | Critical | Medium | 4 hrs | 3 new |
| NetworkPolicy | High | Low | 2 hrs | 1 new |
| Overlays | High | Medium | 4 hrs | 6 new |
| Frontend | Medium | Medium | 3 hrs | 2 new |
| GeoServer | Medium | Medium | 3 hrs | 3 new |
| Postgres | Medium | Medium | 3 hrs | 2 new |
| TLS | Low | High | 8 hrs | 2 new |
| **Total** | | | **34 hrs** | **26 new files** |

---

## 🚀 DAY-BY-DAY EXECUTION PLAN

### Day 1: Critical Fixes (4-6 hrs)
- [ ] 1. Fix Keycloak namespace to satmonitor-platform
- [ ] 2. Add ConfigMap + Secrets for backend
- [ ] 3. Fix discovery URL reference
- [ ] 4. Test: `kubectl apply -k apps/satmonitor/base`

### Day 2: Resource & Health (6-8 hrs)
- [ ] 1. Add resource requests/limits to all Deployments
- [ ] 2. Add liveness/readiness probes to all
- [ ] 3. Test: Pods should stay running without restarts
- [ ] 4. Monitor: `kubectl top pods`

### Day 3: HA & Storage (6-8 hrs)
- [ ] 1. Update replicas: 2 for Backend, Frontend, Keycloak
- [ ] 2. Create PVCs for Postgres, GeoServer
- [ ] 3. Add pod anti-affinity
- [ ] 4. Test: `kubectl describe pod <pod-name>`

### Day 4: RBAC & Ingress (6-8 hrs)
- [ ] 1. Create ServiceAccounts + Roles for each service
- [ ] 2. Create separate platform Ingress (keycloak)
- [ ] 3. Move app Ingress to clusters/ folder
- [ ] 4. Test: Access satmonitor.example.com and keycloak.example.com

### Day 5: Overlays & Testing (6-8 hrs)
- [ ] 1. Populate overlays/dev and overlays/prod
- [ ] 2. Test: `kubectl apply -k apps/satmonitor/overlays/dev`
- [ ] 3. Verify: Different replicas, resources, endpoints per environment
- [ ] 4. Update Argo CD to reference overlays

### Day 6: Validation & Documentation (4-6 hrs)
- [ ] 1. Run deployment on staging cluster
- [ ] 2. Verify: All pods running, healthy, not restarting
- [ ] 3. Performance test: Load testing frontend
- [ ] 4. Document: Ops runbook, troubleshooting guide

---

## 🧪 TESTING COMMANDS

```bash
# Check all pods are running and healthy
kubectl get pods -A --no-headers | grep -v Running

# Check resource usage
kubectl top pods -A --containers

# Check pod restart count
kubectl get pods -A -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount

# Verify Ingress routes
kubectl describe ingress -A

# Check service endpoints
kubectl get endpoints -A

# Test cross-namespace service discovery
kubectl exec -it <pod> -n satmonitor-app -- curl http://keycloak.satmonitor-platform.svc:8080

# View pod logs
kubectl logs -f <pod-name> -n <namespace>

# Port forward for testing
kubectl port-forward -n satmonitor-app svc/satmonitor-backend-login 8080:8080
```

---

## 📚 DOCUMENTATION TO UPDATE

- [ ] README.md - Update architecture section
- [ ] Add DEPLOYMENT.md - Step-by-step deployment guide
- [ ] Add TROUBLESHOOTING.md - Common issues and solutions
- [ ] Add MONITORING.md - Prometheus metrics and dashboards
- [ ] Update .github/workflows/ - CI/CD pipeline updates

---

## ✅ FINAL VALIDATION CHECKLIST

Before deploying to production:

- [ ] All pods have livenessProbes
- [ ] All pods have readinessProbes  
- [ ] All pods have resource requests
- [ ] All pods have resource limits
- [ ] All services use consistent labels
- [ ] All ConfigMaps are referenced in Deployments
- [ ] All Secrets are referenced in Deployments
- [ ] No hardcoded passwords in YAML files (use Secrets)
- [ ] No hardcoded hostnames (use ConfigMaps with overlays)
- [ ] Replicas: 2+ for all stateless services
- [ ] Replicas: 1 for all stateful services  
- [ ] Pod anti-affinity configured
- [ ] NetworkPolicies defined
- [ ] Ingress routes all working
- [ ] TLS certificates configured (optional but recommended)
- [ ] RBAC roles are minimal (least privilege)
- [ ] PVCs defined for persistent data
- [ ] Dev and Prod overlays working
