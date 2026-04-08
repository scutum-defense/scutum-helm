```
 ____            _                     _   _      _
/ ___|  ___ _   _| |_ _   _ _ __ ___  | | | | ___| |_ __ ___
\___ \ / __| | | | __| | | | '_ ` _ \ | |_| |/ _ \ | '_ ` _ \
 ___) | (__| |_| | |_| |_| | | | | | ||  _  |  __/ | | | | | |
|____/ \___|\__,_|\__|\__,_|_| |_| |_||_| |_|\___|_|_| |_| |_|

Sovereign Kubernetes Deployment Charts
```

<p align="center">
  <a href="https://helm.sh"><img src="https://img.shields.io/badge/Helm-3.x-0F1689?logo=helm&logoColor=white" alt="Helm 3"></a>
  <a href="https://kubernetes.io"><img src="https://img.shields.io/badge/Kubernetes-1.28+-326CE5?logo=kubernetes&logoColor=white" alt="Kubernetes 1.28+"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="Apache 2.0"></a>
  <img src="https://img.shields.io/badge/Charts-2-green" alt="Charts">
  <img src="https://img.shields.io/badge/Version-0.1.0-orange" alt="Version 0.1.0">
</p>

<p align="center">
  <strong>Helm charts for sovereign single-tenant deployment of the Scutum Command Platform.</strong>
</p>

---

## Overview

**scutum-helm** provides production-grade Kubernetes Helm charts for deploying the Scutum Command Platform in sovereign, single-tenant environments. Each deployment is fully isolated, ensuring complete data residency and operational sovereignty for defense and critical infrastructure operators.

These charts are designed for air-gapped, high-security environments where data must never leave the operator's sovereign boundary. Every component runs within a single Kubernetes namespace with strict network policies, pod security contexts, and encrypted storage.

---

## Architecture

A standard Scutum Platform deployment provisions the following components within a single Kubernetes cluster:

```
+------------------------------------------------------------------+
|                    Kubernetes Cluster (Sovereign)                  |
|                                                                    |
|  +-------------------+    +-------------------+                    |
|  |   scutum-api (2)  |    |   scutum-web (2)  |                    |
|  |   Port: 4000      |    |   Port: 3000      |                    |
|  +--------+----------+    +--------+----------+                    |
|           |                         |                              |
|           +------------+------------+                              |
|                        |                                           |
|  +-------------------+ | +-------------------+                     |
|  | scutum-ingestion  | | | scutum-audit-log  |                     |
|  | (Sensor Pipeline) | | | (Immutable Audit) |                     |
|  +--------+----------+ | +--------+----------+                     |
|           |             |          |                                |
|           +------+------+------+---+                               |
|                  |             |                                    |
|        +---------+---+  +-----+--------+                           |
|        | PostgreSQL  |  |    Redis     |                           |
|        | (Primary)   |  | (Standalone) |                           |
|        | 50Gi PVC    |  |              |                           |
|        +-------------+  +--------------+                           |
|                                                                    |
|  [Network Policies: Default Deny + Allow Internal Only]            |
+------------------------------------------------------------------+
```

### Components

| Component | Description | Default Replicas |
|-----------|-------------|-----------------|
| **scutum-api** | Core REST API serving the Command Platform backend | 2 |
| **scutum-web** | Next.js frontend application | 2 |
| **scutum-ingestion** | Sensor data ingestion pipeline for real-time feeds | 1 |
| **scutum-audit-log** | Immutable audit log service for compliance | 1 |
| **PostgreSQL** | Primary relational database (Bitnami sub-chart) | 1 |
| **Redis** | Cache and session store (Bitnami sub-chart) | 1 |

---

## Prerequisites

Before deploying the Scutum Platform, ensure the following requirements are met:

| Requirement | Minimum Version | Notes |
|-------------|----------------|-------|
| **Kubernetes** | 1.28+ | EKS, AKS, GKE, or bare-metal |
| **Helm** | 3.12+ | Helm 2 is not supported |
| **kubectl** | 1.28+ | Must match cluster version |
| **Storage Class** | - | Default StorageClass or explicit configuration |
| **Container Registry** | - | Access to `ghcr.io/scutum-defense` or private mirror |
| **Secrets** | - | Pre-created Kubernetes secrets for database credentials |

### Required Secrets

Before installing, create the following Kubernetes secrets in the target namespace:

```bash
# Database credentials
kubectl create secret generic scutum-db-credentials \
  --from-literal=postgres-password='<POSTGRES_PASSWORD>' \
  --from-literal=password='<SCUTUM_USER_PASSWORD>' \
  -n scutum

