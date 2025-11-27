# Stage 1: build the Angular app
FROM node:18-alpine AS builder
WORKDIR /usr/src/app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
RUN ng serve --port 8081 
