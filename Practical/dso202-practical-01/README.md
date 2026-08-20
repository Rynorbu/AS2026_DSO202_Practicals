# DSO202: Practical 1
## Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

## 1. Purpose of This Repository

This repository contains the complete, reproducible deliverable for **DSO202 Practical 1** (BE in Software Engineering, Module: *Scaling, Orchestration, Monitoring & Observability*).

It builds a local three-node Kubernetes cluster using **kind** (Kubernetes IN Docker), and uses that cluster to create, inspect, and manage five categories of core Kubernetes object — a Namespace governed by a ResourceQuota and LimitRange, a bare Pod, a Deployment (with rolling update, rollback, and self-healing), and two types of Service (ClusterIP and NodePort) — all backing a single static `nginx` web workload.

Every object in this repository is defined **declaratively**, in version-controlled YAML, so that the entire environment can be destroyed and rebuilt from this repository alone, on any machine, with no manual steps and no hidden state. That reproducibility is the primary grading criterion this repository is built to satisfy: the repository — not any individual laptop — is the source of truth.

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


