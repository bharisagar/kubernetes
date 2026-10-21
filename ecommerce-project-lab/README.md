# Kubernetes Mini E-Commerce Application — Beginner Hands-On Lab

This student project combines Kubernetes architecture, Services, networking, scheduling, node affinity, Pod affinity, Pod anti-affinity, scaling, and troubleshooting in one small application.

## What you build

The Mini Shop has a static Nginx frontend and a dependency-free Python backend. The browser calls `/api/products` on the same origin. Nginx reverse-proxies that path to the internal `backend-service` Service. The browser never tries to resolve Kubernetes DNS such as `backend-service.mini-shop.svc.cluster.local`; only Nginx inside the cluster uses that name.

The cluster has one k3d server and two worker nodes. Both workers are labeled `lab-role=app`. Backend replicas use required hostname anti-affinity, so two replicas spread across the two workers. The frontend uses required Pod affinity, so it must schedule onto a worker that already has a backend Pod. This makes the backend prerequisite visible to beginners.

[View the Mermaid architecture diagrams](architecture.md).

[Read the complete concept reference: What We Learned](WHAT-WE-LEARNED.md).

## Prerequisites and resource impact

Use a Windows 10/11 laptop with PowerShell 5.1 or PowerShell 7, internet access, and about 4 GB of free memory while the lab is running. Docker Desktop runs the Linux containers, k3d creates one server and two worker containers, and the project adds two small application containers. On an 8 GB laptop, close unused applications and stop the cluster after class. No public container registry is required.

The setup script pins k3s `v1.33.5-k3s1` because newer k3s releases can reject some Docker Desktop environments that expose cgroup v1. Docker Desktop must be running before creating the cluster.

### Windows installation

Open **PowerShell as Administrator** and install the tools with Windows Package Manager (`winget`):

```powershell
winget install --id Docker.DockerDesktop -e --source winget
winget install --id Kubernetes.kubectl -e --source winget
winget install --id Rancher.k3d -e --source winget
winget install --id Git.Git -e --source winget
```

If `winget` is unavailable, install **App Installer** from the Microsoft Store, then rerun the commands. Docker Desktop also requires the WSL 2 backend. If Docker Desktop reports that WSL is missing, run these commands in an Administrator PowerShell, restart Windows, and start Docker Desktop:

```powershell
wsl --install
wsl --update
```

Close and reopen PowerShell after installation so the new commands are on `PATH`. Start Docker Desktop, wait until it says **Engine running**, and select **Linux containers** from its tray menu if that option is shown.

Verify every prerequisite before continuing:

```powershell
docker --version
docker info
kubectl version --client
k3d version
git --version
wsl --status
```

Expected output: each command prints a version or status, and `docker info` returns server information without a daemon error. If `docker info` fails, start Docker Desktop and wait for the engine. If a command is not recognized, reopen PowerShell after installation. Do not run `scripts/01-create-cluster.ps1` until these checks succeed.

## File map

```text
app/backend/       Dependency-free Python API and image
app/frontend/      Nginx site, reverse proxy, and image
k8s/               Namespace, Deployments, Services, NetworkPolicy
scripts/           PowerShell setup, build, deploy, verify, cleanup
architecture.md    Mermaid request and scheduling diagrams
screenshots/       Student GitHub screenshot checklist
```

## Lab 1 — Understand architecture and identify nodes

Objective: see the one control plane and two workers before deploying an application.

Concept: the server runs the Kubernetes control-plane components; workers run application Pods. Scheduling decisions happen through the API server and scheduler, while kubelets start Pods on workers.

Commands:

```powershell
cd C:\bari_sagar\Kubernetes\ecommerce-project-lab
.\scripts\01-create-cluster.ps1
kubectl get nodes -o wide
kubectl get nodes --show-labels
```

Expected output: one server/control-plane node, two agent/worker nodes, and `lab-role=app` on both workers.

