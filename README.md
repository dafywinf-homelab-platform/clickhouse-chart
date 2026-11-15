# 🚀 ClickHouse Altinity Deployment Guide

> [!WARNING]
> **Status: DISABLED (decommissioned 2026-11-15).** ClickHouse is no longer deployed on the Proxmox K3s cluster.
> `enabled: false` in `values.yaml` stops this chart rendering any ClickHouse resources. It was removed because its
> 20 GiB data volume filled the 30 GiB root disk of `k3s-worker-2` (DiskPressure, evicted pods). See
> [Disabling / Decommissioning](#-disabling--decommissioning) below, and read
> [Before re-enabling](#before-re-enabling) first if you bring it back.

## 🧱 Prerequisites

* Helm 3.x installed
* Kubernetes cluster (e.g. kind, Minikube, EKS)
* Access to the correct namespace (or create one)

---

## 🔧 Install and Deploy

### Installation Steps:
```bash
# Add the Altinity ClickHouse Operator Helm chart repository
helm repo add clickhouse-operator https://docs.altinity.com/clickhouse-operator/

# Update local chart repository index
helm repo update

# View available chart versions (latest version appears first)
helm search repo clickhouse-operator

# (Optional) Update chart dependencies (if using Chart.yaml with dependencies)
helm dependency update
```

### Deployment Steps:
```bash
# Deploy your ClickHouse chart for the first time
helm upgrade --install clickhouse . -n clickhouse --create-namespace
```


## 🛑 Disabling / Decommissioning

The chart has a top-level `enabled` flag (currently `false`). Every template is wrapped in `{{- if .Values.enabled }}`,
but the Altinity operator dependency is always installed. That ordering matters: the operator owns the finalizers on
the `ClickHouseInstallation` and `ClickHouseKeeperInstallation`, so it must still be running while they are deleted.

**This permanently deletes all ClickHouse data.**

```bash
# 1. Render with enabled=false: the operator tears down the cluster + keeper (pods, services, statefulsets)
helm upgrade clickhouse . -n clickhouse --set enabled=false
kubectl -n clickhouse get chi,chk        # wait until both are gone

# 2. The operator keeps PVCs by default - delete them to free the local-path disk on the worker nodes
kubectl -n clickhouse delete pvc --all

# 3. Remove the operator and the release, then the namespace
helm uninstall clickhouse -n clickhouse
kubectl delete namespace clickhouse

# 4. (Optional) Helm never deletes CRDs - remove the Altinity ones if nothing else uses them
kubectl get crd -o name | grep altinity.com | xargs kubectl delete
```

### Before re-enabling

* `local-path-mnt-data` does **not** put data on `/mnt/data`: the local-path provisioner ignores the StorageClass
  `path` parameter, so volumes land in `/var/lib/rancher/k3s/storage` on the 30 GiB root disk. Fix the storage class
  (in `kube-storage`) or use Longhorn before giving ClickHouse a 60 GiB volume again.
* Grafana in `kube-monitoring-chart` still has ClickHouse plugins and datasources configured.
* Then set `enabled: true` and run the deployment steps above.

### Uninstallation Challenges

The clickhouseinstallation custom resource may not be deleted automatically. This has a finalizer that prevents
deletion. Deletion can be forced by removing the finalizer by editing the customer resource in k9s.

The deployment also *may* have finalizers on the persistent volume claims (PVCs) that prevent deletion. Again these can
be removed by editing the PVCs in k9s.

## Clickhouse Client Access

Install the ClickHouse client to interact with your ClickHouse instance:

https://clickhouse.com/docs/install

*Note:* There is some complexity in this process, so refer to the official documentation for detailed instructions.

Find out the IP address MetalLB has assigned the ClickHouse service:

```bash
# clickhouse-tcp service
kubectl get svc -n clickhouse
```

```bash
clickhouse client --host 192.168.86.13 --port 9000
```

---

## Links

* [ClickHouse Altinity Kubernetes Operator Documentation](https://docs.altinity.com/altinitykubernetesoperator/kubernetesoperatorguide/)
