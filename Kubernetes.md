# Kubernetes (1.28+) - Deep Interview Notes for a 5-Year Java Developer

> Companion: container internals (namespaces, cgroups, image layers, runc) are in `Docker.md`. This file assumes you know what a container is and covers how Kubernetes runs, scales, heals and exposes them.
> Version notes: native sidecar containers = beta/default-on in 1.29, GA in 1.33. `PodSecurityPolicy` was removed in 1.25 (replaced by Pod Security Admission). `ValidatingAdmissionPolicy` GA in 1.30. Where a feature's maturity varies by version, it is flagged.

## Table of contents
1. 60-second mental model
2. Deep internals (control plane, `kubectl apply` trace, pod lifecycle, termination, probes, resources, networking, storage, scaling, security)
3. Full YAML for a Spring Boot service, line by line
4. Helm, Kustomize, GitOps, observability, local practice
5. Production war stories and troubleshooting playbook (decision trees)
6. Interview questions (Easy / Medium / Hard) with follow-up chains and wrong answers
7. Cheat sheets (one-page + kubectl)

---

# 1. The 60-second mental model

**Kubernetes = a database of desired state + a swarm of control loops that push reality toward it.**

- You write YAML: "I want 3 copies of `order-service:1.4.0`, reachable at a stable address."
- `kubectl` sends it to the **API server**, which validates it and stores it in **etcd**.
- **Controllers** (small loops) watch the API server, compare *desired* vs *actual*, and act: create pods, replace dead ones, add endpoints.
- The **scheduler** picks a node for each unplaced pod. The **kubelet** on that node makes the container runtime actually run it.
- Nothing "commands" anything imperatively; every component only reads/writes objects through the API server. That is why the system self-heals: a loop that sees drift simply fixes it, forever.

**Analogy: a restaurant with a manager's whiteboard.**
The whiteboard (etcd, guarded by one clerk = API server) says "Table 5 wants 3 pizzas." Shift supervisors (controllers) each watch one line of the board: the pizza supervisor counts pizzas in the kitchen; if 2, they write a ticket for one more. The dispatcher (scheduler) assigns each ticket to a chef with free counter space. The chef (kubelet) cooks it and reports back on the board. If a chef collapses, the supervisor notices "only 2 pizzas" and writes another ticket. Nobody phones anybody; everyone reads and writes the whiteboard.

```
   You ── kubectl apply ──▶ [ API server ] ◀──watch── controllers / scheduler / kubelets
                                 │
                              [ etcd ]      desired state (spec)  +  observed state (status)
   Level-triggered reconciliation:   loop { diff(spec, status) -> act }   (never "event happened once")
```

**Key vocabulary in one line each**
| Term | One-liner |
|---|---|
| Pod | 1+ containers sharing network namespace (one IP), volumes; the unit of scheduling; ephemeral |
| ReplicaSet | keeps N identical pods alive (you rarely create it directly) |
| Deployment | owns ReplicaSets; does rolling updates/rollbacks |
| Service | stable virtual IP + DNS name load-balancing over ready pods matched by label selector |
| Ingress / Gateway | L7 (HTTP) entry from outside into Services |
| ConfigMap / Secret | config injected as env vars or files |
| PV/PVC/StorageClass | durable storage request/claim/provisioner |
| StatefulSet | stable network identity + per-pod storage + ordered rollout |
| HPA | scales replica count on metrics |
| Namespace | name scope + quota/RBAC/policy boundary |

---

# 2. Deep internals

## 2.1 Architecture in depth

```
 ┌──────────────────────────── CONTROL PLANE ────────────────────────────┐
 │                                                                        │
 │   kubectl / CI / controllers / kubelets  ──HTTPS(TLS)──▶ kube-apiserver │
 │                                                             │  ▲       │
 │                     ┌───────────────────────────────────────┘  │       │
 │                     ▼  (ONLY component that talks to etcd)      │watch  │
 │                  ┌──────┐   Raft quorum: 3 or 5 members          │       │
 │                  │ etcd │◀─▶ etcd ◀─▶ etcd                       │       │
 │                  └──────┘                                        │       │
 │   kube-scheduler ──watch unscheduled pods, write Binding─────────┤       │
 │   kube-controller-manager (Deployment, ReplicaSet, Node, Job,    │       │
 │       EndpointSlice, ServiceAccount, Namespace... controllers) ──┘       │
 │   cloud-controller-manager (LB, node lifecycle, routes on AWS/GCP/Azure) │
 └────────────────────────────────────────────────────────────────────────┘
                                   │ watch pods where spec.nodeName == me
 ┌──────────────── WORKER NODE ─────▼───────────────────────────────────┐
 │  kubelet ──CRI(gRPC)──▶ containerd ──▶ runc ──▶ containers (cgroups/ns)│
 │     │  └─CNI plugin (pod IP)   └─CSI (volumes)   └─probes, cgroup mgmt │
 │  kube-proxy (iptables/IPVS rules for Services)                          │
 │  [Pod: pause + app + sidecar]  [Pod]  [Pod]     CoreDNS runs as pods    │
 └────────────────────────────────────────────────────────────────────────┘
```

### kube-apiserver
- The **only** front door: REST over HTTPS. Stateless (scale horizontally behind an LB). Every other component, including kubelet and scheduler, is just an API client.
- Request pipeline: **authentication -> authorization -> mutating admission -> schema validation -> validating admission -> persist to etcd** -> notify watchers.
- Provides **watch** (long-lived streaming) so controllers react in milliseconds instead of polling. Uses `resourceVersion` for **optimistic concurrency**: an update with a stale version gets `409 Conflict`; the client re-reads and retries.
- Also serves aggregated APIs (metrics-server, CRDs).

### etcd
- Distributed key-value store, the cluster's only source of truth. **Raft** consensus: one leader; a write is committed once a **majority (quorum = n/2+1)** of members persist it. 3 members tolerate 1 failure, 5 tolerate 2. Even numbers add no fault tolerance (4 tolerates 1, same as 3). Lose quorum -> the API becomes read-only/unavailable, but **running pods keep running** (kubelets/containers don't need the control plane to keep existing workloads alive).
- Only the apiserver talks to etcd -> one place for auth, validation, and encryption at rest. Latency-sensitive (fsync): use SSD, keep DB small (default ~2 GB quota, commonly tuned up to 8 GB); defragment/compact regularly.
- Managed clusters (EKS/GKE/AKS) hide etcd from you.

### kube-scheduler
Assigns a node to each pod where `spec.nodeName` is empty. For each pod:
1. **Queue** (priority order).
2. **Filter** (hard constraints, "can it run here?"): enough allocatable CPU/memory **based on requests, not actual usage**; nodeSelector/affinity; taints vs tolerations; volume zone/topology; ports; topology spread hard rules.
3. **Score** (soft preferences, "which is best?"): spread across nodes, image already present, resource balance, preferred affinity. Highest total wins (ties random).
4. **Reserve/Permit/Bind**: write a `Binding` (sets `spec.nodeName`) via the apiserver.
5. If nothing passes Filter: pod stays `Pending` with event `FailedScheduling`; **PostFilter = preemption** may evict lower-priority pods.
Key interview point: the scheduler never looks at real CPU usage; a node at 5% actual usage but 100% *requested* is "full". That is why requests must be honest.

### kube-controller-manager and the reconciliation idea
One binary running many controllers. Each is:
```
for {
   desired := read spec from API (via informer cache)
   actual  := read current world (status/owned objects)
   if desired != actual { act to converge; write status }
}
```
- **Level-triggered, not edge-triggered**: it doesn't matter if you missed an event; on the next sync it sees the difference and fixes it. Idempotent by design.
- Chain reaction: Deployment controller -> ReplicaSet -> ReplicaSet controller -> Pods; Node controller marks nodes NotReady and taints them; EndpointSlice controller builds Service backends; Job controller, GC (via `ownerReferences`: delete the Deployment and children are garbage-collected).
- Custom controllers/Operators = the same pattern for your own CRD (e.g. a Kafka operator).

### kubelet (node agent)
- Watches pods bound to its node; makes the runtime match. Handles: pulling images, creating pod sandbox, volumes, running probes, restarting containers per `restartPolicy`, reporting `status`, cgroup setup for requests/limits, node-pressure **eviction**, node heartbeat (a `Lease` object in `kube-node-lease`, ~10s; plus `NodeStatus`).
- Talks to the runtime over **CRI** (gRPC), so Kubernetes is runtime-agnostic. Dockershim was removed in 1.24; containerd (or CRI-O) is now standard. `docker build` images still run fine (OCI images).

### Container runtime, CNI, CSI
- **containerd** implements CRI: `RunPodSandbox`, `PullImage`, `CreateContainer`, `StartContainer`; uses **runc** to create the actual namespaces/cgroups (see Docker.md).
- **CNI** (Container Network Interface): a plugin binary/daemon (AWS VPC CNI, Calico, Cilium, Flannel) that gives each pod a routable IP and wires it up. Kubernetes network model: every pod gets its own IP; pods reach any other pod without NAT.
- **CSI**: storage plugin interface (EBS CSI driver etc.).

### kube-proxy and Services
Runs on every node, watches Services + EndpointSlices, programs the node's kernel so a Service **virtual IP** (which no interface owns) is DNAT'd to a backend pod IP.
| Mode | How | Notes |
|---|---|---|
| **iptables** (default long-time) | chains of rules; random-probability rule per endpoint | O(n) rule evaluation; rule updates get slow with thousands of Services |
| **IPVS** | kernel L4 load balancer with hash tables | O(1) lookup, more algorithms (rr, lc, sh...); better for huge clusters |
| nftables | newer kernel backend (alpha 1.29, beta 1.31, GA 1.33) | successor to iptables mode |
Some CNIs (Cilium) replace kube-proxy entirely with eBPF. A ClusterIP is not pingable: it is only rules for the Service's ports.

### CoreDNS
Runs as a Deployment in `kube-system`, exposed via the `kube-dns` Service; each pod's `/etc/resolv.conf` points to that Service IP. Watches Services/EndpointSlices and answers names:
- `my-svc` (same namespace) / `my-svc.ns` / `my-svc.ns.svc` / **`my-svc.ns.svc.cluster.local`** -> ClusterIP.
- Headless service -> A records of each pod IP; StatefulSet pod -> `kafka-0.kafka-headless.ns.svc.cluster.local`.
- `resolv.conf` has `search ns.svc.cluster.local svc.cluster.local cluster.local` and `options ndots:5`: a name with fewer than 5 dots (like `api.example.com`) is first tried with each search suffix -> several wasted NXDOMAIN lookups before the real one. Mitigate with trailing-dot FQDNs, `dnsConfig.options ndots`, or NodeLocal DNSCache.

## 2.2 What EXACTLY happens on `kubectl apply -f deployment.yaml`

```
kubectl ──1──▶ apiserver ──▶ authn ─▶ authz ─▶ mutating adm ─▶ validate ─▶ validating adm ─▶ etcd
                                                                                      │
      ┌─────────────────────────── watch events fan out ──────────────────────────────┘
      ▼
 Deployment ctrl ─2─▶ creates ReplicaSet ─3─▶ RS ctrl creates N Pods (nodeName empty)
      scheduler ─4─▶ Filter+Score ─▶ Binding (nodeName=node-b)
      kubelet(node-b) ─5─▶ CRI: sandbox + CNI IP, pull image, start container
      kubelet ─6─▶ startup/readiness probes ─▶ Pod Ready=True
      EndpointSlice ctrl ─7─▶ adds pod IP to slice ─▶ kube-proxy/ingress update rules ─8─▶ traffic flows
```

1. **Client side**: `kubectl` reads kubeconfig (server URL, credentials), fetches API discovery, parses YAML, and sends a `PATCH`/`POST` to `/apis/apps/v1/namespaces/<ns>/deployments`. (`apply` records the last-applied config annotation, or with `--server-side` uses field managers.)
2. **Authentication**: who are you? (client cert, bearer/ServiceAccount token, OIDC, EKS IAM authenticator). Failure -> 401.
3. **Authorization**: may this identity `create deployments` in this namespace? (RBAC; also Node authorizer for kubelets). Failure -> 403.
4. **Admission - mutating**: defaults injected (ServiceAccount, default StorageClass, LimitRanger defaults, sidecar injectors like Istio, custom webhooks). **Schema validation** of the (possibly mutated) object. **Admission - validating**: ResourceQuota, Pod Security Admission, ValidatingAdmissionPolicy, policy webhooks (OPA Gatekeeper/Kyverno). Any rejection -> 4xx with a message.
5. **Persist**: the Deployment is written to etcd. `kubectl` returns "deployment.apps/x created" **now**; nothing is running yet. Everything after is asynchronous.
6. **Deployment controller** (watching Deployments) creates a **ReplicaSet** named `<deploy>-<pod-template-hash>` with `ownerReferences` -> Deployment.
7. **ReplicaSet controller** sees "want 3, have 0", creates 3 **Pod** objects (again via apiserver; passes admission). Pods have no `nodeName`, phase `Pending`.
8. **Scheduler** picks them up: Filter -> Score -> writes Binding. Event: `Scheduled`.
9. **kubelet** on the chosen node (watching pods for its node) admits the pod, mounts volumes (mounts ConfigMaps/Secrets), then via CRI: **RunPodSandbox** (creates the pause container that holds the network namespace; the **CNI** plugin assigns the pod IP), **PullImage** (respecting `imagePullPolicy` and pull secrets), runs **init containers** sequentially, then **CreateContainer/StartContainer** for app containers. Events: `Pulling`, `Pulled`, `Created`, `Started`.
10. **Probes**: startup probe (if defined) gates the others; readiness probe decides `Ready`; liveness runs continuously. Kubelet PATCHes pod `status` (phase Running, conditions).
11. **EndpointSlice controller** adds the pod IP to the Service's EndpointSlice **only when the pod is Ready**. (`Endpoints` object is the legacy equivalent; its API is deprecated from 1.33; EndpointSlice is the real mechanism.)
12. **kube-proxy** (on every node) sees the slice change and updates iptables/IPVS; ingress controllers also watch endpoints and update their upstream lists.
13. **Traffic**: client -> Service VIP -> DNAT to a ready pod IP.
14. The Deployment controller updates `status` (`readyReplicas`, conditions `Progressing`/`Available`); `kubectl rollout status` reports success.

