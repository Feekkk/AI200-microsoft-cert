# Domain 1 — Develop containerized solutions on Azure (20–25%)

Hosting and orchestrating container images, scaling, and troubleshooting.

Official split:

- **Implement container application hosting** — ACR, ACR Tasks, App Service for Containers
- **Implement container-orchestrated solutions** — ACA + KEDA, AKS manifests, logs / events / connectivity

Pick the host by how much Kubernetes you want:

| Host | You manage | Best for |
| --- | --- | --- |
| App Service for Containers | almost nothing | one web app (or simple Compose) |
| Azure Container Apps | env, revisions, scale rules | microservices, event-driven jobs, scale-to-zero |
| AKS | cluster + YAML | full Kubernetes control |

ACI (Container Instances) is **not** a Domain 1 bullet. Do not pick it when the question is about revisions, KEDA, or manifests.

---

# Implement container application hosting

## Azure Container Registry (ACR) and ACR Tasks

Private Docker registry for storing, versioning, and deploying images.

### SKU tiers

- **Basic** — entry-level, dev/test
- **Standard** — mid-size production
- **Premium** — required for geo-replication, private endpoints / Private Link, customer-managed keys, content trust, dedicated data endpoints, zone redundancy, higher throughput

If the scenario says geo-replication or private endpoint, the answer is Premium. Standard cannot do those.

### Authentication and RBAC

- Production pattern: **managed identity + `AcrPull`**. Do not use the registry admin user.
- `AcrPull` — pull only
- `AcrPush` — pull and push
- `Contributor` / `Owner` on the **registry resource** does **not** grant image pull. Image access is data-plane (`AcrPull` / `AcrPush` or a scoped token).
- Repository-scoped access: **tokens + scope maps** (pull/push per repo), not the admin user.
- AKS / ACA / App Service pull from ACR with a user-assigned or system-assigned identity that has `AcrPull`. AKS can also use the kubelet identity or an image-pull secret; identity is the exam answer.

### Image tagging

Avoid `:latest` in production. It is mutable, so rollbacks and slot swaps become guesswork. Use semantic tags (`1.4.2`) or unique tags (git SHA). Pin the digest (`@sha256:...`) when you need an immutable pull.

### ACR Tasks

Build in Azure so you do not need Docker on a laptop or in CI agents.

- **Quick tasks** — on-demand build and push: `az acr build`
- **Automated triggers** — git commit / PR, cron schedule, or **base-image update** (rebuild when `python:3.12` gets a patch)
- **Multi-step tasks** — YAML that chains build → test → push

`az acr build` uploads context to ACR and runs the build there. `docker build` + `docker push` is the local path; the exam prefers ACR Tasks when the goal is “build in Azure” or “rebuild when the base image changes.”

---

## Azure App Service for Containers

Fully managed PaaS. Best for a **single web container**, or a simple multi-container **Compose** setup, with almost no cluster work.

### Config and secrets

- **App settings** become environment variables inside the container.
- **Key Vault references** + managed identity is the preferred way to inject secrets. Do not bake secrets into the image or put them in plain app settings if Key Vault is an option.
- Container listen port: set `WEBSITES_PORT` (or `PORT`) to the port the process actually binds. Wrong port = app never becomes healthy.

### Deployment slots

- Slots give a staging environment and a swap for near-zero downtime.
- Settings that must **not** swap (DB connection, slot hostname, slot-specific secrets): mark them **slot-specific** / sticky. That is the usual exam answer for “connection string changed after swap.”
- Swap is a traffic cutover of the already-running slot, not a new image build.

App Service is the wrong pick when you need scale-to-zero, KEDA scalers, or many microservices with per-service revisions.

---

# Implement container-orchestrated solutions

## Azure Container Apps (ACA) and KEDA

Fully managed serverless containers on Kubernetes, KEDA, Envoy, and optional Dapr. You do not manage nodes or YAML for Deployments.

<img src="../assets/Azure-ACA%20&%20KEDA.png" alt="Azure Container Apps (ACA) and KEDA" style="max-width: 100%; height: auto;" />

### Environment

Apps live in a **Container Apps environment** (shared networking, logging, Dapr).

