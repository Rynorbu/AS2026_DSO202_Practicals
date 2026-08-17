# DSO202 — Practical 1 Report
## Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

**Module:** DSO202 — Scaling, Orchestration, Monitoring & Observability
**Programme:** BE in Software Engineering
**Practical number:** 1 of 10

---

## 1. Objective

The purpose of this practical was to build, from scratch, the local Kubernetes environment that every subsequent practical in DSO202 depends on, and to use that environment to create, inspect, break, and repair five categories of core Kubernetes object: Namespaces with ResourceQuotas and LimitRanges, Pods, Deployments, and Services.

Rather than working with a real application, the practical deliberately used a single static `nginx` web server throughout, so that the focus stayed on the behaviour of the Kubernetes objects themselves rather than on application code. This let each stage isolate one concept at a time — cluster topology, control-plane components, multi-tenancy primitives, Pod lifecycle, controller-managed workloads, and service discovery — while building toward a single working, self-healing, load-balanced deployment reachable both from inside and outside the cluster.

In descriptor terms, the practical addressed Unit I sections 1.1, 1.2.1–1.2.4, 1.3.1–1.3.3, 1.4.1, and 1.5.1/1.5.3, and covered:

- **LO1** — understanding Kubernetes core concepts and architecture (Stage 2, cluster inspection)
- **LO2** — deploying and managing applications using various resource types (Stages 4–7: Pods, Deployments, Services, cleanup/rebuild)
- **LO3** — operating `kubectl` for cluster management and troubleshooting (used throughout, most heavily in Stages 2, 4, and the debugging commands)
- **LO5 (part)** — applying namespace-based multi-tenancy (Stage 3: Namespaces, ResourceQuotas, LimitRanges)

Persistent storage (LO4) and the service-registry half of LO5 were explicitly out of scope, deferred to Practicals 2 and 6 respectively.

---

## 2. Environment

| Component | Version / Detail |
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

All four prerequisite checks (`docker info`, `kind version`, `kubectl version --client`, and available memory/CPU/disk) passed before Stage 1 began, confirming Docker had at least 4 GB memory, 2 CPUs, and 15 GB free disk allocated.

---

## 3. Procedure and Observations

### Stage 1 — Creating the Three-Node Cluster

The cluster was created from `cluster/kind-cluster.yaml` (Listing 1) using `kind create cluster --config cluster/kind-cluster.yaml`. The build completed through the expected sequence — ensuring the node image, preparing nodes, writing configuration, starting the control plane, installing the CNI and StorageClass, and joining the two worker nodes — and set the active `kubectl` context to `kind-dso202`.

