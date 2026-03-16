# my-webapp Helm Chart — v0.2.0

A Helm chart for deploying a basic web application on **OpenShift Container Platform 4.x**.

---

## What Changed in v0.2.0

| Area | v0.1.0 | v0.2.0 |
|------|--------|--------|
| Web server | `nginx:latest` (runs as root, blocked by OCP SCC) | `ubi9/httpd-24` (non-root, OCP-certified) |
| Internal port | 80 | 8080 (httpd-24 default) |
| Volume mount | `/usr/share/nginx/html` | `/var/www/html` |
| Security context | None | `restricted-v2` compliant (drop ALL caps) |
| Database | None | Optional PostgreSQL or MySQL StatefulSet |
| DB PVCs | None | Separate PVC per engine |
| ConfigMap | None | App env var ConfigMap |
| Secret | None | DB credentials Secret |
| DB Service | None | ClusterIP Service for in-cluster DB |

---

## Prerequisites

- OpenShift 4.x cluster
- Helm 3.x
- A default StorageClass (or set `persistence.storageClass`)

---

## Quick Start

```bash
# Basic deploy (no DB)
helm upgrade --install my-webapp . -n my-web-app --create-namespace

# Deploy with PostgreSQL
helm upgrade --install my-webapp . -n my-web-app --create-namespace \
  --set database.enabled=true \
  --set database.engine=postgresql \
  --set database.credentials.password=supersecret

# Deploy with MySQL
helm upgrade --install my-webapp . -n my-web-app --create-namespace \
  --set database.enabled=true \
  --set database.engine=mysql \
  --set database.credentials.password=supersecret
```

---

## Why nginx was replaced

The official `nginx:latest` image:
- Runs the master process as **root** (UID 0)
- Binds to port **80** (privileged port, < 1024)

OpenShift's default **restricted-v2** SCC blocks both. The `ubi9/httpd-24` image:
- Runs as **UID 1001** (non-root)
- Listens on port **8080** / **8443**
- Is Red Hat certified and receives CVE patches

---

## PV / PVC Design

| PVC Name | Purpose | Default Size | Access Mode |
|----------|---------|-------------|-------------|
| `<release>-app` | Web content / uploads | 1Gi | ReadWriteOnce |
| `<release>-db-postgresql` | PostgreSQL data | 5Gi | ReadWriteOnce |
| `<release>-db-mysql` | MySQL data | 5Gi | ReadWriteOnce |

**Multi-replica note:** If `replicaCount > 1`, change `persistence.accessMode` to `ReadWriteMany` and use an RWX-capable StorageClass (ODF/Ceph, NFS, Azure File).

---

## Security Context

All containers run with:
```yaml
runAsNonRoot: true
allowPrivilegeEscalation: false
seccompProfile:
  type: RuntimeDefault
capabilities:
  drop: [ALL]
```
This satisfies OCP `restricted-v2` SCC without granting any elevated privileges.

---

## Database Credentials

**Never commit passwords to Git.** Supply them at deploy time:

```bash
--set database.credentials.password=<value>
```

For production, use **Sealed Secrets** or **HashiCorp Vault with the Agent Injector**.