- **Consumption** — pay per use, scale to zero
- **Workload profiles** — dedicated compute for predictable CPU/memory, still scale rules on top

One environment, many apps. Apps in the same environment can talk over the internal mesh (`http://<app-name>`).

### Apps vs jobs

- **App** — long-running (HTTP API, worker that stays up, min replicas ≥ 0)
- **Job** — run to completion: manual, scheduled (cron), or event-driven. Use a job for “process this queue message then exit,” not a always-on app with min replicas = 1

### Revisions

A revision is an immutable snapshot of the container image + env + scale config.

- New image or config change → **new revision**
- **Single revision mode** — only one revision gets traffic (simple apps)
- **Multiple revision mode** — split traffic (10% / 90%) for canary / blue-green
- You do not “edit” a live revision. You create another one and shift traffic.

If the question is about canary or splitting traffic between two container versions **without AKS**, the answer is ACA revisions + traffic weights.

### Ingress

- **External** — public HTTPS
- **Internal** — only inside the environment / VNet
- Ingress is Envoy in front of replicas. Target port must match the container.
- Ingress off = no HTTP URL; other apps can still call it only if you set up that path, so most APIs want ingress on.

### Secrets and identity

- Secrets at app scope, mapped as env vars or volume mounts
- Managed identity for ACR pull, Key Vault, Service Bus, Storage
- Dapr is optional sidecar (pub/sub, state, bindings). Do not turn it on unless the scenario needs it.

### KEDA scaling

ACA scale rules are KEDA under the hood.

- **min replicas = 0** → scale to zero (HTTP or event) and pay nothing when idle
- **max replicas** — cap
- HTTP / TCP concurrent-request rules for APIs
- Event scalers: Service Bus queue/topic length, Event Hubs, Storage queues, Redis lists, Cron, CPU/memory

Typical exam mapping:

| Signal | Scale rule |
| --- | --- |
| HTTP request count / concurrency | HTTP scaler |
| Service Bus queue depth | KEDA Service Bus scaler |
| Nightly batch, then exit | Container Apps **job** + schedule or event |
| Always at least one replica | min replicas = 1 (no scale-to-zero) |

KEDA scales **replicas of the app**, not VM nodes. Node count is an AKS / cluster-autoscaler concern.

Polling interval and cooldown matter: a queue scaler that is too slow looks like “messages stuck”; check scale rule auth (managed identity / connection) before blaming the image.

---

## Azure Kubernetes Service (AKS)

Managed Kubernetes. You own the YAML: Deployments, Services, Ingress, probes, HPA.

<img src="../assets/Azure-Kubernetes.png" alt="Azure Kubernetes" style="max-width: 100%; height: auto;" />

### Architecture

- **Control plane** — API server, scheduler, etcd. Azure manages it. **Free** has no SLA. Production needs **Standard** or **Premium**.
- **Node pools**
  - System pool — CoreDNS, metrics, CSI, kube-system. Taint so app pods stay off it.
  - User pool — application workloads
- Separate node pools for GPU, spot, or Windows. Do not mix system add-ons onto the only user pool if the question asks for isolation.

### Deploy with manifests

Exam skill: deploy and manage with **manifest files**, not the portal click-ops story.

Typical set:

- `Namespace`
- `Deployment` (replicas, image, resources, probes)
- `Service` (`ClusterIP` internal, `LoadBalancer` public Azure LB)
- `Ingress` or Gateway API for HTTP routing
- `Secret` / `ConfigMap` (or CSI Secret Store for Key Vault)
- `HorizontalPodAutoscaler`

Apply: `kubectl apply -f`. Update image by changing the tag in the Deployment and applying again, or `kubectl set image`.

Probes:

- **liveness** — restart the container if it is dead
- **readiness** — take it out of the Service until it can take traffic
- **startup** — slow-starting apps (common for AI runtimes)

Missing readiness probe is a classic “traffic hits pods that are still loading the model” failure.

### Networking models

- **Azure CNI** — each pod gets a VNet IP. Other VNet resources can route to pods. Uses more IPs.
- **Azure CNI Overlay** — pods get overlay IPs; nodes sit on the VNet. Fewer VNet IPs, still Azure CNI features. Common modern default.
- **Kubenet** — overlay + NAT. Smaller IP burn, less direct pod reachability, older pattern.

