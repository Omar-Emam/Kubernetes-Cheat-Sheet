# ☸️ Kubernetes Cheat Sheet

A refresher and copy-paste-ready reference for Kubernetes. **Part 1** explains the concepts and the different types of each thing (Services, workloads, Ingress, volumes and so on). **Part 2** lists the commands for `kubectl` and the surrounding tooling (`minikube`, `kubeadm`, `crictl`, `etcdctl`, `helm`, `kustomize`), each with a short description.

> **Conventions used below**
> `<pod>` = resource name · `-n <ns>` = namespace · `-A` = all namespaces · `-l` = label selector
> Most `kubectl` commands accept `-n`, `-A`, `-o`, `-l`, `--context`. They are only repeated where it matters.

---

## 📑 Table of Contents

### Part 1 — Concepts Refresher

- [Architecture](#architecture)
- [How Kubernetes Works: Desired State](#how-kubernetes-works-desired-state)
- [Workload Types](#workload-types)
- [Inside a Pod](#inside-a-pod)
- [Probes](#probes)
- [Deployment Strategies](#deployment-strategies)
- [Service Types](#service-types)
- [Ingress and Gateway API](#ingress-and-gateway-api)
- [Networking and Network Policies](#networking-and-network-policies)
- [Configuration: ConfigMaps and Secrets](#configuration-configmaps-and-secrets)
- [Volume and Storage Types](#volume-and-storage-types)
- [Resources and QoS Classes](#resources-and-qos-classes)
- [Scheduling Controls](#scheduling-controls)
- [Autoscaling Types](#autoscaling-types)
- [Availability and Disruptions](#availability-and-disruptions)
- [Security and Access](#security-and-access)
- [Organizing: Namespaces, Labels, Annotations](#organizing-namespaces-labels-annotations)
- [Extending Kubernetes](#extending-kubernetes)

### Part 2 — Commands

1. [Setup, Aliases & Autocompletion](#1-setup-aliases--autocompletion)
2. [Local Cluster with Minikube](#2-local-cluster-with-minikube)
3. [Kubeconfig, Contexts & Clusters](#3-kubeconfig-contexts--clusters)
4. [Cluster Information](#4-cluster-information)
5. [Resource Types & Short Names](#5-resource-types--short-names)
6. [Viewing Resources (get / describe)](#6-viewing-resources-get--describe)
7. [Output Formats, JSONPath & Custom Columns](#7-output-formats-jsonpath--custom-columns)
8. [Creating Resources (Imperative)](#8-creating-resources-imperative)
9. [Declarative Management (apply / diff)](#9-declarative-management-apply--diff)
10. [Generating YAML (dry-run)](#10-generating-yaml-dry-run)
11. [Updating Resources (edit / patch / set)](#11-updating-resources-edit--patch--set)
12. [Deleting Resources](#12-deleting-resources)
13. [Pods: Logs, Exec, Copy, Attach](#13-pods-logs-exec-copy-attach)
14. [Deployments & Rollouts](#14-deployments--rollouts)
15. [Scaling & Autoscaling](#15-scaling--autoscaling)
16. [Services, Networking & Port Forwarding](#16-services-networking--port-forwarding)
17. [ConfigMaps & Secrets](#17-configmaps--secrets)
18. [Storage (PV / PVC / StorageClass)](#18-storage-pv--pvc--storageclass)
19. [Jobs & CronJobs](#19-jobs--cronjobs)
20. [Labels, Annotations & Selectors](#20-labels-annotations--selectors)
21. [Namespaces](#21-namespaces)
22. [Nodes: Cordon, Drain, Taints](#22-nodes-cordon-drain-taints)
23. [RBAC, ServiceAccounts & Auth](#23-rbac-serviceaccounts--auth)
24. [Resource Usage & Events](#24-resource-usage--events)
25. [Debugging & Troubleshooting](#25-debugging--troubleshooting)
26. [Waiting & Scripting](#26-waiting--scripting)
27. [Kustomize](#27-kustomize)
28. [Plugins & Krew](#28-plugins--krew)
29. [kubeadm (Cluster Lifecycle)](#29-kubeadm-cluster-lifecycle)
30. [etcd Backup & Restore](#30-etcd-backup--restore)
31. [Container Runtime (crictl)](#31-container-runtime-crictl)
32. [Node-level Troubleshooting](#32-node-level-troubleshooting)
33. [Helm Essentials](#33-helm-essentials)
34. [Global Flags Reference](#34-global-flags-reference)
35. [Pod Phases & Common Errors](#35-pod-phases--common-errors)
36. [YAML Skeletons](#36-yaml-skeletons)

---

# Part 1 — Concepts Refresher

## Architecture

A cluster has a **control plane** that makes decisions and **worker nodes** that run your containers.

```
                ┌──────────────────── Control Plane ────────────────────┐
 kubectl ─────► │ kube-apiserver ◄──► etcd                               │
                │      ▲   ▲                                             │
                │      │   └── kube-scheduler                            │
                │      └────── kube-controller-manager / cloud-controller│
                └──────┼─────────────────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
   ┌── Worker Node ──┐          ┌── Worker Node ──┐
   │ kubelet         │          │ kubelet         │
   │ kube-proxy      │          │ kube-proxy      │
   │ container runtime│         │ container runtime│
   │ [pods...]       │          │ [pods...]       │
   └─────────────────┘          └─────────────────┘
```

| Component | Where | What it does |
|---|---|---|
| **kube-apiserver** | Control plane | The front door. Every command and component talks to the cluster through it. Validates and stores objects. |
| **etcd** | Control plane | Key-value database holding the entire cluster state. Back it up. |
| **kube-scheduler** | Control plane | Picks which node each new pod runs on, based on resources, affinity, taints and so on. |
| **kube-controller-manager** | Control plane | Runs the controllers (Deployment, ReplicaSet, Node, Job…) that keep actual state matching desired state. |
| **cloud-controller-manager** | Control plane | Talks to the cloud provider: creates load balancers, manages node lifecycle and routes. |
| **kubelet** | Every node | Agent that makes sure the containers described in pod specs are running and healthy on its node. |
| **kube-proxy** | Every node | Programs iptables/IPVS rules so Service IPs route to pods. Some CNIs (e.g. Cilium) replace it. |
| **Container runtime** | Every node | Actually runs containers (containerd, CRI-O) through the Container Runtime Interface (CRI). |
| **CoreDNS** | Add-on (pods) | Cluster DNS, so `my-svc.my-ns` resolves to a Service IP. |
| **CNI plugin** | Add-on | Gives pods IPs and connects them across nodes (Calico, Cilium, Flannel, Azure CNI…). |

**Managed vs self-managed:** on AKS, EKS and GKE the cloud runs the control plane for you; you only see worker nodes. With kubeadm or minikube you run everything yourself.

---

## How Kubernetes Works: Desired State

- You **declare** what you want (a YAML object with a `spec`).
- Controllers **observe** the actual state (the object's `status`) and **act** to close the gap. This is the reconciliation (control) loop.
- Example: a Deployment says `replicas: 3`. If a pod dies, the ReplicaSet controller sees 2 and creates a new one.

Every object has the same basic shape:

```yaml
apiVersion: apps/v1     # API group/version
kind: Deployment        # Object type
metadata:               # Name, namespace, labels, annotations
  name: web
spec: {}                # What you want
status: {}              # What actually exists (written by Kubernetes)
```

**Owner chain:** `Deployment → ReplicaSet → Pod`, `CronJob → Job → Pod`. Deleting the owner deletes its children (cascading deletion).

---

## Workload Types

| Type | Use it for | Key behaviour |
|---|---|---|
| **Pod** | Smallest deployable unit; one or more containers sharing network and storage | Not self-healing on its own. Rarely created directly. |
| **ReplicaSet** | Keeping N identical pods running | Managed by Deployments; you almost never create one yourself. |
| **Deployment** | Stateless apps (web, APIs) | Rolling updates, rollbacks, scaling. Pods are interchangeable with random names. |
| **StatefulSet** | Stateful apps (databases, Kafka, ZooKeeper) | Stable names (`db-0`, `db-1`), stable per-pod storage, ordered start/stop. Needs a headless Service. |
| **DaemonSet** | One pod per node (log agents, monitoring, CNI, CSI) | New nodes get the pod automatically. |
| **Job** | Run-to-completion tasks (migrations, batch) | Retries on failure (`backoffLimit`); supports `parallelism` and `completions`. |
| **CronJob** | Scheduled Jobs (backups, reports) | Creates a Job on a cron schedule. |
| **ReplicationController** | Legacy | Replaced by ReplicaSet/Deployment. |

**Quick pick:** stateless → Deployment · needs stable identity or disk → StatefulSet · on every node → DaemonSet · runs once → Job · runs on a schedule → CronJob.

---

## Inside a Pod

**Container types within a pod**

| Type | Purpose |
|---|---|
| **App container** | Your main workload. |
| **Init container** | Runs to completion **before** app containers start (wait for DB, run migrations, fetch config). Run one at a time, in order. |
| **Sidecar container** | Helper that runs **alongside** the app for the pod's lifetime (log shipper, proxy, service mesh). Native sidecars are init containers with `restartPolicy: Always`. |
| **Ephemeral container** | Added temporarily to a running pod for debugging (`kubectl debug`). |

**Pod phases:** `Pending` → `Running` → `Succeeded` / `Failed` (or `Unknown` if the node is unreachable).

**restartPolicy** (pod level)

| Value | Meaning | Typical for |
|---|---|---|
| `Always` | Always restart containers (default) | Deployments, StatefulSets, DaemonSets |
| `OnFailure` | Restart only on non-zero exit | Jobs |
| `Never` | Never restart | One-off pods, Jobs |

**imagePullPolicy**

| Value | Meaning |
|---|---|
| `Always` | Pull every time the container starts (default for `:latest` or no tag). |
| `IfNotPresent` | Pull only if the image is not on the node (default for specific tags). |
| `Never` | Only use a local image. |

---

## Probes

Probes are health checks the kubelet runs against containers.

| Probe | Question it answers | On failure |
|---|---|---|
| **startupProbe** | Has the app finished starting? | Keeps retrying; other probes wait until it passes. Container restarted if it never passes. Use for slow-starting apps. |
| **readinessProbe** | Can it receive traffic right now? | Pod removed from Service endpoints (no restart). |
| **livenessProbe** | Is it still alive or stuck? | Container is **restarted**. |

**Probe mechanisms:** `httpGet` (2xx/3xx = healthy) · `tcpSocket` (port opens) · `exec` (command exits 0) · `grpc` (gRPC health check).

**Tuning fields:** `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`, `successThreshold`.

> 💡 Never point the liveness probe at a dependency (like the database). If the DB goes down, every pod restarts in a loop.

---

## Deployment Strategies

| Strategy | How it works | Built in? |
|---|---|---|
| **RollingUpdate** | Replaces pods gradually. `maxSurge` = extra pods allowed; `maxUnavailable` = pods allowed down. | ✅ Default |
| **Recreate** | Kills all old pods, then starts new ones. Causes downtime. Use when two versions can't run together. | ✅ |
| **Blue/Green** | Run old (blue) and new (green) side by side, then switch the Service selector. | Manual, or Argo Rollouts / Flagger |
| **Canary** | Send a small % of traffic to the new version, increase gradually. | Gateway API weights, service mesh, Argo Rollouts / Flagger |

---

## Service Types

A **Service** gives a stable IP and DNS name to a changing set of pods, selected by labels.

| Type | Reachable from | How |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Virtual IP, load-balanced across pods. |
| **NodePort** | Outside, via `<any-node-IP>:<port>` | Opens the same port (30000–32767) on every node, plus a ClusterIP. |
| **LoadBalancer** | Outside, via a cloud load balancer | Asks the cloud provider for an external LB, plus NodePort and ClusterIP. |
| **ExternalName** | Inside the cluster | DNS CNAME to an external hostname. No proxying, no selector. |
| **Headless** (`clusterIP: None`) | Inside the cluster | No virtual IP. DNS returns individual pod IPs. Used by StatefulSets. |

**Port terms:** `port` = the Service's port · `targetPort` = the container's port · `nodePort` = the port on each node.

**Other options:** `sessionAffinity: ClientIP` for sticky sessions · `externalTrafficPolicy: Local` keeps the client source IP and avoids extra hops · `internalTrafficPolicy: Local` routes only to pods on the same node.

**Without a selector:** a Service with no selector plus manual EndpointSlices points at things outside the cluster (e.g. an external DB by IP).

---

## Ingress and Gateway API

Services work at layer 4 (TCP/UDP). For **HTTP(S) routing** by hostname and path, with TLS termination, you use **Ingress** or its successor **Gateway API**.

**Ingress**

- An **Ingress** object holds routing rules: `host` + `path` → Service.
- An **Ingress object does nothing by itself.** You need an **Ingress controller** running in the cluster to read the rules and route traffic.
- **IngressClass** says which controller handles which Ingress (`ingressClassName: nginx`).
- **pathType:** `Prefix` (matches `/api` and `/api/...`) · `Exact` (only that path) · `ImplementationSpecific` (controller decides).
- **TLS:** reference a `kubernetes.io/tls` Secret in `spec.tls`.
- Advanced features (rewrites, rate limits, auth) are controller-specific **annotations**, so they don't carry over between controllers.
- The Ingress API is **frozen**: still supported, but no new features.

**Common Ingress controllers:** Traefik · HAProxy · Kong · F5 NGINX Ingress Controller · Contour · Azure Application Gateway for Containers · AWS Load Balancer Controller · GKE Ingress.

> ⚠️ The community **ingress-nginx** controller (the one in most tutorials) was **retired in March 2026**: no more releases or security fixes. Existing installs keep working. For new setups, use Gateway API or another maintained controller. To check whether you run it: `kubectl get pods -A -l app.kubernetes.io/name=ingress-nginx`. It is a different project from F5's NGINX Ingress Controller, which is still maintained.

**Ingress routing patterns (types of Ingress)**

| Pattern | What it does | Example |
|---|---|---|
| **Single service / default backend** | All traffic goes to one Service, no rules | Everything → `web` |
| **Simple fanout** | **One host**, different **paths** → different Services | `shop.com/api` → `api`, `shop.com/` → `web` |
| **Name-based virtual hosting** | **Different hosts** on the same IP → different Services | `api.shop.com` → `api`, `blog.shop.com` → `blog` |
| **TLS termination** | HTTPS ends at the Ingress, plain HTTP to pods | Cert from a `kubernetes.io/tls` Secret |
| **Wildcard host** | Match all subdomains | `*.shop.com` → `tenant-router` |
| **Catch-all rule** | A rule with no `host` matches requests for any host | Fallback for unknown hosts |

<details>
<summary><b>Single service (default backend)</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single
spec:
  ingressClassName: <class>
  defaultBackend:                      # No rules: everything goes here
    service: { name: web, port: { number: 80 } }
```
</details>

<details>
<summary><b>Simple fanout</b> — one host, many paths</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fanout
spec:
  ingressClassName: <class>
  rules:
    - host: shop.com
      http:
        paths:
          - path: /api                 # shop.com/api/* → api
            pathType: Prefix
            backend: { service: { name: api, port: { number: 8080 } } }
          - path: /images              # shop.com/images/* → images
            pathType: Prefix
            backend: { service: { name: images, port: { number: 80 } } }
          - path: /                    # Everything else → web
            pathType: Prefix
            backend: { service: { name: web, port: { number: 80 } } }
```

> 💡 The longest matching path wins, so `/api` beats `/`. The request path is forwarded **unchanged**: the `api` Service receives `/api/users`, not `/users`. Stripping the prefix needs a controller-specific rewrite annotation (or a `URLRewrite` filter in Gateway API).
</details>

<details>
<summary><b>Name-based virtual hosting</b> — many hosts, one IP</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vhosts
spec:
  ingressClassName: <class>
  rules:
    - host: api.shop.com               # Matched on the HTTP Host header
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 8080 } } }
    - host: blog.shop.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: blog, port: { number: 80 } } }
    - http:                            # No host = catch-all for any other hostname
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: default-site, port: { number: 80 } } }
```
</details>

<details>
<summary><b>TLS termination</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example
spec:
  ingressClassName: <class>
  tls:
    - hosts: [api.shop.com, blog.shop.com]   # Hosts covered by the cert
      secretName: shop-tls                   # kubernetes.io/tls Secret, same namespace
  rules:
    - host: api.shop.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 8080 } } }
```

> 💡 Ingress supports one TLS port (443) and terminates TLS. TLS passthrough (pod handles TLS itself) needs controller-specific options, or a `TLSRoute` in Gateway API. cert-manager can issue and renew the Secret automatically.
</details>

**Same patterns in Gateway API**

| Ingress pattern | Gateway API equivalent |
|---|---|
| Default backend | `HTTPRoute` with one rule and no `matches` |
| Simple fanout | One `HTTPRoute`, several `rules` with `PathPrefix` matches |
| Name-based virtual hosting | Separate `HTTPRoute`s with different `hostnames` (or several listeners on the Gateway) |
| TLS termination | Gateway listener with `protocol: HTTPS` and `certificateRefs` |
| TLS passthrough | Listener `protocol: TLS` + `tls.mode: Passthrough` + `TLSRoute` |
| Path rewrite | `URLRewrite` filter (standard, no annotations) |
| Canary / weighted split | `backendRefs` with `weight` |
| Header-based routing | `matches.headers` |

**Gateway API** is the successor to Ingress. It splits responsibilities by role:

| Resource | Owned by | Purpose |
|---|---|---|
| **GatewayClass** | Infrastructure provider | Which implementation (like IngressClass). |
| **Gateway** | Cluster operator | The actual listener: ports, protocols, hostnames, TLS certs. |
| **HTTPRoute** | App developer | Routing rules: host, path, headers, query → Services, with **traffic weights** for canaries. |
| **GRPCRoute / TLSRoute / TCPRoute / UDPRoute** | App developer | Routing for other protocols (some are experimental). |
| **ReferenceGrant** | Namespace owner | Allows a route to reference a Service in another namespace. |

**Ingress vs Gateway API at a glance**

| | Ingress | Gateway API |
|---|---|---|
| Protocols | HTTP/HTTPS | HTTP, HTTPS, gRPC, TLS, TCP, UDP |
| Traffic splitting | Annotations (controller-specific) | Built in (`weight`) |
| Header/method matching | Annotations | Built in |
| Role separation | One object for everything | Infra / operator / developer split |
| Portability | Low (annotations differ) | High (standard spec) |
| Status | Frozen | Actively developed, recommended |

**Gateway API implementations:** Envoy Gateway · Istio · Cilium · NGINX Gateway Fabric · Traefik · Kong · Contour · cloud implementations (GKE Gateway, AKS Application Gateway for Containers, AWS Gateway API Controller). The `ingress2gateway` tool helps translate existing Ingress objects.

---

## Networking and Network Policies

**The Kubernetes network model**
- Every pod gets its own IP.
- Pods can reach all other pods without NAT, across nodes.
- The **CNI plugin** implements this.

**Traffic types**

| Direction | Handled by |
|---|---|
| Pod ↔ Pod | CNI |
| Pod → Service | kube-proxy (iptables/IPVS) or eBPF (Cilium) |
| Outside → Service | NodePort / LoadBalancer / Ingress / Gateway |
| Pod → Outside | Node SNAT or egress gateway |

**DNS names:** `<svc>.<ns>.svc.cluster.local` · short form `<svc>` within the same namespace, `<svc>.<ns>` across namespaces.

**NetworkPolicy** is a firewall for pods.
- By default **all traffic is allowed**.
- Once any policy selects a pod, only traffic explicitly allowed for that direction is permitted.
- `policyTypes`: `Ingress` (incoming), `Egress` (outgoing).
- Selectors: `podSelector`, `namespaceSelector`, `ipBlock`.
- **Requires a CNI that enforces it** (Calico, Cilium, Azure NPM…). Flannel alone ignores NetworkPolicies.
- Common pattern: default-deny all, then allow specific flows. Remember to allow DNS egress (port 53) when you deny egress.

**Popular CNIs:** Calico (policies, BGP) · Cilium (eBPF, observability, can replace kube-proxy) · Flannel (simple overlay) · Azure CNI / AWS VPC CNI (pods get VNet/VPC IPs).

---

## Configuration: ConfigMaps and Secrets

| Object | Holds | Notes |
|---|---|---|
| **ConfigMap** | Non-sensitive config (key/values or whole files) | Max 1 MiB. Can be `immutable: true`. |
| **Secret** | Sensitive data | Base64-encoded, **not encrypted** by default. Enable encryption at rest and limit RBAC. |

**Ways to consume them in a pod**

| Method | Updates live when changed? |
|---|---|
| Env var from one key (`valueFrom`) | ❌ Needs a pod restart |
| All keys as env vars (`envFrom`) | ❌ Needs a pod restart |
| Mounted as a volume (files) | ✅ Eventually (not with `subPath`) |

**Secret types**

| Type | Use |
|---|---|
| `Opaque` | Generic key/values (default). |
| `kubernetes.io/tls` | TLS cert and key (`tls.crt`, `tls.key`) for Ingress/Gateway. |
| `kubernetes.io/dockerconfigjson` | Private registry credentials (`imagePullSecrets`). |
| `kubernetes.io/basic-auth` | Username and password. |
| `kubernetes.io/ssh-auth` | SSH private key. |
| `kubernetes.io/service-account-token` | Long-lived SA token (legacy; prefer `kubectl create token` / projected tokens). |
| `bootstrap.kubernetes.io/token` | Node bootstrap tokens (kubeadm join). |

**External secret managers:** External Secrets Operator · Secrets Store CSI Driver (Azure Key Vault, AWS Secrets Manager, Vault) · Sealed Secrets (encrypted secrets safe to commit to Git).

---

## Volume and Storage Types

Container filesystems are temporary. Volumes give data a life beyond a single container.

**Volume types (in the pod spec)**

| Type | Lifetime | Use |
|---|---|---|
| `emptyDir` | Pod's lifetime | Scratch space, sharing files between containers. `medium: Memory` for tmpfs. |
| `hostPath` | Node's lifetime | Mount a node folder. Avoid in apps (security risk); used by system DaemonSets. |
| `configMap` / `secret` | Object's lifetime | Mount config or secrets as files. |
| `projected` | — | Combine configMap, secret, downwardAPI and SA tokens in one mount. |
| `downwardAPI` | — | Expose pod info (name, labels, limits) as files. |
| `persistentVolumeClaim` | Independent of the pod | Durable storage via a PVC. |
| `ephemeral` | Pod's lifetime | Per-pod volume created from a StorageClass, deleted with the pod. |
| `csi` | Depends on driver | Inline CSI volumes (e.g. Secrets Store CSI). |
| `nfs` | External | Mount an NFS share directly. |

**Persistent storage objects**

| Object | Role |
|---|---|
| **PersistentVolume (PV)** | A piece of actual storage (disk, file share). Cluster-scoped. |
| **PersistentVolumeClaim (PVC)** | A request for storage by size and access mode. Namespaced. Binds to a PV. |
| **StorageClass (SC)** | Template for **dynamic provisioning**: creates PVs on demand when a PVC asks. |
| **CSI driver** | Plugin that connects Kubernetes to a storage system (Azure Disk, EBS, Ceph…). |
| **VolumeSnapshot** | Point-in-time snapshot of a PVC (needs CSI snapshot support). |

**Provisioning:** *static* (admin creates PVs first) vs *dynamic* (StorageClass creates them automatically, the norm on cloud).

**Access modes**

| Mode | Short | Meaning |
|---|---|---|
| ReadWriteOnce | `RWO` | Read-write by pods on **one node**. Typical for block disks. |
| ReadOnlyMany | `ROX` | Read-only by many nodes. |
| ReadWriteMany | `RWX` | Read-write by many nodes. Needs file storage (NFS, Azure Files, EFS). |
| ReadWriteOncePod | `RWOP` | Read-write by **one pod** only. |

**Reclaim policy** (what happens to the PV when the PVC is deleted): `Delete` (default for dynamic, data gone) · `Retain` (keep data, manual cleanup).

**volumeBindingMode:** `Immediate` (create the volume now) · `WaitForFirstConsumer` (wait until a pod is scheduled, so the disk is created in the right zone).

**volumeMode:** `Filesystem` (default, mounted as a folder) · `Block` (raw block device).

---

## Resources and QoS Classes

| Field | Meaning |
|---|---|
| **requests** | What the scheduler reserves for the container. Used to decide placement. |
| **limits** | The hard ceiling. Exceed **memory** → container is **OOMKilled**. Exceed **CPU** → container is **throttled** (not killed). |

**Units:** CPU `1` = 1 vCPU, `500m` = half a core · Memory `Mi`/`Gi` (binary) vs `M`/`G` (decimal).

**QoS classes** (decide who gets evicted first when a node runs out of memory)

| Class | Condition | Eviction order |
|---|---|---|
| **Guaranteed** | Every container has requests = limits for CPU and memory | Last |
| **Burstable** | At least one request or limit set, but not Guaranteed | Middle |
| **BestEffort** | No requests or limits at all | First |

**Namespace guardrails:** **ResourceQuota** caps total usage in a namespace · **LimitRange** sets default and min/max requests/limits per container.

---

## Scheduling Controls

Ways to influence **where** pods run.

| Mechanism | Who it applies to | Effect |
|---|---|---|
| **nodeSelector** | Pod | Run only on nodes with these labels. Simple, hard requirement. |
| **Node affinity** | Pod | Like nodeSelector but expressive. `required...` (hard) or `preferred...` (soft). |
| **Pod affinity** | Pod | Run **near** pods with certain labels (same node/zone), e.g. app next to its cache. |
| **Pod anti-affinity** | Pod | Run **away** from certain pods, e.g. spread replicas across nodes. |
| **Taints** | Node | Repel pods. Effects: `NoSchedule`, `PreferNoSchedule`, `NoExecute` (also evicts running pods). |
| **Tolerations** | Pod | Allow a pod onto a tainted node. It doesn't force it there; combine with affinity for dedicated nodes. |
| **Topology spread constraints** | Pod | Spread replicas evenly across zones or nodes (`maxSkew`). |
| **PriorityClass** | Pod | Higher-priority pods schedule first and can **preempt** (evict) lower ones. |
| **nodeName** | Pod | Skip the scheduler and pin to a specific node. Mostly for debugging. |

**Taints vs affinity in one line:** taints keep pods **out**; affinity pulls pods **in**.

---

## Autoscaling Types

| Autoscaler | Scales | Based on |
|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Number of **pods** | CPU, memory, or custom/external metrics. Needs metrics-server. |
| **VPA** (Vertical Pod Autoscaler) | **Requests/limits** of pods | Historical usage. Add-on; don't combine with HPA on the same CPU/memory metric. |
| **Cluster Autoscaler** | Number of **nodes** | Pending pods that can't fit / underused nodes. |
| **Karpenter** | **Nodes** (provisions right-sized ones) | Pending pods. Popular on AWS, also available for AKS (Node Auto Provisioning). |
| **KEDA** | **Pods**, including to zero | Event sources (queue length, Kafka lag, cron, HTTP). |

---

## Availability and Disruptions

| Concept | Meaning |
|---|---|
| **Voluntary disruption** | Planned: drain, upgrade, scale-down, deleting a pod. |
| **Involuntary disruption** | Unplanned: node crash, kernel panic, out of resources. |
| **PodDisruptionBudget (PDB)** | Limits how many pods of an app can be down during **voluntary** disruptions (`minAvailable` or `maxUnavailable`). `kubectl drain` respects it. |
| **Replicas + anti-affinity / topology spread** | Protect against **involuntary** disruptions by not putting all pods on one node or zone. |
| **Graceful shutdown** | On delete, the pod gets `SIGTERM`, runs any `preStop` hook, then `SIGKILL` after `terminationGracePeriodSeconds` (default 30s). |

---

## Security and Access

**How a request is checked:** Authentication (who are you?) → Authorization (are you allowed?) → Admission control (is this object acceptable?).

**Identities**

| Type | What it is |
|---|---|
| **User** | A human. Kubernetes has **no User object**; users come from certificates, OIDC (Entra ID, Google) or cloud IAM. |
| **Group** | A set of users, also from the identity provider. |
| **ServiceAccount** | An identity for **pods/processes**. A real namespaced object. Tokens are short-lived and auto-mounted. |

**RBAC objects**

| Object | Scope | Purpose |
|---|---|---|
| **Role** | One namespace | Set of permissions (verbs on resources). |
| **ClusterRole** | Cluster-wide | Permissions for cluster resources (nodes, PVs) or reusable across namespaces. |
| **RoleBinding** | One namespace | Grants a Role **or ClusterRole** to subjects, within that namespace. |
| **ClusterRoleBinding** | Cluster-wide | Grants a ClusterRole across the whole cluster. |

RBAC is **additive only**: there are no "deny" rules. **Verbs:** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`, plus special ones like `impersonate`, `bind`, `escalate`.

**Pod Security Standards** (enforced per namespace by Pod Security Admission via labels like `pod-security.kubernetes.io/enforce=restricted`)

| Level | Meaning |
|---|---|
| **Privileged** | No restrictions. For system components. |
| **Baseline** | Blocks known privilege escalations (no privileged containers, no hostPath, no host networking). |
| **Restricted** | Hardened: must run as non-root, drop all capabilities, use seccomp. |

**Modes:** `enforce` (reject) · `audit` (log) · `warn` (warn the user).

**securityContext highlights:** `runAsNonRoot`, `runAsUser`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`, `seccompProfile`.

**Policy engines** for custom rules: Kyverno · OPA Gatekeeper · built-in ValidatingAdmissionPolicy (CEL expressions).

---

## Organizing: Namespaces, Labels, Annotations

| Concept | Purpose | Example |
|---|---|---|
| **Namespace** | Virtual cluster for isolation, quotas and RBAC scope | `dev`, `prod`, `team-a` |
| **Label** | Key/value used to **select and group** objects | `app=web`, `env=prod` |
| **Selector** | Query on labels (Services, Deployments, kubectl `-l` use them) | `app=web,tier!=db` |
| **Annotation** | Key/value for **non-identifying metadata**; tools read it, selectors don't | `kubernetes.io/change-cause`, ingress controller settings |
| **Field selector** | Query on object fields, not labels | `status.phase=Running` |
| **Finalizer** | Blocks deletion until a controller finishes cleanup | Why namespaces/PVCs get stuck in `Terminating` |
| **Owner reference** | Links child to parent for cascading deletion | Pod → ReplicaSet |

**Default namespaces:** `default` · `kube-system` (cluster components) · `kube-public` (readable by all) · `kube-node-lease` (node heartbeats).

**Cluster-scoped (not in any namespace):** Nodes, PersistentVolumes, StorageClasses, Namespaces, ClusterRoles, CRDs, IngressClasses, PriorityClasses. Check with `kubectl api-resources --namespaced=false`.

**Recommended labels:** `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`, `app.kubernetes.io/managed-by`.

---

## Extending Kubernetes

| Concept | What it is |
|---|---|
| **CustomResourceDefinition (CRD)** | Adds a new object type to the API (e.g. `Certificate`, `Gateway`, `Prometheus`). |
| **Custom Resource (CR)** | An instance of a CRD. |
| **Controller** | A loop that watches objects and acts on them. |
| **Operator** | A CRD plus a controller that encodes operational knowledge (e.g. a Postgres operator handles backups and failover). |
| **Admission webhooks** | **Mutating** (change objects on create, e.g. inject sidecars) and **Validating** (accept/reject). |
| **Aggregated API** | Extra API servers behind the main one (e.g. metrics-server serves `metrics.k8s.io`). |

**Ecosystem worth knowing:** Helm (packaging) · Kustomize (overlays) · Argo CD / Flux (GitOps) · cert-manager (TLS certs) · Prometheus + Grafana (monitoring) · Istio / Linkerd (service mesh) · Velero (backup).

---

# Part 2 — Commands

## 1. Setup, Aliases & Autocompletion

Check your kubectl version and set up shortcuts that save a lot of typing.

```bash
# Version
kubectl version                          # Show client and server versions
kubectl version --client                 # Show only the local kubectl version
kubectl version -o yaml                  # Version details in YAML

# Bash completion
source <(kubectl completion bash)                          # Enable completion in the current shell
echo 'source <(kubectl completion bash)' >> ~/.bashrc      # Enable completion permanently

# Zsh completion
source <(kubectl completion zsh)                           # Enable completion in the current shell
echo 'source <(kubectl completion zsh)' >> ~/.zshrc        # Enable completion permanently

# PowerShell completion
kubectl completion powershell | Out-String | Invoke-Expression   # Enable completion in PowerShell

# Alias + completion for the alias
alias k=kubectl                                  # Short alias for kubectl
complete -o default -F __start_kubectl k         # Make completion work with the alias

# Handy exports
export do="--dry-run=client -o yaml"     # Usage: k create deploy x --image=nginx $do
export now="--force --grace-period=0"    # Usage: k delete pod x $now
```

**Popular aliases**

```bash
alias kg='kubectl get'                    # List resources
alias kd='kubectl describe'               # Describe a resource
alias kdel='kubectl delete'               # Delete a resource
alias kl='kubectl logs'                   # Show logs
alias kex='kubectl exec -it'              # Interactive exec into a pod
alias kaf='kubectl apply -f'              # Apply a manifest
alias kgp='kubectl get pods'              # List pods
alias kgpa='kubectl get pods -A'          # List pods in all namespaces
alias kgs='kubectl get svc'               # List services
alias kgn='kubectl get nodes -o wide'     # List nodes with details
alias kns='kubectl config set-context --current --namespace'   # Switch default namespace
```

---

## 2. Local Cluster with Minikube

Minikube runs a local Kubernetes cluster on your machine, which is ideal for learning and testing.

```bash
# Install check
minikube version                                  # Show the installed minikube version

# Create / start a cluster
minikube start                                    # Start a cluster with default settings
minikube start --driver=docker                    # Use Docker as the driver (also: hyperv, virtualbox, kvm2, podman)
minikube start --cpus=4 --memory=8192             # Set CPU count and memory (MB)
minikube start --disk-size=40g                    # Set the VM/container disk size
minikube start --kubernetes-version=v1.31.0       # Run a specific Kubernetes version
minikube start --nodes=3                          # Create a multi-node cluster
minikube start -p dev                             # Create a separate cluster called "dev" (profile)
minikube start --container-runtime=containerd     # Choose the runtime (docker, containerd, cri-o)
minikube start --addons=ingress,metrics-server    # Enable addons at startup

# Cluster lifecycle
minikube status                                   # Show the state of the cluster components
minikube stop                                     # Stop the cluster but keep its state
minikube pause                                    # Pause Kubernetes without stopping the VM
minikube unpause                                  # Resume a paused cluster
minikube delete                                   # Delete the current cluster
minikube delete -p dev                            # Delete a specific profile
minikube delete --all                             # Delete every minikube cluster
minikube delete --all --purge                     # Delete everything, including ~/.minikube

# Profiles (multiple clusters)
minikube profile list                             # List all minikube clusters
minikube profile dev                              # Switch the active profile to "dev"

# Nodes
minikube node list                                # List nodes in the cluster
minikube node add                                 # Add a worker node
minikube node delete <node>                       # Remove a node

# Access & networking
minikube ip                                       # Show the cluster IP address
minikube ssh                                      # SSH into the minikube node
minikube ssh -n <node>                            # SSH into a specific node
minikube service <svc>                            # Open a service in the browser
minikube service <svc> --url                      # Print the service URL instead of opening it
minikube service list                             # List all services and their URLs
minikube tunnel                                   # Expose LoadBalancer services (run in a separate terminal)
minikube dashboard                                # Open the Kubernetes dashboard
minikube dashboard --url                          # Print the dashboard URL only

# Addons
minikube addons list                              # Show all addons and whether they are enabled
minikube addons enable ingress                    # Enable the NGINX ingress controller
minikube addons enable metrics-server             # Enable metrics (needed for kubectl top / HPA)
minikube addons enable dashboard                  # Enable the web dashboard
minikube addons enable registry                   # Enable a local image registry
minikube addons disable <addon>                   # Disable an addon
minikube addons open <addon>                      # Open an addon's UI in the browser

# Images
minikube image ls                                 # List images inside the cluster
minikube image load myapp:1.0                     # Copy a local image into the cluster
minikube image build -t myapp:1.0 .               # Build an image directly inside the cluster
minikube image rm myapp:1.0                       # Remove an image from the cluster
eval $(minikube docker-env)                       # Point your Docker CLI at minikube's Docker daemon
eval $(minikube docker-env -u)                    # Point Docker back to your local daemon
minikube docker-env --shell powershell | Invoke-Expression   # Same as above, for PowerShell

# Files & kubectl
minikube mount ./data:/data                       # Mount a local folder into the node
minikube cp ./file.txt /home/docker/file.txt      # Copy a file into the node
minikube kubectl -- get pods                      # Use minikube's bundled kubectl
minikube update-context                           # Fix kubeconfig if the cluster IP changed

# Config & troubleshooting
minikube config set driver docker                 # Set the default driver
minikube config set memory 8192                   # Set default memory for new clusters
minikube config set cpus 4                        # Set default CPUs for new clusters
minikube config view                              # Show saved minikube settings
minikube logs                                     # Show cluster logs
minikube logs -f                                  # Follow cluster logs
minikube logs --file=logs.txt                     # Save logs to a file (useful for bug reports)
```

> 💡 With the Docker driver on Windows/macOS, `NodePort` services are not reachable on `minikube ip`. Use `minikube service <svc> --url` or `kubectl port-forward` instead.

---

## 3. Kubeconfig, Contexts & Clusters

Kubeconfig tells kubectl which cluster to talk to and as which user. A context ties a cluster, user and namespace together.

```bash
kubectl config view                                  # Show the merged kubeconfig
kubectl config view --minify                         # Show only the current context
kubectl config view --raw                            # Show including certs and tokens
kubectl config view -o jsonpath='{.users[*].name}'   # List user names from kubeconfig

kubectl config get-contexts                          # List all contexts
kubectl config current-context                       # Show the active context
kubectl config use-context <context>                 # Switch to another context
kubectl config rename-context <old> <new>            # Rename a context
kubectl config delete-context <context>              # Remove a context

kubectl config get-clusters                          # List clusters in kubeconfig
kubectl config get-users                             # List users in kubeconfig
kubectl config delete-cluster <cluster>              # Remove a cluster entry
kubectl config delete-user <user>                    # Remove a user entry

kubectl config set-context --current --namespace=<ns>   # Change the default namespace

# Build a context manually
kubectl config set-cluster <cluster> --server=https://<ip>:6443 --certificate-authority=ca.crt --embed-certs   # Add a cluster
kubectl config set-credentials <user> --client-certificate=user.crt --client-key=user.key --embed-certs      # Add a cert-based user
kubectl config set-credentials <user> --token=<token>                                                         # Add a token-based user
kubectl config set-context <ctx> --cluster=<cluster> --user=<user> --namespace=<ns>                          # Create a context

kubectl config unset users.<user>                    # Remove a specific kubeconfig value

# Use multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/cluster2.yaml       # Merge several kubeconfig files
kubectl config view --flatten > ~/.kube/merged.yaml          # Save merged config into one file

# Per-command overrides
kubectl get pods --kubeconfig=./other.yaml           # Use a different kubeconfig file once
kubectl get pods --context=<ctx>                     # Use a different context once

# Cloud provider kubeconfig
az aks get-credentials -g <rg> -n <cluster>                          # Get AKS credentials
az aks get-credentials -g <rg> -n <cluster> --admin                  # Get AKS admin credentials
aws eks update-kubeconfig --region <region> --name <cluster>         # Get EKS credentials
gcloud container clusters get-credentials <cluster> --region <region> # Get GKE credentials
```

---

## 4. Cluster Information

Inspect the cluster itself: API endpoints, health, available APIs and resource documentation.

```bash
kubectl cluster-info                                 # Show control plane and DNS addresses
kubectl cluster-info dump                            # Dump full cluster state for debugging
kubectl cluster-info dump --output-directory=./dump  # Save the dump to a folder

kubectl get nodes                                    # List nodes and their status
kubectl get nodes -o wide                            # Nodes with IPs, OS, kernel, runtime
kubectl get --raw='/readyz?verbose'                  # API server readiness checks
kubectl get --raw='/livez?verbose'                   # API server liveness checks
kubectl get --raw /metrics | head                    # Raw API server metrics
kubectl get componentstatuses                        # Control-plane component health (deprecated)

kubectl api-versions                                 # List all API group versions
kubectl api-resources                                # List all resource kinds
kubectl api-resources --namespaced=true              # Only namespaced resources
kubectl api-resources --namespaced=false             # Only cluster-scoped resources
kubectl api-resources --api-group=apps               # Resources in a specific API group
kubectl api-resources -o wide                        # Include supported verbs
kubectl api-resources --verbs=list,get               # Resources that support these verbs

kubectl explain pod                                  # Built-in docs for a resource
kubectl explain pod.spec.containers                  # Docs for a nested field
kubectl explain deployment.spec.strategy --recursive # Show all nested fields
kubectl explain pods --api-version=v1                # Docs for a specific API version
```

---

## 5. Resource Types & Short Names

Short names let you type `po` instead of `pods`, `svc` instead of `services`, and so on.

| Resource | Short | Resource | Short |
|---|---|---|---|
| pods | `po` | services | `svc` |
| deployments | `deploy` | replicasets | `rs` |
| statefulsets | `sts` | daemonsets | `ds` |
| namespaces | `ns` | nodes | `no` |
| configmaps | `cm` | secrets | — |
| persistentvolumes | `pv` | persistentvolumeclaims | `pvc` |
| storageclasses | `sc` | serviceaccounts | `sa` |
| ingresses | `ing` | ingressclasses | — |
| networkpolicies | `netpol` | endpoints | `ep` |
| endpointslices | — | events | `ev` |
| horizontalpodautoscalers | `hpa` | poddisruptionbudgets | `pdb` |
| cronjobs | `cj` | jobs | — |
| replicationcontrollers | `rc` | resourcequotas | `quota` |
| limitranges | `limits` | customresourcedefinitions | `crd`, `crds` |
| priorityclasses | `pc` | certificatesigningrequests | `csr` |
| roles / rolebindings | — | clusterroles / clusterrolebindings | — |

```bash
kubectl get all                     # Common workload resources (not cm/secrets/pvc/ingress)
kubectl get all -A                  # Same, across all namespaces
kubectl get cm,secret,pvc,ing -n <ns>   # Resources that "get all" leaves out
```

---

## 6. Viewing Resources (get / describe)

`get` lists resources in a table; `describe` shows full details and recent events.

```bash
kubectl get pods                                     # List pods in the current namespace
kubectl get pods -A                                  # List pods in all namespaces
kubectl get pods -n <ns>                             # List pods in a specific namespace
kubectl get pods -o wide                             # Include node, pod IP and more
kubectl get pods --show-labels                       # Show all labels
kubectl get pods -L app,tier                         # Show specific labels as columns
kubectl get pods -l app=web                          # Filter by label
kubectl get pods -l 'env in (prod,staging)'          # Filter by a set of label values
kubectl get pods -l 'app,!canary'                    # Has "app", does not have "canary"
kubectl get pods --field-selector status.phase=Running        # Filter by a field value
kubectl get pods --field-selector spec.nodeName=<node>        # Pods on a specific node
kubectl get pods --field-selector metadata.namespace!=default # Exclude a namespace
kubectl get pods --sort-by=.metadata.creationTimestamp        # Sort by age
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'   # Sort by restarts
kubectl get pods -w                                  # Watch for changes live
kubectl get pods --watch-only                        # Watch changes without initial list
kubectl get pods --no-headers                        # Omit the header row
kubectl get pods <pod1> <pod2>                       # Get specific pods
kubectl get deploy,svc,pods                          # Several kinds at once
kubectl get pod <pod> -o yaml                        # Full object as YAML
kubectl get pod <pod> -o yaml --show-managed-fields=false   # YAML without managedFields noise
kubectl get pods --chunk-size=500                    # Page through very large lists
kubectl get pods --ignore-not-found                  # Do not error if missing
kubectl get -f manifest.yaml                         # Get resources defined in a file
kubectl get -k ./overlay                             # Get resources from a kustomization

kubectl describe pod <pod>                           # Details and events for a pod
kubectl describe pods -l app=web                     # Describe all matching pods
kubectl describe node <node>                         # Node capacity, conditions, pods
kubectl describe deploy <deploy>                     # Deployment strategy, replicas, events
kubectl describe -f manifest.yaml                    # Describe resources in a file
kubectl describe pod <pod> --show-events=false       # Describe without events
```

---

## 7. Output Formats, JSONPath & Custom Columns

Control the output with `-o` and extract exactly the fields you need.

| Flag | Description |
|---|---|
| `-o wide` | Extra columns |
| `-o yaml` / `-o json` | Full object |
| `-o name` | `kind/name` only (great for piping) |
| `-o jsonpath='{...}'` | Extract fields |
| `-o jsonpath-file=file` | JSONPath from file |
| `-o custom-columns=...` | Build your own table |
| `-o custom-columns-file=file` | Columns from file |
| `-o go-template='...'` | Go template |

```bash
# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'          # All pod names
kubectl get pod <pod> -o jsonpath='{.status.podIP}'                # A pod's IP
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'   # Node internal IPs
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'   # Pod → node, one per line
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u        # Unique images
kubectl get secret <s> -o jsonpath='{.data.password}' | base64 -d                                  # Decode a secret value
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'                # Only running pods

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP        # Name, node, IP table
kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGES:.spec.containers[*].image'         # Pod and its images
kubectl get nodes -o custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory   # Node capacity

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'   # Pod names with Go template

# With jq
kubectl get pods -o json | jq -r '.items[] | select(.status.phase!="Running") | .metadata.name'   # Non-running pods
```

---

## 8. Creating Resources (Imperative)

Create resources directly from the command line without writing YAML first.

```bash
# Pods
kubectl run nginx --image=nginx                              # Run a single pod
kubectl run nginx --image=nginx --port=80 --labels=app=web,tier=fe   # Pod with port and labels
kubectl run nginx --image=nginx --env=KEY=value --env=FOO=bar        # Pod with env variables
kubectl run nginx --image=nginx --restart=Never              # Plain pod, no restarts
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh   # Throwaway shell, deleted on exit
kubectl run nginx --image=nginx --expose --port=80           # Pod plus a ClusterIP service
kubectl run nginx --image=nginx --command -- sleep 3600      # Override the container command
kubectl run nginx --image=nginx -- arg1 arg2                 # Override the container args
kubectl run nginx --image=nginx --overrides='{"spec":{"nodeName":"node1"}}'   # Inject raw JSON into the spec

# Deployments
kubectl create deployment web --image=nginx                  # Create a deployment
kubectl create deployment web --image=nginx --replicas=3 --port=80   # With replicas and port
kubectl create deployment web --image=nginx --image=redis    # Multi-container deployment

# Services
kubectl expose deployment web --port=80 --target-port=8080   # ClusterIP service for a deployment
kubectl expose deployment web --type=NodePort --port=80      # Expose on every node's port
kubectl expose deployment web --type=LoadBalancer --port=80 --name=web-lb   # Cloud load balancer
kubectl expose pod nginx --port=80 --name=nginx-svc          # Service for a single pod
kubectl create service clusterip my-svc --tcp=80:8080        # Standalone ClusterIP service
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30080   # NodePort with fixed port
kubectl create service loadbalancer my-svc --tcp=80:8080     # LoadBalancer service
kubectl create service externalname my-svc --external-name=example.com   # DNS alias to external host

# Other kinds
kubectl create namespace dev                                 # Create a namespace
kubectl create configmap app-cfg --from-literal=KEY=value    # Create a ConfigMap
kubectl create secret generic db --from-literal=password=s3cr3t   # Create a Secret
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'   # Run a one-off job
kubectl create job manual-run --from=cronjob/<cj>            # Run a CronJob immediately
kubectl create cronjob backup --image=busybox --schedule="0 2 * * *" -- /bin/sh -c 'date'   # Scheduled job
kubectl create serviceaccount app-sa                         # Create a service account
kubectl create quota team-q --hard=cpu=4,memory=8Gi,pods=20  # Limit namespace resources
kubectl create priorityclass high --value=1000 --description="high prio"   # Pod scheduling priority
kubectl create poddisruptionbudget web-pdb --selector=app=web --min-available=2   # Protect during drains
kubectl create ingress web --class=<class> --rule="example.com/*=web:80"          # Single host, all paths → web
kubectl create ingress single --class=<class> --default-backend=web:80             # Default backend: everything → web
kubectl create ingress fanout --class=<class> \
  --rule="shop.com/api*=api:8080" --rule="shop.com/images*=images:80" \
  --rule="shop.com/*=web:80"                                                       # Simple fanout: paths → services
kubectl create ingress vhosts --class=<class> \
  --rule="api.shop.com/*=api:8080" --rule="blog.shop.com/*=blog:80"                # Name-based virtual hosting
kubectl create ingress secure --class=<class> \
  --rule="api.shop.com/*=api:8080,tls=shop-tls"                                    # TLS termination with a Secret
kubectl create ingress exact --class=<class> --rule="shop.com/health=web:80"       # No trailing * = Exact path match
kubectl create ingress web --class=<class> --rule="shop.com/*=web:80" \
  --annotation=<controller-annotation>=<value>                                     # Add controller-specific annotations
kubectl create token <sa> --duration=1h                      # Short-lived token for a service account

# From files
kubectl create -f manifest.yaml                              # Create from a file
kubectl create -f ./dir/                                     # Create from all files in a folder
kubectl create -f https://example.com/manifest.yaml          # Create from a URL
kubectl create -f - <<EOF                                    # Create from inline YAML (heredoc)
apiVersion: v1
kind: Namespace
metadata:
  name: demo
EOF
```

---

## 9. Declarative Management (apply / diff)

Describe the desired state in files and let Kubernetes reconcile it. This is the recommended approach for real projects.

```bash
kubectl apply -f manifest.yaml                   # Create or update from a file
kubectl apply -f ./dir/                          # Apply every file in a folder
kubectl apply -f ./dir/ -R                       # Apply folder recursively
kubectl apply -f https://url/manifest.yaml       # Apply from a URL
kubectl apply -k ./overlay                       # Apply a kustomization
cat x.yaml | kubectl apply -f -                  # Apply from stdin

kubectl apply -f x.yaml --dry-run=client         # Validate locally, change nothing
kubectl apply -f x.yaml --dry-run=server         # Validate on the API server, change nothing
kubectl apply -f x.yaml --server-side            # Server-side apply (field ownership tracking)
kubectl apply -f x.yaml --server-side --force-conflicts   # Take ownership of conflicting fields
kubectl apply -f x.yaml --field-manager=my-tool  # Name the field manager
kubectl apply -f ./dir --prune -l app=web        # Delete objects removed from the files (use with care)
kubectl apply -f x.yaml --validate=strict        # Fail on unknown or duplicate fields

kubectl diff -f manifest.yaml                    # Preview changes before applying
kubectl diff -k ./overlay                        # Preview kustomization changes

kubectl replace -f manifest.yaml                 # Replace the whole object
kubectl replace --force -f manifest.yaml         # Delete and recreate the object

# Last-applied configuration
kubectl apply view-last-applied deploy/<name>    # Show what was last applied
kubectl apply set-last-applied -f manifest.yaml --create-annotation   # Set the last-applied annotation
kubectl apply edit-last-applied deploy/<name>    # Edit the last-applied annotation
```

---

## 10. Generating YAML (dry-run)

The fastest way to write manifests: let kubectl generate a template, then edit it.

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml                          # Pod template
kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml   # Deployment template
kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml                  # Service template
kubectl create configmap cfg --from-file=app.conf --dry-run=client -o yaml > cm.yaml         # ConfigMap template
kubectl create secret generic s --from-literal=k=v --dry-run=client -o yaml > secret.yaml    # Secret template
kubectl create job j --image=busybox --dry-run=client -o yaml -- echo hi > job.yaml          # Job template
kubectl create cronjob cj --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- date   # CronJob template
kubectl create role r --verb=get,list --resource=pods --dry-run=client -o yaml               # Role template
kubectl create ingress ing --rule="a.com/*=svc:80" --dry-run=client -o yaml                  # Ingress template

kubectl get deploy web -o yaml --show-managed-fields=false > web.yaml   # Export a live object to YAML
```

---

## 11. Updating Resources (edit / patch / set)

Change running resources: open them in an editor, patch specific fields, or use `set` shortcuts.

```bash
# Edit in $EDITOR
kubectl edit deploy/<name>                       # Open a resource in your editor
KUBE_EDITOR="nano" kubectl edit svc/<name>       # Use a specific editor
kubectl edit deploy/<name> -o json               # Edit as JSON instead of YAML

# set — targeted updates
kubectl set image deploy/web nginx=nginx:1.27            # Change one container's image
kubectl set image deploy/web *=nginx:1.27                # Change the image of all containers
kubectl set env deploy/web KEY=value                     # Add or update an env variable
kubectl set env deploy/web KEY-                          # Remove an env variable
kubectl set env deploy/web --from=configmap/app-cfg      # Import all keys from a ConfigMap
kubectl set env deploy/web --from=secret/db --prefix=DB_ # Import Secret keys with a prefix
kubectl set env deploy/web --list                        # List current env variables
kubectl set resources deploy/web -c=nginx --limits=cpu=500m,memory=512Mi --requests=cpu=100m,memory=128Mi   # Set requests/limits
kubectl set serviceaccount deploy/web app-sa             # Change the service account
kubectl set selector svc/web app=web-v2                  # Change a service's selector
kubectl set subject rolebinding/rb --user=alice --serviceaccount=ns:sa   # Add subjects to a binding

# patch — strategic merge (default)
kubectl patch deploy web -p '{"spec":{"replicas":5}}'                    # Patch a field
kubectl patch node <node> -p '{"spec":{"unschedulable":true}}'           # Mark node unschedulable
# JSON merge patch
kubectl patch svc web --type=merge -p '{"spec":{"type":"NodePort"}}'     # Merge-patch a field
# JSON patch (RFC 6902)
kubectl patch pod <pod> --type=json -p='[{"op":"replace","path":"/spec/containers/0/image","value":"nginx:1.27"}]'   # Replace by path
kubectl patch deploy web --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/env/0"}]'           # Remove by path
# Patch from file / subresource
kubectl patch deploy web --patch-file=patch.yaml                         # Patch from a file
kubectl patch deploy web --subresource=scale -p '{"spec":{"replicas":2}}'   # Patch the scale subresource
# Remove finalizers from a stuck object
kubectl patch <kind>/<name> -p '{"metadata":{"finalizers":null}}' --type=merge   # Unblock stuck deletion
```

---

## 12. Deleting Resources

Remove resources by name, file, label or field. Force options skip graceful shutdown, so use them carefully.

```bash
kubectl delete pod <pod>                                 # Delete a pod
kubectl delete pod <pod1> <pod2>                         # Delete several pods
kubectl delete -f manifest.yaml                          # Delete resources in a file
kubectl delete -k ./overlay                              # Delete a kustomization
kubectl delete pods -l app=web                           # Delete pods by label
kubectl delete pods --all -n <ns>                        # Delete all pods in a namespace
kubectl delete all --all -n <ns>                         # Delete common workloads (not cm/secrets/pvc)
kubectl delete ns <ns>                                   # Delete a namespace and everything in it
kubectl delete pod <pod> --grace-period=0 --force        # Force kill immediately
kubectl delete pod <pod> --now                           # Delete with a 1-second grace period
kubectl delete deploy web --cascade=orphan               # Delete deployment but keep its pods
kubectl delete deploy web --cascade=foreground           # Delete children first, then owner
kubectl delete pod <pod> --wait=false                    # Return without waiting
kubectl delete pods --field-selector=status.phase=Failed -A      # Clean up failed pods
kubectl delete pods --field-selector=status.phase=Succeeded -A   # Clean up completed pods
kubectl delete pod <pod> --dry-run=server                # Simulate the deletion
kubectl delete pods -l app=web -i                        # Ask for confirmation first
```

---

## 13. Pods: Logs, Exec, Copy, Attach

Interact with running containers: read logs, open a shell, and move files in and out.

```bash
# Logs
kubectl logs <pod>                                       # Show pod logs
kubectl logs <pod> -c <container>                        # Logs from a specific container
kubectl logs <pod> --all-containers                      # Logs from all containers
kubectl logs <pod> -f                                    # Stream logs live
kubectl logs <pod> --previous                            # Logs from the previous (crashed) container
kubectl logs <pod> --tail=100                            # Last 100 lines
kubectl logs <pod> --since=1h                            # Logs from the last hour
kubectl logs <pod> --since-time=2026-01-01T10:00:00Z     # Logs since a timestamp
kubectl logs <pod> --timestamps                          # Prefix each line with a timestamp
kubectl logs -l app=web --all-containers --prefix --max-log-requests=10   # Logs from many pods, labelled
kubectl logs deploy/web                                  # Logs from one pod of a deployment
kubectl logs job/<job>                                   # Logs from a job's pod
kubectl logs <pod> --limit-bytes=10000                   # Cap log output size

# Exec
kubectl exec <pod> -- ls /                               # Run a command in a pod
kubectl exec -it <pod> -- sh                             # Open an interactive shell
kubectl exec -it <pod> -c <container> -- bash            # Shell into a specific container
kubectl exec deploy/web -- env                           # Run a command in a deployment's pod
kubectl exec <pod> -- cat /etc/resolv.conf               # Check the pod's DNS config

# Attach
kubectl attach <pod> -it                                 # Attach to the main process
kubectl attach <pod> -c <container>                      # Attach to a specific container

# Copy (requires tar in the container)
kubectl cp <pod>:/path/file ./file                       # Copy from pod to local
kubectl cp ./file <pod>:/path/file                       # Copy from local to pod
kubectl cp <ns>/<pod>:/path ./local -c <container>       # Copy from a container in another namespace
kubectl cp ./dir <pod>:/tmp/dir --retries=3              # Copy a folder with retries
```

---

## 14. Deployments & Rollouts

Manage application updates: watch progress, see history, roll back, and restart.

```bash
kubectl rollout status deploy/web                        # Watch a rollout until it finishes
kubectl rollout status deploy/web --timeout=120s         # Fail if not done in time
kubectl rollout history deploy/web                       # List previous revisions
kubectl rollout history deploy/web --revision=3          # Details of a specific revision
kubectl rollout undo deploy/web                          # Roll back to the previous revision
kubectl rollout undo deploy/web --to-revision=2          # Roll back to a specific revision
kubectl rollout restart deploy/web                       # Restart pods without changing the spec
kubectl rollout restart deploy -n <ns>                   # Restart all deployments in a namespace
kubectl rollout pause deploy/web                         # Pause to batch several changes
kubectl rollout resume deploy/web                        # Resume a paused rollout

# Works for daemonsets and statefulsets too
kubectl rollout restart ds/<ds>                          # Restart a DaemonSet
kubectl rollout status sts/<sts>                         # Watch a StatefulSet rollout

kubectl annotate deploy/web kubernetes.io/change-cause="upgrade to 1.27" --overwrite   # Record why (shows in history)

kubectl get rs -l app=web                                # ReplicaSets behind a deployment
```

---

## 15. Scaling & Autoscaling

Change the number of replicas manually or let the Horizontal Pod Autoscaler do it based on load.

```bash
kubectl scale deploy/web --replicas=5                        # Set replica count
kubectl scale deploy/web --current-replicas=5 --replicas=3   # Scale only if currently at 5
kubectl scale sts/db --replicas=3                            # Scale a StatefulSet
kubectl scale --replicas=0 deploy --all -n <ns>              # Scale every deployment to zero
kubectl scale -f deploy.yaml --replicas=2                    # Scale resources from a file

# HPA (requires metrics-server)
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=70   # Autoscale on CPU usage
kubectl get hpa                                              # List autoscalers
kubectl get hpa -w                                           # Watch autoscaler decisions
kubectl describe hpa web                                     # Current metrics and events
kubectl delete hpa web                                       # Remove the autoscaler
```

---

## 16. Services, Networking & Port Forwarding

Inspect how traffic reaches your pods, and reach them from your own machine for testing.

```bash
kubectl get svc                                          # List services
kubectl get svc -o wide                                  # Services with selectors
kubectl get endpoints <svc>                              # Pod IPs behind a service
kubectl get endpointslices -l kubernetes.io/service-name=<svc>   # EndpointSlices for a service
kubectl get ingress                                      # List ingresses
kubectl describe ingress <ing>                           # Ingress rules and backends
kubectl get ingressclass                                 # Available ingress controllers
kubectl get networkpolicy                                # List network policies
kubectl describe networkpolicy <np>                      # Show policy rules

# Gateway API (once its CRDs and an implementation are installed)
kubectl get gatewayclass                                 # Available Gateway implementations
kubectl get gateway -A                                   # Gateways (listeners) and their addresses
kubectl describe gateway <gw>                            # Listener status and attached routes
kubectl get httproute -A                                 # HTTP routing rules
kubectl describe httproute <route>                       # Check if the route was accepted by the Gateway
kubectl get grpcroute,tlsroute -A                        # Routes for other protocols
kubectl get referencegrant -A                            # Cross-namespace reference permissions
kubectl get crd | grep gateway.networking.k8s.io         # Is Gateway API installed?
kubectl get pods -A -l app.kubernetes.io/name=ingress-nginx   # Are you still running retired ingress-nginx?

# Port forwarding
kubectl port-forward pod/<pod> 8080:80                   # localhost:8080 → pod port 80
kubectl port-forward svc/<svc> 8080:80                   # localhost:8080 → service port 80
kubectl port-forward deploy/<deploy> 8080:80             # Forward to a deployment's pod
kubectl port-forward pod/<pod> 8080:80 9090:9090         # Forward several ports
kubectl port-forward pod/<pod> :80                       # Pick a random local port
kubectl port-forward --address 0.0.0.0 svc/<svc> 8080:80 # Listen on all interfaces

# API proxy
kubectl proxy                                            # Proxy the API to http://127.0.0.1:8001
kubectl proxy --port=8080                                # Proxy on a different port
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods                       # Call the API through the proxy
curl http://127.0.0.1:8001/api/v1/namespaces/<ns>/services/<svc>:<port>/proxy/  # Reach a service through the proxy

# DNS & connectivity tests
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup <svc>.<ns>.svc.cluster.local   # Test DNS
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl -sv http://<svc>.<ns>:80           # Test HTTP
kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash                              # Full network toolbox
```

**DNS format:** `<service>.<namespace>.svc.cluster.local` · Pod: `<pod-ip-dashed>.<namespace>.pod.cluster.local`

---

## 17. ConfigMaps & Secrets

Keep configuration and credentials out of your images and inject them into pods.

```bash
# ConfigMaps
kubectl create configmap cfg --from-literal=A=1 --from-literal=B=2   # From key/value pairs
kubectl create configmap cfg --from-file=app.properties              # From a file (key = file name)
kubectl create configmap cfg --from-file=custom-key=app.properties   # From a file with a custom key
kubectl create configmap cfg --from-file=./config-dir/               # From every file in a folder
kubectl create configmap cfg --from-env-file=.env                    # From a .env file
kubectl get cm cfg -o yaml                                           # Show ConfigMap contents
kubectl describe cm cfg                                              # Describe a ConfigMap

# Secrets
kubectl create secret generic db --from-literal=user=admin --from-literal=pass='p@ss'   # From literals
kubectl create secret generic db --from-file=./username.txt --from-file=./password.txt # From files
kubectl create secret generic db --from-env-file=.env                                  # From a .env file
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key                         # TLS certificate secret
kubectl create secret docker-registry regcred \
  --docker-server=<registry> --docker-username=<u> --docker-password=<p> --docker-email=<e>   # Private registry login
kubectl create secret generic regcred --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson                                                # Registry secret from Docker config

# Read / decode
kubectl get secret db -o jsonpath='{.data.pass}' | base64 -d                           # Decode one key
kubectl get secret db -o go-template='{{range $k,$v := .data}}{{$k}}={{$v|base64decode}}{{"\n"}}{{end}}'   # Decode all keys

kubectl create configmap cfg --from-file=app.conf --dry-run=client -o yaml | kubectl apply -f -   # Update a ConfigMap in place

echo -n 'value' | base64                                 # Encode a value
echo 'dmFsdWU=' | base64 -d                              # Decode a value
```

> ⚠️ Secrets are only **base64-encoded**, not encrypted. Enable encryption at rest and use RBAC / an external secret store.

---

## 18. Storage (PV / PVC / StorageClass)

PersistentVolumes provide storage, PersistentVolumeClaims request it, and StorageClasses provision it dynamically.

```bash
kubectl get pv                                       # List persistent volumes
kubectl get pvc -A                                   # List claims in all namespaces
kubectl get sc                                       # List storage classes
kubectl get volumeattachments                        # Volumes attached to nodes
kubectl get csidrivers                               # Installed CSI storage drivers
kubectl describe pvc <pvc>                           # Claim status and binding events
kubectl describe pv <pv>                             # Volume details and source

kubectl patch sc <sc> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'   # Set default StorageClass
kubectl patch pvc <pvc> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'   # Expand a PVC (if allowed)
kubectl patch pv <pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'       # Keep data after PVC deletion
kubectl patch pv <pv> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'     # Make a Released PV bindable again
```

**Access modes:** `RWO` ReadWriteOnce · `ROX` ReadOnlyMany · `RWX` ReadWriteMany · `RWOP` ReadWriteOncePod
**Reclaim policies:** `Retain` · `Delete` · (`Recycle` deprecated)

---

## 19. Jobs & CronJobs

Jobs run a task to completion; CronJobs run Jobs on a schedule.

```bash
kubectl create job hello --image=busybox -- echo "hello"     # Run a one-off job
kubectl get jobs                                             # List jobs
kubectl describe job hello                                   # Job status and events
kubectl logs job/hello                                       # Job output
kubectl wait --for=condition=complete job/hello --timeout=60s   # Wait until the job finishes
kubectl delete job hello                                     # Delete a job and its pods

kubectl create cronjob nightly --image=busybox --schedule="0 0 * * *" -- echo run   # Create a CronJob
kubectl get cronjobs                                         # List CronJobs and last run time
kubectl create job nightly-manual --from=cronjob/nightly     # Trigger a CronJob now
kubectl patch cronjob nightly -p '{"spec":{"suspend":true}}'    # Pause a CronJob
kubectl patch cronjob nightly -p '{"spec":{"suspend":false}}'   # Resume a CronJob
kubectl get pods -l job-name=hello                           # Pods created by a job
```

**Cron format:** `minute hour day-of-month month day-of-week` → `*/15 * * * *` every 15 min

---

## 20. Labels, Annotations & Selectors

Labels group and select resources; annotations store extra metadata that is not used for selection.

```bash
kubectl label pod <pod> env=prod                         # Add a label
kubectl label pod <pod> env=staging --overwrite          # Change an existing label
kubectl label pod <pod> env-                             # Remove a label
kubectl label pods --all team=platform                   # Label every pod
kubectl label pods -l app=web tier=frontend              # Label pods matching a selector
kubectl label node <node> disktype=ssd                   # Label a node (for scheduling)

kubectl annotate pod <pod> owner="omar"                  # Add an annotation
kubectl annotate pod <pod> owner-                        # Remove an annotation
kubectl annotate deploy web description="frontend" --overwrite   # Change an annotation

# Selectors
kubectl get pods -l env=prod                             # Label equals value
kubectl get pods -l env!=prod                            # Label not equal to value
kubectl get pods -l 'env in (prod,qa)'                   # Label in a set
kubectl get pods -l 'env notin (dev)'                    # Label not in a set
kubectl get pods -l 'app'                                # Label exists
kubectl get pods -l '!app'                               # Label does not exist
kubectl get pods -l app=web,tier=frontend                # Both conditions (AND)
```

---

## 21. Namespaces

Namespaces split one cluster into isolated virtual clusters for teams or environments.

```bash
kubectl get ns                                           # List namespaces
kubectl create ns dev                                    # Create a namespace
kubectl delete ns dev                                    # Delete a namespace and its contents
kubectl config set-context --current --namespace=dev     # Make "dev" the default namespace
kubectl config view --minify -o jsonpath='{..namespace}' # Show the current default namespace
kubectl get all -n dev                                   # Workloads in a namespace
kubectl get resourcequota -n dev                         # Quotas in a namespace
kubectl get limitrange -n dev                            # Default limits in a namespace
kubectl describe quota -n dev                            # Quota usage vs. limits

# Namespace stuck in "Terminating"
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <ns>   # Find leftover resources
kubectl get ns <ns> -o json | jq '.spec.finalizers'                 # Show finalizers blocking deletion
```

---

## 22. Nodes: Cordon, Drain, Taints

Prepare nodes for maintenance and control which pods can be scheduled on them.

```bash
kubectl get nodes -o wide                                # Nodes with IPs and versions
kubectl describe node <node>                             # Conditions, capacity, running pods
kubectl get node <node> -o jsonpath='{.status.allocatable}'   # Resources available to pods
kubectl get nodes --show-labels                          # Node labels
kubectl get nodes -l node-role.kubernetes.io/control-plane    # Control-plane nodes only

kubectl cordon <node>                                     # Stop new pods from scheduling here
kubectl uncordon <node>                                   # Allow scheduling again
kubectl drain <node>                                      # Evict pods for maintenance
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # Typical drain command
kubectl drain <node> --ignore-daemonsets --force --grace-period=30   # Also evict unmanaged pods
kubectl drain <node> --pod-selector='app!=critical'       # Drain only matching pods
kubectl drain <node> --timeout=5m --disable-eviction      # Delete pods, bypassing PDBs (careful)

# Taints
kubectl taint nodes <node> key=value:NoSchedule           # Block new pods without a toleration
kubectl taint nodes <node> key=value:PreferNoSchedule     # Avoid scheduling if possible
kubectl taint nodes <node> key=value:NoExecute            # Evict pods without a toleration
kubectl taint nodes <node> key=value:NoSchedule-          # Remove a specific taint
kubectl taint nodes <node> key-                           # Remove all taints with this key
kubectl taint nodes <node> node-role.kubernetes.io/control-plane:NoSchedule-   # Allow pods on control plane
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints    # Show taints per node

kubectl delete node <node>                                # Remove a node from the cluster
```

---

## 23. RBAC, ServiceAccounts & Auth

Control who can do what. Roles define permissions; bindings grant them to users, groups or service accounts.

```bash
# Roles (namespaced)
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev     # Read-only pods role
kubectl create role cm-editor --verb=* --resource=configmaps --resource-name=app-cfg   # Role for one named object
kubectl create rolebinding read-pods --role=pod-reader --user=alice -n dev      # Grant role to a user
kubectl create rolebinding sa-read --role=pod-reader --serviceaccount=dev:app-sa -n dev   # Grant to a service account
kubectl create rolebinding grp-read --role=pod-reader --group=devs -n dev       # Grant to a group

# ClusterRoles (cluster-wide)
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes   # Cluster-wide read on nodes
kubectl create clusterrole aggregated --aggregation-rule="rbac.example.com/aggregate=true"   # Aggregate other roles
kubectl create clusterrolebinding node-read --clusterrole=node-reader --user=alice   # Grant cluster-wide
kubectl create rolebinding view-dev --clusterrole=view --user=bob -n dev        # Reuse a ClusterRole in one namespace

kubectl get roles,rolebindings -n dev                    # List namespaced RBAC
kubectl get clusterroles,clusterrolebindings             # List cluster RBAC
kubectl describe clusterrole admin                       # Show a role's rules

# Permission checks
kubectl auth can-i create deployments                    # Can I create deployments?
kubectl auth can-i delete pods -n prod                   # Can I delete pods in prod?
kubectl auth can-i '*' '*'                               # Am I cluster-admin?
kubectl auth can-i --list -n dev                         # Everything I can do in a namespace
kubectl auth can-i get pods --as=alice                   # Check another user's permissions
kubectl auth can-i get pods --as=system:serviceaccount:dev:app-sa   # Check a service account
kubectl auth can-i get pods --as=bob --as-group=devs     # Check a user as a group member
kubectl auth whoami                                      # Show my authenticated identity
kubectl auth reconcile -f rbac.yaml                      # Safely create/update RBAC objects

kubectl get pods --as=alice -n dev                       # Run any command as another user

# ServiceAccounts
kubectl create sa app-sa                                 # Create a service account
kubectl get sa                                           # List service accounts
kubectl create token app-sa                              # Issue a short-lived token
kubectl create token app-sa --duration=24h --audience=api   # Token with custom lifetime and audience

# Certificate signing requests (user certs)
kubectl get csr                                          # List certificate requests
kubectl certificate approve <csr>                        # Approve a request
kubectl certificate deny <csr>                           # Deny a request
kubectl get csr <csr> -o jsonpath='{.status.certificate}' | base64 -d > user.crt   # Save the signed cert
```

**Built-in ClusterRoles:** `cluster-admin` · `admin` · `edit` · `view`

---

## 24. Resource Usage & Events

See CPU and memory usage (needs metrics-server) and the event stream that explains what the cluster is doing.

```bash
kubectl top nodes                                        # CPU/memory per node
kubectl top nodes --sort-by=cpu                          # Nodes sorted by CPU
kubectl top pods                                         # CPU/memory per pod
kubectl top pods -A --sort-by=memory                     # Heaviest pods cluster-wide
kubectl top pod <pod> --containers                       # Usage per container
kubectl top pods -l app=web                              # Usage for matching pods

# Events
kubectl get events                                       # Events in the current namespace
kubectl get events -A --sort-by=.lastTimestamp           # All events, newest last
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp   # Events in order of creation
kubectl get events --field-selector type=Warning         # Only warnings
kubectl get events --field-selector involvedObject.name=<pod>      # Events for one object
kubectl get events -w                                    # Watch events live
kubectl events                                           # Newer, friendlier events command
kubectl events --for pod/<pod>                           # Events for one object
kubectl events --types=Warning -A                        # Warnings across the cluster
kubectl events --watch                                   # Stream new events
```

---

## 25. Debugging & Troubleshooting

Tools for finding out why something is broken: debug containers, node shells, verbose API output.

```bash
# Ephemeral debug container in a running pod
kubectl debug -it <pod> --image=busybox --target=<container>         # Attach a debug container sharing the process namespace
kubectl debug -it <pod> --image=nicolaka/netshoot --profile=netadmin # Debug container with network tools and capabilities

# Copy a pod and debug the copy
kubectl debug <pod> -it --copy-to=<pod>-dbg --container=<container> -- sh   # Copy the pod and open a shell
kubectl debug <pod> --copy-to=<pod>-dbg --set-image=*=busybox               # Copy with a different image
kubectl debug <pod> -it --copy-to=<pod>-dbg --share-processes --image=busybox   # Copy with shared processes

# Debug a node
kubectl debug node/<node> -it --image=ubuntu             # Shell on a node; host filesystem at /host
#   then: chroot /host                                   # Act as if logged into the node

# Quick health triage
kubectl get pods -A | grep -vE 'Running|Completed'       # Find unhealthy pods
kubectl get pods -A --field-selector=status.phase!=Running   # Pods not in Running phase
kubectl describe pod <pod> | sed -n '/Events/,$p'        # Show only the events section
kubectl logs <pod> --previous                            # Why did the last container crash?
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state}'                       # Current container state
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'  # Last termination reason

# Verbose API calls
kubectl get pods -v=6          # Show HTTP requests made
kubectl get pods -v=8          # Include request/response bodies
kubectl get pods -v=9          # Maximum verbosity, with curl equivalents

# Raw API access
kubectl get --raw /api/v1/namespaces/default/pods        # Raw GET on an API path
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes     # Raw node metrics
kubectl create --raw /api/v1/namespaces -f ns.json       # Raw POST
kubectl delete --raw /api/v1/namespaces/<ns>/pods/<pod>  # Raw DELETE

# Control-plane pods (kubeadm clusters)
kubectl get pods -n kube-system                          # System components
kubectl logs -n kube-system kube-apiserver-<node>        # API server logs
kubectl logs -n kube-system -l k8s-app=kube-dns          # CoreDNS logs
```

---

## 26. Waiting & Scripting

Wait for conditions in CI/CD pipelines and combine kubectl with shell tools for bulk operations.

```bash
kubectl wait --for=condition=Ready pod/<pod> --timeout=60s           # Wait until a pod is ready
kubectl wait --for=condition=Ready pods -l app=web --timeout=120s    # Wait for all matching pods
kubectl wait --for=condition=Available deploy/web                    # Wait for a deployment
kubectl wait --for=condition=complete job/<job> --timeout=300s       # Wait for a job to finish
kubectl wait --for=delete pod/<pod> --timeout=60s                    # Wait until deleted
kubectl wait --for=jsonpath='{.status.phase}'=Running pod/<pod>      # Wait for a field value
kubectl wait --for=jsonpath='{.status.loadBalancer.ingress}' svc/<svc>   # Wait for a LoadBalancer IP

for p in $(kubectl get pods -l app=web -o name); do kubectl logs "$p" --tail=5; done   # Loop over pods
kubectl get pods -o name | xargs -I{} kubectl describe {}                              # Describe every pod
kubectl get deploy -n <ns> -o name | xargs -n1 kubectl rollout restart -n <ns>         # Restart every deployment

kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u   # All images in the cluster
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>                      # All pods on one node
```

---

## 27. Kustomize

Customize plain YAML for different environments (dev, staging, prod) without templates. Built into kubectl.

```bash
kubectl kustomize ./overlay                    # Render the final YAML
kubectl apply -k ./overlay                     # Apply a kustomization
kubectl diff -k ./overlay                      # Preview changes
kubectl delete -k ./overlay                    # Delete everything it defines

# Standalone kustomize CLI
kustomize build ./overlay                      # Render the final YAML
kustomize create --autodetect                  # Create kustomization.yaml from files in the folder
kustomize edit set image nginx=nginx:1.27      # Change an image tag
kustomize edit add resource deploy.yaml        # Add a resource file
kustomize edit set namespace dev               # Set the namespace for all resources
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev                 # Namespace applied to everything
resources:
  - ../../base                 # Base manifests to start from
images:
  - name: nginx
    newTag: "1.27"             # Override the image tag
patches:
  - path: replica-patch.yaml   # Environment-specific patch
configMapGenerator:
  - name: app-cfg
    literals: [ENV=dev]        # Generate a ConfigMap
```

---

## 28. Plugins & Krew

Extend kubectl with plugins. Krew is the plugin manager.

```bash
kubectl plugin list                    # List installed plugins
# Any executable named kubectl-<name> on $PATH becomes `kubectl <name>`

# Krew (plugin manager)
kubectl krew update                    # Refresh the plugin index
kubectl krew search                    # List available plugins
kubectl krew install ctx ns neat tree stern view-secret   # Install useful plugins
kubectl krew list                      # List installed plugins
kubectl krew upgrade                   # Upgrade all plugins
kubectl krew uninstall <plugin>        # Remove a plugin

# Popular plugins
kubectl ctx <context>                  # Switch context quickly
kubectl ns <namespace>                 # Switch namespace quickly
kubectl neat get pod <pod> -o yaml     # Clean YAML without status and managed fields
kubectl tree deploy <name>             # Show owner → child resource tree
kubectl stern <pattern>                # Tail logs from many pods with colours
kubectl view-secret <secret>           # Show decoded secret values
```

**Related tools:** `k9s` (terminal UI) · `kubectx`/`kubens` · `stern` · `kubeconform` (manifest validation) · `kube-linter`

---

## 29. kubeadm (Cluster Lifecycle)

kubeadm bootstraps, joins, upgrades and resets self-managed clusters.

```bash
# Initialize control plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16                   # Create a cluster
sudo kubeadm init --control-plane-endpoint=<lb>:6443 --upload-certs  # HA-ready control plane
sudo kubeadm init --config=kubeadm-config.yaml                       # Init from a config file
sudo kubeadm config print init-defaults > kubeadm-config.yaml        # Generate a default config
sudo kubeadm config images list                                      # Images the cluster needs
sudo kubeadm config images pull                                      # Pre-pull those images

# Configure kubectl after init
mkdir -p $HOME/.kube                                       # Create the kubeconfig folder
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config   # Copy the admin kubeconfig
sudo chown $(id -u):$(id -g) $HOME/.kube/config            # Make it owned by your user

# Join nodes
kubeadm token create --print-join-command                  # Print a ready-to-use join command
kubeadm token list                                         # List bootstrap tokens
kubeadm token delete <token>                               # Revoke a token
sudo kubeadm join <ip>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<hash>   # Join a worker
sudo kubeadm join <ip>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>                  # Join an extra control-plane node
sudo kubeadm init phase upload-certs --upload-certs        # Generate a new certificate key

# Upgrades
sudo kubeadm upgrade plan                                  # Show available upgrade versions
sudo kubeadm upgrade apply v1.X.Y                          # Upgrade the first control-plane node
sudo kubeadm upgrade node                                  # Upgrade other nodes

# Certificates
sudo kubeadm certs check-expiration                        # Show certificate expiry dates
sudo kubeadm certs renew all                               # Renew all certificates
sudo kubeadm certs renew apiserver                         # Renew one certificate

# Tear down
sudo kubeadm reset                                         # Undo kubeadm init/join on this node
sudo kubeadm reset --force                                 # Reset without confirmation
```

**Typical node upgrade flow:**
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # 1. Move workloads off the node
# 2. Upgrade kubeadm package → kubeadm upgrade (apply|node) → upgrade kubelet & kubectl packages
sudo systemctl daemon-reload && sudo systemctl restart kubelet    # 3. Restart kubelet on the new version
kubectl uncordon <node>                                           # 4. Allow scheduling again
```

**Important paths:**
| Path | Purpose |
|---|---|
| `/etc/kubernetes/manifests/` | Static pod manifests (apiserver, etcd, scheduler, controller-manager) |
| `/etc/kubernetes/pki/` | Cluster certificates |
| `/etc/kubernetes/admin.conf` | Admin kubeconfig |
| `/var/lib/kubelet/config.yaml` | Kubelet config |
| `/var/lib/etcd/` | etcd data dir |

---

## 30. etcd Backup & Restore

etcd stores all cluster state. Back it up regularly on self-managed clusters.

```bash
export ETCDCTL_API=3                                   # Use the v3 API
CERTS="--cacert=/etc/kubernetes/pki/etcd/ca.crt \
       --cert=/etc/kubernetes/pki/etcd/server.crt \
       --key=/etc/kubernetes/pki/etcd/server.key \
       --endpoints=https://127.0.0.1:2379"             # Reusable connection flags

etcdctl $CERTS endpoint health                         # Is etcd healthy?
etcdctl $CERTS endpoint status -w table                # Leader, DB size, revision
etcdctl $CERTS member list -w table                    # List etcd members

etcdctl $CERTS snapshot save /backup/etcd-$(date +%F).db   # Take a snapshot backup

etcdutl snapshot status /backup/etcd.db -w table       # Verify a snapshot

etcdutl snapshot restore /backup/etcd.db --data-dir=/var/lib/etcd-restored   # Restore to a new data dir
#   then edit /etc/kubernetes/manifests/etcd.yaml → hostPath for etcd-data = /var/lib/etcd-restored

etcdctl $CERTS get /registry/pods --prefix --keys-only | head   # Browse stored keys
```

---

## 31. Container Runtime (crictl)

`crictl` talks directly to the container runtime on a node. Useful when kubectl or the API server is down.

```bash
sudo crictl info                   # Runtime status and config
sudo crictl ps                     # Running containers
sudo crictl ps -a                  # All containers, including exited
sudo crictl pods                   # Pod sandboxes on this node
sudo crictl pods --name <pod>      # Find a pod by name
sudo crictl images                 # Images on this node
sudo crictl pull nginx:1.27        # Pull an image
sudo crictl rmi <image>            # Remove an image
sudo crictl rmi --prune            # Remove all unused images
sudo crictl logs <container-id>    # Container logs
sudo crictl logs -f --tail=50 <container-id>   # Follow the last 50 lines
sudo crictl exec -it <container-id> sh         # Shell into a container
sudo crictl inspect <container-id> # Container details (JSON)
sudo crictl inspectp <pod-id>      # Pod sandbox details
sudo crictl stats                  # Container resource usage
sudo crictl stop <container-id>    # Stop a container
sudo crictl rm <container-id>      # Remove a container
sudo crictl stopp <pod-id> && sudo crictl rmp <pod-id>   # Stop and remove a pod sandbox

# containerd native CLI
sudo ctr -n k8s.io containers list # Containers in the Kubernetes namespace
sudo ctr -n k8s.io images list     # Images in the Kubernetes namespace
sudo nerdctl -n k8s.io ps          # Docker-like CLI for containerd (if installed)
```

---

## 32. Node-level Troubleshooting

When a node is `NotReady`, check kubelet, the runtime and networking directly on the node.

```bash
systemctl status kubelet                           # Is kubelet running?
sudo systemctl restart kubelet                     # Restart kubelet
journalctl -u kubelet -f                           # Follow kubelet logs
journalctl -u kubelet --since "10 min ago" --no-pager   # Recent kubelet logs

systemctl status containerd                        # Is the runtime running?
journalctl -u containerd -f                        # Follow runtime logs

ls /etc/kubernetes/manifests/                      # Static pod manifests
ls /var/log/pods/                                  # Pod log folders
ls /var/log/containers/                            # Container log symlinks

ip a; ip route                                     # Interfaces and routes
sudo iptables -t nat -L KUBE-SERVICES | head       # kube-proxy service rules (iptables mode)
sudo ipvsadm -Ln                                   # kube-proxy rules (IPVS mode)
cat /etc/cni/net.d/*                               # CNI network config
ls /opt/cni/bin/                                   # Installed CNI plugins

df -h; free -m; top                                # Disk, memory and CPU on the node
```

---

## 33. Helm Essentials

Helm is the package manager for Kubernetes. Charts are packages; releases are installed instances.

```bash
helm version                                           # Show Helm version
helm repo add bitnami https://charts.bitnami.com/bitnami   # Add a chart repository
helm repo update                                       # Refresh repository indexes
helm repo list                                         # List added repositories
helm search repo nginx                                 # Search added repos
helm search hub prometheus                             # Search Artifact Hub
helm show values bitnami/nginx > values.yaml           # Save a chart's default values
helm show chart bitnami/nginx                          # Show chart metadata

helm install web bitnami/nginx -n web --create-namespace   # Install a chart
helm install web bitnami/nginx -f values.yaml --set replicaCount=3   # Install with custom values
helm install web ./mychart --dry-run --debug           # Render and validate without installing
helm upgrade web bitnami/nginx -f values.yaml          # Upgrade a release
helm upgrade --install web bitnami/nginx -n web --create-namespace --atomic --wait   # Install or upgrade, roll back on failure
helm list -A                                           # List releases in all namespaces
helm status web -n web                                 # Release status and notes
helm history web -n web                                # Release revision history
helm rollback web 1 -n web                             # Roll back to revision 1
helm get values web -n web                             # Values used by a release
helm get manifest web -n web                           # Rendered YAML of a release
helm uninstall web -n web                              # Remove a release

helm template web ./mychart -f values.yaml > rendered.yaml   # Render templates locally
helm lint ./mychart                                    # Check a chart for errors
helm create mychart                                    # Scaffold a new chart
helm package ./mychart                                 # Package a chart into a .tgz
helm dependency update ./mychart                       # Download chart dependencies
helm pull bitnami/nginx --untar                        # Download and extract a chart
helm install web oci://registry.example.com/charts/web --version 1.2.3   # Install from an OCI registry
```

---

## 34. Global Flags Reference

Flags that work with most kubectl commands.

| Flag | Purpose |
|---|---|
| `-n, --namespace` | Target namespace |
| `-A, --all-namespaces` | All namespaces |
| `-o, --output` | Output format |
| `-l, --selector` | Label selector |
| `--field-selector` | Field selector |
| `-f, --filename` | File, dir or URL |
| `-k, --kustomize` | Kustomize dir |
| `-R, --recursive` | Recurse into dirs with `-f` |
| `-w, --watch` | Watch for changes |
| `--context` | Use a kubeconfig context |
| `--kubeconfig` | Use a kubeconfig file |
| `--as`, `--as-group` | Impersonate user/group |
| `--dry-run=client\|server` | Simulate |
| `--server-side` | Server-side apply |
| `--request-timeout` | e.g. `30s` |
| `-v=<0-9>` | Log verbosity |
| `--sort-by` | Sort by JSONPath |
| `--show-labels` | Show labels column |
| `--no-headers` | Hide header row |
| `--grace-period`, `--force` | Deletion behaviour |
| `--insecure-skip-tls-verify` | Skip TLS checks (avoid in prod) |

```bash
kubectl options            # List all global flags
kubectl <command> --help   # Help and examples for any command
```

---

## 35. Pod Phases & Common Errors

What each common pod status usually means and the first command to run.

| Status | Usual cause | First command |
|---|---|---|
| `Pending` | No node fits (resources, taints, affinity), unbound PVC | `kubectl describe pod` → Events |
| `ContainerCreating` | Image pull, volume mount, CNI issue | `kubectl describe pod` |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image/tag, private registry without `imagePullSecrets` | `kubectl describe pod` |
| `CrashLoopBackOff` | App exits on start, bad command/config, failing probes | `kubectl logs --previous` |
| `OOMKilled` | Memory limit exceeded | `kubectl describe pod` → Last State |
| `CreateContainerConfigError` | Missing ConfigMap/Secret/key | `kubectl describe pod` |
| `Evicted` | Node pressure (disk/memory) | `kubectl describe node` |
| `Terminating` (stuck) | Finalizers, unreachable node | `kubectl delete --force --grace-period=0` |
| `Init:Error` / `Init:CrashLoopBackOff` | Init container failing | `kubectl logs <pod> -c <init>` |
| Node `NotReady` | kubelet / runtime / CNI down | `journalctl -u kubelet` |

---

## 36. YAML Skeletons

Minimal, working manifests to start from.

<details>
<summary><b>Pod</b> — single container with resources and probes</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels: { app: web }
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports: [{ containerPort: 80 }]
      resources:
        requests: { cpu: 100m, memory: 128Mi }   # Guaranteed minimum
        limits:   { cpu: 500m, memory: 256Mi }   # Hard maximum
      readinessProbe:                            # When to send traffic
        httpGet: { path: /, port: 80 }
        initialDelaySeconds: 5
      livenessProbe:                             # When to restart
        httpGet: { path: /, port: 80 }
        periodSeconds: 10
```
</details>

<details>
<summary><b>Deployment + Service</b> — replicated app exposed inside the cluster</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }   # Zero-downtime updates
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          envFrom:
            - configMapRef: { name: app-cfg }   # Load all keys as env vars
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector: { app: web }                         # Routes to pods with this label
  ports:
    - port: 80
      targetPort: 80
```
</details>

<details>
<summary><b>Ingress</b> — HTTP(S) routing from outside the cluster</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  tls:
    - hosts: [example.com]
      secretName: web-tls                        # TLS certificate secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```
</details>

<details>
<summary><b>Gateway + HTTPRoute</b> — Gateway API routing with a canary split</summary>

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gw
spec:
  gatewayClassName: <your-gateway-class>         # From: kubectl get gatewayclass
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: example.com
      tls:
        certificateRefs:
          - name: web-tls                        # TLS certificate secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
spec:
  parentRefs:
    - name: web-gw                               # Attach to the Gateway above
  hostnames: [example.com]
  rules:
    - matches:
        - path: { type: PathPrefix, value: / }
      backendRefs:
        - name: web-v1                           # 90% of traffic
          port: 80
          weight: 90
        - name: web-v2                           # 10% canary
          port: 80
          weight: 10
```
</details>

<details>
<summary><b>PersistentVolumeClaim</b> — request storage</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standard
  resources:
    requests: { storage: 10Gi }
```
</details>

<details>
<summary><b>NetworkPolicy</b> — default deny, then allow frontend → api</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}                                # Applies to all pods in the namespace
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
spec:
  podSelector:
    matchLabels: { app: api }
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: frontend }
      ports:
        - { protocol: TCP, port: 8080 }
```
</details>

<details>
<summary><b>CronJob</b> — scheduled task</summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"                          # Every day at 02:00
  concurrencyPolicy: Forbid                      # Skip if previous run still active
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox
              command: ["sh", "-c", "date; echo backing up"]
```
</details>

<details>
<summary><b>Role + RoleBinding</b> — read-only pod access for a service account</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
</details>

---

## 📚 References

- [kubectl reference](https://kubernetes.io/docs/reference/kubectl/)
- [Official kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [JSONPath support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Ingress NGINX retirement announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Minikube docs](https://minikube.sigs.k8s.io/docs/)
- [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Helm docs](https://helm.sh/docs/)

---

⭐ If this helped you, star the repo! Contributions and PRs are welcome.
