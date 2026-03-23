# Logistics API
## Kubernetes Deployment Guide
 
---
 
## Overview
 
This repository contains Kubernetes manifests for deploying the Logistics API Django application ([Located Here](https://github.com/AbbyyRigsby/LogisticsProject_Backend)) and its PostgreSQL database on a local Minikube cluster. The manifests were initially generated using kompose from a `docker-compose.yaml` file, then manually modified to follow Kubernetes best practices.
 
The application exposes a Django REST API on port 8000, backed by a PostgreSQL database with persistent storage. Graph processing for logistics routing is initialised at startup using airport and seaport datasets baked into the Docker image.
 
---
 
## Architecture
 
### Cluster Layout
 
| Resource | Kind | Purpose |
|---|---|---|
| web | Deployment | Django application server (port 8000) |
| postgres | StatefulSet | PostgreSQL database (port 5432) |
| web | Service (NodePort) | Exposes the API externally on port 30370 |
| postgres | Service (ClusterIP) | Internal-only database access |
| postgres-data | PersistentVolumeClaim | Persistent storage for database files |
| logistics-secret | Secret | Database credentials and dataset paths |
 
### Request Flow
 
External traffic enters via the `NodePort` service on port 30370, which forwards to the Django pod on port 8000. The Django application connects to PostgreSQL through the internal `ClusterIP` service. On Windows with the Docker driver, Minikube creates a localhost tunnel so the NodePort is accessible at `127.0.0.1:<tunnel-port>` rather than directly at the Minikube node IP.
 
### Startup Sequence
 
Pod initialisation follows this order:
 
1. The postgres StatefulSet starts and mounts the PersistentVolumeClaim.
2. The web Deployment starts an `initContainer` that runs Django migrations against the database.
3. Once the `initContainer` exits successfully, the main web container starts the Django development server.
 
> **Note:** There is no formal readiness dependency between the postgres pod and the web `initContainer`. If postgres is slow to start, the `initContainer` may fail and Kubernetes will restart it automatically until the database is available.
 
---
 
## Architecture Decisions
 
### StatefulSet for PostgreSQL
 
kompose generates a `Deployment` for every service by default. The postgres service was manually changed to a `StatefulSet` for the following reasons:
 
- **Stable network identity:** a StatefulSet gives the pod a predictable DNS name (`postgres-0`) that the web pod can rely on for database connections.
- **Ordered startup and shutdown:** Kubernetes guarantees the pod comes up and tears down gracefully, reducing the risk of data corruption.
- **Persistent volume binding:** a StatefulSet keeps the same PVC bound across pod restarts, ensuring database files survive pod recreation.
 
### initContainer for Migrations
 
The original `docker-compose.yaml` ran `python manage.py migrate` as part of the web container's startup command. In Kubernetes this was separated into an `initContainer` for two reasons:
 
- **Separation of concerns:** migrations are a one-time setup step, not part of the server's runtime. Mixing them into the main container command makes restarts slower and logs harder to read.
- **Failure isolation:** if migrations fail, only the `initContainer` restarts rather than the entire pod entering a crash loop with the server also attempting to start.
 
### ClusterIP for PostgreSQL
 
The postgres service uses `ClusterIP` (internal-only), matching the network isolation of the original `logistics_network` bridge in docker-compose. This means the database is not reachable from outside the cluster, reducing the attack surface.
 
### NodePort for the Web Service
 
The web service was patched from `ClusterIP` (kompose default) to `NodePort` to allow external access without requiring `kubectl port-forward` or an ingress controller. Port 30370 was assigned automatically by Kubernetes.
 
### Secret for Credentials
 
All database credentials and dataset paths are stored in a Kubernetes Secret using `stringData` (plain text, encoded by Kubernetes at apply time) rather than hardcoding them in the deployment manifests. This means the manifests can be committed to version control without exposing sensitive values.
 
> **Warning:** For production use, replace the Secret with an external secrets manager such as HashiCorp Vault or AWS Secrets Manager. The current approach is suitable for local development only.
 
### Dataset Files
 
The airport and seaport CSV datasets are baked into the Docker image at build time via the `COPY . /app/` instruction in the Dockerfile. They are excluded from the Git repository via `.gitignore` but are not in `.dockerignore`, so they are present inside the container at runtime. The paths are passed to the application as environment variables in the Secret.
 
`graph_setup.py` was also modified to use `os.path.abspath(__file__)` for path resolution rather than walking four directory levels up, which was machine-specific and broke inside the container.
 
### imagePullPolicy: Never
 
Both containers in the web deployment specify `imagePullPolicy: Never`. This tells Minikube to use the locally built image rather than attempting to pull from a remote registry. The image must be built inside Minikube's Docker context using `minikube docker-env` before applying the manifests.
 
---
 
## Prerequisites
 
Install the following tools before attempting to deploy:
 
| Tool | Install |
|---|---|
| Docker Desktop | https://docs.docker.com/get-docker/ |
| Minikube | `winget install Kubernetes.minikube` |
| kubectl | `winget install Kubernetes.kubectl` |
| kompose (optional) | `winget install kubernetes.kompose` |
 
Ensure Docker Desktop is running and WSL 2 is enabled (Docker Desktop > Settings > General > Use the WSL 2 based engine) before starting Minikube.
 
---
 
## Repository Structure
 
```
k8s-logistics/
├── README.md
├── secret.yaml                               # DB credentials + dataset paths
├── postgres/
│   ├── postgres-deployment.yaml             # StatefulSet (not a Deployment)
│   ├── postgres-service.yaml                # ClusterIP service
│   └── postgres-data-persistentvolumeclaim.yaml
└── web/
    ├── web-deployment.yaml                  # Deployment with initContainer
    └── web-service.yaml                     # NodePort service
```
 
---
 
## Deploying the Stack
 
### Step 1: Clone both repositories
 
```powershell
# Clone the application source
git clone <your-app-repo-url>
cd <your-app-repo>
 
# Clone the manifests repo into a separate folder
git clone <this-repo-url>
```
 
### Step 2: Add dataset files
 
The CSV datasets are not in version control. Copy them into the application source before building the image:
 
```
datasets/
  UPPLY-AIRPORTS.csv
  UPPLY-SEAPORTS.csv
```
 
### Step 3: Start Minikube
 
```powershell
minikube start --driver=docker
```
 
If Minikube stalls on `Pulling base image`, pull the kicbase image manually first:
 
```powershell
docker pull kicbase/stable:v0.0.50
minikube start --driver=docker --base-image="kicbase/stable:v0.0.50"
```
 
### Step 4: Build the Docker image inside Minikube
 
This step must be done in every new terminal session before applying manifests. It points your Docker CLI at Minikube's internal Docker daemon so the image is visible to the cluster.
 
```powershell
minikube docker-env --shell powershell | Invoke-Expression
docker build -t web .
```
 
### Step 5: Edit secret.yaml
 
Before applying, open `secret.yaml` and confirm the values match your environment:
 
```yaml
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
  POSTGRES_DB: logisticsdb
  AIR_DATASET: ../../../datasets/UPPLY-AIRPORTS.csv
  SEA_DATASET: ../../../datasets/UPPLY-SEAPORTS.csv
```
 
### Step 6: Apply the manifests
 
Order matters. The Secret and PVC must exist before the pods that reference them.
 
```powershell
kubectl apply -f secret.yaml
kubectl apply -f postgres/postgres-data-persistentvolumeclaim.yaml
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f postgres/postgres-service.yaml
kubectl apply -f web/web-deployment.yaml
kubectl apply -f web/web-service.yaml
```
 
### Step 7: Verify the cluster
 
```powershell
# Watch pods come up (Ctrl+C to stop watching)
kubectl get pods -w
 
# Expected output when healthy:
# NAME                   READY   STATUS    RESTARTS   AGE
# postgres-0             1/1     Running   0          ...
# web-<hash>             1/1     Running   0          ...
```
 
Check migration logs to confirm the database initialised correctly:
 
```powershell
kubectl logs -c migrate deployment/web
 
# Expected output:
# Graph processing initialized...
# Datasets loaded and concatenated.
# Edges added to the graph.
# Running migrations:
#   No migrations to apply.    (or a list of OK migrations on first run)
```
 
### Step 8: Access the application
 
```powershell
minikube service web
```
 
This opens a browser tunnel and prints the local URL. On Windows with the Docker driver, a tunnel terminal must remain open for the application to be accessible. The URL will be in the format `http://127.0.0.1:<port>`.
 
---
 
## Redeploying After Changes
 
### After changing application code
 
```powershell
minikube docker-env --shell powershell | Invoke-Expression
docker build -t web .
kubectl rollout restart deployment/web
```
 
### After changing manifests
 
```powershell
kubectl apply -f <changed-file>.yaml
```
 
`kubectl apply` is idempotent — running it on an already-applied manifest updates only what changed.
 
### After changing the Secret
 
```powershell
kubectl apply -f secret.yaml
kubectl rollout restart deployment/web
```
 
> **Note:** Kubernetes does not automatically restart pods when a Secret changes. The rollout restart is required to pick up new values.
 
### Full teardown and redeploy
 
```powershell
kubectl delete namespace default
kubectl apply -f secret.yaml
kubectl apply -f postgres/postgres-data-persistentvolumeclaim.yaml
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f postgres/postgres-service.yaml
kubectl apply -f web/web-deployment.yaml
kubectl apply -f web/web-service.yaml
```
 
> **Warning:** Deleting the namespace removes the PersistentVolumeClaim, which deletes all database data. Only do this when you want a clean slate.
 
---
 
## Useful Commands
 
| Command | Purpose |
|---|---|
| `kubectl get pods` | List all pods and their status |
| `kubectl get svc` | List all services and their ports |
| `kubectl logs deployment/web` | View web server logs |
| `kubectl logs -c migrate deployment/web` | View migration logs |
| `kubectl logs statefulset/postgres` | View database logs |
| `kubectl exec -it deployment/web -- sh` | Open a shell inside the web pod |
| `kubectl exec -it postgres-0 -- psql -U postgres` | Open a psql session |
| `kubectl describe pod <name>` | Detailed pod status and events |
| `minikube stop` | Stop the cluster without deleting it |
| `minikube delete` | Delete the cluster entirely |
| `minikube dashboard` | Open the Kubernetes web dashboard |
 
---
 
## Troubleshooting
 
### Minikube stalls on "Pulling base image"
 
Pull the kicbase image manually using `docker pull kicbase/stable:v0.0.50`, then start with the `--base-image` flag as shown in Step 3.
 
### Web pod starts before postgres is ready
 
The `initContainer` will fail and Kubernetes will restart it automatically. Watch `kubectl get pods -w` until the init container completes and the web pod reaches `Running` status. This usually resolves within a minute.
 
### Graph processing error: NoneType
 
The `AIR_DATASET` or `SEA_DATASET` environment variables are not set, or the CSV files are missing from the image. Confirm the `datasets/` folder exists in the application source before building, and that `secret.yaml` contains the correct relative paths.
 
### Service has no node port
 
If the web service was created as `ClusterIP`, patch it to `NodePort`:
 
```powershell
kubectl patch svc web -p "{\"spec\": {\"type\": \"NodePort\"}}"
```
 
### Access is denied when deleting Minikube
 
Run PowerShell as Administrator and manually take ownership of the locked directory:
 
```powershell
takeown /f "C:\Users\<username>\.minikube\machines\minikube" /r /d y
icacls "C:\Users\<username>\.minikube\machines\minikube" /grant administrators:F /t
Remove-Item -Recurse -Force "C:\Users\<username>\.minikube\machines\minikube"
```