Say in an interview: *"apply only writes desired state to etcd; a chain of independent controllers does the rest, each level-triggered."*

## 2.3 Request path: Client -> Ingress -> Service -> Pod

```
 Browser ─DNS─▶ Cloud LB (ALB/NLB) ─▶ NodePort/target ─▶ ingress-nginx pod  (L7: host+path routing, TLS termination)
                                                              │  (nginx upstream = pod IPs from EndpointSlice,
                                                              │   it usually bypasses the Service VIP)
                                                              ▼
                       (or via ClusterIP VIP: kube-proxy iptables/IPVS DNAT) ─▶ Pod IP:8081 ─▶ Spring Boot
 In-cluster caller: pod ─▶ CoreDNS (order-service.default.svc.cluster.local) ─▶ VIP ─▶ DNAT ─▶ Pod IP
```
With the AWS Load Balancer Controller in `ip` target mode, the ALB targets pod IPs directly (skips a node hop). Pods must then be deregistered gracefully (see termination race below).

## 2.4 Pod lifecycle

### Phases (`status.phase`, coarse)
| Phase | Meaning |
|---|---|
| Pending | accepted but not all containers running: unscheduled, pulling images, waiting on init containers |
| Running | bound to a node, >=1 container running/starting/restarting |
| Succeeded | all containers exited 0, won't restart (Jobs) |
| Failed | all terminated, >=1 non-zero / killed |
| Unknown | node unreachable (rare) |
`CrashLoopBackOff`, `ImagePullBackOff`, `OOMKilled`, `Evicted`, `Terminating` are **not phases**; they are container waiting/terminated reasons or kubectl display statuses.

**Pod conditions**: `PodScheduled` -> `Initialized` -> `ContainersReady` -> `Ready`. Only `Ready=True` pods receive Service traffic.

### Container states
`Waiting` (reason: ContainerCreating, ImagePullBackOff, CrashLoopBackOff, CreateContainerConfigError) -> `Running` -> `Terminated` (reason: Completed, Error, OOMKilled; exitCode; signal). `lastState` shows why the previous instance died: `kubectl describe pod`.

### restartPolicy
`Always` (default; Deployments require it), `OnFailure`, `Never` (Jobs use the latter two). Restart is done by the **kubelet on the same node** (the pod is not rescheduled). Backoff: 10s, 20s, 40s ... capped at **5 minutes**, reset after ~10 minutes of clean running. That is the "BackOff" in CrashLoopBackOff.

### Init containers
Run **one at a time, to completion, in order** before app containers start. Use for: wait-for-dependency, DB migration (careful: with many replicas run migrations as a Job/Flyway lock instead), fetching config, chmod on volumes. If one fails, pod restarts per `restartPolicy` (init is retried). Effective pod request = max(largest init request, sum of app requests).

### Sidecars
Classic pattern (log shipper, Envoy proxy) had ordering problems: no start order, and a sidecar in a Job kept the pod from completing. **Native sidecar containers** (beta and on by default from 1.29; GA 1.33): declare them under `initContainers` with `restartPolicy: Always`. They start in order before app containers, keep running for the pod's life, restart independently, and are terminated *after* the main containers (reverse order), so a proxy can outlive the app during shutdown and Jobs can complete. See `sidecar-demo` in section 3.4.

## 2.5 Pod termination sequence and the zero-downtime race

```
 t=0   kubectl delete / rolling update / drain / scale-down
        apiserver sets metadata.deletionTimestamp; pod = Terminating
        ├─ (A) EndpointSlice controller removes pod IP  ──▶ kube-proxy/ingress/ALB reprogram (takes seconds; async!)
        └─ (B) kubelet:  run preStop hook  ──▶ then send SIGTERM to PID 1 of each container
                         wait until terminationGracePeriodSeconds (default 30, counted from t=0, INCLUDES preStop)
                         then SIGKILL (exit 137)
```
**The race**: (A) and (B) happen concurrently and (A) propagates slowly through many components. If the app receives SIGTERM immediately and closes its listener, requests that some nodes/ingress still route to the dying pod get connection refused/502. Fix: **`preStop: sleep 10`** (keeps serving while endpoint removal propagates), then SIGTERM triggers graceful drain.

**Spring Boot alignment** (`application.yml`):
```yaml
server:
  shutdown: graceful                       # stop accepting new, finish in-flight
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s        # max drain time
```
Budget: `preStop (10s) + timeout-per-shutdown-phase (30s) + margin <= terminationGracePeriodSeconds (60s)`. On SIGTERM Spring flips readiness to `REFUSING_TRAFFIC`, drains, closes Kafka consumers/DB pools. Java exits with **143** (128+15) on normal SIGTERM. Make sure PID 1 is the JVM (use `exec java ...` in shell entrypoints, or `ENTRYPOINT ["java", ...]`), otherwise a shell PID 1 swallows SIGTERM and you hit SIGKILL after 30s. Distroless/no-shell images cannot run `exec: sleep`; use the built-in `preStop: sleep: seconds: 10` action (available in newer versions; beta 1.30, verify for your cluster) or an HTTP preStop to an actuator/admin endpoint.
Kafka consumer note: graceful shutdown should `close()` the consumer so the group rebalances promptly rather than waiting for `session.timeout.ms`.

## 2.6 Probes in depth

| Probe | Question | On failure | Typical use |
|---|---|---|---|
| **startup** | has the app finished booting? | kill+restart container after `failureThreshold*periodSeconds` | slow JVM start; **disables liveness/readiness until it succeeds** |
| **liveness** | is it stuck/deadlocked beyond recovery? | **restart container** | catch deadlocks only |
| **readiness** | can it take traffic *right now*? | **remove from Service endpoints** (no restart) | warm-up, overload, dependency down |

Mechanisms: `httpGet` (2xx/3xx = success), `tcpSocket`, `exec` (exit 0), `grpc` (GA 1.27). Defaults: `periodSeconds 10`, `timeoutSeconds 1`, `failureThreshold 3`, `successThreshold 1` (must be 1 for liveness/startup). Probes are executed by the **kubelet** (from the node, hitting pod IP).

**Startup budget example**: JVM takes up to 90s. `startupProbe: periodSeconds: 5, failureThreshold: 36` = 180s allowed; once it passes, liveness with a tight 10s period takes over.

**Liveness too aggressive - classic outage**: `livenessProbe.initialDelaySeconds: 10` on an app that needs 60s -> kubelet kills it at ~10+3*10=40s -> restart -> same again -> `CrashLoopBackOff` forever, and exit code is 137/143 with *no application error in logs* (the app never got to fail). Under load it's worse: GC pause or CPU throttling makes `/health` exceed `timeoutSeconds 1`, all replicas restart together, load shifts, cascading failure. Fixes: startupProbe, longer timeout/threshold, liveness on a cheap endpoint.

**Spring Boot Actuator**: when running on Kubernetes (detected via env) or with `management.endpoint.health.probes.enabled=true`, you get health groups `/actuator/health/liveness` (`LivenessState`: CORRECT/BROKEN) and `/actuator/health/readiness` (`ReadinessState`: ACCEPTING_TRAFFIC/REFUSING_TRAFFIC). By default readiness does **not** include external dependencies. Rules:
- **Liveness must never depend on DB/Kafka/downstream**. If the DB is down and liveness fails, K8s restarts every pod, making recovery worse (thundering herd).
- Readiness may include what *this* pod needs to serve (e.g. local cache warmed); including shared dependencies means all pods go unready at once = full outage instead of degraded (choose deliberately).
- Set `management.endpoint.health.probes.enabled=true` and optionally a separate management port; keep the probe path unauthenticated.

Common misconfigurations: probe port/path wrong (readiness never true -> rollout stuck); readiness = liveness path; `initialDelaySeconds` guessing instead of startupProbe; probing through auth; timeout 1s on a throttled JVM; forgetting `containerPort` naming mismatch (`port: http` needs a named port).

## 2.7 Resource management

### Requests vs limits
- **request** = what the *scheduler* reserves (Filter uses sum of requests vs node **allocatable**) and the kubelet turns into CPU weight (`cpu.weight` / shares) and eviction ranking.
- **limit** = hard ceiling enforced by cgroups. CPU units: `1` = one core, `500m` = half. Memory: `Mi`/`Gi` (powers of two; `M`/`G` decimal).
- CPU is **compressible** (throttled), memory is **incompressible** (killed).

### CPU throttling (CFS quota)
A CPU limit becomes a CFS quota: `cpu.max = limit * 100ms per 100ms period`. A limit of `1` CPU = 100ms of CPU time per 100ms window across all threads. A multi-threaded JVM (GC threads, JIT compiler, request threads) can burn the quota in, say, 30ms on 4 threads, then **all threads freeze for the rest of the 100ms window**: p99 latency spikes, health probes time out, even though average CPU looks low (~50%). Symptom: `container_cpu_cfs_throttled_periods_total` rising; latency spikes at low utilization. Also slow startup (JIT). Mitigation: raise/remove CPU limits (keep requests honest), tune `-XX:ActiveProcessorCount`/thread pools, monitor throttling ratio. Many teams set **CPU request, no CPU limit** and **memory request == memory limit**. That gives QoS Burstable (see below) but avoids throttling; trade-off = noisy neighbors compete by request-weight.

### Memory limit => OOMKilled
Exceed the memory limit (container total RSS incl. off-heap, page cache charged to cgroup) -> kernel OOM killer SIGKILLs the process -> `lastState.terminated.reason: OOMKilled`, **exit code 137** (128+9). The pod restarts (restartPolicy Always) -> may go CrashLoopBackOff. Note: `137` alone can also be the kubelet killing after grace period or a failed liveness probe; `reason: OOMKilled` disambiguates.

### JVM in containers
- Java 10+ (and 8u191+) is **container-aware** (`-XX:+UseContainerSupport` default): reads cgroup memory limit and CPU quota.
- Default max heap = **25%** of container memory (too low). Set **`-XX:MaxRAMPercentage=70`** (or 60-75). Never leave the heap at 100%: total JVM = heap + metaspace + thread stacks (`-Xss` x threads) + code cache + direct/NIO buffers + GC structures + native libs. Example: 1Gi limit x 70% = ~717 MiB heap; remaining ~300 MiB for non-heap.
- `-Xmx` fixed values ignore container size: fine if intentional, dangerous if the limit later shrinks.
- CPU count: JVM sets `availableProcessors()` from the CPU **limit** (quota, rounded up). With **no CPU limit** the JVM sees all node cores (e.g. 16) -> sizes GC/ForkJoin/Tomcat-related pools accordingly; with a tiny limit (e.g. 250m) it sees 1 CPU -> may choose SerialGC (JVM picks Serial on <2 CPUs or <~1792MB memory "not server class"). Set `-XX:ActiveProcessorCount=N` explicitly if needed. (JDK 19+ no longer derives CPU count from `cpu.shares` requests; verify behavior on your JDK.)
- `-XX:+ExitOnOutOfMemoryError` makes a Java heap OOM crash the container quickly (so K8s restarts) instead of a zombie JVM. Heap OOM (`java.lang.OutOfMemoryError`) is different from container OOMKilled (kernel).

### QoS classes and eviction
| Class | Rule | Eviction priority |
|---|---|---|
| **Guaranteed** | every container has cpu **and** memory requests == limits | last to be evicted; lowest `oom_score_adj` |
| **Burstable** | at least one request/limit, not Guaranteed | middle; ranked by usage above requests |
| **BestEffort** | no requests/limits at all | first to go |
Under node memory pressure the **kubelet evicts** pods (soft/hard thresholds e.g. `memory.available<100Mi`): BestEffort and Burstable pods **exceeding their requests** first, ordered by priority then by how far over request. Evicted pods end `Failed` with reason `Evicted` (disk pressure, ephemeral storage too). Distinct from the kernel OOM killer (per-cgroup) which also uses `oom_score_adj` derived from QoS. `ephemeral-storage` requests/limits also exist.

### LimitRange and ResourceQuota
- **LimitRange** (namespace): default/min/max requests+limits per container/pod so a pod with none still gets sane values.
- **ResourceQuota** (namespace): caps totals: `requests.cpu`, `limits.memory`, `pods`, `persistentvolumeclaims`, object counts. If a quota on cpu/memory exists, pods without requests/limits are **rejected** (unless LimitRange supplies defaults).

## 2.8 Deployments