# Redis credentials
kubectl create secret generic scutum-redis-credentials \
  --from-literal=redis-password='<REDIS_PASSWORD>' \
  -n scutum
```

---

## Quick Start

### 1. Add the Helm Repository

```bash
helm repo add scutum https://charts.scutum.defense
helm repo update
```

### 2. Create Namespace

```bash
kubectl create namespace scutum
```

### 3. Create Required Secrets

```bash
kubectl create secret generic scutum-db-credentials \
  --from-literal=postgres-password='$(openssl rand -base64 32)' \
  --from-literal=password='$(openssl rand -base64 32)' \
  -n scutum

kubectl create secret generic scutum-redis-credentials \
  --from-literal=redis-password='$(openssl rand -base64 32)' \
  -n scutum
```

### 4. Install the Platform

```bash
helm install scutum scutum/scutum-platform \
  --namespace scutum \
  --values my-values.yaml \
  --wait --timeout 10m
```

### 5. Verify the Deployment

```bash
kubectl get pods -n scutum
kubectl get services -n scutum
helm test scutum -n scutum
```

---

## Configuration

### Key Values

The following table lists the most commonly configured values. For the complete list, see [`charts/scutum-platform/values.yaml`](./charts/scutum-platform/values.yaml).

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.sovereignty.region` | Sovereign deployment region identifier | `"abudhabi"` |
| `global.sovereignty.tenant` | Tenant identifier for single-tenant isolation | `"default"` |
| `global.sovereignty.dataResidency` | Data residency zone label | `"sovereign-primary"` |
| `global.sovereignty.classification` | Data classification level | `"restricted"` |
| `global.image.registry` | Container image registry | `ghcr.io/scutum-defense` |
| `global.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `api.enabled` | Enable the API deployment | `true` |
| `api.replicas` | Number of API pod replicas | `2` |
| `api.image.tag` | API container image tag | `"0.1.0"` |
| `api.port` | API service port | `4000` |
| `api.resources.requests.cpu` | API CPU request | `250m` |
| `api.resources.requests.memory` | API memory request | `512Mi` |
| `api.resources.limits.cpu` | API CPU limit | `1000m` |
| `api.resources.limits.memory` | API memory limit | `1Gi` |
| `api.env.SCUTUM_ENV` | Application environment | `production` |
| `api.env.SCUTUM_LOG_LEVEL` | Logging verbosity | `info` |
| `web.enabled` | Enable the Web deployment | `true` |
| `web.replicas` | Number of Web pod replicas | `2` |
| `web.image.tag` | Web container image tag | `"0.1.0"` |
| `web.port` | Web service port | `3000` |
| `web.resources.requests.cpu` | Web CPU request | `100m` |
| `web.resources.requests.memory` | Web memory request | `256Mi` |
| `web.resources.limits.cpu` | Web CPU limit | `500m` |
| `web.resources.limits.memory` | Web memory limit | `512Mi` |
| `ingestion.enabled` | Enable the ingestion pipeline | `true` |
| `ingestion.replicas` | Number of ingestion pod replicas | `1` |
| `auditLog.enabled` | Enable the audit log service | `true` |
| `auditLog.replicas` | Number of audit log pod replicas | `1` |
| `postgresql.enabled` | Deploy PostgreSQL sub-chart | `true` |
| `postgresql.auth.database` | PostgreSQL database name | `scutum` |
| `postgresql.auth.username` | PostgreSQL username | `scutum` |
| `postgresql.primary.persistence.size` | PostgreSQL PVC size | `50Gi` |
| `redis.enabled` | Deploy Redis sub-chart | `true` |
| `redis.architecture` | Redis architecture | `standalone` |
| `ingress.enabled` | Enable Kubernetes Ingress | `false` |
| `ingress.className` | Ingress class name | `""` |
| `networkPolicies.enabled` | Enable default-deny network policies | `true` |
| `podSecurityContext.runAsNonRoot` | Enforce non-root containers | `true` |
| `securityContext.allowPrivilegeEscalation` | Prevent privilege escalation | `false` |
| `securityContext.readOnlyRootFilesystem` | Mount root filesystem as read-only | `true` |

### Custom Values File

Create a `my-values.yaml` for your deployment:

```yaml
global:
  sovereignty:
    region: "abudhabi"
    tenant: "adnoc-pilot"
    dataResidency: "sovereign-primary"
    classification: "secret"

api:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi

web:
  replicas: 3

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: scutum.internal.example.com
      paths:
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: scutum-api
              port: 4000
        - path: /
          pathType: Prefix
          backend:
            service:
              name: scutum-web
              port: 3000
  tls:
    - secretName: scutum-tls
      hosts:
        - scutum.internal.example.com
