# Stage 1: install dependencies
FROM node:18-alpine AS deps
WORKDIR /usr/src/app
COPY package.json package-lock.json* ./
RUN npm ci --omit=dev || npm install --production

# Stage 2: copy source
FROM node:18-alpine AS build
WORKDIR /usr/src/app
COPY --from=deps /usr/src/app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production

# Expose the port the app listens on
EXPOSE 8080

# Start the app
CMD ["node", "server.js"]