What happened: k3d created a lightweight k3s cluster in Docker and the script labeled only agent nodes. The workloads use that label through required node affinity.

Troubleshooting: if `k3d` is not recognized, install it and reopen PowerShell. If Docker cannot be reached, start Docker Desktop. If the node count is wrong, run `k3d cluster list` and inspect `k3d cluster create --help`.

Practice: Which nodes are eligible for an application Pod, and why is the server excluded?

## Lab 2 — Build images and import them into k3d

Objective: build both images locally without a public registry.

Concept: Docker builds the images on your laptop. `k3d image import` copies them into the k3d node runtime, and `imagePullPolicy: IfNotPresent` lets Kubernetes use the imported tag.

Commands:

```powershell
.\scripts\02-build-images.ps1
docker images mini-shop-backend mini-shop-frontend
kubectl get nodes
```

Expected output: local tags `mini-shop-backend:local` and `mini-shop-frontend:local` and a successful image import.

What happened: no registry credentials or push were needed. The image tag is deliberately explicit so students can connect the Deployment image field to the imported image.

Troubleshooting: run `docker info` if the build cannot reach Docker. If a later Pod reports `ErrImageNeverPull` or `ImagePullBackOff`, check the image tag and rerun the import for the exact cluster name.

Practice: Why would `imagePullPolicy: Always` be inconvenient for this offline-style lab?

## Lab 3 — Create namespace, Deployments, and Services

Objective: deploy the backend first, then the frontend, and inspect selectors and probes.

Concept: a Deployment maintains Pods; a Service provides a stable destination. The backend Service is ClusterIP and internal. The frontend Service is NodePort for laptop access.

### Scheduling prerequisite demonstration (run on a fresh namespace)

Before the normal deployment, students can prove that required Pod affinity is a real prerequisite. This demonstration deletes only this project's namespace, so do not run it if you have another experiment in `mini-shop`:

```powershell
kubectl delete namespace mini-shop --ignore-not-found
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/20-frontend.yaml
Start-Sleep -Seconds 10
kubectl get pods -n mini-shop -o wide
kubectl describe pod -n mini-shop -l app.kubernetes.io/name=frontend | Select-String -Pattern 'Events|pod affinity|did not satisfy' -Context 0,3
```

Expected output: the frontend Pod is `Pending`, and its scheduler events explain that no existing Pod matches the required `app.kubernetes.io/name=backend` affinity rule. The frontend image can already be imported; the blocker is scheduling, not image startup.

Now create the backend and watch the prerequisite disappear:

```powershell
kubectl apply -f k8s/10-backend.yaml
kubectl rollout status deployment/backend -n mini-shop --timeout=180s
kubectl rollout status deployment/frontend -n mini-shop --timeout=180s
kubectl get pods -n mini-shop -o wide
```

Expected output: two backend Pods become Ready on different app workers, then the frontend schedules onto one of those worker hostnames and becomes Ready. If you skip this demonstration, run `scripts/03-deploy.ps1`, which applies the backend before the frontend.

Commands:

```powershell
.\scripts\03-deploy.ps1
kubectl get all -n mini-shop
kubectl describe service backend-service -n mini-shop
kubectl get endpointslice -n mini-shop
```

Expected output: two Ready backend Pods, one Ready frontend Pod, a backend ClusterIP on port 8000, and frontend NodePort 30080.

What happened: the backend was applied first because the frontend has required Pod affinity to backend Pods. Readiness probes make Pods eligible only after their HTTP endpoints respond.

Troubleshooting: inspect `kubectl get pods -n mini-shop -o wide`, `kubectl describe pod -n mini-shop <pod-name>`, and `kubectl logs -n mini-shop deploy/backend`.

Practice: Which label connects `backend-service` to backend Pods?

## Lab 4 — Access the frontend from localhost:8080

Objective: follow host port 8080 through k3d's load balancer to the frontend NodePort.

