# Docker TP1 — Multi-Container Application

**Author:** Ryan Mballa Kadjio
**School:** EFREI Paris
**Course:** Docker — TP1

## 1. Project Description

This project demonstrates how to containerize and orchestrate a multi-tier web application using Docker.

The application consists of three main services:

* **PostgreSQL** — database service
* **Spring Boot** — backend REST API
* **Apache HTTPD** — HTTP server and reverse proxy

Docker Compose is used to manage the three services, their network, their dependencies and the persistent database volume.

The final architecture is:

```text
                     Host Machine
                          │
                          │ HTTP :80
                          ▼
                 ┌─────────────────┐
                 │  Apache HTTPD   │
                 │ Reverse Proxy   │
                 └────────┬────────┘
                          │
                    backend:8080
                          │
                          ▼
                 ┌─────────────────┐
                 │ Spring Boot API │
                 └────────┬────────┘
                          │
                    database:5432
                          │
                          ▼
                 ┌─────────────────┐
                 │   PostgreSQL    │
                 └─────────────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ postgres-data   │
                 │ Named Volume    │
                 └─────────────────┘
```

Only Apache HTTPD is exposed to the host machine. The backend and database communicate through the internal Docker network.

---

# 2. Project Structure

```text
TP1/
│
├── database/
│   ├── Dockerfile
│   ├── 01-CreateScheme.sql
│   └── 02-InsertData.sql
│
├── backend/
│   ├── hello/
│   │   ├── Dockerfile
│   │   ├── Main.java
│   │   └── Main.class
│   │
│   └── simpleapi/
│       ├── Dockerfile
│       ├── pom.xml
│       └── src/
│           ├── main/
│           └── test/
│
├── httpd/
│   ├── Dockerfile
│   ├── index.html
│   └── httpd.conf
│
└── docker-compose.yml
```

The `hello` directory contains an initial Docker exercise, while `simpleapi` contains the Spring Boot backend used by the final multi-container application.

---

# 3. Docker Images

## Database

The database image is based on PostgreSQL:

```dockerfile
FROM postgres:17.2-alpine
```

Initialization SQL scripts are copied into PostgreSQL's initialization directory so that the database schema and initial data can be created when the database is initialized.

## Backend

The backend is a Spring Boot application running on Java 21.

The Dockerfile uses a multi-stage build:

1. A JDK/Maven image compiles the application.
2. A smaller JRE image runs the resulting JAR.

This separates the build environment from the runtime environment and avoids including unnecessary build tools in the final image.

## HTTP Server

The HTTP server is based on Apache HTTPD:

```dockerfile
FROM httpd:2.4
```

Apache serves as the entry point to the application and acts as a reverse proxy for the Spring Boot backend.

---

# 4. Docker Network

The services communicate through a dedicated Docker network:

```yaml
networks:
  app-network:
```

All three services are connected to this network.

Docker's internal DNS allows services to communicate using their Compose service names.

For example, the backend connects to PostgreSQL using:

```text
database:5432
```

rather than:

```text
localhost:5432
```

Similarly, Apache communicates with the backend using:

```text
backend:8080
```

rather than `localhost:8080`.

---

# 5. Reverse Proxy

Apache HTTPD is configured as a reverse proxy.

The relevant configuration is:

```apache
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://backend:8080/
    ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

The required Apache proxy modules are also enabled:

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

The client therefore does not communicate directly with the Spring Boot application.

Instead:

```text
Client
  │
  ▼
Apache :80
  │
  ▼
backend:8080
  │
  ▼
PostgreSQL :5432
```

This keeps the backend and database internal to the Docker network.

---

# 6. Docker Compose

The complete application is described in `docker-compose.yml`.

The main services are:

```yaml
services:
  database:
    ...

  backend:
    ...

  httpd:
    ...
```

The database uses a named volume:

```yaml
volumes:
  postgres-data:
```

Apache is the only service exposing a host port:

```yaml
ports:
  - "80:80"
```

The backend and database do not expose ports to the host.

---

# 7. Building the Application

From the project root:

```powershell
cd C:\Users\mball\Desktop\EFREI\I2\docker\TP1
```

Build all images:

```powershell
docker compose build
```

---

# 8. Starting the Application

Start all services in detached mode:

```powershell
docker compose up -d
```

Check the status:

```powershell
docker compose ps
```

The three services should be running:

```text
database
backend
httpd
```

---

# 9. Viewing Logs

Display all logs:

```powershell
docker compose logs
```

Display backend logs:

```powershell
docker compose logs backend
```

Display database logs:

```powershell
docker compose logs database
```

Display Apache logs:

```powershell
docker compose logs httpd
```

Follow logs continuously:

```powershell
docker compose logs -f
```

---

# 10. Testing the Application

The application can be accessed through Apache on port 80.

For example:

```text
http://localhost/departments/IRC/students
```

The request is forwarded by Apache to the Spring Boot backend.

The backend then communicates with PostgreSQL to retrieve the required data.

The complete request path is:

```text
http://localhost/departments/IRC/students
                    │
                    ▼
              Apache HTTPD
                    │
                    ▼
          Spring Boot Backend
                    │
                    ▼
               PostgreSQL
