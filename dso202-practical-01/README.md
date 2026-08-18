# DSO202 Practical 01 — Local Kubernetes Cluster with kind

This repository builds a three-node Kubernetes cluster on one machine with
[kind](https://kind.sigs.k8s.io/), then deploys and exposes a static nginx web
server using five kinds of Kubernetes object: Namespace, ResourceQuota /
LimitRange, Pod, Deployment, and Service (ClusterIP and NodePort).

Everything the cluster runs is declared in version-controlled files, so the
whole practical can be destroyed and rebuilt from this repository alone.

## Repository layout

```
dso202-practical-01/
├── README.md                        # this file
├── cluster/
│   └── kind-cluster.yaml            # Listing 1 - three-node kind cluster
├── manifests/
│   ├── 00-namespace.yaml            # Listing 2 - Namespace
│   ├── 01-quota-and-limits.yaml     # Listing 3 - ResourceQuota + LimitRange
│   ├── 02-pod-web.yaml              # Listing 4 - bare nginx Pod
│   ├── 03-deployment-web.yaml       # Listing 5 - nginx Deployment (3 replicas)
│   ├── 04-service-clusterip.yaml    # Listing 6 - ClusterIP Service
│   ├── 05-service-nodeport.yaml     # Listing 7 - NodePort Service (30080)
│   └── 06-pod-client.yaml           # Listing 8 - busybox client Pod
├── evidence/                        # captured command output and screenshots
└── report/
    └── practical-01-report.md       # the assessed report
```

The manifest files are numbered so that `kubectl apply -f manifests/` applies
them in dependency order (the namespace first).

## Software versions

| Software           | Version used          |
| ------------------ | --------------------- |
| Operating system   | macOS (Apple Silicon) |
| Docker Desktop     | 29.x                  |
| kind               | v0.32.0               |
| kubectl            | v1.36 (must be within one minor version of the cluster) |
| Kubernetes (nodes) | v1.36.1 (`kindest/node:v1.36.1`) |

Docker needs at least 4 GB of memory, 2 CPUs and 15 GB of free disk. Host port
`30080` must be free.

## Rebuild from an empty machine

```bash
# 0. Verify the tools
docker info --format '{{.ServerVersion}} {{.OperatingSystem}}'
kind version
kubectl version --client

# 1. Create the cluster (context kind-dso202 is selected automatically)
kind create cluster --config cluster/kind-cluster.yaml
kubectl get nodes            # control-plane, worker-node-1, worker-node-2

# 2. Apply every manifest in order
kubectl apply -f manifests/
kubectl config set-context --current --namespace=dso202-practical-01

# 3. Wait for the workloads
kubectl rollout status deployment/web-deployment
kubectl wait --for=condition=Ready pod/client-pod --timeout=60s

# 4. Verify
kubectl get all
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
kubectl exec client-pod -- wget -qO- http://web-clusterip | head -4
curl -s http://localhost:30080 | head -4
kubectl diff -f manifests/ && echo "cluster matches repository"
```

## Cleanup

```bash
# Remove the workloads declaratively, in reverse order
kubectl delete -f manifests/06-pod-client.yaml
kubectl delete -f manifests/05-service-nodeport.yaml
kubectl delete -f manifests/04-service-clusterip.yaml
kubectl delete -f manifests/03-deployment-web.yaml
kubectl delete -f manifests/02-pod-web.yaml

# Reset the default namespace and delete the cluster
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202

# Optional: reclaim the ~1 GB node image
docker image rm kindest/node:v1.36.1
```
