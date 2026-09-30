# Docker

## Idea
"Works on my machine" solved: package app + JRE + libs into an **image**; run it anywhere as a **container** (isolated process sharing host kernel; lighter than a VM).

```
Dockerfile ──docker build──► Image (read-only layers) ──docker run──► Container (running instance)
                                   │ docker push/pull
                              Registry (Docker Hub / AWS ECR)
```

## Dockerfile for Spring Boot (multi-stage = small & safe)
```dockerfile
# ---- stage 1: build
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -q dependency:go-offline           # cached layer unless pom changes
COPY src ./src
RUN mvn -q package -DskipTests

# ---- stage 2: runtime (only JRE + jar)
FROM eclipse-temurin:21-jre
RUN useradd -r appuser                     # don't run as root
USER appuser
WORKDIR /app
COPY --from=build /app/target/order-service.jar app.jar
EXPOSE 8081
HEALTHCHECK CMD curl -f http://localhost:8081/actuator/health || exit 1
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-jar","app.jar"]
```
`.dockerignore`: `target/`, `.git`, `node_modules`.

## docker-compose (whole system locally)
```yaml
services:
  mysql:
    image: mysql:8
    environment: { MYSQL_ROOT_PASSWORD: root, MYSQL_DATABASE: shop }
    volumes: [db-data:/var/lib/mysql]
  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]
  kafka:
    image: bitnami/kafka:latest          # (KRaft mode) plus env config
    ports: ["9092:9092"]
  order-service:
    build: ./order-service
    ports: ["8081:8081"]
    environment: { DB_HOST: mysql, DB_USER: root, DB_PASS: root, SPRING_RABBITMQ_HOST: rabbitmq }
    depends_on: [mysql, rabbitmq, kafka]
  frontend:
    build: ./frontend            # nginx serving React build
    ports: ["3000:80"]
volumes: { db-data: {} }
```
`docker compose up -d` → everything on one network; services reach each other by **service name** (`mysql`).

## Commands & concepts
`docker ps`, `docker logs -f <c>`, `docker exec -it <c> sh`, `docker stop/rm`, `docker images`, `docker build -t shop/order:1.0 .`.
**Volumes** persist data (containers are ephemeral). **Networks** let containers talk. **Layers** cached → order Dockerfile from least to most changing. CMD vs ENTRYPOINT · COPY vs ADD · image vs container · container vs VM · never bake secrets in images (use env/secret managers) · pin versions (not `latest`) · scan images (Trivy).

React Dockerfile:
```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```