```
Deployment ──owns──▶ ReplicaSet(v1: 0 pods)   ← kept for rollback (revisionHistoryLimit, default 10)
           └─owns──▶ ReplicaSet(v2: N pods)
```
A change to `spec.template` (image, env, resources...) creates a new ReplicaSet; scale-only changes don't. Changing a ConfigMap does **not** (that is why config changes need `kubectl rollout restart` or a checksum annotation).

### Rolling update math
- `maxSurge`: extra pods above `replicas` allowed during update (absolute or %; % rounds **up**).
- `maxUnavailable`: pods below `replicas` allowed to be unready (% rounds **down**).
- Bounds: total pods <= `replicas + maxSurge`; available pods >= `replicas - maxUnavailable`.

Example `replicas: 10`, `maxSurge: 25%`, `maxUnavailable: 25%`: surge = ceil(2.5) = **3**, unavailable = floor(2.5) = **2**. So at most **13** pods exist and at least **8** are Ready at any moment. Defaults are 25%/25%.
Example `replicas: 3`, `maxSurge: 1`, `maxUnavailable: 0` (zero-downtime, needs spare capacity):
```
t0   v1 v1 v1                       ready=3 total=3
t1   v1 v1 v1 [v2 starting]         total=4 (surge=1), ready=3   (v2 not Ready yet - old not touched)
t2   v1 v1 v1  v2(Ready)            ready=4
t3   v1 v1 (v1 Terminating) v2      old pod removed -> ready=3
t4   v1 v1 v2 [v2 starting] ... repeat until v2 v2 v2
```
`maxUnavailable: 0` + `maxSurge: 0` is invalid (deadlock). If the new pod never becomes Ready, the rollout **stalls safely** with old pods still serving; after `progressDeadlineSeconds` (default 600) the Deployment condition `Progressing=False, reason ProgressDeadlineExceeded` (it does not auto-roll back).
`minReadySeconds`: pod must be Ready this long before counted available (guards against flapping).

### Commands and strategies
```
kubectl rollout status deploy/order-service           # blocks until done or deadline; non-zero exit on failure (CI!)
kubectl rollout history deploy/order-service [--revision=3]
kubectl rollout undo deploy/order-service [--to-revision=2]
kubectl rollout pause|resume deploy/order-service     # batch several edits into one rollout
kubectl rollout restart deploy/order-service          # new template annotation -> rolling restart
```
- **Recreate**: kill all old, then start new (downtime; used when two versions can't coexist, e.g. schema-incompatible singleton).
- **Blue-green**: run v1 and v2 Deployments; flip Service `selector` (`version: green`) or Ingress backend atomically; instant rollback; costs 2x capacity.
- **Canary**: small share to v2. Simple: two Deployments behind one Service (traffic ~ replica ratio, 1 of 10 pods = ~10%). Precise: ingress-nginx canary annotations (`nginx.ingress.kubernetes.io/canary-weight`), Gateway API `HTTPRoute` weighted `backendRefs`, service mesh, or **Argo Rollouts** (`Rollout` CRD with `steps: setWeight, pause, analysis` that auto-aborts on bad Prometheus metrics).
- Database schema must be **backward compatible** (expand/contract) because old and new pods run simultaneously and rollbacks happen.

## 2.9 Services, DNS, Ingress, Gateway API, NetworkPolicy

### Service types
| Type | What | Use |
|---|---|---|
| **ClusterIP** (default) | virtual IP reachable in-cluster only | service-to-service |
| **NodePort** | opens a port (30000-32767) on every node -> Service | dev/bare metal; base of LB |
| **LoadBalancer** | asks cloud for an external LB (NLB/ALB), implies NodePort+ClusterIP | one Service = one paid LB |
| **ExternalName** | DNS CNAME to an external name; no proxying | alias to RDS/SaaS |
| **Headless** (`clusterIP: None`) | no VIP; DNS returns pod IPs | StatefulSets, client-side LB, gRPC |
- Backends = pods matching `selector` **and Ready** (via EndpointSlices, up to 100 endpoints per slice). Selector typo -> Service with **no endpoints** (`kubectl get endpointslices` / `kubectl get endpoints`).
- `port` (Service) vs `targetPort` (container; can be a named port) vs `nodePort`.
- `sessionAffinity: ClientIP` pins a client IP to a pod (timeout default 10800s). Otherwise load balancing is per **connection**, not per request: long-lived HTTP/2/gRPC connections stick to one pod (need headless+client LB or a mesh/L7 proxy).
- `externalTrafficPolicy: Local` preserves client source IP and skips a hop but only nodes with local pods pass LB health checks.

### Ingress vs Gateway API
- **Ingress**: an API object (host/path -> Service, TLS). Does nothing alone; needs an **ingress controller** (ingress-nginx, AWS Load Balancer Controller creating ALBs, Traefik, HAProxy, Istio). `ingressClassName` picks the controller. Limitations: HTTP only, behavior via vendor-specific annotations, no role separation. Note: the community ingress-nginx project announced retirement/end of maintenance in 2026; verify current status before choosing it for new clusters.
- **Gateway API** (GA v1.0 for core resources): `GatewayClass` (infra provider) -> `Gateway` (listeners/ports/TLS, owned by platform team) -> `HTTPRoute`/`GRPCRoute`/`TCPRoute` (app teams: matches, header rules, **weighted backends, redirects, mirroring** natively). Role-oriented, portable, more expressive; it is where the ecosystem is moving.

### NetworkPolicy
Default: **all pods can talk to all pods**. A NetworkPolicy selects pods and, once any policy selects a pod for a direction, that direction becomes **default deny except what's allowed**. Enforced by the **CNI** (Calico, Cilium; the default AWS VPC CNI needs network policy support enabled, Flannel alone doesn't enforce). Pattern: namespace-wide default-deny ingress+egress, then allow: ingress from ingress-controller namespace, egress to DNS (UDP/TCP 53) + DB. Forgetting egress to CoreDNS is a classic self-inflicted outage.

## 2.10 ConfigMap and Secret

| | Env var (`env`/`envFrom`) | Volume mount |
|---|---|---|
| Update propagation | **never** (fixed at container start) | updates in place eventually (kubelet sync, up to ~1-2 min); **not** for `subPath` mounts |
| App must | be restarted | re-read the file (Spring Cloud Kubernetes reload, or restart) |
| Notes | simple, visible in `describe`/`env` | good for files (`application.yml`, certs) |
- To roll pods on config change: `kubectl rollout restart`, Helm checksum annotation (`checksum/config: {{ include ... | sha256sum }}`), Reloader/operator, or immutable ConfigMaps with versioned names (`immutable: true` also reduces apiserver load).
- **Secrets are only base64-encoded**, not encrypted: anyone with `get secret` (or etcd access) reads them: `kubectl get secret x -o jsonpath='{.data.PW}' | base64 -d`. Hardening: **encryption at rest** (apiserver `EncryptionConfiguration`; with a **KMS provider** for envelope encryption; EKS has envelope encryption with KMS), tight RBAC (nobody gets `list secrets` broadly), avoid env-var secrets where logs/crash dumps leak them (prefer files), **external secret managers**: External Secrets Operator or Secrets Store CSI Driver syncing from AWS Secrets Manager/Vault, Sealed Secrets/SOPS for GitOps (never commit plain Secret YAML), rotation strategy.
- Size limit 1 MiB per object. `stringData` accepts plaintext and is stored base64 in `data`.

## 2.11 Storage

```
 Pod ──volumeMounts──▶ volume ──▶ PVC (claim: "10Gi RWO gp3") ──binds──▶ PV (actual EBS vol)
                                             ▲                              ▲
                                     StorageClass (provisioner: ebs.csi.aws.com) dynamically creates PV
```
- **PV**: cluster-level storage resource. **PVC**: namespaced request. **StorageClass**: how to provision (type, `reclaimPolicy`, `volumeBindingMode`, `allowVolumeExpansion`).
- **Access modes**: `ReadWriteOnce` (one *node*; many pods on that node can share), `ReadOnlyMany`, `ReadWriteMany` (needs NFS/EFS/CephFS), `ReadWriteOncePod` (single pod, GA 1.29). EBS is RWO and **zonal**.
- **volumeBindingMode: WaitForFirstConsumer** delays provisioning until the pod is scheduled so the volume is created in the pod's zone (avoids "volume node affinity conflict" Pending pods).
- **Reclaim policy**: `Delete` (PV and backing disk removed with PVC; dynamic default) vs `Retain` (data kept; manual cleanup). Use Retain/snapshots for anything precious.
- Ephemeral: `emptyDir` (dies with pod; also `medium: Memory`), needed for `readOnlyRootFilesystem` (`/tmp`).

### StatefulSet
- **Stable identity**: pods `kafka-0, kafka-1, kafka-2` (ordinal names survive rescheduling), stable DNS via a **headless Service** (`kafka-0.kafka-headless.ns.svc.cluster.local`), **per-pod PVC** from `volumeClaimTemplates` (`data-kafka-0`) that re-attaches to the same ordinal.
- **Ordered**: create 0->N-1 (each Ready before next), delete/scale-down N-1->0, rolling update reverse-ordinal. `podManagementPolicy: Parallel` relaxes start order. `updateStrategy.rollingUpdate.partition` for staged updates.
- PVCs are **not deleted** on scale-down/delete by default (`persistentVolumeClaimRetentionPolicy` can change this in newer versions).
- **Why running Kafka/DBs on K8s needs care**: data on zonal network disks (pod must reschedule into the same zone); pod restarts trigger replication/rebalance/leader election storms; PDBs and drain ordering must respect quorum; fsync latency of network storage; upgrades of stateful software need operator logic; node autoscaler scale-down evicting brokers; JVM/page-cache memory limits. Recommended: use an **Operator** (Strimzi for Kafka, CloudNativePG/Percona for databases), anti-affinity across zones, `minAvailable` PDB, or just use the managed service (MSK, RDS). Say "I run stateless on K8s; stateful data stores as managed services unless there's a strong reason."

