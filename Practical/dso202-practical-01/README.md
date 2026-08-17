# DSO202 — Practical 1
## Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

## 1. Purpose of This Repository

This repository contains the complete, reproducible deliverable for **DSO202 Practical 1** (BE in Software Engineering, Module: *Scaling, Orchestration, Monitoring & Observability*).

It builds a local three-node Kubernetes cluster using **kind** (Kubernetes IN Docker), and uses that cluster to create, inspect, and manage five categories of core Kubernetes object — a Namespace governed by a ResourceQuota and LimitRange, a bare Pod, a Deployment (with rolling update, rollback, and self-healing), and two types of Service (ClusterIP and NodePort) — all backing a single static `nginx` web workload.

Every object in this repository is defined **declaratively**, in version-controlled YAML, so that the entire environment can be destroyed and rebuilt from this repository alone, on any machine, with no manual steps and no hidden state. That reproducibility is the primary grading criterion this repository is built to satisfy: the repository — not any individual laptop — is the source of truth.

**Descriptor coverage:** Unit I — 1.1, 1.2.1–1.2.4, 1.3.1–1.3.3, 1.4.1, 1.5.1, 1.5.3
**Learning outcomes addressed:** LO1, LO2, LO3, and the multi-tenancy half of LO5

---

## 2. Repository Structure

```
dso202-practical-01/
├── README.md                          # this file
├── cluster/
│   └── kind-cluster.yaml              # 3-node kind cluster definition (Listing 1)
├── manifests/
│   ├── 00-namespace.yaml              # Namespace (Listing 2)
│   ├── 01-quota-and-limits.yaml       # ResourceQuota + LimitRange (Listing 3)
│   ├── 02-pod-web.yaml                # bare Pod (Listing 4)
│   ├── 03-deployment-web.yaml         # Deployment (Listing 5)
│   ├── 04-service-clusterip.yaml      # ClusterIP Service (Listing 6)
│   ├── 05-service-nodeport.yaml       # NodePort Service (Listing 7)
│   └── 06-pod-client.yaml             # debug/client Pod (Listing 8)
├── evidence/                          # command output (.txt) and screenshots (.png) for every stage
└── report/
    └── practical-01-report.md         # the assessed report
```

`kubectl apply -f manifests/` applies every file in this directory in lexical order — this is precisely why the manifest filenames are numbered.

---

## 3. Software Versions Used

| Software | Version verified against |
|---|---|
| Docker Engine / Docker Desktop | 28.1.1 |
| kind | v0.32.0 (go1.24.4, linux/amd64) |
| kubectl (client) | v1.36.0 |
| Kustomize (bundled with kubectl) | v5.7.1 |
| Kubernetes cluster | v1.36.1 (`kindest/node:v1.36.1`) |
| Container runtime | containerd 2.1.4 |
| CNI | kindnet (kind default) |
| Node OS image | Debian GNU/Linux 12 (bookworm), kernel 6.10.14 |

`kubectl` is supported within one minor version of the cluster it addresses. Since this cluster runs Kubernetes v1.36.1, any `kubectl` client from v1.35 through v1.37 will work correctly.

---

## 4. Prerequisites

Before rebuilding this practical, confirm the following are installed and reachable:

- **Docker Engine or Docker Desktop** ≥ 24.0, daemon running
- **kind** ≥ 0.32.0
- **kubectl** v1.35 or v1.36
- At least **4 GB memory**, **2 CPUs**, and **15 GB free disk** allocated to Docker

Windows users must enable the **WSL 2** backend in Docker Desktop and run every command below from inside a WSL 2 Linux shell. Running `kind` from PowerShell directly is not supported for this practical.

**Installation references** (commands change frequently — always use the official pages):
- Docker: https://docs.docker.com/engine/install/
- kind: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
- kubectl: https://kubernetes.io/docs/tasks/tools/

---

## 5. Rebuilding the Practical from an Empty Machine

Run the following commands in order, from the root of this repository.

### Step 1 — Verify prerequisites

