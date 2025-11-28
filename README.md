In this DevOps task, I am building and deploying a full-stack CRUD application using the MEAN stack (MongoDB, Express, Angular 15, and Node.js). The backend is Node/Express (REST API) and the frontend is an Angular application.

This README documents:
- What was source code changes were done to make it Container friendly
- How to run locally with Docker
- How to deploy on an EC2 Linux host using Docker Compose
- A GitHub Actions CI/CD workflow and the required secrets

## What was added (summary)

I added containerization and deployment artifacts so we can build and deploy

- `backend/Dockerfile` — multi-stage Node image (build/runtime) for the API server
- `backend/.dockerignore`
- `frontend/Dockerfile` — multi-stage Node builder + nginx static server for the Angular app
- `frontend/.dockerignore`
- `docker-compose.yml` — Compose manifest (current version uses host networking for EC2 compatibility)
- `.env.example` — example environment variables

Important: The application source files names were changed from turorial to tutorial for example ("backend/app/routes/turorial.routes.js to backend/app/routes/tutorial.routes.js") and its references in files.

- Backend: default MongoDB URI was `mongodb://localhost:27017/dd_db` which was changed to `mongodb://mongo:27017/dd_db`
- Frontend: `baseUrl` was `http://localhost:8080/api/tutorials` which was changed to `http://public-ip:8080/api/tutorials`

## Build and run with Docker (single-service)

Build the backend image (from repo root):

```powershell
docker build -f backend/Dockerfile -t discoverdoller_backend:latest backend
```

Run the backend:

```powershell
docker run --rm -p 8080:8080 discoverdoller_backend:latest
```

Build and run the frontend image:

```powershell
docker build -f frontend/Dockerfile -t discoverdoller_frontend:latest frontend
docker run --rm -p 8081:80 discoverdoller_frontend:latest
```

Visit `http://localhost:8081/` (frontend) and the frontend will call the backend at `http://localhost:8080/api/...` if both services run on the same host.

### Quick EC2 deployment (Linux)

1) Launch an EC2 Linux instance (Ubuntu or Amazon Linux). Configure the instance security group to allow:

- SSH (22) from your IP
- Frontend port (8081) from client CIDR(s)

Do NOT expose MongoDB port 27017 to the public internet in production.

2) Install Docker & Docker Compose (Ubuntu example):

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
newgrp docker
```

3) Deploy the stack on the instance:

```bash
git clone <repo-url> discoverdoller
cd discoverdoller
cp .env.example .env   # edit if needed
docker compose up -d --build
```

4) Open `http://<ec2-public-ip>/` in a browser.

## Recommended production improvements (optional but recommended)

1. Make the backend and frontend environment-aware:
	 - Backend: read `MONGODB_URI` (or `MONGO_HOST`, `MONGO_PORT`, `MONGO_DB`) from environment and construct the connection string.
	 - Frontend: read an `API_BASE_URL` at build time or via runtime-config to avoid hard-coding `http://localhost:8080`.

2. Switch `docker-compose.yml` to bridge networking and refer to services by name (e.g., `backend:8080`), which is portable and secure.

3. Secure MongoDB (authentication, network ACLs) or use a managed database (MongoDB Atlas) and do not expose port 27017 publicly.

4. Use TLS/HTTPS (ALB, nginx, or a CDN) in front of the frontend for production.


---

## Screenshots (showcase)

Added screenshots to showcase the running application and help reviewers verify the deployment quickly.

### Screenshots

Docker Containers:
<img width="1919" height="925" alt="Screenshot 2025-11-28 135918" src="https://github.com/user-attachments/assets/23df1f1a-c00b-46f1-80b1-fcc856af612f" />

Tutorial Edit:
<img width="1919" height="1032" alt="Screenshot 2025-11-28 140033" src="https://github.com/user-attachments/assets/c8e092f4-9bb4-4641-80b4-63f72410f781" />

Tutorial details:
<img width="1919" height="1035" alt="Screenshot 2025-11-28 141537" src="https://github.com/user-attachments/assets/3886bfb7-acf8-44b8-8b1b-45faee9dd903" />

## CI / CD with GitHub Actions (example)

This example covers:

- CI: install/build the frontend and backend
- CD: build and push Docker images to a registry, then SSH to an EC2 host and run `docker compose up -d`

Required repository secrets for the workflow

- `DOCKER_USER` and `DOCKER_PASS` (or credentials for your container registry)
- `PUBLIC_IP`, `USERNAME`, `KEY_PAIR` (private key)

Notes on CI/CD design

- The action builds the frontend in the runner (so the built `dist/` is available if you want to build an image locally) and uses `docker/build-push-action` to push images to a registry.
- The deploy step SSHs to the target EC2 host and runs `docker compose up -d --build`. For production you may want the runner to only `docker pull` pre-built images and avoid building on the host.

## Troubleshooting

- If the frontend cannot reach the API, check `frontend/src/app/services/tutorial.service.ts` for `baseUrl`.

---