Concept: the mapping is `localhost:8080 -> k3d load balancer -> NodePort 30080 -> frontend Pod:80`.

Commands:

```powershell
Invoke-WebRequest -UseBasicParsing http://localhost:8080 | Select-Object -ExpandProperty Content
Start-Process http://localhost:8080
```

Expected output: the Mini Shop page with four product cards.

What happened: the browser talks to the frontend only. Its JavaScript calls same-origin `/api/products`; Nginx handles the internal backend Service lookup.

Troubleshooting: confirm `kubectl get svc -n mini-shop`, `kubectl get pods -n mini-shop`, and `k3d cluster list`. If port 8080 is busy, stop the conflicting process and recreate the cluster with a different host port, then update the documentation command.

Practice: Why should browser JavaScript not use `backend-service.mini-shop.svc.cluster.local`?

## Lab 5 — Test frontend-to-backend communication and CoreDNS

Objective: prove internal Service DNS and the Nginx reverse proxy separately.

Concept: CoreDNS resolves `backend-service` from inside a Pod. A ClusterIP forwards traffic to ready backend endpoints. The browser does not participate in this internal DNS lookup.

Commands:

```powershell
kubectl delete pod api-check -n mini-shop --ignore-not-found
kubectl run api-check -n mini-shop --labels=app.kubernetes.io/name=frontend --restart=Never --image=busybox:1.36 --command -- sleep 60
kubectl wait --for=condition=Ready pod/api-check -n mini-shop --timeout=120s
kubectl exec -n mini-shop api-check -- nslookup backend-service
kubectl exec -n mini-shop api-check -- wget -qO- http://backend-service:8000/api/products
kubectl delete pod api-check -n mini-shop
Invoke-WebRequest -UseBasicParsing http://localhost:8080/api/products | Select-Object -ExpandProperty Content
```

Expected output: DNS returns the backend ClusterIP; the API returns four products; the host request also returns the same JSON through Nginx.

What happened: the first command tests naming, the second tests Service forwarding, and the third tests the complete browser-facing path.

Troubleshooting: check `kubectl get pods -n kube-system`, `kubectl get endpointslice -n mini-shop`, and `kubectl describe service backend-service -n mini-shop`.

Practice: What is the difference between a Service ClusterIP and a backend Pod IP?

## Lab 6 — Verify node and Pod affinity

Objective: confirm backend and frontend placement rules.

Concept: required node affinity says `lab-role=app` must be present. Required frontend Pod affinity says a backend Pod must exist on the same hostname. These are scheduling constraints, not application-level network rules.

Commands:

```powershell
kubectl get pods -n mini-shop -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'
kubectl get pods -n mini-shop -o yaml | Select-String -Pattern 'affinity|lab-role|topologyKey' -Context 0,3
kubectl describe pod -n mini-shop -l app.kubernetes.io/name=frontend
```

Expected output: frontend and at least one backend share a worker hostname, and all application Pods run on nodes labeled `lab-role=app`.

What happened: the scheduler satisfied both required rules. The frontend is not merely near a backend by chance; its required affinity caused the relationship.

Troubleshooting: inspect `kubectl get nodes --show-labels`. If affinity is unsatisfied, check the exact label key/value and topology key.

Practice: What would happen if all backend Pods were deleted while the frontend Deployment still existed?

## Lab 7 — Verify backend anti-affinity across workers

Objective: prove the two backend replicas use different hostnames.

Concept: required hostname anti-affinity prevents two backend Pods with the matching label from sharing a worker. This improves failure isolation in the lab.

Commands:

```powershell
kubectl get pods -n mini-shop -l app.kubernetes.io/name=backend -o wide
kubectl get pods -n mini-shop -l app.kubernetes.io/name=backend -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName'
```

Expected output: the two backend Pods are on different agent nodes.

What happened: the first replica occupied one eligible hostname and the second required another hostname. With only one eligible worker, the second replica would remain Pending.

