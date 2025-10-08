# Docker Compose & Makefile Integration for Flask-PostgreSQL API

This README covers best practices, explanations, and troubleshooting for using Docker Compose to orchestrate a Flask REST API and its dependent services (PostgreSQL, Nginx). It also includes common interview questions for Docker Compose.

---

## 1. Docker Compose Best Practices

- **Service Isolation**: Define each component (API, DB, web server) as a separate service for modular development and easy scaling.
- **Environment Variables**: Use `env_file` or environment variables for secrets/configs (never hardcode).
- **Volumes**: Persist data by mounting volumes, especially for databases.
- **Healthchecks**: Use healthchecks to ensure services (like Postgres) are ready before dependent services start.
- **depends_on**: Use `depends_on` with healthcheck conditions to control service startup order.
- **Restart Policies**: Always set `restart: always` for production, ensuring services recover from crashes.
- **Minimal Images**: Use official, minimal images (e.g., `python:alpine`, `nginx:alpine`, `postgres:15`) for efficiency.
- **Configuration Management**: Mount config files (e.g., Nginx) as read-only volumes.
- **Clean Networking**: Compose automatically creates a network for your project; services can reference each other by name.
- **Resource Limits**: For production, set CPU/memory limits (not shown here).
- **Version Pinning**: Use explicit image versions to avoid unexpected changes.

---

## 2. Explanation of the Provided Docker Compose File

```yaml
services:
  flask-app:
    build: 
      context: ./app
      dockerfile: Dockerfile
    image: flask-app:1.0.0
    restart: always
    env_file:
      - ${ENV_FILE}
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./:/app

  postgres:
    image: postgres:15
    container_name: postgres-container
    restart: always
    env_file:
      - ${ENV_FILE}
    ports:
      - "5432:5432"   
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres}"]
      interval: 10s
      timeout: 20s
      retries: 5

  nginx:
    image: nginx:alpine
    container_name: nginx-container
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - flask-app

volumes:
  pgdata:
    driver: local
```

### **Service Breakdown**

- **flask-app**:
  - Builds from Dockerfile in `./app`, tags as `flask-app:1.0.0`.
  - Uses environment variables from `.env`.
  - Depends on healthy Postgres service.
  - Mounts project directory for code sharing/hot reload (dev mode).
- **postgres**:
  - Uses official Postgres 15 image.
  - Persists data with `pgdata` volume.
  - Reads environment from `.env`.
  - Healthcheck ensures DB is ready before API starts.
  - Exposes port 5432 for local access.
- **nginx**:
  - Uses lightweight Nginx image.
  - Mounts custom config file.
  - Depends on flask-app.
  - Exposes port 80 for HTTP traffic.

### **Volumes**
- `pgdata` is a named volume that keeps PostgreSQL data persistent across container restarts.

### **Healthcheck**
- Ensures Postgres is ready before API attempts to connect.

---

## 3. Docker Compose Commands

#### **Build and Start Services**

```sh
# Build all services (API, DB, Nginx)
docker-compose build

# Start all services in foreground
docker-compose up

# Start all services in detached mode (recommended)
docker-compose up -d
```

#### **Stop and Remove Services**

```sh
# Stop and remove all running containers, networks, and volumes defined in the compose file
docker-compose down

# Stop only the running containers (does not remove networks/volumes)
docker-compose stop

# Remove stopped service containers
docker-compose rm <service>
```

#### **Service Management**

```sh
# List running containers and their status
docker-compose ps

# Restart a specific service
docker-compose restart flask-app
```

#### **Logs and Debugging**

```sh
# View logs for all services
docker-compose logs

# View logs for a specific service
docker-compose logs postgres
```

#### **Accessing Containers**

```sh
# Start a shell in a running container
docker-compose exec flask-app /bin/sh

# Run a command inside a service (e.g., database migration)
docker-compose run flask-app flask db upgrade
```

#### **Build/Start Specific Services**

```sh
# Build only the flask-app image
docker-compose build flask-app

# Start only the postgres service
docker-compose up postgres
```

#### **Clean Up Volumes**

```sh
# Stop and remove all containers, networks, and volumes
docker-compose down -v
```

#### Database Commands (Inside Container)

```sh
# Connect to PostgreSQL from inside the container
docker-compose exec postgres psql -U <user> -d <db>
```

#### Maintenance & Troubleshooting

```sh
# Check environment variables inside a running container
docker-compose exec flask-app env

# Remove unused images, containers, and networks
docker system prune -a
```
---

## 4. Issues Faced & Troubleshooting Steps

