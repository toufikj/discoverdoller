In this DevOps task, you need to build and deploy a full-stack CRUD application using the MEAN stack (MongoDB, Express, Angular 15, and Node.js). The backend is Node/Express (REST API) and the frontend is an Angular application.

This README documents:
- What was added to the repo so the app can run (no application source changes required)
- How to run locally with Docker
- How to deploy on an EC2 Linux host using Docker Compose
- A sample GitHub Actions CI/CD workflow and the required secrets

## What was added (summary)

I added containerization and deployment artifacts so you can build and deploy the app without modifying the application source code:

- `backend/Dockerfile` — multi-stage Node image (build/runtime) for the API server
- `backend/.dockerignore`
- `frontend/Dockerfile` — multi-stage Node builder + nginx static server for the Angular app
- `frontend/.dockerignore`
- `docker-compose.yml` — Compose manifest (current version uses host networking for EC2 compatibility)
- `.env.example` — example environment variables

Important: the application source files were not changed. The backend config (`backend/app/config/db.config.js`) and the frontend service (`frontend/src/app/services/tutorial.service.ts`) still reference `localhost` by default:

- Backend: default MongoDB URI is `mongodb://localhost:27017/dd_db` (see `backend/app/config/db.config.js`).
- Frontend: `baseUrl` is `http://localhost:8080/api/tutorials` (see `frontend/src/app/services/tutorial.service.ts`).

Because the apps expect `localhost` addresses, the provided `docker-compose.yml` can be used with host networking on Linux (EC2) so containers and host share loopback and the apps work without code changes. For portable/production deployments I recommend switching the apps to read configuration from environment variables and using bridge networking — see the Recommended production improvements section.

## Build and run with Docker (single-service)

Build the backend image (from repo root):

```powershell
docker build -f backend/Dockerfile -t discoverdoller-backend:latest backend
```

Run the backend:

```powershell
docker run --rm -p 8080:8080 discoverdoller-backend:latest
```

Build and run the frontend image:

```powershell
docker build -f frontend/Dockerfile -t discoverdoller-frontend:latest frontend
docker run --rm -p 8081:80 discoverdoller-frontend:latest
```

Visit `http://localhost:8081/` (frontend) and the frontend will call the backend at `http://localhost:8080/api/...` if both services run on the same host.

## Docker Compose (EC2-friendly current setup)

The `docker-compose.yml` in this repository is configured to use host networking on Linux so the apps' default `localhost` configuration continues to work without modifying source files. Use this on an EC2 Linux instance.

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

4) Open `http://<ec2-public-ip>:8081/` in a browser.

Notes:

- Host networking exposes container ports on the host network; make sure there are no port conflicts.
- Host networking is supported on Linux hosts (EC2). It is not portable to Docker Desktop on macOS/Windows.

## Recommended production improvements (optional but recommended)

1. Make the backend and frontend environment-aware:
	 - Backend: read `MONGODB_URI` (or `MONGO_HOST`, `MONGO_PORT`, `MONGO_DB`) from environment and construct the connection string.
	 - Frontend: read an `API_BASE_URL` at build time or via runtime-config to avoid hard-coding `http://localhost:8080`.

2. Switch `docker-compose.yml` to bridge networking and refer to services by name (e.g., `backend:8080`), which is portable and secure.

3. Secure MongoDB (authentication, network ACLs) or use a managed database (MongoDB Atlas) and do not expose port 27017 publicly.

4. Use TLS/HTTPS (ALB, nginx, or a CDN) in front of the frontend for production.

I can provide the minimal code diffs required to make the apps read environment variables and a portable `docker-compose` if you want — tell me and I'll prepare the changes.

## CI / CD with GitHub Actions (example)

This example covers:

- CI: install/build the frontend and backend
- CD: build and push Docker images to a registry, then SSH to an EC2 host and run `docker compose up -d`

Create `.github/workflows/ci-cd.yml` with the following (example):

```yaml
name: CI-CD

on:
	push:
		branches: [ main, master, initial ]

jobs:
	build_and_push:
		runs-on: ubuntu-latest
		steps:
			- uses: actions/checkout@v4

			- name: Set up Node.js
				uses: actions/setup-node@v4
				with:
					node-version: '18'

			- name: Build frontend
				working-directory: ./frontend
				run: |
					npm ci
					npm run build -- --configuration production

			- name: Build backend dependencies
				working-directory: ./backend
				run: |
					npm ci --production

			- name: Set up Docker Buildx
				uses: docker/setup-buildx-action@v2

			- name: Login to Docker Hub
				uses: docker/login-action@v2
				with:
					username: ${{ secrets.DOCKERHUB_USERNAME }}
					password: ${{ secrets.DOCKERHUB_TOKEN }}

			- name: Build and push backend image
				uses: docker/build-push-action@v4
				with:
					context: ./backend
					push: true
					tags: ${{ secrets.DOCKERHUB_USERNAME }}/discoverdoller-backend:latest

			- name: Build and push frontend image
				uses: docker/build-push-action@v4
				with:
					context: ./frontend
					push: true
					tags: ${{ secrets.DOCKERHUB_USERNAME }}/discoverdoller-frontend:latest

	deploy_via_ssh:
		needs: build_and_push
		runs-on: ubuntu-latest
		steps:
			- name: Checkout
				uses: actions/checkout@v4

			- name: Deploy to EC2 via SSH
				uses: appleboy/ssh-action@v0.1.9
				with:
					host: ${{ secrets.EC2_HOST }}
					username: ${{ secrets.EC2_USER }}
					key: ${{ secrets.EC2_SSH_KEY }}
					port: ${{ secrets.EC2_SSH_PORT || '22' }}
					script: |
						cd /home/${{ secrets.EC2_USER }}/discoverdoller || git clone https://github.com/${{ github.repository }} discoverdoller && cd discoverdoller
						docker compose pull || true
						docker compose up -d --build

```

Required repository secrets for the workflow

- `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` (or credentials for your container registry)
- `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY` (private key), and optionally `EC2_SSH_PORT`

Notes on CI/CD design

- The action builds the frontend in the runner (so the built `dist/` is available if you want to build an image locally) and uses `docker/build-push-action` to push images to a registry.
- The deploy step SSHs to the target EC2 host and runs `docker compose up -d --build`. For production you may want the runner to only `docker pull` pre-built images and avoid building on the host.

## Troubleshooting

- If the backend cannot connect to MongoDB, check `backend/app/config/db.config.js` — it may expect `mongodb://localhost:27017/dd_db`. On bridge networks you must supply a `MONGODB_URI` or change the config to read `MONGO_HOST`/`MONGO_PORT`.
- If the frontend cannot reach the API, check `frontend/src/app/services/tutorial.service.ts` for `baseUrl`.

---

If you want, I can now:
- Add the `.github/workflows/ci-cd.yml` file with the example pipeline, or
- Make the minimal, safe edits to backend and frontend to read environment variables and convert `docker-compose.yml` to bridge networking (recommended for portability and security).
Tell me which you prefer and I'll proceed.