```

---

## Chart Structure

```
scutum-helm/
  charts/
    scutum-platform/           # Main platform chart
      Chart.yaml               # Chart metadata and dependencies
      values.yaml              # Default configuration values
      templates/
        deployment-api.yaml    # API server deployment
        deployment-web.yaml    # Web frontend deployment
        service-api.yaml       # API ClusterIP service
        service-web.yaml       # Web ClusterIP service
        networkpolicy.yaml     # Default-deny network policy
      crds/                    # Custom Resource Definitions (future)
    scutum-operator/           # Kubernetes operator chart (future)
      templates/
  docs/                        # Additional documentation
  tests/                       # Chart test suites
  .github/
    workflows/                 # CI/CD pipelines
  CODEOWNERS                   # Repository ownership
  CHANGELOG.md                 # Version history
  LICENSE                      # Apache 2.0
  README.md                    # This file
```

---

## Sovereign Deployment Model

The Scutum Platform follows a **strict single-tenant deployment model**. This means:

### Isolation Guarantees

1. **Namespace Isolation** -- Each operator gets a dedicated Kubernetes namespace with RBAC boundaries. No cross-namespace communication is permitted.

2. **Network Isolation** -- Default-deny network policies block all ingress and egress traffic. Only explicitly allowed pod-to-pod communication within the same release is permitted. DNS resolution (port 53) is the only external egress allowed.

3. **Data Residency** -- All persistent data (PostgreSQL, Redis) remains within the sovereign cluster. The `global.sovereignty.dataResidency` label propagates to all resources for policy enforcement.

4. **Classification Labels** -- Every resource is tagged with `global.sovereignty.classification` to enable automated policy engines (OPA/Gatekeeper) to enforce data handling rules.

5. **No Shared State** -- There is no multi-tenant database, no shared cache, and no cross-operator data flow. Each deployment is a complete, self-contained instance.

### Sovereignty Labels

All deployed resources carry the following labels for policy enforcement:

```yaml
scutum.defense/sovereignty-region: "abudhabi"
scutum.defense/sovereignty-tenant: "default"
scutum.defense/data-residency: "sovereign-primary"
scutum.defense/classification: "restricted"
```

These labels integrate with Kubernetes admission controllers (OPA Gatekeeper, Kyverno) to enforce data residency and classification policies at the cluster level.

---

## Environment Matrix

The following environments are supported with recommended configurations:

| Environment | API Replicas | Web Replicas | PostgreSQL Size | Network Policies | Purpose |
|-------------|-------------|-------------|----------------|-----------------|---------|
| **dev** | 1 | 1 | 10Gi | Disabled | Local development and testing |
| **staging** | 2 | 2 | 25Gi | Enabled | Integration testing and QA |
| **pilot** | 2 | 2 | 50Gi | Enabled | Customer pilot deployments |
| **production** | 3+ | 3+ | 100Gi+ | Enabled | Full production sovereign deployment |

### Environment-Specific Values

Store environment-specific overrides in separate files:

```bash
# Development
helm install scutum ./charts/scutum-platform -f values/dev.yaml

# Staging
helm install scutum ./charts/scutum-platform -f values/staging.yaml

# Production
helm install scutum ./charts/scutum-platform -f values/production.yaml
```

Example `values/dev.yaml`:

```yaml
global:
  sovereignty:
    region: "local"
    tenant: "dev"
    classification: "unclassified"

api:
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

web:
  replicas: 1

postgresql:
  primary:
    persistence:
      size: 10Gi

networkPolicies:
  enabled: false
