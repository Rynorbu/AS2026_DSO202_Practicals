# DSO202: Practical 1

## Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads


## Objective

The purpose of this practical was to build, from scratch, the local Kubernetes environment that every subsequent practical in DSO202 depends on, and to use that environment to create, inspect, break, and repair five categories of core Kubernetes object: Namespaces with ResourceQuotas and LimitRanges, Pods, Deployments, and Services.

Rather than working with a real application, the practical deliberately used a single static nginx web server throughout, so that the focus stayed on the behaviour of the Kubernetes objects themselves rather than on application code. This let each stage isolate one concept at a time, cluster topology, control-plane components, multi-tenancy primitives, Pod lifecycle, controller-managed workloads, and service discovery, while building toward a single working, self-healing, load-balanced deployment reachable both from inside and outside the cluster.

## 2. Environment

| Component | Version/Detail |
|---|---|
| Operating system | Linux (WSL2 backend used where applicable) |
| Docker Engine / Docker Desktop | 28.1.1 |
| kind | v0.32.0 (go1.24.4, linux/amd64) |
| kubectl (client) | v1.36.0 |
| Kustomize (bundled with kubectl) | v5.7.1 |
| Kubernetes cluster version | v1.36.1 |
| Cluster topology | 1 control-plane node + 2 worker nodes |
| Container runtime | containerd 2.1.4 |
| CNI | kindnet (default) |
| Default StorageClass provider | local-path-storage (added by kind) |