```bash
docker info --format '{{.ServerVersion}} {{.OperatingSystem}}'
kind version
kubectl version --client
```

All three commands must print a version before continuing.

### Step 2 — Create the three-node cluster

```bash
kind create cluster --config cluster/kind-cluster.yaml
```

This creates one control-plane node and two worker nodes, installs the CNI and default StorageClass, and automatically sets the `kubectl` context to `kind-dso202`.

Verify:

```bash
kind get clusters
kind get nodes --name dso202
kubectl config current-context
kubectl get nodes
```

Expected: `kind get clusters` prints `dso202`; all three nodes report `Ready`.

### Step 3 — Apply every manifest

```bash
kubectl apply -f manifests/
```

This creates, in order: the `dso202-practical` Namespace, the ResourceQuota and LimitRange, the `web-pod` Pod, the `web-deployment` Deployment, the `web-clusterip` and `web-nodeport` Services, and the `client-pod` debug Pod.

### Step 4 — Set the namespace as the kubectl default (optional but recommended)

```bash
kubectl config set-context --current --namespace=dso202-practical
```

### Step 5 — Verify the rebuild

```bash
kubectl get all
kubectl get resourcequota,limitrange,endpointslice
kubectl rollout status deployment/web-deployment
curl -s http://localhost:30080
```

Expected: `web-deployment` reports `3/3`; the `curl` against the NodePort returns the nginx welcome page (or the custom "served by `<pod-name>`" page if Stage 6's load-balancing demo has been re-run) directly from the host, with no `port-forward` needed.

### Step 6 — (Optional) Confirm reproducibility by tearing down and rebuilding

```bash
kubectl delete -f manifests/06-pod-client.yaml
kubectl delete -f manifests/05-service-nodeport.yaml
kubectl delete -f manifests/04-service-clusterip.yaml
kubectl delete -f manifests/03-deployment-web.yaml
kubectl delete -f manifests/02-pod-web.yaml
kubectl get all                      # should report: No resources found
kubectl apply -f manifests/          # rebuilds everything in one command
```

---

## 6. Cleanup Commands

To fully remove the practical and reclaim resources:

```bash
# Reset the kubectl default namespace
kubectl config set-context --current --namespace=default

# Delete the entire cluster (removes all nodes, all objects, and the kubeconfig context)
kind delete cluster --name dso202

# Confirm the cluster and its containers are gone
kind get clusters
docker ps
```

Expected: `kind get clusters` reports no clusters; `docker ps` shows no `kindest/node` containers remaining. Deleting the cluster automatically removes the `kind-dso202` context from the local kubeconfig — no manual `kubectl config` cleanup is required.

**Optional** — reclaim the ~1 GB node image (only if disk space is short; keeping it makes the next `kind create cluster` much faster):

```bash
docker image ls | grep kindest
docker image rm kindest/node:v1.36.1
```

---

## 7. Manifest Comments Policy

Every file in `manifests/` includes inline comments (`#`) explaining:
- the purpose of the object,
- any non-obvious field (e.g. why `nodePort: 30080` is fixed rather than auto-allocated, or why `maxUnavailable: 0` is set on the Deployment), and
- which Listing number in `DSO202_Practical1_Manifests.md` it corresponds to.

This is required so that any reader — not only the author — can understand each object's intent directly from version control, without cross-referencing the practical brief.

---

## 8. Evidence

The `evidence/` directory contains:
- Screenshots (`.png`) captured at each stage checkpoint, named `fig-<stage>-<n>-<short-description>.png`
- Captured command output (`.txt`) for key verification steps, including the final-state dump taken immediately before cleanup (`final-state-all.txt`, `final-state-nodes.txt`, `final-state-events.txt`)

See `report/practical-01-report.md`, Section 3 ("Procedure and Observations"), for the full mapping of each screenshot to the step and command it documents.

---

## 9. Deliverable Checklist

- [x] `cluster/kind-cluster.yaml`
- [x] All manifests in `manifests/`, commented
- [x] `README.md` (this file)
- [x] `evidence/` (screenshots + captured output)
- [x] `report/practical-01-report.md`