`kind get clusters` returned `dso202`, and `docker ps` confirmed three running containers: `dso202-control-plane`, `dso202-worker`, and `dso202-worker2`. This demonstrates the distinction between the **Docker container name** (fixed by kind's naming pattern) and the **Kubernetes Node object name** (set separately via the kubeadm patch in Listing 1, producing `control-plane`, `worker-node-1`, and `worker-node-2`). `kubectl config current-context` confirmed `kind-dso202` was selected automatically — no manual context switch was needed.

### Stage 2 — Inspecting the Cluster and Its Components

`kubectl cluster-info` showed the API server reachable at a randomly assigned local port, confirming that the port is chosen per-cluster and must always be read live rather than assumed. `kubectl get nodes` listed all three nodes as `Ready`, with the two workers showing `ROLES: <none>` since kind assigns no role label to them by default.

`kubectl describe node worker-node-1` confirmed the custom labels `dso202/node-role=worker` and `dso202/node-index=1` applied by Listing 1, alongside its Capacity/Allocatable figures and any Pods scheduled there.

Listing the control-plane components in `kube-system` (`kubectl get pods -n kube-system -o wide`) showed three clear patterns worth recording:

1. `etcd`, `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` each ran exactly once, all on the `control-plane` node — these are the core control-plane components from Unit I 1.1.1.
2. `kube-proxy` and `kindnet` each ran three times, one per node, illustrating the DaemonSet pattern for components that must run everywhere.
3. `coredns` ran twice, both scheduled on the control-plane node, because CoreDNS tolerates the control-plane taint that ordinary Pods do not.

`kubectl api-resources` confirmed the namespaced/cluster-scoped split: Pods, Services, Deployments, ResourceQuotas, and LimitRanges are namespaced, while Nodes, Namespaces, PersistentVolumes, and StorageClasses are cluster-scoped.

**Checkpoint met:** all three nodes `Ready`; all `kube-system` Pods `Running` at `1/1`.

### Stage 3 — Namespaces, Resource Quotas, and Limit Ranges

A scratch namespace was created and deleted imperatively purely to observe the syntax, before applying the graded declarative version. `kubectl apply -f manifests/00-namespace.yaml` created `dso202-practical`, and `kubectl config set-context --current --namespace=dso202-practical` set it as the context default for the remainder of the practical.

Applying Listing 3 (`01-quota-and-limits.yaml`) created both a `ResourceQuota` and a `LimitRange` in one file. `kubectl describe resourcequota dso202-quota` showed `count/configmaps` already at `1/10` before any work was done — this is the automatically injected `kube-root-ca.crt` ConfigMap that every namespace receives, and is a useful reminder that a quota's `Used` column counts cluster-created objects too, not only user-created ones.

The LimitRange was proven functional directly: a bare Pod (`limitrange-check`) was started with **no** resource declarations at all. Reading it back with `kubectl get pod ... -o jsonpath='{.spec.containers[0].resources}'` showed the cluster had injected `limits: {cpu:200m, memory:128Mi}` and `requests: {cpu:50m, memory:64Mi}` — values that came entirely from the LimitRange defaults, and without which the ResourceQuota (which mandates `requests.cpu`/`requests.memory`/`limits.cpu`/`limits.memory` declarations) would have rejected the Pod outright. The check Pod was deleted immediately afterward, since it was not part of the deliverable.

**Checkpoint met:** `kubectl get resourcequota,limitrange` listed both objects; default namespace confirmed as `dso202-practical`.

### Stage 4 — Pods

A Pod was first created imperatively (`kubectl run web-imperative ...`) to observe the transition from `ContainerCreating` to `Running`, then captured to YAML with `kubectl get pod ... -o yaml`. Comparing that captured file against Listing 4 made clear how much the cluster silently adds at runtime — `status`, `nodeName`, service account, `terminationGracePeriodSeconds`, tolerations, and LimitRange-injected resources — none of which belongs in a manifest intended for version control. The imperative Pod was deleted afterward.

The declarative Pod (`manifests/02-pod-web.yaml`, Listing 4) was then applied. Re-applying the identical file a second time returned `pod/web-pod unchanged`, demonstrating the idempotency that is the entire point of declarative management: the command states desired state, and nothing happens if that state already holds.

`kubectl get pod web-pod -o wide` confirmed placement on a worker node (the scheduler's choice, not the manifest's) and an internal Pod IP from the cluster's Pod CIDR — explicitly unreachable from the host. `kubectl describe pod web-pod` was used to read the `Events:` timeline (`Scheduled → Pulling → Pulled → Created → Started`), which mapped directly onto the `default-scheduler` and `kubelet` components named in Unit I 1.1.

**Labels and selectors** were exercised with `--show-labels`, label-based filtering (`-l app=web`, set-based `-l 'tier in (frontend,backend)'`, negation `-l app!=web`), and runtime label add/remove with `kubectl label`. An annotation was added separately with `kubectl annotate` to demonstrate that annotations — unlike labels — cannot be selected on.

**Debugging commands** exercised: `kubectl logs`, `kubectl exec -it ... -- sh` (Alpine ships `sh`, not `bash`), `kubectl exec` for one-off commands, `kubectl port-forward pod/web-pod 8080:80` (confirmed with `curl http://localhost:8080` returning the nginx welcome page), and `kubectl explain` for live schema lookups.

**Checkpoint met:** `web-pod` `Running`/`1/1`; labels visible; logs, exec, and port-forward all produced expected output.

### Stage 5 — Deployments

An imperative Deployment was generated with `--dry-run=client -o yaml` purely for comparison — it omitted labels, resource declarations, probes, and any rollout strategy, all of which Listing 5 supplies explicitly.

Applying `manifests/03-deployment-web.yaml` and running `kubectl rollout status deployment/web-deployment` confirmed a successful rollout to `3/3` replicas. `kubectl get deployment,replicaset,pod -l app=web` made the ownership chain concrete: **Deployment → ReplicaSet → Pods**, with the ReplicaSet name derived from a hash of the Pod template and each Pod name suffixed randomly from that. This was confirmed programmatically via the ReplicaSet's `ownerReferences`, which pointed back to `Deployment/web-deployment`.

**Self-healing** was demonstrated by deleting one Pod directly and immediately re-listing: a replacement appeared within seconds, and `kubectl get events --field-selector reason=SuccessfulCreate` confirmed the **ReplicaSet**, not the Deployment, performed the recreation — the Deployment delegates.

**Scaling:** imperative scale-up to 5 replicas succeeded, but re-applying the unmodified manifest silently reverted the Deployment back to 3 — a deliberate illustration of why imperative changes are unsafe for graded/production work, since the next `apply` always wins. This is exactly the property ArgoCD enforces continuously in Unit IV.

**Rolling update:** changing the image to `nginx:1.31-alpine` via `kubectl set image`, with a `kubernetes.io/change-cause` annotation for traceability, produced a textbook rolling update — one new Pod created and made Ready before one old Pod was terminated, keeping ready replicas at 3 throughout (`maxUnavailable: 0`). Two ReplicaSets existed afterward, one scaled to zero and retained specifically to support rollback.

**Rollback:** `kubectl rollout history` showed two revisions with the change-cause annotation correctly attached to revision 2. `kubectl rollout undo` successfully reverted the image to `nginx:1.30-alpine`.

**Failed rollout recovery:** deploying a deliberately non-existent tag (`nginx:9.99-does-not-exist`) caused the new Pod to enter `ImagePullBackOff` and the rollout to stall — critically, the three healthy Pods were **never removed**, because the strategy's `maxUnavailable: 0` meant the stalled rollout produced no outage, only a stalled state. `kubectl rollout undo` recovered cleanly. Finally, `kubectl apply -f manifests/03-deployment-web.yaml` followed by `kubectl diff` confirmed the cluster and the repository manifest were back in agreement (no diff output).

**Checkpoint met:** `3/3` replicas on `nginx:1.30-alpine`; rollout history showed ≥2 revisions; `kubectl diff` reported no drift.

### Stage 6 — Services

`manifests/04-service-clusterip.yaml` (Listing 6) was applied, producing a `ClusterIP` Service with an allocated virtual IP. `kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip` listed three Pod addresses, matching the three ready replicas — using `EndpointSlice` rather than the deprecated `Endpoints` API.

A dedicated `client-pod` (Listing 8) was applied to issue requests from inside the cluster. `nslookup web-clusterip` resolved correctly to the Service's ClusterIP through CoreDNS, confirming the standard `<service>.<namespace>.svc.cluster.local` naming pattern, and `wget`/`curl` through the Service returned the nginx welcome page.

**Load balancing** was made visible by writing a distinct `index.html` (containing `$HOSTNAME`) into each of the three Pods and issuing nine sequential requests through the Service: all three Pod names appeared roughly evenly in the response distribution, confirming traffic was spread across all ready endpoints.

**Readiness gating** was demonstrated by deleting `index.html` on one Pod (breaking its readiness probe while its TCP liveness probe kept passing, so the container was not restarted). Within one probe period that Pod's address was removed from the EndpointSlice, and repeating the load-balancing loop confirmed only the two remaining Pods answered. Restoring the file caused the address to reappear in the EndpointSlice automatically — the same mechanism that made the earlier zero-downtime rolling update possible.

A **broken Service** was deliberately created imperatively (`kubectl create service clusterip broken-service --tcp=80:80`), whose auto-generated selector matched no Pod. Its EndpointSlice came back empty — the diagnostic signature of a selector mismatch — and the Service was deleted afterward.

`manifests/05-service-nodeport.yaml` (Listing 7) was then applied, fixing `nodePort: 30080` to match the host port published by Listing 1. `curl http://localhost:30080` succeeded from the host with **no port-forward running**, and repeated requests again showed different Pod names. `docker exec dso202-worker curl -s http://localhost:30080` confirmed the node port was open on every node, not only the one published to the host. Finally, a `LoadBalancer` Service was created and observed to remain `EXTERNAL-IP: <pending>` indefinitely, since kind provides no cloud load-balancer integration — expected behaviour, not a fault — and was deleted.

**Checkpoint met:** `curl http://localhost:30080` answered from the host; DNS resolution succeeded from `client-pod`; the `web-clusterip` EndpointSlice listed three addresses.

### Stage 7 — Cleanup and Reproducibility

Final evidence was captured with `kubectl get all -o wide`, plus a separate query for ResourceQuotas/LimitRanges/EndpointSlices (since `kubectl get all` does not include these), node state, and the full event log — all saved under `evidence/`.

Workload objects were deleted declaratively, in reverse creation order, from the same manifest files that created them (`client-pod` → NodePort Service → ClusterIP Service → Deployment → Pod), leaving only the Namespace, ResourceQuota, and LimitRange behind. Running `kubectl apply -f manifests/` against the now-empty namespace rebuilt every object in one command, in the lexical order guaranteed by the numbered filenames, proving the repository — not the local machine — is the source of truth.

The kubectl default namespace was reset to `default`, and the cluster was deleted with `kind delete cluster --name dso202`. `kind get clusters` returned no clusters, and `docker ps` showed no remaining node containers, confirming clean teardown.

**Checkpoint met:** no clusters remained; the repository alone was sufficient to rebuild the entire practical from nothing.

---

## 4. Analysis

**On architecture (LO1):** Stage 2 made the theoretical control-plane/data-plane split from lecture concrete and verifiable. Seeing `etcd`, the API server, the scheduler, and the controller-manager as ordinary Pods confined to the control-plane node — while `kube-proxy` and the CNI ran as DaemonSets on every node — clarified *why* Kubernetes treats the control plane as a set of workloads rather than a monolith: it is upgraded, scaled, and debugged with the exact same tools used for application Pods.

**On resource governance (LO5, part):** The ResourceQuota/LimitRange pairing in Stage 3 demonstrated a design pattern rather than two unrelated objects — a quota alone would simply reject any Pod lacking explicit resource declarations, which is unfriendly for quick imperative testing; the LimitRange supplies sane defaults so that governance and convenience coexist. This is the practical mechanism behind namespace-based multi-tenancy: several teams can share one cluster because no namespace can consume more than its quota allows, and no container can silently go undeclared.

**On workload management (LO2):** The Deployment → ReplicaSet → Pod ownership chain, and the self-healing/rolling-update/rollback behaviours built on top of it, were the clearest illustration in the practical of Kubernetes' reconciliation model: the cluster continuously drives observed state toward declared state. The stalled-rollout exercise in Stage 5 was particularly instructive — `maxUnavailable: 0` converted what would otherwise be a failed deployment into a *safely stalled* one, with zero customer-facing impact, which is the entire rationale for rollout strategies existing at all.

**On service discovery (LO2):** Stage 6 showed that a Service is fundamentally a level-4, selector-driven abstraction over EndpointSlices, not a proxy in any richer sense — it does not understand HTTP paths, cannot terminate TLS, and performs no retries. The readiness-gating exercise reinforced that "traffic reaches only ready Pods" is not a passive property but an actively maintained one, continuously re-evaluated by the EndpointSlice controller.

**On CLI operation (LO3):** The recurring theme across every stage was that `kubectl describe`'s `Events:` section, not `kubectl get`, is the first place to look when something misbehaves — it names both *what* happened and *which component* (scheduler vs. kubelet vs. ReplicaSet) performed it, which is essential for correctly attributing faults during troubleshooting.

---

## 5. Reflection

The most difficult part of this practical was correctly diagnosing the deliberately broken states rather than the "happy path" steps — specifically, distinguishing a Service with a **selector mismatch** (Stage 6, Step 8) from a Service whose selector was correct but pointed at Pods that were simply not yet ready. Both produce an empty or short `EndpointSlice`, and at first glance they look identical. The diagnostic step that resolved this was checking the Pod's `READY` column directly with `kubectl get pods` *before* inspecting the Service — a correct selector with `0/1` ready Pods behaves exactly like a broken selector from the Service's point of view, and only cross-referencing Pod readiness against the EndpointSlice contents made the distinction clear.

A second error encountered was during the rolling-update failure exercise (Stage 5, Step 15): the deliberately invalid image tag caused the new ReplicaSet's Pod to sit in `ImagePullBackOff`, and `kubectl rollout status --timeout=60s` correctly timed out rather than hanging indefinitely. The instinct at first was to delete the failed Pod directly, but this achieves nothing — the ReplicaSet immediately recreates it with the same broken image. The correct fix was `kubectl rollout undo`, which reverts the entire Deployment's Pod template rather than acting on an individual Pod, reinforcing that Pods are disposable and the Deployment/ReplicaSet template is the actual unit of control.

If repeating this practical, I would capture evidence continuously (screenshots and command output) at the end of each stage rather than relying on the final `evidence/final-state-*.txt` dump at Stage 7, since several intermediate states (e.g., the two-ReplicaSet state mid-rollout, or the EndpointSlice with only two addresses during the readiness-gating demonstration) are transient and cannot be reconstructed after the fact.

One thing that remains unclear to me is exactly how the EndpointSlice controller batches updates when many Pods change readiness simultaneously at larger scale — the practical's three-replica scale made the propagation feel effectively instantaneous, but I would like to understand the batching/rate-limiting behaviour that presumably exists to protect the API server in clusters with hundreds of endpoints per Service, which I expect is covered in a later unit on scaling and observability.

---

## 6. References

- Kubernetes Documentation — Tools, *kubectl* installation and version skew policy. https://kubernetes.io/docs/tasks/tools/ (accessed 17 August 2026)
- kind Documentation — Quick Start / Installation. https://kind.sigs.k8s.io/docs/user/quick-start/#installation (accessed 17 August 2026)
- Docker Documentation — Engine installation. https://docs.docker.com/engine/install/ (accessed 17 August 2026)
- DSO202_Practical1_Manifests.md — companion listings file (Listings 1–8), module resource.
- DSO202 Practical 1 brief — "Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads," module handout.