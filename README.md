# self-healing-service

Kubernetes manifests that run a containerized fraud-detection API as two replicas, replace
copies that disappear, restart copies that stop answering their health check, and expose the
service on a stable address. Everything runs on a local cluster, so it costs nothing.

The API (a Flask app that serves a scikit-learn model) is built into a Docker image in
[`06-docker`](../06-docker). This folder holds the two Kubernetes files that deploy it.

## Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Project layout](#project-layout)
- [Prerequisites](#prerequisites)
- [The manifests, field by field](#the-manifests-field-by-field)
- [Run it](#run-it)
- [Demonstrate self-healing](#demonstrate-self-healing)
- [Scale up and down](#scale-up-and-down)
- [How the two kinds of healing differ](#how-the-two-kinds-of-healing-differ)
- [Design decisions](#design-decisions)
- [Limitations and next steps](#limitations-and-next-steps)
- [Troubleshooting](#troubleshooting)
- [Clean up](#clean-up)
- [Skills demonstrated](#skills-demonstrated)

---

## What it does

- Keeps **2 replicas** of the API running at all times (`deployment.yaml`).
- Replaces any replica that is deleted or whose node fails, without manual action.
- Probes `GET /health` on each replica every 30 seconds and restarts a container that stops
  responding (liveness probe).
- Exposes the API through one stable address that load-balances across healthy replicas
  (`service.yaml`, type `NodePort`, port `30080`).
- Can be resized with one command (`kubectl scale`).

## Architecture

```text
                      curl http://localhost:30080/health
                                   |
                                   v
                 +-------------------------------------+
                 |  Service: zero2ai-fraud-svc         |
                 |  type NodePort                      |
                 |  nodePort 30080 -> port 80 -> 5000  |
                 +------------------+------------------+
                                    | selector: app=zero2ai-fraud-api
                      +-------------+-------------+
                      v                           v
              +---------------+           +---------------+
              |  Pod (copy 1) |           |  Pod (copy 2) |
              |  fraud-api    |           |  fraud-api    |
              |  :5000        |           |  :5000        |
              +---------------+           +---------------+
                      ^                           ^
                      +-------------+-------------+
                                    |
                 Deployment: zero2ai-fraud-api (replicas: 2)
                 keeps the count at 2 and probes /health
```

## Project layout

```text
ai-ml-platform/
├── 05-api/                  # the Flask API source
├── 06-docker/               # Dockerfile; builds the image zero2ai-fraud-api:v1.0
└── self-healing-service/
    ├── deployment.yaml      # replica count, image, health probe
    ├── service.yaml         # stable address and port mapping
    └── README.md
```

## Prerequisites

| Requirement | Why |
|-------------|-----|
| `kubectl` | The command-line client for Kubernetes. |
| A local cluster | **Docker Desktop's built-in Kubernetes** (Settings → Kubernetes → Enable) or **minikube**. |
| The image `zero2ai-fraud-api:v1.0` | The manifests reference it by name. Build it from `06-docker`. |

Build the image:

```bash
cd 06-docker
docker compose build
```

`docker-compose.yml` tags the result `zero2ai-fraud-api:v1.0`. Confirm with
`docker images zero2ai-fraud-api`.

Make sure the cluster is up:

```bash
kubectl get nodes
```

At least one node should show `Ready`.

**Getting the image into the cluster.** Docker Desktop's Kubernetes can usually see images you
built locally. minikube runs its own container runtime and cannot, so load the image into it:

```bash
minikube image load zero2ai-fraud-api:v1.0
```

## The manifests, field by field

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero2ai-fraud-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: zero2ai-fraud-api
  template:
    metadata:
      labels:
        app: zero2ai-fraud-api
    spec:
      containers:
      - name: fraud-api
        image: zero2ai-fraud-api:v1.0
        ports:
        - containerPort: 5000
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 30
```

| Field | Meaning |
|-------|---------|
| `kind: Deployment` | Declares the desired state for a set of identical pods. A Deployment manages a ReplicaSet, which does the actual counting. |
| `replicas: 2` | Desired number of pods. If the real number differs, Kubernetes creates or deletes pods to match. |
| `selector.matchLabels` | How the Deployment finds the pods it owns. Must match the pod template's labels. |
| `template` | The blueprint for each pod, including its labels and containers. |
| `image: zero2ai-fraud-api:v1.0` | The container image to run. With an explicit version tag, Kubernetes only pulls it if it is not already on the node. |
| `containerPort: 5000` | The port the Flask app listens on. Documentation for readers and tools; it does not open anything by itself. |
| `livenessProbe` | A repeated health check. `GET /health` on port 5000. The first check happens 10 seconds after the container starts (`initialDelaySeconds`), then every 30 seconds (`periodSeconds`). |

The probe uses Kubernetes defaults for the rest: a 1-second timeout and three consecutive
failures (`failureThreshold: 3`) before the container is declared unhealthy. With a 30-second
period, a hung container is therefore restarted after roughly 90 seconds.

The API's `/health` route returns `{"status": "healthy", ...}` (see `05-api/app/api.py`).

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zero2ai-fraud-svc
spec:
  type: NodePort
  selector:
    app: zero2ai-fraud-api
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30080
```

| Field | Meaning |
|-------|---------|
| `kind: Service` | A stable network address in front of a changing set of pods. Pods get new IPs whenever they are replaced; the Service address does not change. |
| `type: NodePort` | Opens the same port (`nodePort`) on every node so traffic from outside the cluster can reach the Service. |
| `selector` | Sends traffic only to pods labelled `app: zero2ai-fraud-api`. |
| `port: 80` | The port other workloads inside the cluster use to reach the Service. |
| `targetPort: 5000` | The port on the pod that receives the traffic. |
| `nodePort: 30080` | The port exposed on the node. The allowed range is 30000–32767. |

Traffic path: `localhost:30080` → Service port `80` → a pod on port `5000`.

## Run it

From this folder:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods
kubectl get service zero2ai-fraud-svc
```

`kubectl apply -f` reads a manifest and makes the cluster match it. It is safe to run again; it
only changes what differs.

Expected: two pods named `zero2ai-fraud-api-<hash>-<id>` with status `Running` and `READY 1/1`.
Wait for that before testing.

Call the API through the Service:

```bash
curl http://localhost:30080/health
```

Expected: a JSON body containing `"status": "healthy"`.

With minikube, `localhost:30080` may not be reachable on some platforms. Use:

```bash
minikube service zero2ai-fraud-svc --url
```

and `curl` the URL it prints.

## Demonstrate self-healing

```bash
kubectl get pods                    # note both pod names
kubectl delete pod <one-pod-name>   # remove a replica on purpose
kubectl get pods                    # a replacement appears within seconds
```

Watch it live in a second terminal:

```bash
kubectl get pods --watch
```

You should see the deleted pod go to `Terminating` while a new pod appears in `Pending`,
`ContainerCreating` and then `Running`. The count returns to 2 without any command from you.
Throughout, `curl http://localhost:30080/health` keeps working because the other replica is
still serving.

## Scale up and down

```bash
kubectl scale deployment zero2ai-fraud-api --replicas=5
kubectl get pods
kubectl scale deployment zero2ai-fraud-api --replicas=2
```

Note that `kubectl scale` changes the live cluster only. `deployment.yaml` still says `2`. To
make a change permanent, edit `replicas` in the file and run `kubectl apply -f deployment.yaml`.
Keeping the file as the source of truth is what lets you rebuild the setup exactly.

## How the two kinds of healing differ

This setup heals in two different ways, and it is worth keeping them apart:

| Failure | What detects it | What happens |
|---------|-----------------|--------------|
| A **pod is deleted**, or its node dies | The ReplicaSet notices the count is below `replicas` | A **new pod** is created, with a new name and IP |
| The **app hangs** but its container is still running | The **liveness probe** fails three times in a row | The kubelet **restarts the container** inside the same pod. The pod keeps its name and IP, and its `RESTARTS` count goes up |

To see the second kind, make the API stop answering `/health` (for example, by running a
modified image whose `/health` route returns an error) and watch `RESTARTS` increase in
`kubectl get pods`. The deleted-pod demo above only exercises the first kind.

## Design decisions

- **Two replicas, not one.** One copy means any restart causes downtime. Two lets one serve
  while the other is replaced.
- **A Deployment, not bare pods.** A pod created by itself is gone for good when it dies. A
  Deployment recreates it.
- **Labels and selectors for wiring.** The Service finds pods by label, not by name or IP, so
  replacing pods requires no change to the Service.
- **NodePort for local use.** It needs no cloud load balancer, so it works on a laptop. See
  limitations for what production would use.
- **A versioned image tag (`v1.0`).** Avoids `latest`, which makes it unclear which code is
  running.
- **The container does not run as root.** The Dockerfile in `06-docker` creates and switches
  to a non-root user.

## Limitations and next steps

This is a minimal manifest set. Things a production deployment would add, roughly by value:

1. **A readiness probe.** Right now the Service sends traffic to a pod as soon as its container
   starts, even if the model has not finished loading. A `readinessProbe` keeps unready pods
   out of rotation. This is the most important gap.
2. **Resource requests and limits.** Without them the scheduler cannot place pods sensibly and
   one pod can starve others of memory or CPU.
3. **Rolling-update settings and a rollback drill.** Change the image tag, watch
   `kubectl rollout status`, then practice `kubectl rollout undo`.
4. **A security context.** Set `runAsNonRoot: true`, drop Linux capabilities and make the root
   filesystem read-only.
5. **Autoscaling.** A HorizontalPodAutoscaler to change the replica count based on CPU load.
6. **Ingress or a LoadBalancer Service** instead of NodePort, with TLS.
7. **Configuration and secrets** moved out of the image into a ConfigMap and Secret.
8. **Namespaces and a registry.** Deploy into a dedicated namespace and pull the image from a
   container registry rather than the local machine.
9. **Packaging with Helm or Kustomize** so dev, qa and prod can share the same manifests with
   different settings.

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `kubectl: command not found` | Install `kubectl`, or enable Kubernetes in Docker Desktop. |
| `kubectl get nodes` shows no `Ready` node | The cluster is still starting. Wait, or restart it (`minikube start`, or toggle Kubernetes in Docker Desktop). |
| Pods in `ErrImagePull` or `ImagePullBackOff` | The cluster cannot find `zero2ai-fraud-api:v1.0`. Build it (`docker compose build` in `06-docker`) and, for minikube, run `minikube image load zero2ai-fraud-api:v1.0`. |
| Pods in `CrashLoopBackOff` | The container starts and exits. Read its log: `kubectl logs <pod-name>`. |
| Pod `Running` but `RESTARTS` keeps rising | The liveness probe is failing. Run `kubectl describe pod <pod-name>` and read the Events section. Check the app really serves `/health` on port 5000. |
| `curl: Connection refused` on 30080 | Pods may not be `Running` yet, or the Service is missing. Check `kubectl get pods` and `kubectl get service`. On minikube use `minikube service zero2ai-fraud-svc --url`. |
| `error validating data` on apply | YAML indentation. Use two spaces per level and no tabs. |
| `provided port is not in the valid range` | `nodePort` must be between 30000 and 32767. |

Two commands solve most problems: `kubectl describe pod <name>` (events and probe results) and
`kubectl logs <name>` (the app's output).

## Clean up

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl get pods        # should report nothing, or pods in Terminating
```

If you used minikube and are finished for the day: `minikube stop`.

## Skills demonstrated

- Writing Kubernetes Deployment and Service manifests
- Declarative configuration: describing desired state and letting the cluster converge on it
- Labels and selectors, and how a Service finds its pods
- Health checking with liveness probes, and the difference between restarting a container and
  replacing a pod
- Exposing a workload with NodePort and tracing the port mapping end to end
- Operating a workload with `kubectl` (apply, get, describe, logs, delete, scale)
- Recognizing the gaps between a minimal setup and a production one
