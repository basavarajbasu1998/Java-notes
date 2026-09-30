# Kubernetes

## Why
One container is easy; 50 containers across 10 machines need: auto-restart, scaling, load balancing, rolling updates, config/secrets. K8s = **container orchestrator** ("declare desired state, K8s makes it true").

## Architecture
```
            CONTROL PLANE                               WORKER NODES (EC2 machines)
 ┌───────────────────────────────┐             ┌────────────────────────────────────┐
 │ API Server (kubectl talks here)│             │ kubelet (node agent)                │
 │ etcd (cluster state DB)        │◄───────────►│ container runtime                   │
 │ Scheduler (picks node for pod) │             │ kube-proxy (networking)             │
 │ Controller Manager (fixes drift)│            │ [Pod][Pod][Pod] each = 1+ containers│
 └───────────────────────────────┘             └────────────────────────────────────┘
```
Deploy flow: `kubectl apply -f x.yaml` → API server stores desired state in etcd → scheduler assigns pod to a node → kubelet pulls image & starts container → controller keeps replicas = desired (crash → restart / recreate).

## Objects you must know
| Object | Meaning |
|---|---|
| **Pod** | smallest unit; 1+ containers sharing IP; ephemeral |
| **Deployment** | manages replicas of pods, rolling updates, rollback |
| **ReplicaSet** | (created by Deployment) keeps N pods alive |
| **Service** | stable virtual IP/DNS in front of changing pods (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | HTTP routing rules (host/path → service) via ingress controller (nginx/ALB) |
| **ConfigMap / Secret** | non-secret config / sensitive values |
| **PersistentVolume(Claim)** | storage that survives pod death |
| **StatefulSet** | pods with stable identity+storage (DB, Kafka) |
| **DaemonSet** | one pod per node (log agents) |
| **HPA** | autoscale pods on CPU/custom metrics |
| **Namespace** | logical isolation (dev/test/prod) |
| **Job/CronJob** | run-to-completion / scheduled tasks |

## Manifests for Order Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: order-service }
spec:
  replicas: 3
  selector: { matchLabels: { app: order-service } }
  strategy: { type: RollingUpdate, rollingUpdate: { maxUnavailable: 0, maxSurge: 1 } }
  template:
    metadata: { labels: { app: order-service } }
    spec:
      containers:
        - name: order
          image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.4.0
          ports: [{ containerPort: 8081 }]
          envFrom: [{ configMapRef: { name: order-config } }, { secretRef: { name: order-secret } }]
          resources: { requests: { cpu: 250m, memory: 512Mi }, limits: { cpu: "1", memory: 1Gi } }
          readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8081 }, initialDelaySeconds: 20 }
          livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8081 }, initialDelaySeconds: 40 }
---
apiVersion: v1
kind: Service
metadata: { name: order-service }
spec: { selector: { app: order-service }, ports: [{ port: 80, targetPort: 8081 }] }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: order-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: order-service }
  minReplicas: 3
  maxReplicas: 10
  metrics: [{ type: Resource, resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } } }]
```

## Rolling update & rollback flow
```
kubectl set image deploy/order-service order=...:1.5.0
   ▼
Start 1 new pod (v1.5) → wait until READINESS probe passes → route traffic
   ▼
Terminate 1 old pod (v1.4) → repeat until all replaced   (zero downtime)
   ▼ problem?  kubectl rollout undo deploy/order-service  → back to v1.4
```
Liveness = "is it dead? restart it". Readiness = "ready for traffic? if not, remove from Service". Requests/limits: request = scheduling guarantee; limit = hard cap (memory over limit → **OOMKilled**).

Debugging flow:
```
kubectl get pods                       → status (CrashLoopBackOff? Pending? ImagePullBackOff?)
kubectl describe pod <p>               → events (why scheduling/pulling failed)
kubectl logs <p> [--previous]          → app logs (previous = before crash)
kubectl exec -it <p> -- sh             → look inside
kubectl top pod                        → CPU/memory
```
Common states: **CrashLoopBackOff** (app crashes on start – check logs/config), **ImagePullBackOff** (wrong image/credentials), **Pending** (no node has enough resources), **OOMKilled** (raise memory limit / fix leak). Helm = package manager for K8s manifests (templated charts). Config change: update ConfigMap and restart rollout.