If pods must be directly reachable from a VNet (private DB firewall on pod IPs, NVA inspection), pick Azure CNI, not kubenet.

### Autoscaling (do not mix these up)

| Mechanism | What it scales | When |
| --- | --- | --- |
| **HPA** | pod replicas in a Deployment | CPU, memory, or custom metrics |
| **Cluster autoscaler** | **nodes** in a node pool | pods are `Pending` / unschedulable |
| **KEDA** (if installed) | replicas from events | queue length, etc. ACA has this built in; AKS you add it |
| **VPA** | CPU/memory **requests** on pods | not a substitute for HPA |

Pods `Pending` + `Insufficient cpu` → cluster autoscaler / bigger nodes, not HPA. HPA already wants more replicas; there is nowhere to put them.

### Identity and ACR

Attach ACR with `az aks update --attach-acr` (AcrPull on the kubelet / kubelet identity). ImagePullBackOff + 401 from ACR is almost always missing `AcrPull` or wrong identity.

---

# Monitor and troubleshoot (AKS and Container Apps)

The study guide calls this out on its own: inspect **logs**, **events**, and **end-to-end connectivity**.

## Container Apps

- `az containerapp logs show` / console log stream — stdout/stderr of a replica
- System logs — revision activation, scale, probe failures
- Environment → Log Analytics (or storage) for queries
- Revision status: `Running` vs `Failed` vs `Stopped`; click the revision for replica count
- Scale: replica count stuck at 0 with HTTP 503 → ingress on, min replicas, or a failing startup probe / image pull
- Image pull errors → identity + `AcrPull`, tag exists in the registry

Connectivity:

- External ingress 404 / 502 → wrong target port, app not listening on `0.0.0.0`, revision not getting traffic
- App A cannot reach app B → B ingress internal, name is the ACA app name, same environment
- VNet-injected environment cannot reach a private PaaS → DNS (Azure Private DNS) and NSGs, not “restart the container”

## AKS

Order that matches how questions are written:

1. **Events** — `kubectl describe pod` / `kubectl get events`. ImagePullBackOff, CrashLoopBackOff, FailedScheduling, FailedMount show up here first.
2. **Logs** — `kubectl logs` (and `-p` for the previous crash). App exceptions live here, not in events.
3. **Connectivity** — Service endpoints, NSG, UDR, CoreDNS, NetworkPolicy, NSG on the subnet, private endpoint DNS.

Common mappings:

| Symptom | Look at |
| --- | --- |
| `ImagePullBackOff` | ACR auth, tag, `imagePullSecrets` / AcrPull |
| `CrashLoopBackOff` | logs, command/args, required env, port |
| `Pending` | resources, node selectors/taints, cluster autoscaler |
| 503 from LoadBalancer | readiness probe, endpoints empty (`kubectl get endpoints`) |
| DNS fail inside pod | CoreDNS, VNet DNS, private zone links |
| Intermittent timeouts to Cosmos/Postgres | NSG, firewall IP of the **node** (kubenet/NAT) vs **pod** (CNI), SNAT ports |

`kubectl get events` without looking at logs is wrong for an app exception. Logs without events is wrong for a scheduling/pull failure.

---

# Exam traps (Domain 1)

- **Contributor on ACR ≠ pull images.** Need `AcrPull`.
- **Admin user on ACR** is never the production answer.
- **`:latest`** is the wrong tag strategy in production.
- **Premium ACR** for geo-replication and private endpoints.
- **Slot-specific settings** stop connection strings from swapping.
- **ACA revisions + traffic split**, not AKS, when they want canary without cluster management.
- **ACA job** for run-to-completion; **app** for HTTP that stays up.
- **KEDA / ACA scale rules** scale replicas. **Cluster autoscaler** scales nodes.
- **HPA** vs **cluster autoscaler**: replica count vs node count.
- **Free AKS** has no control-plane SLA.
- **System vs user node pools** for isolation.
- Troubleshoot **events first** for pull/schedule, **logs** for crashes, **DNS/NSG** for “cannot connect.”
