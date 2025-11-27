In this DevOps task, you need to build and deploy a full-stack CRUD application using the MEAN stack (MongoDB, Express, Angular 15, and Node.js). The backend will be developed with Node.js and Express to provide REST APIs, connecting to a MongoDB database. The frontend will be an Angular application utilizing HTTPClient for communication.  

The application will manage a collection of tutorials, where each tutorial includes an ID, title, description, and published status. Users will be able to create, retrieve, update, and delete tutorials. Additionally, a search box will allow users to find tutorials by title.

## Project setup

### Node.js Server

cd backend

npm install

You can update the MongoDB credentials by modifying the `db.config.js` file located in `app/config/`.

Run `node server.js`

### Angular Client

cd frontend

npm install

Run `ng serve --port 8081`

You can modify the `src/app/services/tutorial.service.ts` file to adjust how the frontend interacts with the backend.

Navigate to `http://localhost:8081/`

## Docker

This repository includes multi-stage Dockerfiles for both the backend (Node/Express) and the frontend (Angular) to produce small, production-ready images.

Files added
- `backend/Dockerfile` - multi-stage Node image for the API server (listens on 8080)
- `backend/.dockerignore`
- `frontend/Dockerfile` - multi-stage build using Node for build and nginx to serve static files
- `frontend/.dockerignore`
- `frontend/nginx-spa.conf` - nginx config to support SPA routing

Build and run

1) Backend

Build image (from repository root):

```powershell
docker build -f backend/Dockerfile -t discoverdoller-backend:latest .
```

Run the backend (maps container 8080 -> host 8080):

```powershell
docker run --rm -p 8080:8080 \
	-e MONGO_HOST=<your-mongo-host> \
	-e MONGO_PORT=<your-mongo-port> \
	-e MONGO_DB=<your-db-name> \
	discoverdoller-backend:latest
```

Note: This project reads MongoDB connection info from `app/models/index.js` (see `db.config.js`). Provide the appropriate environment variables or edit the config before building.

2) Frontend

Build the Angular production image:

```powershell
docker build -f frontend/Dockerfile -t discoverdoller-frontend:latest frontend
```

Run the frontend (maps container 80 -> host 8081):

```powershell
docker run --rm -p 8081:80 discoverdoller-frontend:latest
```

Then open the app at `http://localhost:8081/`.

Development notes
- To run the backend without Docker:

```powershell
cd backend; npm install; node server.js
```

- To run the Angular dev server (live-reload):

```powershell
cd frontend; npm install; npx ng serve --port 8081
```

If you'd like, I can also add Docker Compose to run both services plus a MongoDB container and wire environment variables. Tell me if you prefer that and any specific MongoDB credentials/ports you want.

### Docker Compose (local or EC2)

I added a `docker-compose.yml` at the repository root to run MongoDB, the backend, and the frontend together.

Quick start on an EC2 instance (Amazon Linux / Ubuntu):

1. Open the instance security group to allow inbound traffic on the ports you want to expose (e.g., 8081 for the frontend). For testing you may also open 8080 and 27017 but DO NOT expose MongoDB to the public internet in production.

2. SSH into your EC2 instance, then install Docker and Docker Compose (example for Ubuntu):

```powershell
# Ubuntu example (run on the instance shell)
sudo apt update; sudo apt install -y docker.io docker-compose; sudo usermod -aG docker $USER
newgrp docker
```

3. Clone this repo and start the stack:

```powershell
git clone <repo-url> discoverdoller; cd discoverdoller
cp .env.example .env   # edit .env if you need custom values
docker compose up -d --build
```

4. Visit the frontend at `http://<ec2-public-ip>:8081/`.

Notes and security:
- Use a managed MongoDB service (Atlas) or run MongoDB in a private subnet and don't expose port 27017 to the internet.
- For production, add credentials and enable authentication for MongoDB. Update `docker-compose.yml` and `app/config/db.config.js` to use those credentials or a `MONGODB_URI`.
- Consider adding TLS termination (e.g., behind an ALB or nginx proxy) and a systemd unit or ECS/EKS for production orchestration.