## 2.12 Other workloads
- **DaemonSet**: one pod per (matching) node: log agents (Fluent Bit), node exporters, CNI/CSI agents. Tolerate taints as needed.
- **Job**: run-to-completion: `completions`, `parallelism`, `backoffLimit`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished` (auto-clean). `restartPolicy` must be `Never`/`OnFailure`. Indexed Jobs for sharding work.
- **CronJob**: creates Jobs on a cron `schedule` (`timeZone` field GA 1.27, else controller-manager TZ). `concurrencyPolicy`: `Allow` (default; overlapping runs), `Forbid` (skip new run if previous still going), `Replace` (kill old, start new). `startingDeadlineSeconds` (missed-run tolerance; >100 missed schedules without a deadline = stops scheduling), `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`. Jobs must be idempotent (at-least-once, rare double runs).

## 2.13 Scaling and placement

### HPA (Horizontal Pod Autoscaler)
Control loop (default every 15s) reading **metrics-server** (CPU/memory; `kubectl top` uses it) or custom/external metrics adapters:
```
desiredReplicas = ceil( currentReplicas * currentMetricValue / targetMetricValue )
```
Worked example: 4 pods, target 70% CPU **utilization (of requests!)**, current average 105% -> `ceil(4 * 105/70) = ceil(6.0) = 6`. Current 40% -> `ceil(4*40/70) = ceil(2.29) = 3`. Details: ignored inside a ~10% **tolerance** (0.9-1.1 ratio -> no change); not-Ready/just-started pods are treated conservatively so they don't distort; clamped to `min/maxReplicas`; multiple metrics -> the **max** proposal wins.
**Utilization is relative to *requests***: no CPU request = HPA can't compute; a request that is too high = HPA never scales; too low = scales too eagerly.
**Stabilization windows** (`behavior`): scale-down default **300s** (uses the highest recommendation in the window; prevents flapping), scale-up default 0s. `policies` cap rate (e.g. max +100% or 4 pods per 15s; -1 pod/min).
Don't combine HPA-on-CPU with VPA-auto on CPU for the same workload. Don't set `replicas` in the Deployment manifest under GitOps if HPA owns it (Argo will fight; omit or ignore the diff).
**Custom/external metrics**: Prometheus Adapter or **KEDA** (event-driven; scales 0->N; scalers for Kafka consumer-group **lag**, SQS depth, cron, Prometheus queries; creates and manages an HPA). Kafka caveat: replicas beyond partition count are idle.
- **VPA**: recommends/sets requests (modes Off/Initial/Auto (evicts to apply; in-place resize maturing in newer versions)). Great in `Off` mode to right-size requests.
- **Cluster Autoscaler**: adds nodes when pods are **Pending due to insufficient resources**, removes underutilized nodes (evicting pods, respects PDBs, `safe-to-evict` annotation). **Karpenter** (AWS): provisions right-sized instances directly from pending pods (NodePool/EC2NodeClass), consolidates nodes; faster and cheaper than node-group-based scaling. Chain: load up -> HPA adds pods -> pods Pending -> autoscaler adds nodes.

### PodDisruptionBudget
Limits **voluntary** disruptions (node drain, `kubectl drain`, autoscaler scale-down, eviction API): `minAvailable: 2` or `maxUnavailable: 1`. Doesn't protect against involuntary loss (node crash) or delete/rolling update of your own Deployment (that uses maxUnavailable). Pitfall: `minAvailable` == replicas (or `maxUnavailable: 0`) **blocks node drains and upgrades forever**.

### Placement
- `nodeSelector` (simple label match); **nodeAffinity** (`required...IgnoredDuringExecution` = hard, `preferred...` = weighted soft).
- **podAntiAffinity** (`topologyKey: kubernetes.io/hostname` or `topology.kubernetes.io/zone`): keep replicas apart; `podAffinity` to co-locate (cache next to app). Required anti-affinity across hostnames with more replicas than nodes leaves pods Pending; prefer soft or topology spread.
- **topologySpreadConstraints** (`maxSkew`, `topologyKey`, `whenUnsatisfiable: DoNotSchedule|ScheduleAnyway`, `labelSelector`): even spread across zones/nodes, modern replacement for most anti-affinity uses.
- **Taints/tolerations**: taint repels (`key=value:NoSchedule|PreferNoSchedule|NoExecute`), toleration allows (does not attract; combine with nodeSelector/affinity to dedicate nodes, e.g. GPU pools). Node lifecycle taints: `node.kubernetes.io/not-ready`, `unreachable`, `memory-pressure`, `disk-pressure`; pods get a default 300s toleration for not-ready/unreachable before `NoExecute` eviction.
- **PriorityClass**: numeric priority; when a high-priority pod is Pending, the scheduler can **preempt** (evict) lower-priority pods. `system-cluster-critical`/`system-node-critical` reserved for core add-ons. `preemptionPolicy: Never` for non-disruptive priority.

## 2.14 RBAC and security

### RBAC
- **Role** (namespaced) / **ClusterRole** (cluster-wide or non-namespaced resources): `rules: apiGroups, resources, verbs` (additive only, no deny).
- **RoleBinding** / **ClusterRoleBinding** bind a role to subjects (User, Group, **ServiceAccount**). A RoleBinding may reference a ClusterRole to reuse it inside one namespace.
- **ServiceAccount**: identity for pods; token projected into the pod (bound, expiring, audience-scoped since 1.22+). Set `automountServiceAccountToken: false` if the app doesn't call the API. Each namespace has a `default` SA; don't grant it anything; use one SA per workload.
- Least privilege: specific verbs and `resourceNames`; avoid `*`; avoid `secrets` list/watch, `pods/exec`, `escalate/bind/impersonate`, `create pods` (a way to mount any secret). Check: `kubectl auth can-i list secrets --as system:serviceaccount:prod:order-sa -n prod`; `kubectl auth can-i --list`.
- **IRSA on EKS**: annotate the ServiceAccount `eks.amazonaws.com/role-arn: arn:aws:iam::123:role/order-s3`; the projected OIDC token is exchanged with STS (`AssumeRoleWithWebIdentity`) by the AWS SDK -> pod gets temporary IAM credentials scoped to *that pod's* role, no node-wide instance-profile access and no static keys. (EKS Pod Identity is the newer alternative.) Java: use the default credentials provider chain; it picks up `AWS_WEB_IDENTITY_TOKEN_FILE` automatically.

### Pod security
- **Pod Security Standards** enforced by the built-in **Pod Security Admission** via namespace labels: `pod-security.kubernetes.io/enforce=restricted` (also `audit`, `warn`). Levels: `privileged` (anything), `baseline` (blocks known escalations: hostNetwork, privileged), `restricted` (must runAsNonRoot, drop ALL caps, no privilege escalation, seccomp RuntimeDefault).
- `securityContext`: `runAsNonRoot: true`, `runAsUser`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true` (mount `emptyDir` for `/tmp`; Spring/Tomcat need a writable tmp), `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`, `fsGroup` for volume ownership.
- Supply chain: scan images (Trivy/Grype/ECR scanning) in CI, minimal base images (distroless/jlink), pin by digest, sign (cosign) and verify in admission (Kyverno/Sigstore policy-controller), private registry with `imagePullSecrets` (or node IAM/IRSA for ECR).
- **Admission controllers** = last gate: built-in (`LimitRanger`, `ResourceQuota`, `PodSecurity`, `NodeRestriction`), webhooks, `ValidatingAdmissionPolicy` (CEL, GA 1.30), policy engines (OPA Gatekeeper, Kyverno) to enforce "no `latest` tag, must have limits, no privileged".
- Also: NetworkPolicy, audit logs, etcd encryption, restrict API server endpoint, regular upgrades (support window ~14 months per minor).

### Namespaces and multi-tenancy
Namespaces scope names, RBAC, quotas, policies; they are a **soft** boundary (shared nodes/kernel/cluster-scoped resources). Soft multitenancy: namespace per team/env + RBAC + ResourceQuota + LimitRange + NetworkPolicy + PSA labels. Hard isolation: separate clusters (or vClusters/dedicated node pools with taints). Prefer separate clusters for prod vs non-prod. Cross-namespace call: `svc.namespace`.

### Service mesh (mention)
Istio/Linkerd inject a proxy (sidecar or, in Istio ambient, per-node) for mTLS, retries, timeouts, traffic splitting, L7 metrics, without app changes. Cost: latency, complexity, resource overhead. Use when you need uniform mTLS/traffic shaping across many services; not for one or two apps.

---

# 3. Full YAML: deploying a Spring Boot service end to end

