# Stage 1: build the Angular app
FROM node:18-alpine AS builder
WORKDIR /usr/src/app
COPY package.json package-lock.json* ./
RUN npm install -g @angular/cli && npm install
COPY . .
EXPOSE 8081
CMD ["ng", "serve", "--host", "0.0.0.0", "--port", "8081"]