- **DB Connection Refused**: Check if Postgres is healthy, ports are mapped, and credentials are correct.
- **Service Not Starting**: Use `docker-compose logs <service>` for error insight.
- **Flask Can't Reach DB**: Ensure healthcheck passes before API starts, verify network and env vars.
- **Volume Permission Issues**: Set correct permissions on host for mounted volumes.
- **Container Fails to Build**: Check Dockerfile paths, dependency versions, and syntax.
- **Address Already in Use**: Stop other containers/processes using the same port.
- **Nginx 502 Gateway Error**: Verify flask-app is running and accessible from nginx-container.
---

### **1. DB Connection Refused**

**Symptoms**: Flask app cannot connect to PostgreSQL.  
**Troubleshooting Steps**:

- **Check Postgres Health**  
  ```sh
  docker-compose ps
  docker-compose logs postgres
  ```
  Look for "healthy" status and any error messages.

- **Test DB Connectivity from Container**  
  ```sh
  docker-compose exec postgres psql -U <user> -d <db>
  ```
  Replace `<user>` and `<db>` with your actual DB user and database name.

- **Check Exposed Ports**  
  Ensure `5432:5432` is mapped in `docker-compose.yml` and not blocked by firewall.

- **Verify Credentials & Environment Variables**  
  ```sh
  docker-compose exec flask-app env | grep DATABASE_URL
  ```
  Confirm the database URL matches your config.

---

### **2. Service Not Starting**

**Symptoms**: Any service fails to start, exits, or restarts repeatedly.  
**Troubleshooting Steps**:

- **Check Logs for Specific Service**  
  ```sh
  docker-compose logs <service>
  ```
  Replace `<service>` with the name (e.g., `flask-app`, `postgres`, `nginx`).  
  Review error messages for missing dependencies or misconfiguration.

- **Check Container Status**  
  ```sh
  docker-compose ps
  ```

---

### **3. Flask Can't Reach DB**

**Symptoms**: Flask throws DB connection errors or times out.  
**Troubleshooting Steps**:

- **Ensure Healthcheck Passes Before API Starts**  
  Healthchecks in `docker-compose.yml` ensure the DB is ready before Flask starts.  
  Check:
  ```sh
  docker-compose ps
  ```

- **Validate Network Connectivity**  
  Inside the flask-app container:
  ```sh
  docker-compose exec flask-app ping postgres
  ```

- **Check Environment Variables**  
  ```sh
  docker-compose exec flask-app env | grep DATABASE_URL
  ```

- **Restart Dependent Services**  
  ```sh
  docker-compose restart flask-app
  ```

---

### **4. Volume Permission Issues**

**Symptoms**: Postgres or Flask app fails to read/write to mounted volume.  
**Troubleshooting Steps**:

- **Check Volume Mounts**  
  ```sh
  docker-compose ps
  docker-compose logs postgres
  ```

- **Inspect Permissions on Host**  
  ```sh
  ls -l <host_volume_path>
  sudo chown -R $(whoami):$(whoami) <host_volume_path>
  sudo chmod -R 750 <host_volume_path>
  ```

- **Restart Services After Permission Fix**  
  ```sh
  docker-compose restart postgres
  ```

---

### **5. Container Fails to Build**

**Symptoms**: `docker-compose build` fails with errors.  
**Troubleshooting Steps**:

- **Read Build Output for Errors**  
  Run:
  ```sh
  docker-compose build
  ```

- **Check Dockerfile Path and Syntax**  
  Ensure `Dockerfile` exists at specified location and has correct instructions.

- **Check Dependency Versions**  
  Validate `requirements.txt` and base images.

- **Clean Build Cache and Rebuild**  
  ```sh
  docker-compose down -v
  docker-compose build --no-cache
  docker-compose up -d
  ```

---

### **6. Address Already in Use**

**Symptoms**: Port binding errors, unable to start service.  
**Troubleshooting Steps**:

- **List Processes Using Port**  
  ```sh
  sudo lsof -i :5000
  sudo lsof -i :5432
  ```

- **Stop Conflicting Containers or Services**  
  ```sh
  docker ps
  docker stop <conflicting_container>
  ```

- **Restart Compose Stack**  
  ```sh
  docker-compose down
  docker-compose up -d
  ```

---

### **7. Nginx 502 Gateway Error**

**Symptoms**: Nginx returns 502 Bad Gateway when accessing Flask app.  
**Troubleshooting Steps**:

- **Check Flask-App Service Status**  
  ```sh
  docker-compose ps
  docker-compose logs flask-app
  ```

- **Check Nginx Logs**  
  ```sh
  docker-compose logs nginx
  ```

