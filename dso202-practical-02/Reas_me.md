# DSO202 Practical 02 — Persistent Storage for a Stateful Application

This repository builds a three-node Kubernetes cluster with
[kind](https://kind.sigs.k8s.io/) and uses it to demonstrate how Kubernetes
stores data that must outlive a Pod:

- **static provisioning** — a hand-written PersistentVolume backed by a host
  directory, and what the `Retain` reclaim policy does to it;
- **dynamic provisioning** — claims that the `local-path` provisioner fulfils
  automatically, `WaitForFirstConsumer` binding, and the `Delete` policy;
- **why a Deployment cannot own state** — three replicas sharing one volume
  (a deliberate anti-pattern, file `08`);
- **StatefulSets** — stable names, one claim per replica, per-Pod DNS through a
  headless Service, ordered scaling, retention policies and partitioned updates;
- **PostgreSQL 18** running as a StatefulSet on retained storage, surviving the
  deletion of its Pod.

Every object is declared in a version-controlled file. Command output from a
full run is in `evidence/`, and the assessed write-up is in `report/`.

## Repository layout

```
dso202-practical-02/
├── README.md                          # this file
├── cluster/
│   └── kind-cluster.yaml              # Listing 1  - 3 nodes, host dir mounted into worker-node-1
├── manifests/
│   ├── 00-namespace.yaml              # Listing 2  - Namespace dso202-practical-02
│   ├── 01-quota-and-limits.yaml       # Listing 3  - ResourceQuota (incl. storage caps) + LimitRange
│   ├── 02-storageclass-retain.yaml    # Listing 4  - StorageClass dso202-retain (cluster-scoped)
│   ├── 03-pv-static.yaml              # Listing 5  - static PersistentVolume (cluster-scoped)
│   ├── 04-pvc-static.yaml             # Listing 6  - claim for the static volume
│   ├── 05-pod-static-writer.yaml      # Listing 7  - Pod appending to ledger.txt
│   ├── 06-pvc-dynamic.yaml            # Listing 8  - dynamically provisioned claim
│   ├── 07-pod-dynamic-writer.yaml     # Listing 9  - first consumer of that claim
│   ├── 08-deployment-shared-pvc.yaml  # Listing 10 - ANTI-PATTERN: 3 replicas, 1 volume
│   ├── 09-service-webnote.yaml        # Listing 11 - headless Service
│   ├── 10-statefulset-webnote.yaml    # Listing 12 - webnote StatefulSet (nginx:1.31-alpine after Stage 6)
│   ├── 11-pod-client.yaml             # Listing 13 - busybox client for DNS/HTTP tests
│   ├── 12-secret-postgres.yaml        # Listing 14 - PostgreSQL credentials (practice values only)
│   ├── 13-service-postgres.yaml       # Listing 15 - headless + ClusterIP Services
│   └── 14-statefulset-postgres.yaml   # Listing 16 - PostgreSQL StatefulSet
├── extensions/                        # Extension 1-3 manifests (NOT applied by 'apply -f manifests/')
├── evidence/                          # command output per stage, final state, SQL dump (run 1, 2 Nov)
│   └── logs/                          # plain-text output behind each screenshot, final state, SQL dump (run 2)
├── screenshots/                       # terminal screenshots of every step, named <stage>-<step>.png (run 2, 10 Nov)
└── report/
    └── practical-02-report.md         # the assessed report
```

Files are numbered so that `kubectl apply -f manifests/` applies them in
dependency order. Every namespaced object states `namespace:
dso202-practical-02`; the StorageClass and the PersistentVolume carry no
namespace because they are cluster-scoped.

## Software and image versions

| Software / image   | Version used                                   |
| ------------------ | ---------------------------------------------- |
| Operating system   | macOS 26.7.1 (Apple Silicon, arm64)            |
| Docker Desktop     | Engine 29.9.2                                  |
| kind               | v0.32.0                                        |
| kubectl            | v1.37.2 (within one minor version of the cluster) |
| Kubernetes (nodes) | v1.36.1 (`kindest/node:v1.36.1`, pinned by digest) |
| Storage provisioner| `rancher.io/local-path` (installed by kind)    |
| Images             | `busybox:1.37`, `nginx:1.30-alpine`, `nginx:1.31-alpine`, `postgres:18-alpine` (PostgreSQL 18.6) |

Docker needs at least 4 GB memory, 2 CPUs and about 3 GB of free disk **inside
the Docker VM** (check with `docker system df`; see the report's Reflection).

## Rebuild from an empty machine

```bash
# 0. Tools, and the host directory the static volume lives in.
#    It MUST exist before the cluster is created.
docker info --format '{{.ServerVersion}}'
kind version
kubectl version --client
kind delete cluster --name dso202 2>/dev/null || true   # Practical 1 cluster, if any
mkdir -p /tmp/dso202-p2-storage

# 1. Cluster, namespace, quota, retaining StorageClass
kind create cluster --config cluster/kind-cluster.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=180s
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml

# 2. Static provisioning
kubectl apply -f manifests/03-pv-static.yaml -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl exec static-writer -- cat /data/ledger.txt

# 3. Dynamic provisioning (claim stays Pending until the Pod exists)
kubectl apply -f manifests/06-pvc-dynamic.yaml -f manifests/07-pod-dynamic-writer.yaml
kubectl wait --for=condition=Ready pod/dynamic-writer --timeout=120s

# 4. The anti-pattern - observe, then remove
kubectl apply -f manifests/08-deployment-shared-pvc.yaml
kubectl rollout status deployment/shared-writer --timeout=180s
kubectl delete -f manifests/08-deployment-shared-pvc.yaml

# 5. StatefulSet + headless Service + client
kubectl apply -f manifests/09-service-webnote.yaml
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl rollout status statefulset/webnote --timeout=300s
kubectl apply -f manifests/11-pod-client.yaml
kubectl exec client -- wget -qO- http://webnote-0.webnote.dso202-practical-02.svc.cluster.local

# 6. PostgreSQL
kubectl apply -f manifests/12-secret-postgres.yaml -f manifests/13-service-postgres.yaml
kubectl apply -f manifests/14-statefulset-postgres.yaml
kubectl rollout status statefulset/postgres --timeout=600s

# Optional: restore the captured data
kubectl exec -i postgres-0 -- psql -U taskuser -d tasktracker < evidence/tasktracker-dump.sql
```

`kubectl apply -f manifests/` also works in one step, but it recreates the
Stage 4 anti-pattern (file `08`) along with everything else; delete it again
with `kubectl delete -f manifests/08-deployment-shared-pvc.yaml`.

## Cleanup

Deleting the workloads does **not** delete the storage. Claims generated by a
StatefulSet are in no manifest, and volumes from the `dso202-retain` class are
kept after their claim is deleted.

```bash
# 1. Back up first - a retained volume is not a backup
kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker > evidence/tasktracker-dump.sql

# 2. Workloads
kubectl delete -f manifests/14-statefulset-postgres.yaml
kubectl delete -f manifests/10-statefulset-webnote.yaml
kubectl delete -f manifests/11-pod-client.yaml
kubectl delete -f manifests/05-pod-static-writer.yaml

# 3. Claims no manifest created (StatefulSet claims) and the static claim
kubectl get pvc
kubectl delete pvc --all

# 4. Volumes with reclaimPolicy Retain are now Released, not gone
kubectl get pv
kubectl delete pv pv-web-static
kubectl delete pv <name-of-the-released-postgres-pv>

# 5. Context and cluster
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202-p2

# 6. The host directory survives 'kind delete cluster' - remove it last
ls -l /tmp/dso202-p2-storage/pv-web-static/
rm -rf /tmp/dso202-p2-storage

# Optional: reclaim the ~1 GB node image
docker image rm kindest/node:v1.36.1
```