```

---

<details>
<summary><strong>Security Configuration</strong></summary>

### Pod Security

All pods are configured with security-hardened contexts by default:

- **Non-root execution**: `runAsNonRoot: true` prevents containers from running as root
- **Read-only filesystem**: `readOnlyRootFilesystem: true` prevents runtime filesystem modifications
- **No privilege escalation**: `allowPrivilegeEscalation: false` blocks setuid/setgid binaries
- **Dropped capabilities**: All Linux capabilities are dropped via `capabilities.drop: [ALL]`

### Network Policies

When `networkPolicies.enabled: true`, the chart deploys a default-deny network policy that:

- Blocks all inbound traffic (ingress) by default
- Allows outbound traffic only to pods within the same Helm release
- Allows DNS resolution (port 53 UDP/TCP) for service discovery
- Requires explicit ingress rules for external access (via Ingress controller)

### Secrets Management

The chart does not create secrets directly. Instead, it references pre-existing Kubernetes secrets:

- `scutum-db-credentials` -- PostgreSQL passwords
- `scutum-redis-credentials` -- Redis authentication

This approach supports external secret management tools:
- **HashiCorp Vault** with the Vault Secrets Operator
- **AWS Secrets Manager** with External Secrets Operator
- **Azure Key Vault** with the CSI driver
- **SOPS** with FluxCD

### Image Security

- All images are pulled from `ghcr.io/scutum-defense` (configurable)
- For air-gapped deployments, mirror images to a private registry and update `global.image.registry`
- Image signatures are verified via Sigstore/cosign (when configured)
- No `latest` tags are used; all images are pinned to specific versions

### RBAC

The chart does not create ClusterRoles or ClusterRoleBindings. All RBAC should be configured at the cluster level by the platform administrator. The recommended approach is:

1. Create a dedicated ServiceAccount per component
2. Grant minimal namespace-scoped permissions
3. Use NetworkPolicies in combination with RBAC for defense-in-depth

</details>

---

<details>
<summary><strong>Contributing</strong></summary>

### Development Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/scutum-defense/scutum-helm.git
   cd scutum-helm
   ```

2. Install development dependencies:
   ```bash
   helm plugin install https://github.com/quintush/helm-unittest
   ```

3. Lint the charts:
   ```bash
   helm lint charts/scutum-platform
   ```

4. Run unit tests:
   ```bash
   helm unittest charts/scutum-platform
   ```

5. Template locally to inspect output:
   ```bash
   helm template scutum charts/scutum-platform --debug
   ```

### Pull Request Requirements

- All changes must pass `helm lint`
- Unit tests must cover new template logic
- Values changes must be documented in this README
- Breaking changes require a major version bump
- CHANGELOG.md must be updated

### Code Owners

See [CODEOWNERS](./CODEOWNERS) for ownership rules. All changes to chart templates require approval from `@ScutumDefense/sre-release`.

</details>

---

<details>
<summary><strong>Upgrade Guide</strong></summary>

### Upgrading the Chart

```bash
# Check current release
helm list -n scutum

# Upgrade to a new version
helm upgrade scutum scutum/scutum-platform \
  --namespace scutum \
  --values my-values.yaml \
  --wait --timeout 10m

# Rollback if needed
helm rollback scutum -n scutum
```

### Version Compatibility

| Chart Version | App Version | Kubernetes | Helm | Notes |
|--------------|-------------|-----------|------|-------|
| 0.1.0 | 0.1.0 | 1.28+ | 3.12+ | Initial release |

### Breaking Changes

No breaking changes in the current release.

### Migration Notes

When upgrading between minor versions:

1. Review the [CHANGELOG](./CHANGELOG.md) for new values
2. Compare your `values.yaml` against the new defaults
3. Test the upgrade in a staging environment first
4. Back up the PostgreSQL database before upgrading
5. Use `helm diff` to preview changes:
   ```bash
   helm diff upgrade scutum scutum/scutum-platform \
     --namespace scutum \
     --values my-values.yaml
   ```

</details>

---

<details>
<summary><strong>Troubleshooting</strong></summary>

### Common Issues

**Pods stuck in `Pending` state:**
- Check if PersistentVolumeClaims are bound: `kubectl get pvc -n scutum`
- Verify the StorageClass exists and can provision volumes
- Check node resources: `kubectl describe nodes`

**Database connection failures:**
- Verify the `scutum-db-credentials` secret exists: `kubectl get secret scutum-db-credentials -n scutum`
- Check PostgreSQL pod logs: `kubectl logs -l app.kubernetes.io/name=postgresql -n scutum`
- Ensure network policies allow API-to-PostgreSQL communication

**Image pull errors:**
- Verify registry access: `kubectl get events -n scutum | grep Failed`
- Check image pull secrets if using a private registry
- For air-gapped deployments, ensure images are mirrored locally

**Health check failures:**
- Review probe configuration in the deployment template
- Check application logs: `kubectl logs -l app.kubernetes.io/name=scutum-api -n scutum`
- Increase `initialDelaySeconds` if the application needs more startup time

### Debug Commands

```bash
# Get all resources in the scutum namespace
kubectl get all -n scutum

# Describe a failing pod
kubectl describe pod <pod-name> -n scutum

# View pod logs
kubectl logs <pod-name> -n scutum --tail=100

# Check Helm release status
helm status scutum -n scutum

# Get rendered templates
helm get manifest scutum -n scutum
```

</details>

---

## License

Copyright 2026 Scutum Defense. Licensed under the [Apache License 2.0](./LICENSE).