Troubleshooting: check that both agents have `lab-role=app`, that both are Ready, and that no unrelated workload consumes all resources.

Practice: Why is hostname anti-affinity useful during a worker failure?

## Lab 8 — Scale and troubleshoot a Pending Pod

Objective: observe the capacity limit created by required hostname anti-affinity.

Concept: two eligible worker hostnames can satisfy at most two backend replicas under this required rule. Scaling to three is intentionally a teaching experiment; the third Pod may remain Pending.

Commands:

```powershell
kubectl scale deployment/backend -n mini-shop --replicas=3
kubectl get pods -n mini-shop -l app.kubernetes.io/name=backend -o wide
kubectl describe pod -n mini-shop -l app.kubernetes.io/name=backend | Select-String -Pattern 'Events|did not satisfy|anti-affinity' -Context 0,3
kubectl scale deployment/backend -n mini-shop --replicas=2
kubectl rollout status deployment/backend -n mini-shop --timeout=180s
```

Expected output: two backend Pods run and an extra Pod is Pending with a scheduler event explaining that required anti-affinity cannot be satisfied.

What happened: scaling changed desired replicas, but it did not create a third eligible hostname. Restore two replicas before continuing.

Troubleshooting: verify the worker count and labels. If the third Pod schedules, inspect whether more than two nodes are labeled `lab-role=app`.

Practice: What design change could allow three replicas while keeping two workers?

## Lab 9 — Delete a Pod and observe self-healing

Objective: watch a Deployment replace a deleted backend Pod and let the Service update endpoints.

Concept: the Deployment controller compares desired and actual state. A deleted Pod is recreated; the Service continues to use its stable name while EndpointSlices change.

Commands:

```powershell
$pod = kubectl get pods -n mini-shop -l app.kubernetes.io/name=backend -o jsonpath='{.items[0].metadata.name}'
kubectl delete pod $pod -n mini-shop
kubectl rollout status deployment/backend -n mini-shop --timeout=180s
kubectl get pods,endpointslice -n mini-shop -o wide
Invoke-WebRequest -UseBasicParsing http://localhost:8080/api/products | Select-Object -ExpandProperty Content
```

Expected output: a new backend Pod appears, endpoints recover, and the same frontend URL still returns products.

What happened: the Pod identity changed, but the Service abstraction kept the client route stable.

Troubleshooting: inspect Deployment events and backend readiness. A terminating Pod can briefly cause a failed connection during convergence.

Practice: Which object recreated the deleted Pod?

## Lab 10 — Clean up all resources

Objective: remove the lab without affecting other namespaces or repositories.

Commands:

```powershell
.\scripts\99-cleanup.ps1
```

Expected output: namespace `mini-shop` and k3d cluster `mini-shop` are removed.

What happened: the namespace deletion removes project resources, and the k3d deletion removes only this lab's local cluster containers.

Troubleshooting: if the namespace is terminating, inspect finalizers with `kubectl get namespace mini-shop -o yaml`. If the cluster remains, run `k3d cluster list` and delete only the exact `mini-shop` cluster.

Practice: Why is a dedicated namespace safer than deploying this experiment into `default`?

## Static checks and cluster-dependent checks

Before a cluster is available, inspect YAML syntax and required fields with a YAML parser or `kubectl apply --dry-run=client -f k8s`. The following require a running Docker/k3d/Kubernetes environment: image builds/imports, scheduling behavior, Pod affinity Pending state, Service DNS, NodePort access, probes, scaling, and self-healing. Do not mark those checks as passed from static inspection alone.

Suggested static checks:

```powershell
kubectl apply --dry-run=client -f k8s/00-namespace.yaml
kubectl apply --dry-run=client -f k8s/10-backend.yaml
kubectl apply --dry-run=client -f k8s/20-frontend.yaml
kubectl apply --dry-run=client -f k8s/30-network-policy.yaml
```