```

---

# 11. Database Persistence

PostgreSQL uses a named Docker volume:

```text
postgres-data
```

The volume is mounted at:

```text
/var/lib/postgresql/data
```

This means that database data is not stored only inside the PostgreSQL container.

To stop and remove the containers:

```powershell
docker compose down
```

The named volume remains by default.

The application can then be started again:

```powershell
docker compose up -d
```

The database data remains available.

To remove the containers **and** the associated named volumes, the following command can be used:

```powershell
docker compose down -v
```

This command should not be used when testing persistence because it deletes the volume.

---

# 12. Useful Docker Compose Commands

| Command                               | Purpose                                    |
| ------------------------------------- | ------------------------------------------ |
| `docker compose build`                | Build the service images                   |
| `docker compose up`                   | Start the application                      |
| `docker compose up -d`                | Start in detached mode                     |
| `docker compose down`                 | Stop and remove the application containers |
| `docker compose ps`                   | Display service status                     |
| `docker compose logs`                 | Display logs                               |
| `docker compose logs -f`              | Follow logs                                |
| `docker compose restart`              | Restart services                           |
| `docker compose stop`                 | Stop services                              |
| `docker compose start`                | Start stopped services                     |
| `docker compose exec SERVICE COMMAND` | Execute a command inside a service         |
| `docker compose config`               | Validate the Compose configuration         |

---

# 13. Docker Inspection Commands

During the Docker exercises, several commands can be used to inspect containers.

Display running containers:

```powershell
docker ps
```

Display all containers:

```powershell
docker ps -a
```

Display detailed container information:

```powershell
docker inspect CONTAINER
```

Display resource usage:

```powershell
docker stats
```

Display container logs:

```powershell
docker logs CONTAINER
```

Execute a command inside a running container:

```powershell
docker exec -it CONTAINER COMMAND
```

---

# 14. Publishing Images to Docker Hub

Docker images can be published to Docker Hub so that they can be retrieved from another machine.

First authenticate:

```powershell
docker login
```

Tag an image:

```powershell
docker tag IMAGE USERNAME/IMAGE:latest
```

Push the image:

```powershell
docker push USERNAME/IMAGE:latest
```

For example:

```powershell
docker tag my-httpd YOUR_DOCKERHUB_USERNAME/my-httpd:latest
docker push YOUR_DOCKERHUB_USERNAME/my-httpd:latest
```

The same procedure can be applied to the backend and database images.

---

# 15. Why Use a Docker Registry?

A Docker registry provides centralized storage and distribution for Docker images.

Publishing images makes it possible for other developers, servers or CI/CD systems to retrieve the same image without rebuilding it locally.

This improves:

* Reproducibility
* Collaboration
* Deployment
* Version management
* CI/CD integration

Docker Hub is one example of a Docker registry.

---

# 16. Architecture Advantages

The final architecture provides several advantages:

### Separation of responsibilities

Each container has a specific role:

```text
HTTPD       → HTTP entry point / reverse proxy
Backend     → Business logic / REST API
PostgreSQL  → Data persistence
```

### Isolation

The services run in separate containers and communicate through a dedicated Docker network.

### Security

Only Apache is exposed to the host. The backend and database are not directly accessible through host ports.

### Persistence

The PostgreSQL data is stored in a named volume, allowing it to survive container recreation.

### Reproducibility

Docker Compose describes the complete application architecture, making it possible to recreate the environment consistently.

### Scalability

The architecture can later be extended with additional backend instances, services or reverse-proxy rules.

---

# 17. Conclusion

This TP demonstrated the main concepts required to build a multi-container Docker application.

The final application consists of a PostgreSQL database, a Spring Boot REST API and an Apache HTTPD reverse proxy. Docker Compose is used to orchestrate the services, while a dedicated Docker network enables communication between them.

The database uses a named volume for persistence, and only Apache is exposed to the host. This results in a clear separation between the public HTTP entry point and the internal application services.

The project also introduced Docker image management and the possibility of publishing images to an online Docker registry such as Docker Hub.