Validation status: the manifests below are parsed as valid multi-document YAML (7 + 8 documents, checked with a YAML parser). `kubectl` v1.34.1 is installed on this machine but no cluster was reachable (Docker Desktop's Kubernetes context was down), and even `--dry-run=client` needs API discovery, so schema validation was **not** run. Validate on your cluster with:
```
kubectl apply --dry-run=client -f app.yaml     # local schema check (needs API discovery)
kubectl apply --dry-run=server -f app.yaml     # runs admission too, changes nothing
kubectl diff -f app.yaml                       # what would change
```

## 3.1 The manifest set (`app.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8081"
  KAFKA_BOOTSTRAP: "kafka-0.kafka-headless.data.svc.cluster.local:9092"
---
apiVersion: v1
kind: Secret
metadata:
  name: order-secret
type: Opaque
stringData:
  DB_PASSWORD: "change-me"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: order-service
    spec:
      terminationGracePeriodSeconds: 60
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: order-service
      containers:
        - name: order
          image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.4.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8081
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=70 -XX:+ExitOnOutOfMemoryError"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-secret
                  key: DB_PASSWORD
          envFrom:
            - configMapRef:
                name: order-config
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              memory: 1Gi
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 5
            failureThreshold: 36
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "10"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: orders.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

## 3.2 Line-by-line explanation

**ConfigMap / Secret**
- `SPRING_PROFILES_ACTIVE`, `SERVER_PORT`: Spring's relaxed binding maps env vars to properties (`SERVER_PORT` -> `server.port`). Loaded with `envFrom` so every key becomes an env var.
- `Secret` with `stringData`: convenience for plaintext input; the API stores it base64 in `data`. In real life this is not in Git: use External Secrets/Sealed Secrets. `type: Opaque` = arbitrary key/values.

**Deployment**
- `replicas: 3`: baseline. If the HPA exists, prefer omitting `replicas` under GitOps so the two don't fight.
- `revisionHistoryLimit: 5`: keep 5 old ReplicaSets for `rollout undo` (default 10; each is an etcd object).
- `progressDeadlineSeconds: 300`: mark rollout failed (`ProgressDeadlineExceeded`) if no progress for 5 min. It only reports; it does not roll back.
- `selector.matchLabels` must match `template.metadata.labels`, and is **immutable** after creation. Never let two Deployments' selectors overlap.
- `strategy RollingUpdate maxSurge 1 / maxUnavailable 0`: always at least 3 Ready pods; one extra pod during the update (needs node capacity for +1).
- `terminationGracePeriodSeconds: 60`: total time budget from delete to SIGKILL: preStop (10) + Spring drain (up to 30) + margin.
- Pod-level `securityContext`: `runAsNonRoot` (the image must have a numeric non-root user or you set `runAsUser: 10001`), `seccompProfile: RuntimeDefault` (Pod Security "restricted" requirement).
- `topologySpreadConstraints`: spread pods evenly over zones (`maxSkew: 1`); `ScheduleAnyway` = soft so a zone outage/small cluster doesn't leave pods Pending. `labelSelector` picks which pods count.
- `image: ...:1.4.0`: immutable version tag (never `latest`; prefer digest). `imagePullPolicy: IfNotPresent` is the default for non-latest tags.
- `ports.name: http` lets probes and Services say `port: http`, so port numbers live in one place.
- `JAVA_TOOL_OPTIONS`: picked up automatically by any JVM. `MaxRAMPercentage=70` sizes heap from the container limit (1 Gi -> ~717 MiB); `ExitOnOutOfMemoryError` makes heap OOM kill the JVM so Kubernetes restarts it cleanly.
- `env` `DB_PASSWORD` from `secretKeyRef`: one key from the Secret; `envFrom configMapRef`: all keys of the ConfigMap. (If both define the same key, `env` wins.)
- `resources`: `requests cpu 500m, memory 1Gi` = what the scheduler reserves and HPA's 70% is measured against. `limits.memory 1Gi` (= request) protects the node and makes memory behavior predictable; **no CPU limit on purpose** (avoid CFS throttling). Result QoS = **Burstable**. To get Guaranteed you would add `limits.cpu: 500m` (and accept throttling).
- `startupProbe`: `periodSeconds 5 x failureThreshold 36` = up to 180s to boot. Until it succeeds, liveness/readiness are **not** run.
- `readinessProbe`: every 5s, 2s timeout, 3 consecutive failures = removed from Service. On SIGTERM Spring flips it to REFUSING_TRAFFIC.
- `livenessProbe`: only checks the app's own liveness state, 3 x 10s = 30s of continuous failure before restart; no external dependencies.
- `lifecycle.preStop exec sleep 10`: the endpoint-propagation race fix (see 2.5). Requires a `sleep` binary in the image.
- Container `securityContext`: no privilege escalation, read-only root FS, drop all Linux capabilities.
- `volumeMounts /tmp` + `emptyDir`: Tomcat/JVM need a writable tmp when the root FS is read-only.

**Service**: `ClusterIP`, selector `app: order-service` (the endpoint list = Ready pods with that label), `port: 80` (what callers use) -> `targetPort: http` (named container port 8081).

**Ingress**: requires an installed controller matching `ingressClassName`. Routes `orders.example.com/*` to Service port 80. Add `tls:` with a Secret (cert-manager) for HTTPS.

**HPA** (`autoscaling/v2`): 3-10 replicas, target average CPU utilization 70% of *requests* (500m -> ~350m per pod). `behavior.scaleDown`: wait 300s of consistently lower recommendations, remove max 1 pod/min, so a traffic dip doesn't shred capacity. Requires metrics-server.

**PDB**: `minAvailable: 2` keeps at least 2 pods during voluntary disruptions (node drain, autoscaler). With 3 replicas, one node can drain at a time. Do not set it equal to replicas.

## 3.3 Graceful shutdown and probe wiring for Spring Boot

`application.yml`:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
management:
  endpoint:
    health:
      probes:
        enabled: true          # /actuator/health/liveness and /readiness
      show-details: never
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```
Timeline on a rolling update, one old pod:
```
t=0    pod Terminating. EndpointSlice ctrl removes IP (propagating)  |  kubelet starts preStop: sleep 10
t=0-10 pod STILL serves; ingress/ALB/kube-proxy converge, new requests go elsewhere
t=10   SIGTERM -> Spring: readiness=REFUSING_TRAFFIC, stops accepting, waits for in-flight (<=30s)
t<=40  context closed (Kafka consumers, Hikari pool), JVM exits 143
t=60   (only if still alive) SIGKILL
```

## 3.4 Extra manifests (`extra.yaml`): StatefulSet, headless Service, CronJob, NetworkPolicy, RBAC, native sidecar

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  selector:
    matchLabels: {app: kafka}
  template:
    metadata:
      labels: {app: kafka}
    spec:
      containers:
        - name: kafka
          image: apache/kafka:3.7.0
          volumeMounts:
            - {name: data, mountPath: /var/lib/kafka}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
spec:
  clusterIP: None
  selector: {app: kafka}
  ports:
    - {name: broker, port: 9092}
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
            - {name: report, image: busybox:1.36, command: ["sh","-c","echo report"]}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-only
spec:
  podSelector:
    matchLabels: {app: order-service}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - {protocol: TCP, port: 8081}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cm-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list","watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-cm-reader
subjects:
  - {kind: ServiceAccount, name: order-sa}
roleRef:
  kind: Role
  name: cm-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  initContainers:
    - name: log-shipper
      image: busybox:1.36
      restartPolicy: Always
      command: ["sh","-c","tail -F /var/log/app.log"]
  containers:
    - {name: app, image: busybox:1.36, command: ["sleep","3600"]}
```

## 3.5 Notes on the extras
- **StatefulSet**: `serviceName` must reference the headless Service; `volumeClaimTemplates` creates `data-kafka-0/1/2`; deleting a pod recreates `kafka-1` and re-attaches `data-kafka-1`. (Illustrative skeleton only; real Kafka needs broker config, KRaft/listeners; use Strimzi.)
- **CronJob**: `concurrencyPolicy: Forbid` = skip a run if the previous is still running; `restartPolicy: Never` + `backoffLimit: 2` retry by new pods.
- **NetworkPolicy**: selecting `order-service` pods for Ingress means *only* the `ingress-nginx` namespace on TCP 8081 can reach them; everything else is denied (requires an enforcing CNI). Egress is unrestricted here since `policyTypes` lists only Ingress.
- **RBAC**: `order-sa` can only get/list/watch ConfigMaps in its own namespace (what Spring Cloud Kubernetes config reload needs). Set `serviceAccountName: order-sa` in the pod spec to use it.
- **Native sidecar**: `restartPolicy: Always` inside `initContainers` (1.29+ beta enabled by default) = a sidecar that starts before `app`, runs alongside it, and stops after it.

## 3.6 Helm and Kustomize

### Helm (package manager + templating + release tracking)
```
order-chart/
  Chart.yaml          # name, version (chart), appVersion, dependencies
  values.yaml         # default config (image.tag, replicas, resources, ingress.host)
  values-prod.yaml    # overrides (kept in Git per environment)
  templates/
    deployment.yaml   # Go templates: image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    service.yaml  ingress.yaml  hpa.yaml  configmap.yaml
    _helpers.tpl      # named templates (labels, fullname)
    NOTES.txt  tests/
  charts/             # subcharts / dependencies
```
- Config change rolls pods: `annotations: checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`.
- **Hooks**: annotate a manifest, e.g. a DB-migration Job with `"helm.sh/hook": pre-install,pre-upgrade`, `"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded`. Hooks run outside the normal release manifest lifecycle.
- Commands:
```
helm lint ./order-chart
helm template order ./order-chart -f values-prod.yaml          # render locally, no cluster
helm upgrade --install order ./order-chart -n prod --create-namespace -f values-prod.yaml --set image.tag=1.5.0 --atomic --wait --timeout 5m
helm history order -n prod          # revisions
helm rollback order 3 -n prod       # to revision 3 (creates a NEW revision)
helm uninstall order -n prod
helm list -A ; helm get values order -n prod ; helm get manifest order -n prod
```
`--atomic` rolls back automatically if the upgrade fails/times out (`--wait` waits for readiness). Helm 3 has no Tiller; release state is stored as Secrets (`sh.helm.release.v1.*`) in the namespace. Gotchas: CRDs in `crds/` are installed once and not upgraded by Helm; `helm rollback` doesn't undo DB migrations; `helm diff` plugin previews changes.

### Kustomize (patch, don't template) - built into `kubectl`
```
base/            kustomization.yaml + deployment.yaml service.yaml
overlays/prod/   kustomization.yaml  (resources: [../../base]; patches: replicas 6; images: newTag: 1.5.0; configMapGenerator)
kubectl kustomize overlays/prod        # render
kubectl apply -k overlays/prod
```
`configMapGenerator` appends a content hash to the ConfigMap name, so a config change automatically rolls the Deployment. Helm = parameterized packages and third-party charts; Kustomize = plain YAML with overlays; many teams use both (Argo/Flux can render either).

### GitOps (ArgoCD / Flux)
Git is the source of truth; an in-cluster agent **pulls** and reconciles (same control-loop idea applied to whole environments). CI builds+pushes an image and bumps the tag in the config repo; the agent syncs.
- **Argo CD**: `Application` CR (repo, path, destination cluster/namespace, `syncPolicy.automated: {prune: true, selfHeal: true}`); UI showing drift (OutOfSync) and health; sync waves/hooks; ApplicationSets for many clusters/envs.
- **Flux**: `GitRepository` + `Kustomization`/`HelmRelease` CRs; controllers toolkit, image automation.
- Benefits: audit trail (git log), rollback = `git revert`, drift correction, no cluster credentials in CI. Manual `kubectl edit` in prod is undone by self-heal. Keep secrets out of Git (Sealed Secrets/SOPS/External Secrets).

## 3.7 Observability essentials
- `kubectl logs <pod> [-c container] [--previous] [-f] [-l app=order-service --tail=100]` (only stdout/stderr; log to console in containers; ship with Fluent Bit/Loki/ELK).
- `kubectl top pod|node` (needs metrics-server; instantaneous, not history).
- `kubectl get events --sort-by=.lastTimestamp -A` (events expire after ~1h).
- **Prometheus + Grafana** (kube-prometheus-stack): scrape `/actuator/prometheus` via `ServiceMonitor`/`PodMonitor`; alert on `kube_pod_container_status_restarts_total`, `container_cpu_cfs_throttled_periods_total`, `kube_pod_status_phase`, `kube_deployment_status_replicas_unavailable`, JVM heap/GC, HTTP p99. kube-state-metrics exposes object state; node-exporter exposes host metrics.
- Tracing: OpenTelemetry/Jaeger/Tempo; correlation IDs in logs.
- `kubectl debug` ephemeral containers (`kubectl debug -it pod/x --image=busybox --target=order`) for distroless images; `kubectl debug node/x -it --image=ubuntu`.

## 3.8 Local practice (minikube / kind)
```
minikube version ; kubectl version --client            # checked here: minikube v1.35.0, kubectl client v1.34.1
minikube start --driver=docker --cpus=4 --memory=6g    # single-node cluster
minikube addons enable metrics-server                  # enables kubectl top and HPA
minikube addons enable ingress                         # ingress-nginx
eval $(minikube docker-env)   # build images straight into the cluster's runtime (bash); or: minikube image load order-service:1.4.0
kubectl apply -f app.yaml                              # set image to a local one, imagePullPolicy: IfNotPresent/Never
minikube service order-service --url                   # NodePort/LB access;  minikube tunnel for LoadBalancer
minikube dashboard ; minikube delete
# kind alternative (containers as nodes; supports multi-node):
kind create cluster --name lab ; kind load docker-image order-service:1.4.0 --name lab
```
Experiments worth doing: kill a pod and watch the ReplicaSet recreate it; set a bad readiness path and watch `rollout status` hang while old pods serve; set `limits.memory: 100Mi` to see `OOMKilled`/137; `kubectl drain` with a PDB; load test with `hey`/`k6` and watch HPA (`kubectl get hpa -w`). Note: in local dev use `kubectl port-forward svc/order-service 8080:80`.

---

# 4. Production war stories (symptom -> diagnosis -> fix)

**Story 1: 502s during every deployment**
- Symptom: brief 502/connection-reset spike at each rollout; app logs clean.
- Diagnosis: `kubectl get pod -w` during a rollout; time between `Terminating` and ALB/ingress deregistration; no `preStop`; app closes its port at once on SIGTERM.
- Fix: `preStop sleep 10`, `server.shutdown=graceful`, grace period >= preStop + drain; on ALB, tune deregistration delay/readiness gates. Verify by load-testing a rollout.

**Story 2: CrashLoopBackOff only in prod, logs empty**
- Symptom: pods restart every ~40s; `kubectl logs` shows only Spring startup lines.
- Diagnosis: `kubectl describe pod` -> `Liveness probe failed: Get ... context deadline exceeded`, `Last State: Terminated Exit Code 137`; startup takes 70s under a 250m CPU limit.
- Fix: add `startupProbe`, raise CPU request/remove limit, lengthen timeouts; the app itself was healthy.

**Story 3: OOMKilled every few hours**
- Symptom: `RESTARTS` climbing; `Last State: OOMKilled`, exit 137.
- Diagnosis: `kubectl describe`; Grafana `container_memory_working_set_bytes` flat-lines at limit; heap dump shows heap is 25% of limit (default) but off-heap/direct buffers (Netty, Kafka) and threads growing; `-XX:NativeMemoryTracking=summary` + `jcmd VM.native_memory`.
- Fix: `MaxRAMPercentage=60-70`, cap direct memory (`-XX:MaxDirectMemorySize`), bound thread pools, raise the limit with data; fix the leak if any. If Java heap OOM instead: heap dump on OOM to a volume.

**Story 4: p99 latency spikes at 30% CPU**
- Diagnosis: `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])` high; JVM sees few CPUs but GC + request threads burst.
- Fix: remove/raise CPU limit, right-size request via VPA recommendations, review `ActiveProcessorCount`, thread pools, GC choice.

**Story 5: Pods Pending after a scale-up: "Insufficient cpu"**
- Diagnosis: `kubectl describe pod` -> `0/6 nodes are available: 6 Insufficient cpu`; `kubectl describe node` -> `Allocated resources` near 100% of *requests* while `kubectl top node` shows 30% actual.
- Fix: requests were inflated (copy-paste 2 CPU); right-size with VPA/Goldilocks, add nodes (Cluster Autoscaler/Karpenter check logs and whether the node group max was hit).

**Story 6: Node drain hangs during cluster upgrade**
- Diagnosis: `kubectl drain` says `Cannot evict pod ... would violate the pod's disruption budget`; `kubectl get pdb -A` shows `ALLOWED DISRUPTIONS 0` (minAvailable == replicas, or an unhealthy pod).
- Fix: `maxUnavailable: 1` / minAvailable = replicas-1, ensure replicas > 1, fix unready pods, don't `--disable-eviction` blindly.

**Story 7: Config change didn't take effect**
- Symptom: edited ConfigMap, app still on old value.
- Diagnosis: value injected as env var (never updates) or `subPath` mount; pods not restarted.
- Fix: `kubectl rollout restart`, checksum annotation, or Reloader.

**Story 8: Everything restarted when the DB blipped**
- Diagnosis: liveness endpoint included DB health; DB failover of 40s -> all pods killed simultaneously -> connection storm on recovery.
- Fix: liveness only internal state; readiness may degrade; add retries/circuit breakers; connection pool limits.

**Story 9: Intermittent DNS failures / slow external calls**
- Diagnosis: `ndots:5` making external names hit search domains; CoreDNS pods CPU-throttled or conntrack races; 5s delays (UDP conntrack/parallel A+AAAA race in older kernels).
- Fix: NodeLocal DNSCache, scale CoreDNS, FQDN with trailing dot, `dnsConfig ndots: 2`, JVM `networkaddress.cache.ttl` awareness (DNS caching so pods don't stick to dead IPs).

**Story 10: Rollout stuck at 1 new pod, old pods OK**
- Diagnosis: `kubectl rollout status` waits; new pod `0/1 Running`; readiness probe hitting wrong path or dependent config missing; or `ImagePullBackOff`; or quota exceeded (`kubectl describe rs` -> `exceeded quota`) because maxSurge pod can't be created.
- Fix: fix cause then `kubectl rollout undo` if urgent; CI should fail on `rollout status` timeout.

---

## 4.1 Troubleshooting playbook

Universal first three commands:
```
kubectl get pods -o wide -n <ns>                  # STATUS, RESTARTS, node, IP
kubectl describe pod <pod> -n <ns>                # EVENTS at bottom = root cause 80% of time
kubectl logs <pod> -n <ns> [--previous] [-c <container>]
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

### 4.1.1 Pending
```
Pod Pending
  │ kubectl describe pod -> Events
  ├─ "FailedScheduling ... Insufficient cpu/memory" ─▶ requests too big / cluster full
  │      kubectl describe node | grep -A8 Allocated ; kubectl top node ─▶ right-size, add nodes, check autoscaler
  ├─ "node(s) had untolerated taint" ─▶ add toleration or use other pool: kubectl describe node | grep Taints
  ├─ "didn't match Pod's node affinity/selector" ─▶ kubectl get nodes --show-labels ; fix selector/label
  ├─ "volume node affinity conflict" / "unbound PersistentVolumeClaims"
  │      kubectl get pvc,pv ; kubectl describe pvc <c> ─▶ StorageClass missing? zone mismatch? WaitForFirstConsumer? provisioner (CSI) errors
  ├─ "max node group size reached" / autoscaler not triggering ─▶ kubectl -n kube-system logs deploy/cluster-autoscaler
  ├─ "exceeded quota" (pod not even created) ─▶ kubectl describe quota -n <ns>
  └─ no events at all ─▶ scheduler down? kubectl -n kube-system get pods ; or pod stuck ContainerCreating (see next)
```
ContainerCreating for long: CNI failure (no IPs left in subnet: `failed to assign an IP address`), volume attach/mount failure (`FailedAttachVolume`, `FailedMount`: missing ConfigMap/Secret), image pull in progress.

### 4.1.2 ImagePullBackOff / ErrImagePull
```
kubectl describe pod -> "Failed to pull image ..."
  ├─ "not found"/"manifest unknown" ─▶ wrong image name/tag ─▶ verify: docker manifest inspect / ECR console
  ├─ "unauthorized"/"pull access denied" ─▶ missing/wrong imagePullSecret ─▶ kubectl get secret; kubectl get sa default -o yaml
  │       ECR: token expires in 12h (node role needs ecr:GetAuthorizationToken / IRSA), cross-account policy
  ├─ "timeout"/"no such host" ─▶ node egress/NAT/VPC endpoint/DNS, registry rate limits (Docker Hub 429)
  └─ arch mismatch "exec format error" (arm image on amd node) ─▶ multi-arch build
```
Fix: `kubectl create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...` then reference `imagePullSecrets`.

### 4.1.3 CrashLoopBackOff
```
kubectl describe pod ─▶ Last State / Exit Code / Reason
  ├─ Exit 1 / stacktrace in `kubectl logs --previous` ─▶ app error: bad config/env, DB unreachable, missing Secret key, port in use
  │      (CreateContainerConfigError instead = missing ConfigMap/Secret key referenced)
  ├─ Exit 137 + OOMKilled ─▶ see OOMKilled tree
  ├─ Exit 137 no OOM + "Liveness probe failed" in events ─▶ probe too aggressive / startup too slow ─▶ startupProbe, raise thresholds
  ├─ Exit 143 ─▶ SIGTERM (normal shutdown) - is something (probe, eviction, rollout) terminating it?
  ├─ Exit 139 ─▶ segfault (native lib/JNI)
  ├─ Exit 126/127 ─▶ command not executable / not found (bad ENTRYPOINT, wrong shell, CRLF line endings in scripts)
  └─ Exit 0 quickly ─▶ process finished (a job-like app in a Deployment): wrong workload type or foreground missing
```
Commands: `kubectl logs <p> --previous`, `kubectl describe pod <p>`, `kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState}'`, run the image with a sleep override or `kubectl debug` to inspect, `kubectl exec` during the brief up window, check `kubectl get cm,secret`.

### 4.1.4 OOMKilled
```
lastState.reason == OOMKilled (exit 137)
  ├─ Was it JVM heap OOM (java.lang.OutOfMemoryError in logs)? ─▶ heap too small / leak: heap dump, increase MaxRAMPercentage/limit
  └─ Container-level (no Java OOM message) ─▶ off-heap: metaspace, direct buffers, threads, native
        ─▶ kubectl top pod --containers ; Prometheus working_set vs limit ; NMT (`jcmd <pid> VM.native_memory summary`)
        ─▶ heap% too high (MaxRAMPercentage 90+ leaves nothing) ─▶ lower to 60-70
        ─▶ emptyDir medium:Memory or tmpfs writes count toward memory
  Node-level? `kubectl describe node` "System OOM encountered" ─▶ node overcommitted (limits >> requests) ─▶ set memory requests==limits
```

### 4.1.5 Evicted
```
kubectl get pods | grep Evicted ; kubectl describe pod <p> ─▶ "The node was low on resource: memory/ephemeral-storage"
  ├─ memory pressure ─▶ BestEffort/Burstable over-request pods hit first ─▶ set requests properly, use Guaranteed for critical, add nodes
  ├─ ephemeral-storage/disk pressure ─▶ logs, emptyDir, image layers ─▶ set ephemeral-storage limits, rotate logs, bigger node disk
  └─ node.kubernetes.io/not-ready NoExecute after 300s ─▶ node problem (see NotReady)
Cleanup (evicted pods are just leftover Failed objects): kubectl delete pod --field-selector=status.phase==Failed -n <ns>
```

### 4.1.6 Pod Running but not Ready (readiness failing)
```
READY 0/1, STATUS Running
  kubectl describe pod ─▶ "Readiness probe failed: <reason>"
  ├─ connection refused ─▶ app not listening yet / wrong port / bound to 127.0.0.1 (must bind 0.0.0.0)
  ├─ 404 ─▶ wrong path (actuator probes not enabled) ; 401/403 ─▶ security on health endpoint
  ├─ 503 ─▶ app says not ready (dependency, warm-up): check actuator readiness details/logs
  ├─ timeout ─▶ throttling/GC; raise timeoutSeconds
  └─ test from inside: kubectl exec <p> -- wget -qO- localhost:8081/actuator/health/readiness ; or kubectl port-forward pod/<p> 8081
```

### 4.1.7 Service unreachable
```
Client cannot reach service
  1. Does the Service have endpoints?  kubectl get endpointslices -l kubernetes.io/service-name=<svc> ; kubectl get endpoints <svc>
       ├─ empty ─▶ selector mismatch (kubectl get svc <s> -o yaml vs kubectl get pods --show-labels) or pods not Ready
       └─ present ─▶ 2
  2. Port mapping right?  targetPort must equal the port the app really listens on
  3. Pod reachable directly?  kubectl exec -it debug-pod -- curl -sv http://<podIP>:8081/actuator/health
       ├─ fails ─▶ app/bind address/NetworkPolicy/CNI issue ; kubectl get networkpolicy -n <ns>
       └─ works ─▶ 4
  4. Via ClusterIP/DNS from another pod: curl http://<svc>.<ns>.svc.cluster.local ─▶ if fails but podIP works: kube-proxy/CNI
       kubectl -n kube-system get pods -l k8s-app=kube-proxy ; logs ; iptables -t nat -L | grep <svc> (on node)
  5. From outside: Ingress?  kubectl describe ingress ; ingress-controller logs ; correct ingressClassName/host header ; LB health checks ; security groups
  6. LoadBalancer <pending>: cloud controller / LB controller logs, subnet tags, quotas
```
Quick debug pod: `kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash` (netshoot has curl/dig/tcpdump). Note this **creates** a pod: use only on non-prod or with approval.

### 4.1.8 DNS failing
```
Name does not resolve in pod
  ├─ kubectl exec <p> -- cat /etc/resolv.conf         ─▶ nameserver = kube-dns ClusterIP? search domains?
  ├─ kubectl exec <p> -- nslookup kubernetes.default   ─▶ fails for everything ─▶ CoreDNS or path to it
  │     kubectl -n kube-system get pods -l k8s-app=kube-dns ; kubectl -n kube-system logs -l k8s-app=kube-dns
  │     kubectl -n kube-system get svc kube-dns ; get endpoints kube-dns
  │     NetworkPolicy blocking egress UDP/TCP 53 to kube-system ? node-to-node CNI problem (only pods on some nodes fail)?
  ├─ internal names OK, external fail ─▶ CoreDNS upstream forwarders / VPC DNS / NAT (kubectl -n kube-system get cm coredns -o yaml)
  ├─ NXDOMAIN for my-svc ─▶ wrong namespace/name, or Service doesn't exist (Service names, not Deployment names)
  └─ slow/intermittent ─▶ ndots:5, CoreDNS overloaded/throttled, conntrack ─▶ NodeLocal DNSCache
```

### 4.1.9 Node NotReady
```
kubectl get nodes ; kubectl describe node <n> ─▶ Conditions (Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable), Events
  ├─ Ready=Unknown ("Kubelet stopped posting node status") ─▶ node down/unreachable/kubelet dead
  │     cloud console/instance status ; SSH/SSM: systemctl status kubelet containerd ; journalctl -u kubelet -n 200
  ├─ DiskPressure ─▶ df -h ; image/log cleanup ; crictl rmi --prune
  ├─ MemoryPressure ─▶ evictions in progress, look at pods' memory/limits
  ├─ NetworkUnavailable / CNI not ready ─▶ CNI daemonset pod (aws-node/calico) logs, IPs exhausted
  └─ certificate expiry / API server connectivity ─▶ kubelet logs "Unable to register node" / x509
Timeline: no heartbeat 40s -> NotReady (node-monitor-grace-period) -> taint not-ready/unreachable NoExecute
          -> pods evicted after tolerationSeconds (default 300s) -> ReplicaSets recreate them elsewhere.
Managed nodes: cordon+drain then replace the node (terminate instance; ASG/Karpenter brings a new one).
   kubectl cordon <n> ; kubectl drain <n> --ignore-daemonsets --delete-emptydir-data  (modifies the cluster - ops only)
```

### 4.1.10 Rollout stuck
```
kubectl rollout status deploy/x  hangs or "exceeded its progress deadline"
  kubectl get rs -l app=x ; kubectl get pods -l app=x   ─▶ where are new pods?
  ├─ new pods Pending ─▶ Pending tree (capacity for maxSurge? quota? PDB irrelevant here)
  ├─ ImagePullBackOff ─▶ tag typo/registry auth
  ├─ CrashLoopBackOff ─▶ CrashLoop tree (bad config in new version)
  ├─ Running 0/1 ─▶ readiness tree (new version's probe/path/dependency)
  ├─ ReplicaFailure condition on RS ─▶ kubectl describe rs : quota, admission webhook denied (PSA, policy), LimitRange
  └─ old pods won't die (Terminating forever) ─▶ finalizers, stuck preStop, node down; kubectl delete pod --grace-period=0 --force is LAST resort
  Decide: fix forward or `kubectl rollout undo deploy/x` ; confirm `kubectl rollout status`
```

### 4.1.11 What to say in an incident interview
Structure: **Detect -> Triage/Mitigate -> Diagnose -> Fix -> Prevent**.
1. "First I check blast radius and recent change: `kubectl rollout history`, deploy/config timeline, alerts."
2. "Mitigate first: roll back (`rollout undo` / Argo revert), scale up, shift traffic. Diagnose after service is stable."
3. "Then top-down: is it Pending, crashing, not Ready, or Ready but unreachable? `get`, `describe` (events), `logs --previous`, `top`, then node/network layer."
4. "I distinguish liveness restarts (exit 137 with probe events) from OOMKill (`reason: OOMKilled`) from app crashes (exit 1)."
5. "Prevent: startup probes, PDBs, realistic requests, alerts on restarts/throttling, canary with automated rollback, runbook update, blameless postmortem."
Use one concrete story (Story 1 or 2 above) with timings and numbers.

---

# 5. Interview questions

Format: **Q (difficulty)** -> model answer -> follow-up chain (F) -> common wrong answer (X).

## Easy

**Q1 (Easy). What is Kubernetes and what problem does it solve?**
A container orchestrator: you declare desired state (N replicas of an image, exposed via a Service) and controllers continuously reconcile actual state to it. It gives scheduling, self-healing, scaling, service discovery/load balancing, rolling updates/rollbacks, config/secret injection, storage orchestration.
F: How is it different from Docker/Compose? (Docker runs containers on one host; K8s manages fleets across nodes with self-healing.) F: Do you need K8s for 3 services? (Maybe not: ECS/App Runner/VMs; K8s pays off with many services/teams.)
X: "Kubernetes is a container runtime" (no: it drives runtimes through CRI).

**Q2 (Easy). What is a Pod and why not just a container?**
Smallest deployable unit: one or more containers sharing a network namespace (one IP, localhost), volumes, and lifecycle; scheduled together onto one node. Multi-container pods are for tightly coupled helpers (sidecars, init).
F: Can containers in a pod talk via localhost? Yes. F: Why don't you create bare pods? No self-healing; use Deployment. F: What is the pause container? Holds the network namespace for the pod.
X: "Pod = container" or "one pod can span nodes".

**Q3 (Easy). Explain Deployment vs ReplicaSet vs Pod.**
Deployment declares template+replicas+strategy, creates/updates ReplicaSets; ReplicaSet keeps N pods alive; pods run the containers. On image change a new RS is created and scaled up while the old is scaled down.
F: Who does rollback? Deployment scales the old RS back up. F: What is `pod-template-hash`? Label distinguishing RSs.

**Q4 (Easy). Name the control-plane and node components.**
apiserver, etcd, scheduler, controller-manager (+cloud-controller-manager); kubelet, kube-proxy, container runtime (containerd) on nodes; CNI/CoreDNS as add-ons.
F: Which talks to etcd? Only apiserver. F: If the control plane dies, do running pods die? No, they keep running but nothing is reconciled/scheduled.

**Q5 (Easy). Types of Service?**
ClusterIP, NodePort, LoadBalancer, ExternalName, plus headless (`clusterIP: None`). Explain each and when.
F: Difference between port, targetPort, nodePort? F: Why can't I ping a ClusterIP? It is only iptables/IPVS rules on protocol ports.

**Q6 (Easy). ConfigMap vs Secret. Are Secrets secure?**
Both inject config as env/files; Secret is for sensitive data but only **base64-encoded** by default; enable encryption at rest (KMS), restrict RBAC, use external secret managers.
X: "Secrets are encrypted."
F: Do env-var values update when the ConfigMap changes? No, restart needed; volume-mounted ones update eventually (not `subPath`).

**Q7 (Easy). Liveness vs readiness probe?**
Liveness failure restarts the container; readiness failure removes the pod from Service endpoints without restart.
F: What is a startup probe for? Slow starters; disables the other two until it passes. F: What should liveness check? Only internal stuck state, never downstream.

**Q8 (Easy). requests vs limits?**
Request = scheduling reservation/baseline; limit = hard cap. Over memory limit = OOMKilled; over CPU limit = throttled.
F: What if only limits are set? Requests default to the limits. F: What is allocatable? Node capacity minus system/kubelet reserved.

**Q9 (Easy). What is a Namespace?**
A virtual partition for names, RBAC, quotas and policy within a cluster. Not a hard security boundary; not for nodes/PVs (cluster-scoped).
F: How do you call a service in another namespace? `svc.ns` or full FQDN.

**Q10 (Easy). How do you roll back a bad deployment?**
`kubectl rollout undo deploy/x [--to-revision=N]`; Helm: `helm rollback`; GitOps: revert commit. Check `kubectl rollout history`.
F: Does rollback revert config/DB migrations? Only the pod template; ConfigMaps (if unversioned) and schema changes stay.

**Q11 (Easy). What are the most common pod error states and first steps?**
Pending (scheduling), ImagePullBackOff, CrashLoopBackOff, OOMKilled, CreateContainerConfigError. `get pods` -> `describe pod` (events) -> `logs --previous`.

**Q12 (Easy). What is `kubectl apply` vs `create`?**
`create` is imperative (fails if exists); `apply` is declarative (create or patch to the manifest, three-way merge using last-applied annotation, or server-side apply with field ownership).
F: What is `kubectl replace`? Overwrites whole object. F: `--dry-run=server`? Runs admission, persists nothing.

## Medium

**Q13 (Medium). Walk me through what happens on `kubectl apply -f deployment.yaml`.**
Answer with the 14-step trace in 2.2: kubeconfig/discovery -> authn -> authz -> mutating admission -> validation -> validating admission -> etcd write -> Deployment controller creates RS -> RS controller creates Pods -> scheduler filter/score/bind -> kubelet: CRI sandbox + CNI + image pull + start -> probes -> EndpointSlice -> kube-proxy -> traffic. Stress asynchronous, level-triggered controllers.
F: Where can it fail with `kubectl` returning success? Everything after etcd write (Pending, ImagePull, crash). F: Which step rejects a missing CPU request under ResourceQuota? Validating admission. F: Who sets `nodeName`? The scheduler's Binding. F: How does kubelet know about the pod? A watch on pods for its node.
X: "kubectl talks to the scheduler / kubelet directly."

**Q14 (Medium). How does the scheduler decide the node?**
Filter (requests fit allocatable, affinity, taints, volumes, spread) then Score (spread, resource balance, image locality, preferences), then bind; unschedulable -> Pending and possible preemption. Based on **requests, not usage**.
F: Pod Pending but nodes at 20% CPU? Requests sum to allocatable. F: How do you force a pod onto specific nodes? nodeSelector/affinity + taints/tolerations. F: How do you spread across zones? topologySpreadConstraints.

**Q15 (Medium). Explain the reconciliation/control-loop pattern. Why is it robust?**
Controllers compare desired (spec) with observed (status/world) and act; level-triggered so missed events don't matter; idempotent; independent controllers coordinate only through the API. Enables self-healing and Operators/CRDs.
F: What is an informer/work queue? Cached watch + rate-limited retry queue. F: What are ownerReferences? Link for GC and adoption.

**Q16 (Medium). What is etcd and what happens if it loses quorum?**
Raft KV store of all cluster state; writes need majority. Without quorum, no writes/changes (API mostly unavailable for writes) but running workloads keep going. Use 3/5 members, SSD, backups (`etcdctl snapshot save`), encrypt secrets. Only apiserver reads/writes it.
F: Why odd number? Even adds no tolerance. F: How do you back up a managed cluster? Rely on provider + GitOps for manifests + Velero for PVs.

**Q17 (Medium). Describe the pod termination sequence and why you need a preStop sleep.**
Terminating -> endpoint removal and preStop run concurrently -> SIGTERM after preStop -> SIGKILL at grace period (includes preStop). Endpoint removal propagates asynchronously; without a delay the pod may stop listening while traffic still arrives (502/reset). `preStop: sleep 10` plus Spring `server.shutdown=graceful`; grace >= preStop + drain.
F: What if the app ignores SIGTERM or PID 1 is a shell script? SIGKILL after 30s; use `exec`, tini, or JVM as PID 1. F: What exit code? 143 for graceful SIGTERM, 137 for SIGKILL. F: ALB IP-mode extra concern? Deregistration delay/target propagation is even slower; add pod readiness gates.
X: "SIGTERM first, then endpoints removed."

**Q18 (Medium). Liveness probe is killing your Spring Boot app at startup. Why and how do you fix it?**
Startup exceeds `initialDelaySeconds + failureThreshold*period`; kubelet kills at 137 before the app is up -> loop, empty logs. Fix: `startupProbe` with generous budget (`failureThreshold*period` >= worst startup), then relaxed liveness; also check CPU throttling makes startup slower.
F: Why not just a big initialDelaySeconds? It also delays detecting failures on later restarts and still guesses. F: Would you put DB checks in liveness? No.

**Q19 (Medium). Explain QoS classes and what gets evicted first.**
Guaranteed (all containers requests==limits for cpu+memory), Burstable, BestEffort. Under node pressure kubelet evicts BestEffort and Burstable pods exceeding requests first (by priority and overage); Guaranteed last. OOM killer scores follow QoS.
F: How do you make a pod Guaranteed? F: Is a pod with memory limit only Guaranteed? No (needs cpu too; and requests==limits). F: Eviction vs OOMKill? Kubelet graceful-ish eviction (pod Failed/Evicted) vs kernel kill of a container at cgroup limit.

**Q20 (Medium). Why is my Java app OOMKilled even though heap is only 50% used?**
Container memory = heap + metaspace + thread stacks + code cache + direct buffers + native + GC overhead (+page cache). Limit compares to total RSS/working set. Set `MaxRAMPercentage` 60-75, cap direct memory, bound threads, monitor with NMT.
F: Default heap in container? 25% of limit. F: Why 137? 128+SIGKILL. F: How to distinguish from a Java OOM? `reason: OOMKilled` vs `java.lang.OutOfMemoryError` in logs.

**Q21 (Medium). What is CPU throttling and how does it affect a JVM?**
CPU limit -> CFS quota per 100ms period; multithreaded JVM exhausts quota early in a period, all threads paused until the next; latency spikes, probe timeouts, slow startup at low average CPU. Monitor `container_cpu_cfs_throttled_periods_total`; mitigate by raising/removing CPU limits, sizing thread pools/`ActiveProcessorCount`, honest requests.
F: Do requests throttle? No, they set relative weight. F: Downside of no CPU limit? Noisy neighbors; mitigate with correct requests and QoS/nodepool separation.

**Q22 (Medium). Explain rolling update with maxSurge/maxUnavailable. Compute for 10 replicas with 25%/25%.**
Surge ceil(2.5)=3, unavailable floor(2.5)=2 -> max 13 pods, min 8 available. `maxUnavailable: 0, maxSurge: 1` gives zero-capacity-loss updates at the cost of needing headroom. New pods must pass readiness before old ones are removed. Stuck on unready pods; progressDeadlineSeconds reports failure, no auto-rollback.
F: Both zero? Invalid. F: What if quota is at limit? RS cannot create surge pods, rollout stalls. F: How to speed up? Higher surge, faster readiness, `minReadySeconds` lower.

**Q23 (Medium). How does a Service find its pods, and what are EndpointSlices?**
Label selector; the EndpointSlice controller maintains slices of Ready pod IP:ports; kube-proxy/ingress programs from them. Empty endpoints = selector mismatch or unready pods. Slices scale better than the single monolithic Endpoints object.
F: What about Services without selectors? Create EndpointSlices manually (external backends). F: When does a pod join endpoints? When Ready. F: `publishNotReadyAddresses`? For headless StatefulSet peer discovery.

**Q24 (Medium). Ingress vs LoadBalancer Service vs Gateway API.**
LoadBalancer = one cloud LB per Service, L4. Ingress = shared L7 routing by host/path through a controller; annotations for extras. Gateway API = role-separated, expressive successor (HTTPRoute weights, header matches, GatewayClass/Gateway).
F: What happens if there is no ingress controller? Object does nothing. F: TLS termination where? Controller, with cert Secret/cert-manager.

**Q25 (Medium). How does HPA compute replicas? Example.**
`desired = ceil(current * currentMetric/target)`. 4 pods at 105% vs 70% target -> 6. Utilization is % of **requests**; 10% tolerance; scale-down stabilization 300s; metrics-server required; max across multiple metrics.
F: Why doesn't HPA scale though CPU is high? No requests set, metrics-server missing, at maxReplicas, or unready pods/`FailedGetResourceMetric`. F: Scale on Kafka lag? KEDA/external metrics; replicas <= partitions. F: HPA + Cluster Autoscaler interplay? New pods Pending -> nodes added (Karpenter faster).

**Q26 (Medium). What is a PodDisruptionBudget? What can go wrong?**
Caps voluntary evictions (`minAvailable`/`maxUnavailable`) for drains/autoscaler/upgrades. Does not stop crashes or your own rolling update. `minAvailable == replicas` (or 1 replica with minAvailable 1) blocks drains and node upgrades.
F: Single-replica service? PDB is meaningless (or blocks); make it 2+ replicas.

**Q27 (Medium). ConfigMap changed; how do pods pick it up?**
Env vars: restart required. Mounted files: eventually consistent updates (not with subPath) but Spring must reload. Robust: hash annotation in Helm / Kustomize configMapGenerator suffix / Reloader / `rollout restart`; immutable ConfigMaps with new names.
F: Why doesn't a Deployment roll automatically? Only `spec.template` changes trigger a new RS.

**Q28 (Medium). PV, PVC, StorageClass; access modes; reclaim policy.**
As in 2.11. RWO = single node; RWX needs shared FS (EFS/NFS); RWOP single pod. `Delete` vs `Retain`. `WaitForFirstConsumer` for zonal disks. `allowVolumeExpansion` for growth.
F: Pod stuck Pending with volume node affinity conflict? PV in another AZ than schedulable nodes. F: What happens to PV when PVC deleted with Retain? Released; manual reclaim.

**Q29 (Medium). StatefulSet vs Deployment. When and why?**
Stable ordinal names, stable DNS via headless Service, per-pod PVCs, ordered create/scale/update. Use for stateful, identity-aware software (brokers, DB replicas). Not needed for stateless apps.
F: How does scale-down treat PVCs? Kept. F: Can two pods share the same PVC in a StatefulSet? No, each gets its own.
X: "StatefulSet makes a database highly available" (no: replication is the app's job).

**Q30 (Medium). RBAC design: give a service read access to ConfigMaps only.**
ServiceAccount + Role (`configmaps`: get/list/watch) + RoleBinding, `serviceAccountName` in the pod; check with `kubectl auth can-i --as system:serviceaccount:ns:sa`. Least privilege, per-workload SA, avoid `default`, disable token automount if unused.
F: Role vs ClusterRole? F: How would an EKS pod call S3 securely? IRSA/Pod Identity.

**Q31 (Medium). Helm vs Kustomize; what does `helm upgrade --install --atomic` do?**
Helm templating+release history+rollback+hooks; Kustomize overlay patching in kubectl. `upgrade --install` idempotent deploy; `--atomic --wait --timeout` rolls back if not Ready in time. Release stored as Secrets.
F: `helm rollback` and DB? Not undone. F: Hook types? pre/post install/upgrade/delete/rollback/test.

**Q32 (Medium). Explain GitOps and how you'd deploy with ArgoCD.**
Git holds desired state, Argo watches repo and cluster, syncs and self-heals; CI only updates image tag; rollback = revert. Discuss drift, prune, sync waves, secrets handling, HPA vs replicas ignoreDifferences.

**Q33 (Medium). How do you do zero-downtime deployments?**
Readiness probes, `maxUnavailable: 0`/surge, preStop sleep + graceful shutdown, enough replicas + PDB, topology spread, backward-compatible DB changes, connection draining on LB, canary/blue-green for risk, `rollout status` gate in CI.

**Q34 (Medium). Init container vs sidecar; what's new in 1.29?**
Init: sequential, run-to-completion prerequisites. Sidecar: long-running helper. Native sidecars (`initContainers` with `restartPolicy: Always`, beta-on by default 1.29, GA 1.33) start before and stop after main containers and don't block Job completion.
F: Effective resource request with init containers? max(init max, sum of app) (sidecars count in the sum).

**Q35 (Medium). Pod is Running but Service returns connection refused. Debug.**
Follow tree 4.1.7: endpoints present? targetPort matches listening port? app binds 0.0.0.0 not localhost? curl podIP directly; NetworkPolicy; kube-proxy; then Ingress.

## Hard

**Q36 (Hard). Design zero-downtime shutdown end to end for a Spring Boot + Kafka consumer on EKS behind ALB (IP mode).**
Readiness gate + `preStop sleep` > ALB deregistration convergence, `server.shutdown=graceful`, `timeout-per-shutdown-phase`, ALB `deregistration_delay` aligned, `terminationGracePeriodSeconds` = sum + margin, close Kafka consumer (commit offsets, leave group) to avoid session-timeout rebalance stalls, `maxUnavailable 0`, PDB, static membership (`group.instance.id` from StatefulSet name) or cooperative-sticky assignor to minimize rebalances, idempotent processing for at-least-once. Verify with load test during rollouts.
F: Why still errors with preStop 10? Slow LB deregistration; measure. F: Would readiness fail at preStop start? You can trigger `AvailabilityChangeEvent` REFUSING_TRAFFIC via an actuator/custom preStop HTTP call.

**Q37 (Hard). Explain how a Service ClusterIP actually routes traffic (iptables mode) and its limits versus IPVS.**
kube-proxy programs `nat` PREROUTING/OUTPUT -> KUBE-SERVICES chain matching VIP:port -> KUBE-SVC-xxx chain with probabilistic jumps (`--probability 1/n`) to KUBE-SEP-yyy chains doing DNAT to pod IP:port; conntrack keeps a connection on one backend. Rule count grows O(services*endpoints), updates rewrite tables (slow at scale), sequential match. IPVS uses kernel hash tables and real schedulers; still uses iptables for some masquerade. Connection-level balancing means gRPC needs client-side or L7 balancing. Cilium eBPF removes kube-proxy.
F: What happens to existing connections when a backend is removed? conntrack entries pointing to a dead pod time out/reset. F: Source IP preservation? externalTrafficPolicy Local.

**Q38 (Hard). Your pods are healthy but p99 latency doubled after you "right-sized" CPU limits. Diagnose.**
Throttling from a tight CFS quota with bursty multithreaded JVM: check throttled ratio metric, per-thread stalls, GC pause correlated; JIT compile threads during warm-up. Fix: remove/raise limit, keep request for scheduling, tune pools, consider `-XX:ActiveProcessorCount`, cgroup v2 `cpu.stat`. Also check node CPU steal, noisy neighbors.
F: How do you find the right request? VPA Off-mode recommendations, p95 usage over weeks, load test.

**Q39 (Hard). A node goes NotReady. Describe the timeline and what protects the service.**
Heartbeat Leases stop; node controller marks NotReady after `node-monitor-grace-period` (40s), adds `not-ready`/`unreachable` NoExecute taints; pods evicted after `tolerationSeconds` (300s default); RS creates replacements on other nodes (needs spare capacity); Service endpoints drop the pods as their readiness stops reporting/eviction; PDB does not apply to involuntary. Protect with spread across nodes/zones (topology spread/anti-affinity), replicas >= 3, lower tolerationSeconds for faster failover, StatefulSet caveat (pod stuck Terminating; volume attach to a new node blocked until detached, ~6 min for EBS unless node is confirmed gone).
F: Why not shorter than 40s? Flapping/false positives. F: StatefulSet at-most-one guarantee? It won't replace a pod until the old one is confirmed gone.

**Q40 (Hard). How do you secure a cluster/workloads? Give layers.**
Identity: RBAC least privilege, OIDC/IAM, per-workload SA, IRSA; API server access restrictions; etcd encryption + KMS; PSA `restricted`, securityContext (non-root, read-only FS, drop caps, seccomp); NetworkPolicy default deny; admission policy (Kyverno/Gatekeeper/VAP: no latest, signed images, required limits); image scanning + minimal images + signing; secrets from external manager; audit logging; node hardening/patching, private endpoints; runtime detection (Falco); upgrade cadence; namespace quotas.
F: What replaced PSP? Pod Security Admission. F: Can a pod escape? Privileged/hostPath/hostPID risks; containers share the kernel; gVisor/Kata for untrusted code.

**Q41 (Hard). Why can StatefulSet + EBS pods get stuck during zone or node failure, and how do you design around it?**
RWO EBS is zonal and attached to one instance; a failed node keeps the attachment until the control plane force-detaches (long timeout), and pods need the same zone; if that zone has no capacity the pod stays Pending. Design: replicate at app level across zones (Kafka rack awareness/`min.insync.replicas`), anti-affinity/spread, WaitForFirstConsumer, capacity in every zone, operator-managed recovery, or managed service.
F: Why not RWX EFS for Kafka? Latency/fsync semantics; not suitable for brokers.

**Q42 (Hard). Explain how the Cluster Autoscaler/Karpenter and HPA interact; how do you handle sudden 10x traffic?**
HPA adds pods (15s loop, metrics lag ~1 min, scale-up policies); if nodes lack room pods are Pending -> CA/Karpenter provisions nodes (1-3 min incl. image pull and JVM start). Mitigate: overprovisioning via low-priority pause pods (preempted instantly), KEDA/request-rate metrics for earlier signals, min replicas sized for baseline+burst, pre-pulled images/smaller images, `startupProbe` tuned, readiness fast, rate limiting/queues upstream, scale-up behavior policies.
F: Why not scale on CPU only? Lagging; IO-bound apps show low CPU.

**Q43 (Hard). A rolling deployment is stuck: new pod Ready 0/1 but old pods fine. Production incident: what do you do in the first 5 minutes?**
Confirm impact (old pods still serve -> no outage), pause further changes, `kubectl rollout status`, `get rs/pods`, `describe pod`/events, `logs`, compare diff to previous version (`rollout history --revision`), check config/secrets/quota/dependencies. If unclear or urgent: `rollout undo`. Communicate. Then RCA and add gating (canary/analysis, pre-prod parity, readiness correctness).

**Q44 (Hard). How would you implement canary with automated analysis?**
Argo Rollouts (or Flagger): `Rollout` with steps `setWeight 10 -> pause -> analysis (Prometheus success-rate/latency query) -> 50% -> 100%`; traffic split through ingress/Gateway/mesh for exact percentages (else pod-count ratio); auto-abort and rollback on failed AnalysisRun; keep DB changes compatible; feature flags for behavior.
F: How is that different from Deployment's rolling update? No metrics-based gating and no precise traffic control.

**Q45 (Hard). What is an Operator/CRD, and when would you write one?**
CRD extends the API with a new type; an Operator is a controller implementing domain logic (provision, upgrade, backup, failover) via the same reconcile loop (Strimzi, Prometheus operator, cert-manager). Write one to codify operational knowledge for a repeatable stateful/complex system; avoid if Helm+Job suffice. Concepts: informers, work queues, finalizers, status subresource, idempotent reconcile, leader election (Lease). Frameworks: Kubebuilder/operator-sdk; Java: Fabric8/Java Operator SDK.

**Q46 (Hard). Multi-tenant cluster: how do you isolate teams and prevent noisy neighbors?**
Namespace per team + RBAC + ResourceQuota/LimitRange + NetworkPolicy default deny + PSA + PriorityClasses + dedicated node pools via taints + policy engine + cost allocation labels; for stronger isolation vClusters or separate clusters. Noisy neighbor: requests/limits, QoS, ephemeral-storage limits, PID limits, `max` LimitRange.

**Q47 (Hard). How do DNS and connection reuse interact with rolling deployments for a Java client calling another service?**
JVM caches DNS (`networkaddress.cache.ttl` default 30s without SecurityManager, forever with one), Service VIP is stable so DNS caching is harmless for ClusterIP, but headless/client-side LB targets pod IPs that change -> stale connections. HTTP keep-alive/HTTP2 sticks to old pods: rebalance requires connection max-age, LB-aware clients or a mesh. `ndots` amplification of lookups for external hosts.

**Q48 (Hard). Explain admission controllers vs RBAC vs NetworkPolicy vs PSA: what does each stop?**
RBAC: who can call which API verbs. Admission: what the object may look like/mutations at write time (policies, quotas, PSA is an admission controller). NetworkPolicy: pod-to-pod traffic at runtime (CNI). PSA: pod security fields. None replaces the others.

**Q49 (Hard). etcd is huge and the API server is slow. Causes?**
Too many objects/events, large Secrets/ConfigMaps, Helm release Secrets history (`--history-max`), CRD churn, controllers hot-looping updates, leaked Jobs/pods, unbounded list calls without pagination; fix: TTL cleanup, history limits, defrag/compact, API priority & fairness tuning, fix hot loops, monitor `apiserver_request_duration_seconds`, `etcd_db_total_size_in_bytes`.

**Q50 (Hard). Compare running Postgres/Kafka on K8s vs managed services.**
Consider durability, upgrades, backups, latency, zonal storage, operators' maturity, team skills, cost, compliance. Default: managed for data stores; K8s for stateless plus Operators when portability/cost/control justify (Strimzi on dedicated node pools with local/NVMe or gp3, rack awareness, PDB, anti-affinity, monitored disk/lag, tested restore).

## Common wrong answers / traps (quick list)
| Trap | Reality |
|---|---|
| "A Deployment restarts pods when ConfigMap changes" | No, only template changes trigger rollouts |
| "Secrets are encrypted" | base64 only unless encryption at rest |
| "Liveness should check the database" | Never; causes restart storms |
| "Readiness failure restarts the container" | It only removes endpoints |
| "Limits drive scheduling" | Requests do |
| "HPA uses actual CPU cores" | Utilization vs requests |
| "Namespaces isolate networking" | Not without NetworkPolicy |
| "PDB protects against node failure" | Only voluntary disruptions |
| "Service load-balances requests" | Connections (L4) |
| "kubectl apply waits for pods" | Only writes to etcd; use `rollout status` |
| "OOMKilled = Java OutOfMemoryError" | Different: kernel cgroup kill vs JVM heap exhaustion |
| "CrashLoopBackOff is a failure reason" | It's a back-off state; find the exit code/last state |
| "SIGTERM after endpoints removed" | Concurrent; hence preStop sleep |
| "StatefulSet gives you HA" | Only identity/storage; replication is app-level |
| "ClusterIP is pingable" | Virtual, protocol/port rules only |

---

# 6. Cheat sheets

## 6.1 One-page concept cheat sheet
```
ARCH      apiserver(only door; only one to etcd) | etcd(Raft quorum n/2+1) | scheduler(filter->score->bind, uses REQUESTS)
          controller-mgr(level-triggered loops) | kubelet(CRI, probes, cgroups) | kube-proxy(iptables/IPVS) | CNI | CoreDNS
APPLY     authn->authz->mutate->validate->validating adm->etcd => Deployment ctrl->RS->Pods->sched bind->kubelet(CRI,CNI,pull,start)
          ->probes->EndpointSlice->kube-proxy->traffic          (apply returns after etcd write!)
PHASES    Pending Running Succeeded Failed Unknown | states Waiting Running Terminated | restart backoff 10s..5min
TERMINATE deletionTimestamp -> [endpoint removal || preStop] -> SIGTERM -> grace(30s default, includes preStop) -> SIGKILL(137)
          fix: preStop sleep + server.shutdown=graceful + grace >= preStop+drain ; PID 1 must be JVM (exec)
PROBES    startup(gates others) | liveness(restart; internal only) | readiness(endpoints; no restart)
          startup budget = failureThreshold x period ; defaults period10 timeout1 fail3
RESOURCES request=schedule+weight | limit=cap | CPU throttle (CFS 100ms) | mem over limit => OOMKilled 137
          QoS: Guaranteed(req=lim cpu+mem) > Burstable > BestEffort (evicted first) ; JVM: MaxRAMPercentage 60-75 (default 25)
ROLLING   surge=ceil, unavailable=floor ; 10 pods 25/25 => max13 min8 ; zero-downtime: maxUnavail 0 ; undo: rollout undo
SERVICE   ClusterIP|NodePort|LB|ExternalName|Headless ; DNS svc.ns.svc.cluster.local ; endpoints = Ready pods matching selector
INGRESS   needs controller ; Gateway API = role-oriented successor ; NetworkPolicy needs enforcing CNI
CONFIG    env never updates; volume updates (not subPath); Secret=base64 -> encrypt at rest/KMS/external secrets
STORAGE   PVC->PV via StorageClass ; RWO/ROX/RWX/RWOP ; Retain|Delete ; WaitForFirstConsumer ; StatefulSet: stable id + PVC per pod
HPA       desired=ceil(cur x curMetric/target) ; tol 10% ; scaleDown window 300s ; needs requests + metrics-server ; KEDA for lag
NODES     CA/Karpenter add nodes for Pending pods ; PDB voluntary only ; taints repel, tolerations allow, affinity attracts
SECURITY  RBAC least priv ; SA per app ; IRSA ; PSA restricted ; runAsNonRoot, readOnlyRootFS, drop ALL ; scan/sign images
DELIVERY  Helm(upgrade --install --atomic, rollback) | Kustomize overlays | ArgoCD/Flux pull-based, selfHeal
DEBUG     get -> describe(events) -> logs --previous -> top -> exec/debug ; exit 137 OOM/probe, 143 SIGTERM, 1 app error
```

## 6.2 kubectl cheat sheet
```
# Context & discovery
kubectl config get-contexts ; kubectl config use-context <c> ; kubectl config set-context --current --namespace=<ns>
kubectl api-resources ; kubectl explain deployment.spec.strategy --recursive
kubectl cluster-info ; kubectl get nodes -o wide ; kubectl version

# Read
kubectl get pods -A -o wide ; kubectl get pods -l app=x --show-labels ; kubectl get deploy,rs,svc,ing,hpa,pdb -n <ns>
kubectl get pod <p> -o yaml ; kubectl get pod <p> -o jsonpath='{.status.containerStatuses[*].lastState}'
kubectl describe pod|node|svc|ing <name>
kubectl get events --sort-by=.lastTimestamp -A
kubectl get pods -w        # watch
kubectl get endpointslices -l kubernetes.io/service-name=<svc>

# Logs & exec
kubectl logs <p> [-c c] [--previous] [-f] [--tail=200] [--since=10m] ; kubectl logs -l app=x --all-containers --prefix
kubectl exec -it <p> -- sh ; kubectl cp <p>:/tmp/heap.hprof ./heap.hprof
kubectl port-forward svc/x 8080:80 ; kubectl debug -it <p> --image=busybox --target=<container>
kubectl top pod --containers ; kubectl top node

# Change
kubectl apply -f file|dir|-k overlay ; kubectl apply --dry-run=server -f x.yaml ; kubectl diff -f x.yaml
kubectl set image deploy/x c=img:tag ; kubectl scale deploy/x --replicas=5 ; kubectl autoscale deploy/x --min=3 --max=10 --cpu-percent=70
kubectl rollout status|history|undo|pause|resume|restart deploy/x
kubectl label|annotate ... ; kubectl edit ... ; kubectl patch deploy/x -p '{"spec":{"replicas":4}}'
kubectl delete -f x.yaml ; kubectl delete pod <p> --grace-period=0 --force   # last resort

# Node ops
kubectl cordon <n> ; kubectl drain <n> --ignore-daemonsets --delete-emptydir-data ; kubectl uncordon <n>
kubectl taint nodes <n> key=value:NoSchedule ; kubectl taint nodes <n> key-

# Access & secrets
kubectl auth can-i create pods -n <ns> --as system:serviceaccount:<ns>:<sa> ; kubectl auth can-i --list
kubectl get secret x -o jsonpath='{.data.KEY}' | base64 -d
kubectl create secret generic x --from-literal=K=v ; kubectl create configmap c --from-file=application.yml

# Tips
alias k=kubectl ; source <(kubectl completion bash) ; kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
kubectl get pods --field-selector=status.phase!=Running -A ; kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
```
Rolling update timeline reminder (3 replicas, surge 1, unavailable 0): add v2 -> wait Ready -> remove one v1 (preStop sleep, SIGTERM, drain) -> repeat.