- **Validate Nginx Configuration**  
  Ensure upstream points to correct flask-app service and port.

- **Test Connectivity from Nginx Container**  
  ```sh
  docker-compose exec nginx ping flask-app
  ```

- **Restart Services**  
  ```sh
  docker-compose restart flask-app
  docker-compose restart nginx
  ```

---

### **8. PostgreSQL Configuration Issues**

- **Check PostgreSQL Status**
  ```sh
  sudo systemctl status postgresql
  ```
- **Fix PostgreSQL Configuration**

  - Edit pg_hba.conf:
  ```sh
  sudo vi /etc/postgresql/16/main/pg_hba.conf
  ```
- **Correct configuration:**
  ```sh
  # Database administrative login by Unix domain socket
  local   all             postgres                                md5
  # TYPE  DATABASE        USER            ADDRESS                 METHOD
  # "local" is for Unix domain socket connections only
  local   all             all                                     md5
  # IPv4 local connections:
  host    all             all             [IP_ADDRESS]            md5
  host    all             all             [IP_ADDRESS]            md5
  # IPv6 local connections:
  host    all             all             ::1/128                 md5
  # Allow replication connections
  local   replication     all                                     md5
  host    replication     all             [IP_ADDRESS]            md5
  host    replication     all             ::1/128                 md5
  ```
- **Edit postgresql.conf:**
  ```sh
  sudo vi /etc/postgresql/16/main/postgresql.conf
  
  # Modify:
  listen_addresses = '*'
  ```
- **PostgreSQL Service Management**
  ```sh
  # Restart PostgreSQL
  sudo systemctl restart postgresql

  # Check cluster status
  pg_lsclusters

  # If cluster is down, start it
  sudo pg_ctlcluster 16 main start
  ```
---


### **9. Database Connection Issues**
  
- **Set PostgreSQL Password**
  ```sh
  sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '[PASSWORD]';"
  ```
- **Verify Database Connection**
  ```sh
  # Connect to PostgreSQL
  psql -U postgres -h [IP_ADDRESS] -d studentdb -W
  
  # List databases
  \l

  # List tables
  \dt
  ```
---

### **General Troubleshooting Tips**

- **View All Container Logs**  
  ```sh
  docker-compose logs
  ```

- **Remove and Recreate Volumes**  
  ```sh
  docker-compose down -v
  docker-compose up -d
  ```

- **Check Service Health**  
  ```sh
  docker-compose ps
  ```

- **Inspect Environment Variables**  
  ```sh
  docker-compose exec <service> env
  ```
---

# Docker Compose Interview Questions

This document provides common interview questions related to Docker compose and best practices, along with sample answers.

---

#### **1: What is Docker Compose and why is it useful?**
**A:** Docker Compose is a tool for defining and running multi-container Docker applications using a single YAML file. It simplifies setup, orchestration, and management of interdependent services.

---

#### **2: How does `depends_on` work in Docker Compose?**
**A:** `depends_on` controls the startup order of services. With healthchecks, it can wait for a service to be healthy before starting the dependent service.

---

#### **3: How do you persist data in containers using Docker Compose?**
**A:** By defining named volumes (e.g., `pgdata` for PostgreSQL), data remains intact even if the container is recreated or stopped.

---

#### **4: How can you pass environment variables to services in Docker Compose?**
**A:** Use `env_file` or the `environment:` section in the compose file. Avoid hardcoding sensitive information.

---

#### **5: How does Docker Compose networking work?**
**A:** Compose automatically creates a network for the project. Services can communicate using their service names as hostnames.

---

#### **6: How do you run database migrations with Docker Compose?**
**A:** Use `docker-compose run <service> <migration-command>` (e.g., `flask db upgrade`). Or add migration as a startup step in the API container.

---

#### **7: What is the difference between `build` and `image` in a compose file?**
**A:** `build` lets Compose build an image from a Dockerfile; `image` pulls from a registry or uses a pre-built image.

---

#### **8: How can you troubleshoot a service that is not starting?**
**A:** Use `docker-compose logs <service>`, check healthchecks, inspect environment variables, and verify port bindings.

---

#### **9: How does volume mounting work in Docker Compose?**
**A:** Volumes map host directories/files into containers for persistence or config sharing.

---

#### **10: How do you upgrade a service in Docker Compose?**
**A:** Rebuild the image (`docker-compose build`), then restart the service (`docker-compose up -d <service>`).

---

## References

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Compose Best Practices](https://docs.docker.com/compose/best-practices/)
- [Compose Healthchecks](https://docs.docker.com/compose/compose-file/compose-file-v3/#healthcheck)
