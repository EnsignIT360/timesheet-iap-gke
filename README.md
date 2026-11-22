# 4.4 Team Lab 2: Deploying the Timesheet App to Kubernetes with Secure Architecture

This is a new project repo for Lab 2, based on our working 4.3 timesheet app and current GKE deployment. We will use this baseline to implement and document the secure Kubernetes architecture required for this lab.

A small full-stack employee timesheet app built with React, Express, Docker, and Kubernetes. Team members can submit hours and see the latest entries and running total.

**Starting point:** [4.3 Team Lab 1 repository](https://github.com/EnsignIT360/timesheet-app).

## Current GKE baseline

The existing app is deployed to the `timesheet-gke` cluster in `us-central1`, in the `timesheet` namespace. The frontend and backend are exposed inside the cluster through `ClusterIP` services; the current demo is accessed using `kubectl port-forward`. The manifests in `k8s/` are the starting point for the secure architecture work.

## Baseline screenshots from Lab 1

### Docker containers

The frontend container is running, and the backend container is healthy.

![Docker Compose showing the frontend running and backend healthy](screenshots/docker-containers.png)

### Timesheet UI with submitted entries

![Timesheet UI showing two submitted entries](screenshots/timesheet-ui-two-entries.png)

![Timesheet UI showing one submitted entry](screenshots/timesheet-ui-one-entry.png)

GitHub repository: [EnsignIT360/timesheet-k8s-secure-architecture](https://github.com/EnsignIT360/timesheet-k8s-secure-architecture).

## Run the whole app with Docker

Requirements: Docker Desktop with Docker Compose.

From the project root:

```bash
docker compose up --build
```

Open [http://localhost:3000](http://localhost:3000). NGINX serves the frontend and proxies `/api` requests to the backend. The backend is available locally at [http://localhost:5000](http://localhost:5000); its health endpoint is [http://localhost:5000/api/health](http://localhost:5000/api/health).

To stop the containers, press `Ctrl+C`, then run `docker compose down`.

## Run locally without Docker

```bash
npm install
npm install --prefix backend
npm install --prefix frontend
npm run dev
```

The Vite development server is at `http://localhost:3000` and proxies `/api` requests to the backend at `http://localhost:5000`.

## API

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Check that the backend is running |
| `GET` | `/api/timesheet` | List submitted timesheet entries |
| `POST` | `/api/timesheet` | Add an entry with `employeeName` and `hours` |

Example request body:

```json
{"employeeName":"Jordan Lee","hours":8}
```

Entries are kept in backend memory, matching the lab guide. Restarting the backend clears the data.

## Build containers individually

```bash
docker build -t timesheet-backend ./backend
docker build -t timesheet-frontend ./frontend
```

The frontend NGINX image proxies API requests to the `timesheet-backend` service on the same container network. Use Docker Compose to run the two containers together.

## Deploy to Google Kubernetes Engine (GKE)

The manifests in `k8s/` deploy one backend replica behind an internal `ClusterIP` service named `backend` and one frontend replica behind an internal `ClusterIP` service. The browser uses the frontend's `/api` proxy, so the backend does not need a public IP or a billable public load balancer. Use `kubectl port-forward` to open the frontend locally.

Prerequisites: Google Cloud CLI (`gcloud`), `kubectl`, a Google Cloud project with billing enabled, and permission to create GKE and Artifact Registry resources. Choose a region supported by your GKE setup. The commands below use `us-central1`.

1. Set your project and region, then enable the APIs:

   ```bash
   gcloud config set project PROJECT_ID
   gcloud services enable container.googleapis.com artifactregistry.googleapis.com
   ```

2. Create a Docker repository in Artifact Registry and configure Docker authentication:

   ```bash
   gcloud artifacts repositories create timesheets --repository-format=docker --location=us-central1
   gcloud auth configure-docker us-central1-docker.pkg.dev
   ```

3. Build and push both images. Replace `PROJECT_ID` with your Google Cloud project ID:

   ```bash
   docker build -t us-central1-docker.pkg.dev/PROJECT_ID/timesheets/timesheet-backend:latest ./backend
   docker push us-central1-docker.pkg.dev/PROJECT_ID/timesheets/timesheet-backend:latest
   docker build -t us-central1-docker.pkg.dev/PROJECT_ID/timesheets/timesheet-frontend:latest ./frontend
   docker push us-central1-docker.pkg.dev/PROJECT_ID/timesheets/timesheet-frontend:latest
   ```

4. In `k8s/kustomization.yaml`, replace `PROJECT_ID` in both image paths with your project ID. If using a different region, update `us-central1` there too.

5. Connect `kubectl` to your existing GKE cluster and deploy:

   ```bash
   gcloud container clusters get-credentials CLUSTER_NAME --region us-central1
   kubectl apply -k k8s/
   kubectl -n timesheet get pods
   kubectl -n timesheet port-forward service/timesheet-frontend 8080:80
   ```

   Open [http://localhost:8080](http://localhost:8080) in a browser. If the cluster is in a different project from Artifact Registry, grant the cluster's node service account the Artifact Registry Reader role for the image repository.

The app stores entries in backend memory to match the lab guide. A backend restart clears them, so use a database before relying on GKE for durable timesheet records. GKE and external load balancers can incur Google Cloud charges.

Google Cloud references: [GKE and Artifact Registry](https://cloud.google.com/artifact-registry/docs/integrate-gke) and [Docker authentication for Artifact Registry](https://cloud.google.com/artifact-registry/docs/docker/authentication).

## Team contributions

**Scrum Master:** Alan Sanchez

**Team members present:** Aaron Smith, Abraham Osorio Lagunas, Alan Sanchez

| Team member | Contribution |
| --- | --- |
| Aaron Smith | Built the timesheet application, containerized the services, and deployed the project to Google Kubernetes Engine. |
| Abraham Osorio Lagunas | Attended the team session; contributed project support and Scrum documentation. |
| Alan Sanchez | Scrum Master; supported team coordination. |
| James King JR | Project support. |
| Marco Chiong | Project support. |
| Matias Vazquez | Project support. |

